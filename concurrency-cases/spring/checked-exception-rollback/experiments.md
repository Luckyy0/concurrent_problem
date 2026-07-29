# PostgreSQL integration experiments

## Mục tiêu

Tests phải phân biệt:

- Java method outcome;
- Spring transaction completion status;
- committed PostgreSQL state;
- state mà một actor khác có thể quan sát.

Mock repository hoặc inspect annotation chỉ xác nhận configuration bề mặt. Case
cần gọi bean qua proxy và đọc lại database bằng transaction mới.

> **Nói ngắn gọn:** assert exception chưa đủ; phải chứng minh database đã commit hay
> rollback và dispatcher thấy gì.

## Test topology

```text
test thread
  ├─ request executor -> Spring proxy -> Tx-A -> policy gate -> exception
  ├─ dispatcher executor -> waits for completion -> Tx-B -> query PROCESSING
  └─ inspector bean -> independent Tx-C -> committed-state snapshot
```

Test method không annotated `@Transactional`. Nếu test framework bọc một outer
transaction, service có thể join nó và committed-state assertions bị che bởi
rollback lúc test kết thúc.

## PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class CheckedExceptionRollbackIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Không dùng H2 để kết luận về flush, row lock hoặc committed visibility. Schema tối
thiểu:

```sql
create table wallet_account (
    id uuid primary key,
    available_balance bigint not null
);

create table payout_request (
    id uuid primary key,
    wallet_id uuid not null references wallet_account(id),
    beneficiary_id uuid not null,
    amount bigint not null check (amount > 0),
    status varchar(32) not null
);

create table ledger_entry (
    id uuid primary key,
    payout_id uuid not null references payout_request(id),
    wallet_id uuid not null references wallet_account(id),
    amount bigint not null check (amount > 0),
    entry_type varchar(32) not null,
    constraint uk_ledger_payout_type
        unique (payout_id, entry_type)
);
```

Fixture setup commits:

```text
P-42: RECEIVED, amount 300, beneficiary blocked
W-7:  available_balance 1000
ledger: no entry for P-42
```

Mỗi test dùng IDs riêng hoặc reset data trong committed setup transaction.

## Transaction completion probe

Test-only probe ghi nhận callback của đúng transaction:

```java
@Component
final class TransactionCompletionProbe {
    private final AtomicInteger completionStatus =
        new AtomicInteger(Integer.MIN_VALUE);

    void register() {
        completionStatus.set(Integer.MIN_VALUE);
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void afterCompletion(int status) {
                    completionStatus.set(status);
                }
            }
        );
    }

    int status() {
        return completionStatus.get();
    }
}
```

Fixture service gọi `completionProbe.register()` ngay sau khi vào proxied method.
Sau khi proxy call return hoặc throw tới test, `afterCompletion` đã chạy.

Constants cần assert:

```java
TransactionSynchronization.STATUS_COMMITTED
TransactionSynchronization.STATUS_ROLLED_BACK
```

Probe phục vụ test/diagnostic, không phải business mechanism.

## Committed-state inspector

Snapshot phải đọc trong transaction mới, không reuse persistence context của
failed call:

```java
public record PayoutSnapshot(
    PayoutStatus status,
    long availableBalance,
    long holdCount
) {}
```

```java
@Service
class CommittedStateInspector {
    private final JdbcTemplate jdbc;

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    public PayoutSnapshot read(UUID payoutId, UUID walletId) {
        PayoutStatus status = PayoutStatus.valueOf(
            jdbc.queryForObject(
                "select status from payout_request where id = ?",
                String.class,
                payoutId
            )
        );
        long balance = jdbc.queryForObject(
            "select available_balance from wallet_account where id = ?",
            Long.class,
            walletId
        );
        long holds = jdbc.queryForObject(
            """
            select count(*)
            from ledger_entry
            where payout_id = ?
              and entry_type = 'PAYOUT_HOLD'
            """,
            Long.class,
            payoutId
        );
        return new PayoutSnapshot(status, balance, holds);
    }
}
```

## Experiment 1 — Default rule commits checked rejection

`broken.prepare()` dùng default `@Transactional` và policy reject ngay:

```java
@Test
void checkedBusinessExceptionCommitsWithDefaultRule() {
    assertThatThrownBy(() -> broken.prepare(PAYOUT_ID))
        .isExactlyInstanceOf(BeneficiaryRejectedException.class);

    PayoutSnapshot committed = inspector.read(PAYOUT_ID, WALLET_ID);

    assertThat(completionProbe.status())
        .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);
    assertThat(committed.status()).isEqualTo(PayoutStatus.PROCESSING);
    assertThat(committed.availableBalance()).isEqualTo(700L);
    assertThat(committed.holdCount()).isEqualTo(1L);
}
```

Test intentionally asserts broken actual behavior. Nó bảo vệ tài liệu khỏi một
test false-positive chỉ thấy exception rồi tự kết luận rollback.

## Experiment 2 — `rollbackFor` khôi phục invariant

Fixed fixture chỉ khác annotation:

```java
@Transactional(
    rollbackFor = BeneficiaryRejectedException.class
)
public void prepare(UUID payoutId)
        throws BeneficiaryRejectedException {
    completionProbe.register();
    preparationWork.prepare(payoutId);
}
```

Test:

```java
@Test
void explicitRollbackRuleRollsBackEveryDatabaseMutation() {
    assertThatThrownBy(() -> fixed.prepare(PAYOUT_ID))
        .isExactlyInstanceOf(BeneficiaryRejectedException.class);

    PayoutSnapshot committed = inspector.read(PAYOUT_ID, WALLET_ID);

    assertThat(completionProbe.status())
        .isEqualTo(TransactionSynchronization.STATUS_ROLLED_BACK);
    assertThat(committed.status()).isEqualTo(PayoutStatus.RECEIVED);
    assertThat(committed.availableBalance()).isEqualTo(1000L);
    assertThat(committed.holdCount()).isZero();
    assertThat(dispatcher.hasExecutable(PAYOUT_ID)).isFalse();
}
```

Business assertions quan trọng hơn exception assertion: payout không executable,
funds không bị hold và ledger không có entry.

## Gate cho interleaving Request A / Dispatcher B

```java
final class RejectionGate {
    private final CountDownLatch policyEntered = new CountDownLatch(1);
    private final CountDownLatch allowRejection = new CountDownLatch(1);
    private final CountDownLatch allowDispatcherRead = new CountDownLatch(1);

    void policyEntered() {
        policyEntered.countDown();
        awaitOrFail(allowRejection, Duration.ofSeconds(5));
    }

    void awaitPolicyEntered() {
        awaitOrFail(policyEntered, Duration.ofSeconds(5));
    }

    void releaseRejection() {
        allowRejection.countDown();
    }

    void awaitDispatcherPermission() {
        awaitOrFail(allowDispatcherRead, Duration.ofSeconds(5));
    }

    void releaseDispatcher() {
        allowDispatcherRead.countDown();
    }

    void releaseAll() {
        allowRejection.countDown();
        allowDispatcherRead.countDown();
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for test gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

Blocking policy gọi `gate.policyEntered()` rồi ném checked exception. Mọi wait có
timeout; `finally` release cả hai gates.

## Experiment 3 — Dispatcher quan sát broken commit

```java
@Test
void dispatcherCanObserveExecutableStateAfterCallerGetsRejection()
        throws Exception {
    RejectionGate gate = new RejectionGate();
    policy.install(gate, PAYOUT_ID);
    ExecutorService actors = Executors.newFixedThreadPool(2);

    try {
        Future<Throwable> requestOutcome = actors.submit(() ->
            catchThrowable(() -> broken.prepare(PAYOUT_ID))
        );

        gate.awaitPolicyEntered();

        Future<Boolean> dispatcherOutcome = actors.submit(() -> {
            gate.awaitDispatcherPermission();
            return dispatcher.hasExecutable(PAYOUT_ID);
        });

        gate.releaseRejection();
        Throwable rejected = requestOutcome.get(5, TimeUnit.SECONDS);

        assertThat(rejected)
            .isExactlyInstanceOf(BeneficiaryRejectedException.class);
        assertThat(completionProbe.status())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);

        gate.releaseDispatcher();
        assertThat(dispatcherOutcome.get(5, TimeUnit.SECONDS)).isTrue();
    } finally {
        gate.releaseAll();
        policy.clear();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

`requestOutcome` chỉ hoàn tất sau khi transaction interceptor commit và rethrow.
Dispatcher Tx-B vì vậy đọc committed state; test không đoán timing bằng delay.

Chạy cùng test với `fixed.prepare()` phải đổi hai assertions:

```text
completion = ROLLED_BACK
dispatcherOutcome = false
```

## Experiment 4 — Flush sớm vẫn rollback được

Tạo fixed fixture gọi `entityManager.flush()` sau khi ghi ledger nhưng trước
policy:

```java
ledger.save(LedgerEntry.payoutHold(...));
entityManager.flush();
beneficiaryPolicy.verify(beneficiaryId);
```

Sau checked exception với `rollbackFor`, assert lại toàn bộ initial state và
`STATUS_ROLLED_BACK`. Test chứng minh SQL execution không đồng nghĩa durable
commit.

Một inspector chạy trước rollback có thể block trên row lock hoặc chỉ thấy old
committed version tùy query; test chính nên đọc sau completion để tập trung vào
rollback invariant.

## Experiment 5 — Outer catch gặp rollback-only

Inner fixed bean được gọi qua proxy và join outer transaction:

```java
@Service
class BatchFacade {
    private final FixedPayoutPreparationService preparation;

    @Transactional
    public BatchResult run(UUID payoutId) {
        try {
            preparation.prepare(payoutId);
            return BatchResult.ready();
        } catch (BeneficiaryRejectedException rejected) {
            return BatchResult.rejected();
        }
    }
}
```

Expected:

```java
@Test
void swallowingInnerRollbackSurfacesUnexpectedRollbackAtOuterCommit() {
    assertThatThrownBy(() -> batch.run(PAYOUT_ID))
        .isInstanceOf(UnexpectedRollbackException.class);

    PayoutSnapshot committed = inspector.read(PAYOUT_ID, WALLET_ID);
    assertThat(committed.status()).isEqualTo(PayoutStatus.RECEIVED);
    assertThat(committed.availableBalance()).isEqualTo(1000L);
    assertThat(committed.holdCount()).isZero();
}
```

Điều cần bảo vệ là rollback state. `UnexpectedRollbackException` cảnh báo outer
code đã return bình thường dù physical transaction không thể commit.

## Experiment 6 — Committed `Rejected` result

Với Solution 3:

```java
@Test
void expectedRejectionCommitsOnlyNonExecutableRejectedState() {
    PreparationResult result = resultBased.prepare(PAYOUT_ID);

    assertThat(result)
        .isEqualTo(new PreparationResult.Rejected(
            PAYOUT_ID,
            "BENEFICIARY_BLOCKED"
        ));

    PayoutSnapshot committed = inspector.read(PAYOUT_ID, WALLET_ID);
    assertThat(committed.status()).isEqualTo(PayoutStatus.REJECTED);
    assertThat(committed.availableBalance()).isEqualTo(1000L);
    assertThat(committed.holdCount()).isZero();
    assertThat(dispatcher.hasExecutable(PAYOUT_ID)).isFalse();
    assertThat(outbox.countReadyEvents(PAYOUT_ID)).isZero();
}
```

Đây là commit đúng có chủ đích. Test không kỳ vọng exception hoặc rollback.

## Coverage matrix

| Scenario | Method outcome | Tx outcome | Committed state |
| --- | --- | --- | --- |
| Default + checked rejection | Throws checked | Commit | Broken `PROCESSING` + hold |
| `rollbackFor` + checked rejection | Throws checked | Rollback | Initial state |
| Early flush + `rollbackFor` | Throws checked | Rollback | Initial state |
| Inner rollback, outer catches | Late `UnexpectedRollbackException` | Rollback | Initial state |
| Result-based rejection | Returns `Rejected` | Commit | `REJECTED`, no hold |
| Unchecked rejection | Throws runtime | Rollback | Initial state |

## Chống flaky

- Mọi latch, future và executor termination đều có bounded timeout.
- `finally` release gates trước khi shutdown executor.
- Không bọc test method bằng transaction.
- Inspector luôn dùng persistence context/transaction mới.
- Assert transaction completion và business state.
- Chạy class theo `SAME_THREAD` vì policy gate/completion probe là single-scenario.
- Dùng committed fixture setup và IDs riêng với các test classes khác.
- Dispatcher query chạy qua bean proxy, không qua shared EntityManager.
- Không dùng time-based delay để tạo interleaving.

## Production verification

Theo dõi:

- rejected response cùng payout/correlation ID;
- completion status ở diagnostic environment;
- count `REJECTED` payouts có hold/outbox — phải bằng zero;
- count exception outcome nhưng payout vẫn executable — phải bằng zero;
- `UnexpectedRollbackException` tại outer boundaries;
- dispatcher claims theo state và source transaction;
- ledger/outbox uniqueness violations và duplicate command rate.

Reconciliation query minh họa:

```sql
select p.id, p.status, count(l.id) as hold_count
from payout_request p
left join ledger_entry l
  on l.payout_id = p.id
 and l.entry_type = 'PAYOUT_HOLD'
where p.status = 'REJECTED'
group by p.id, p.status
having count(l.id) > 0;
```

Query phải trả zero rows. Nó là guardrail vận hành, không thay thế transaction
contract và integration tests.
