# Broken two-read refund decision

## Current policy entity

```java
@Entity
@Table(name = "merchant_refund_policy")
public class MerchantRefundPolicy {
    @Id
    @Column(name = "merchant_id", nullable = false)
    private UUID merchantId;

    @Column(name = "auto_refund_limit", nullable = false, precision = 19, scale = 2)
    private BigDecimal autoRefundLimit;

    @Column(nullable = false)
    private boolean active;

    @Column(nullable = false)
    private long revision;

    protected MerchantRefundPolicy() {
    }

    public void changeLimit(BigDecimal newLimit) {
        if (newLimit.signum() < 0) {
            throw new IllegalArgumentException("newLimit must be non-negative");
        }
        autoRefundLimit = newLimit;
        revision++;
    }

    public boolean allows(BigDecimal amount) {
        return active && amount.compareTo(autoRefundLimit) <= 0;
    }

    public BigDecimal getAutoRefundLimit() {
        return autoRefundLimit;
    }

    public long getRevision() {
        return revision;
    }
}
```

`revision` ở đây là business policy revision, chưa phải JPA `@Version`. B update
một row; case không tập trung vào lost update giữa nhiều administrators.

## Scalar projection luôn phát SQL

```java
public interface PolicyView {
    BigDecimal getAutoRefundLimit();
    boolean getActive();
    long getRevision();
}

public interface MerchantRefundPolicyRepository
    extends JpaRepository<MerchantRefundPolicy, UUID> {

    @Query(
        value = """
            select auto_refund_limit as autoRefundLimit,
                   active as active,
                   revision as revision
              from merchant_refund_policy
             where merchant_id = :merchantId
            """,
        nativeQuery = true
    )
    Optional<PolicyView> readPolicy(UUID merchantId);
}
```

Projection scalar không phải managed entity. Mỗi `readPolicy()` thực thi một
SELECT mới, nên PostgreSQL có thể cấp statement snapshot mới.

## Decision entity

```java
@Entity
@Table(
    name = "refund_decision",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_refund_decision_command",
        columnNames = "command_id"
    )
)
public class RefundDecision {
    @Id
    private UUID id;

    @Column(name = "command_id", nullable = false)
    private UUID commandId;

    @Column(name = "merchant_id", nullable = false)
    private UUID merchantId;

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private RefundOutcome outcome;

    @Column(name = "evaluated_limit", nullable = false, precision = 19, scale = 2)
    private BigDecimal evaluatedLimit;

    @Column(name = "policy_revision", nullable = false)
    private long policyRevision;

    protected RefundDecision() {
    }

    public static RefundDecision approved(
        UUID commandId,
        UUID merchantId,
        BigDecimal amount,
        BigDecimal evaluatedLimit,
        long policyRevision
    ) {
        RefundDecision decision = new RefundDecision();
        decision.id = UUID.randomUUID();
        decision.commandId = commandId;
        decision.merchantId = merchantId;
        decision.amount = amount;
        decision.outcome = RefundOutcome.APPROVED;
        decision.evaluatedLimit = evaluatedLimit;
        decision.policyRevision = policyRevision;
        return decision;
    }

    public UUID getId() {
        return id;
    }
}
```

Unique `command_id` bảo vệ duplicate command, không bảo vệ policy consistency.
Đây là hai invariants khác nhau.

## Broken service

```java
@Service
public class RefundDecisionService {
    private final MerchantRefundPolicyRepository policies;
    private final RefundDecisionRepository decisions;
    private final LocalRefundRules localRules;

    public RefundDecisionService(
        MerchantRefundPolicyRepository policies,
        RefundDecisionRepository decisions,
        LocalRefundRules localRules
    ) {
        this.policies = policies;
        this.decisions = decisions;
        this.localRules = localRules;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public RefundResult decide(
        UUID commandId,
        UUID merchantId,
        BigDecimal amount
    ) {
        PolicyView eligibility = policies.readPolicy(merchantId)
            .orElseThrow(PolicyNotFoundException::new);

        if (!eligibility.getActive()
            || amount.compareTo(eligibility.getAutoRefundLimit()) > 0) {
            return RefundResult.manualReview();
        }

        localRules.validate(amount);

        PolicyView audit = policies.readPolicy(merchantId)
            .orElseThrow(PolicyNotFoundException::new);

        RefundDecision saved = decisions.save(
            RefundDecision.approved(
                commandId,
                merchantId,
                amount,
                eligibility.getAutoRefundLimit(),
                audit.getRevision()
            )
        );
        return RefundResult.approved(saved.getId());
    }
}
```

Code không ngớ ngẩn: lần đọc thứ hai thường xuất hiện sau refactor để lấy audit
metadata “mới nhất”, hoặc ở helper khác trong cùng call graph. Bug nằm ở việc
trộn decision từ lần đọc đầu với revision từ lần đọc sau.

> **Nói ngắn gọn:** `@Transactional` giữ atomic commit/rollback, nhưng
> `READ_COMMITTED` không hứa mọi SELECT trong transaction dùng cùng snapshot.

## Concurrent policy update

```java
@Service
public class PolicyAdministrationService {
    private final MerchantRefundPolicyRepository policies;

    public PolicyAdministrationService(
        MerchantRefundPolicyRepository policies
    ) {
        this.policies = policies;
    }

    @Transactional
    public void lowerLimit(UUID merchantId, BigDecimal newLimit) {
        MerchantRefundPolicy policy = policies.findById(merchantId)
            .orElseThrow(PolicyNotFoundException::new);
        policy.changeLimit(newLimit);
    }
}
```

Hibernate flush của B sinh:

```sql
update merchant_refund_policy
   set active = true,
       auto_refund_limit = 50.00,
       revision = 8
 where merchant_id = '...';
```

UPDATE giữ row lock tới B commit. A dùng plain SELECT nên đọc committed version
phù hợp thay vì chờ để “giữ” revision cũ.

## SQL tương đương

```sql
begin isolation level read committed;

select auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId;
-- 100.00, true, 7

-- session B updates revision 8 and commits here

select auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId;
-- 50.00, true, 8

insert into refund_decision(
    id,
    command_id,
    merchant_id,
    amount,
    outcome,
    evaluated_limit,
    policy_revision
)
values (
    :id,
    :commandId,
    :merchantId,
    80.00,
    'APPROVED',
    100.00,
    8
);

commit;
```

## JPA first-level cache có thể che anomaly

Nếu code gọi `findById()` hai lần trong cùng persistence context, Hibernate có
thể trả lại cùng managed entity mà không phát SELECT thứ hai. Khi đó application
thấy revision `7`, nhưng đó là identity-map behavior, không phải bằng chứng
PostgreSQL `READ COMMITTED` dùng stable snapshot.

Các thao tác sau có thể làm database bị đọc lại:

- scalar/native projection như sample;
- `EntityManager.refresh(...)`;
- `EntityManager.clear()` rồi load;
- query path ở một persistence context khác;
- JDBC query trong cùng physical transaction.

Đừng dùng first-level cache làm isolation contract.

## Preconditions tái hiện

1. A và B có physical transactions/connections độc lập.
2. Effective isolation của A là PostgreSQL `READ COMMITTED`.
3. Hai repository calls của A thực sự phát hai SELECT.
4. B commit sau SELECT đầu và trước SELECT thứ hai.
5. A giữ kết quả eligibility từ revision `7`.
6. INSERT không lock hoặc validate policy revision.
7. Không có outer test transaction che transaction boundaries.

## Những cách sửa chưa đủ

### Chỉ thêm `@Transactional`

Đã có transaction nhưng isolation vẫn dùng statement snapshots.

### Chỉ bỏ lần đọc thứ hai

Việc này tạo coherent in-memory decision, nhưng cần lưu đủ evidence và xác định rõ
policy cũ có còn hợp lệ khi concurrent update commit hay không.

### Dùng `synchronized`

Chỉ serialize threads trong một JVM và không chặn Policy Admin chạy trên node
khác hoặc direct database client.

### Gọi `flush()` sớm

Flush INSERT không làm policy snapshot ổn định và không tạo revision predicate.

### Retry trong cùng transaction

Một retry cần transaction/snapshot mới. Lặp lại query trong cùng transaction
`REPEATABLE READ` chỉ tiếp tục thấy snapshot cũ.
