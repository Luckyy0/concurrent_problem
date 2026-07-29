# Giải pháp snapshot, validation và locking

## Mục tiêu thiết kế

Giải pháp phải trả lời trước một câu hỏi nghiệp vụ:

```text
Decision phải hợp lệ theo:
  policy tại lúc đọc,
  policy tại statement ghi decision,
  hay policy bị giữ nguyên tới lúc transaction commit?
```

Mọi lựa chọn đều phải lưu một coherent policy revision, không trộn field từ nhiều
statement snapshots.

## Solution 1 — Đọc một immutable policy snapshot

Nếu policy tại lúc evaluation được phép dùng, đọc đúng một lần và truyền một value
object xuyên suốt:

```java
public record RefundPolicySnapshot(
    UUID merchantId,
    BigDecimal autoRefundLimit,
    boolean active,
    long revision
) {
    boolean allows(BigDecimal amount) {
        return active && amount.compareTo(autoRefundLimit) <= 0;
    }
}
```

Repository trả projection rồi map ngay:

```java
@Repository
public class RefundPolicyReader {
    private final JdbcTemplate jdbc;

    public RefundPolicyReader(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public RefundPolicySnapshot read(UUID merchantId) {
        return jdbc.queryForObject(
            """
            select merchant_id,
                   auto_refund_limit,
                   active,
                   revision
              from merchant_refund_policy
             where merchant_id = ?
            """,
            (rs, rowNum) -> new RefundPolicySnapshot(
                rs.getObject("merchant_id", UUID.class),
                rs.getBigDecimal("auto_refund_limit"),
                rs.getBoolean("active"),
                rs.getLong("revision")
            ),
            merchantId
        );
    }
}
```

Service không query policy lần hai:

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public RefundResult decide(
    UUID commandId,
    UUID merchantId,
    BigDecimal amount
) {
    RefundPolicySnapshot policy = policyReader.read(merchantId);

    if (!policy.allows(amount)) {
        return RefundResult.manualReview();
    }

    localRules.validate(amount);
    RefundDecision saved = decisions.save(
        RefundDecision.approved(
            commandId,
            merchantId,
            amount,
            policy.autoRefundLimit(),
            policy.revision()
        )
    );
    return RefundResult.approved(saved.getId());
}
```

Decision/evidence nhất quán với revision đã đọc. Nhưng concurrent update vẫn có
thể commit sau read; solution này không hứa “latest at decision commit”.

> **Nói ngắn gọn:** đọc một lần sửa việc trộn snapshots; immutable policy history
> mới làm revision cũ còn có ý nghĩa audit lâu dài.

## Solution 2 — Versioned policy history

Mutable current row không đủ để reconstruct revision cũ. Tách policy versions
immutable:

```sql
create table merchant_refund_policy_version (
    merchant_id uuid not null,
    revision bigint not null,
    auto_refund_limit numeric(19, 2) not null,
    active boolean not null,
    created_at timestamptz not null,
    primary key (merchant_id, revision),
    check (auto_refund_limit >= 0)
);

create table merchant_refund_policy_current (
    merchant_id uuid primary key,
    revision bigint not null,
    foreign key (merchant_id, revision)
        references merchant_refund_policy_version(merchant_id, revision)
);

alter table refund_decision
    add constraint fk_refund_decision_policy_version
    foreign key (merchant_id, policy_revision)
    references merchant_refund_policy_version(merchant_id, revision);
```

Admin transaction insert version mới rồi chuyển current pointer:

```sql
begin;

insert into merchant_refund_policy_version(
    merchant_id,
    revision,
    auto_refund_limit,
    active,
    created_at
)
values (:merchantId, :nextRevision, :newLimit, true, clock_timestamp());

update merchant_refund_policy_current
   set revision = :nextRevision
 where merchant_id = :merchantId
   and revision = :expectedRevision;

-- affected row phải là 1
commit;
```

Reader join current pointer với immutable version trong một statement:

```sql
select v.merchant_id,
       v.revision,
       v.auto_refund_limit,
       v.active
from merchant_refund_policy_current c
join merchant_refund_policy_version v
  on v.merchant_id = c.merchant_id
 and v.revision = c.revision
where c.merchant_id = :merchantId;
```

Foreign key bảo đảm decision không tham chiếu revision không tồn tại. Old policy
không bị overwrite, nên audit vẫn tái dựng được decision revision `7` sau current
pointer chuyển sang revision `8`.

Nếu pointer update affected-row `0`, admin attempt thua optimistic conflict và
phải retry toàn transaction với revision mới.

## Solution 3 — Conditional final validation

Nếu contract yêu cầu policy chưa đổi trước statement ghi decision, ghi bằng
`INSERT ... SELECT` và revision predicate:

```sql
insert into refund_decision(
    id,
    command_id,
    merchant_id,
    amount,
    outcome,
    evaluated_limit,
    policy_revision
)
select :decisionId,
       :commandId,
       p.merchant_id,
       :amount,
       'APPROVED',
       p.auto_refund_limit,
       p.revision
from merchant_refund_policy p
where p.merchant_id = :merchantId
  and p.revision = :expectedRevision
  and p.active
  and :amount <= p.auto_refund_limit;
```

Spring repository:

```java
public interface RefundDecisionCommands {

    @Modifying
    @Query(
        value = """
            insert into refund_decision(
                id, command_id, merchant_id, amount, outcome,
                evaluated_limit, policy_revision
            )
            select :decisionId, :commandId, p.merchant_id, :amount,
                   'APPROVED', p.auto_refund_limit, p.revision
              from merchant_refund_policy p
             where p.merchant_id = :merchantId
               and p.revision = :expectedRevision
               and p.active
               and :amount <= p.auto_refund_limit
            """,
        nativeQuery = true
    )
    int insertIfPolicyStillAllows(
        UUID decisionId,
        UUID commandId,
        UUID merchantId,
        BigDecimal amount,
        long expectedRevision
    );
}
```

Service:

```java
@Transactional
public RefundResult decideValidated(
    UUID commandId,
    UUID merchantId,
    BigDecimal amount
) {
    RefundPolicySnapshot policy = policyReader.read(merchantId);
    if (!policy.allows(amount)) {
        return RefundResult.manualReview();
    }

    localRules.validate(amount);

    UUID decisionId = UUID.randomUUID();

    int inserted = commands.insertIfPolicyStillAllows(
        decisionId,
        commandId,
        merchantId,
        amount,
        policy.revision()
    );
    if (inserted == 0) {
        throw new PolicyChangedException(merchantId, policy.revision());
    }
    return RefundResult.approved(decisionId);
}
```

Affected-row `1` nghĩa policy visible cho final statement vẫn đúng revision và
allow amount. `0` nghĩa inactive, limit không đủ, revision đổi hoặc row biến mất;
transaction rollback, outer retry bắt đầu transaction mới và recompute.

Unique `command_id` vẫn cần cho duplicate delivery. Unique violation phải được
phân biệt với revision mismatch và xử lý theo idempotency contract.

Caveat: plain `INSERT ... SELECT` xác nhận state tại statement snapshot. Updater
vẫn có thể commit sau statement nhưng trước decision commit. Nếu requirement cấm
điều đó, dùng row lock.

## Solution 4 — Giữ policy bằng `FOR SHARE`

PostgreSQL `FOR SHARE` giữ row-level lock tới commit/rollback và conflict với
UPDATE/DELETE của policy row:

```sql
select merchant_id, auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId
for share;
```

Spring Data có thể dùng pessimistic read:

```java
public interface LockedPolicyRepository
    extends JpaRepository<MerchantRefundPolicy, UUID> {

    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("""
        select p
          from MerchantRefundPolicy p
         where p.merchantId = :merchantId
        """)
    Optional<MerchantRefundPolicy> findForDecision(UUID merchantId);
}
```

Với PostgreSQL dialect, kiểm tra SQL generated thực sự dùng locking clause phù
hợp; native `FOR SHARE` là contract rõ nhất khi lock mode mapping quan trọng.

```java
@Transactional
public RefundResult decideWhilePolicyLocked(
    UUID commandId,
    UUID merchantId,
    BigDecimal amount
) {
    MerchantRefundPolicy policy = lockedPolicies
        .findForDecision(merchantId)
        .orElseThrow(PolicyNotFoundException::new);

    if (!policy.allows(amount)) {
        return RefundResult.manualReview();
    }

    RefundDecision saved = decisions.save(
        RefundDecision.approved(
            commandId,
            merchantId,
            amount,
            policy.getAutoRefundLimit(),
            policy.getRevision()
        )
    );
    return RefundResult.approved(saved.getId());
}
```

Behavior:

1. A acquire share row lock.
2. B UPDATE cùng policy row chờ.
3. A INSERT decision rồi commit/rollback.
4. Lock release; B tiếp tục trên current row hoặc timeout/fail.

Không đặt remote I/O trong transaction này. Với nhiều policies, lock theo
deterministic order để giảm deadlock. Cấu hình bounded `lock_timeout` và map lỗi
thành retry/rejection.

## Solution 5 — `REPEATABLE READ`

Khi requirement là stable transaction view:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public RefundResult decideWithStableSnapshot(...) {
    RefundPolicySnapshot first = policyReader.read(merchantId);
    localRules.validate(amount);
    RefundPolicySnapshot second = policyReader.read(merchantId);

    if (first.revision() != second.revision()) {
        throw new IllegalStateException("Stable snapshot contract broken");
    }
    // Save using only fields from the same snapshot.
}
```

Hai SELECT thấy cùng transaction snapshot. Cần xác minh effective isolation trên
physical transaction: inner `REQUIRED` annotation không nâng isolation của outer
transaction đã tồn tại.

Trade-off:

- loại non-repeatable read mà không giữ read row lock;
- snapshot có thể stale so với commit mới;
- long transaction giữ old snapshot/resource lâu;
- write conflicts/serialization failures ở flow phức tạp vẫn cần bounded retry;
- không thay thế immutable audit history.

## Solution 6 — Bounded retry ở transaction mới

Retry orchestration phải nằm ngoài transactional worker:

```java
@Service
public class RefundDecisionRetrier {
    private final RefundDecisionAttempt attempt;
    private final Backoff backoff;

    public RefundDecisionRetrier(
        RefundDecisionAttempt attempt,
        Backoff backoff
    ) {
        this.attempt = attempt;
        this.backoff = backoff;
    }

    public RefundResult decide(RefundCommand command) {
        int maxAttempts = 3;
        for (int number = 1; number <= maxAttempts; number++) {
            try {
                return attempt.runInNewTransaction(command);
            } catch (PolicyChangedException | CannotSerializeTransactionException ex) {
                if (number == maxAttempts) {
                    throw ex;
                }
                backoff.pauseWithJitter(number);
            }
        }
        throw new IllegalStateException("unreachable");
    }
}

@Service
public class RefundDecisionAttempt {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public RefundResult runInNewTransaction(RefundCommand command) {
        // Reload current policy, rerun all rules, perform conditional insert.
    }
}
```

`Backoff` phải bounded và interrupt-aware. Không giữ old entity/decision qua retry.
Retry amplification ở hot merchant cần metric và admission control.

## Vì sao `SERIALIZABLE` không có nghĩa “latest”

`SERIALIZABLE` cung cấp kết quả tương đương một serial order, không cam kết reader
đứng sau mọi concurrent writer theo wall-clock. Nếu A có thể serialize trước B,
decision revision `7` và policy update revision `8` vẫn cùng commit hợp lệ.

Dùng `SERIALIZABLE` khi invariant/read-write dependencies phù hợp, đồng thời xử lý
SQLSTATE `40001` bằng full-transaction retry. Không nâng isolation chỉ để tránh
định nghĩa policy timing contract.

## Trade-off comparison

| Cách | Bảo vệ | Loser behavior | Contention/latency | Multi-instance |
| --- | --- | --- | --- | --- |
| Một immutable snapshot | Coherent evidence tại evaluation | Không có loser | Thấp | Có, nếu audit version durable |
| Versioned history + FK | Revision tồn tại, audit tái dựng được | Pointer update có thể no-op | Thêm storage/schema | Có |
| Conditional INSERT | Revision còn đúng tại final statement | affected-row `0`, retry/reject | Thấp–vừa | Có |
| `FOR SHARE` | Policy không đổi tới A commit | Updater block/timeout | Cao hơn trên hot row | Có |
| `REPEATABLE READ` | Stable transaction snapshot | Có thể conflict ở flow khác | Snapshot/retry cost | Có |
| JVM lock | Chỉ threads cùng process | Local thread chờ | Không bảo vệ DB-wide | Không |

## Failure behavior

- A rollback: decision biến mất; lock của A release.
- B rollback: policy revision mới invisible; A tiếp tục với old committed version.
- Conditional mismatch: rollback attempt, reload/recompute trong transaction mới.
- Lock timeout/deadlock: transaction hiện tại đã fail; không reuse persistence
  context.
- Crash sau commit/trước response: replay bằng `command_id`, không chạy lại refund
  side effect mù quáng.
- Policy-history write fail giữa chừng: transaction rollback cả version và pointer.

## Recommendation cho case này

Default:

1. dùng immutable `RefundPolicySnapshot` cho evaluation;
2. lưu policy revision, evaluated limit và outcome cùng decision;
3. giữ immutable policy history với foreign key;
4. thêm conditional final validation nếu policy phải còn current tại statement;
5. dùng `FOR SHARE` chỉ khi business thật sự yêu cầu policy không đổi tới commit.

`REPEATABLE READ` hữu ích khi toàn transaction cần stable view, nhưng không thay
cho việc định nghĩa và lưu decision evidence.

## Production checklist

### Semantics

- [ ] Đã định nghĩa policy có hiệu lực tại read, write statement hay commit.
- [ ] Approved decision join được đúng immutable policy revision.
- [ ] Không trộn fields từ nhiều `PolicyView`.
- [ ] Duplicate command và policy mutation được xử lý như hai invariants riêng.

### Transactions

- [ ] Effective isolation được đo trên physical transaction.
- [ ] Conditional affected-row `0` được map rõ.
- [ ] Retry tạo transaction mới và recompute toàn decision.
- [ ] Không có remote call trong transaction giữ row lock.
- [ ] Lock order, `lock_timeout`, deadlock handling đã xác định.

### Operations

- [ ] Có metric revision mismatch, retry, lock wait và timeout.
- [ ] Có query reconciliation cho decision/policy revision.
- [ ] Trace ghi revision read và revision persisted.
- [ ] Test chạy trên PostgreSQL thật, không suy luận từ H2.
- [ ] Hot merchant contention và connection-pool pressure được theo dõi.
