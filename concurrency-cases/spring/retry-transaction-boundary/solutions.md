# New transaction per attempt

## Mục tiêu thiết kế

Correct retry pipeline:

```text
classify retryable failure
  -> failed transaction rollback completes
  -> persistence context closes
  -> deadline/attempt budget check
  -> bounded backoff outside transaction
  -> new transaction
  -> reload and revalidate
  -> commit or fail
```

> **Nói ngắn gọn:** retry coordinator sở hữu attempts; transaction worker sở hữu
> đúng một attempt.

## Solution 1 — Separate retry coordinator và transactional worker

Đây là cấu trúc khuyến nghị với Spring Retry.

### One-attempt worker

```java
@Service
public class ReservationAttemptService {
    private final InventoryItemRepository inventory;
    private final ReservationRecordRepository reservations;

    public ReservationAttemptService(
        InventoryItemRepository inventory,
        ReservationRecordRepository reservations
    ) {
        this.inventory = inventory;
        this.reservations = reservations;
    }

    @Transactional
    public ReservationResult reserveOnce(
        UUID commandId,
        String sku,
        int quantity
    ) {
        Optional<ReservationRecord> existing =
            reservations.findByCommandId(commandId);

        if (existing.isPresent()) {
            return ReservationResult.replayed(existing.orElseThrow());
        }

        InventoryItem item = inventory.findById(sku)
            .orElseThrow();
        long expectedVersion = item.getVersion();

        item.reserve(quantity); // revalidates stock on fresh state
        reservations.save(ReservationRecord.accepted(
            commandId,
            sku,
            quantity
        ));

        inventory.flush(); // surface conflict inside this attempt boundary

        return ReservationResult.accepted(
            commandId,
            expectedVersion
        );
    }
}
```

Generated write:

```sql
update inventory_item
set available = ?,
    version = ?
where sku = ?
  and version = ?;
```

Unique command key remains required:

```sql
alter table reservation_record
    add constraint uk_reservation_command unique (command_id);
```

### Non-transactional coordinator

```java
@Service
public class ReservationRetryCoordinator {
    private final ReservationAttemptService attempts;

    public ReservationRetryCoordinator(
        ReservationAttemptService attempts
    ) {
        this.attempts = attempts;
    }

    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttemptsExpression =
            "${reservation.retry.max-attempts:4}",
        backoff = @Backoff(
            delayExpression =
                "${reservation.retry.initial-delay-ms:20}",
            maxDelayExpression =
                "${reservation.retry.max-delay-ms:200}",
            multiplierExpression =
                "${reservation.retry.multiplier:2.0}",
            random = true
        )
    )
    public ReservationResult reserve(
        UUID commandId,
        String sku,
        int quantity
    ) {
        return attempts.reserveOnce(commandId, sku, quantity);
    }
}
```

Coordinator không annotated `@Transactional`. Mỗi RetryInterceptor invocation gọi
`attempts` qua một bean proxy khác:

```text
Retry attempt 1
  -> ReservationAttemptService proxy begins Tx-1
  -> conflict
  -> proxy rolls back Tx-1 and rethrows
  -> RetryInterceptor catches

Retry attempt 2
  -> proxy begins Tx-2 with new EntityManager
  -> reload version 8
  -> commit version 9
```

Property defaults chỉ là example development configuration. Production values
phải dựa trên operation deadline, conflict distribution và database capacity.

### Khi attempts exhausted

Để final optimistic exception propagate hoặc dùng `@Recover` để map thành một
stable application result:

```java
@Recover
public ReservationResult exhausted(
    ObjectOptimisticLockingFailureException conflict,
    UUID commandId,
    String sku,
    int quantity
) {
    throw new ReservationContentionException(
        commandId,
        sku,
        conflict
    );
}
```

`@Recover` không được báo success. Caller cần biết operation chưa commit và có thể
query bằng command ID nếu response outcome từng bị ambiguous.

## Guard outer transaction

Solution 1 giả định coordinator là root use-case boundary. Nếu một transactional
caller wrap coordinator, worker `REQUIRED` sẽ join outer transaction.

Các lựa chọn:

- kiến trúc cấm transactional caller và có integration/architecture test;
- dùng `REQUIRES_NEW` cho attempt worker khi independent commit là đúng;
- dùng programmatic template với explicit propagation;
- tách retry thành asynchronous command handler không có outer transaction.

`REQUIRES_NEW` guarantee new attempt transaction nhưng suspend outer transaction,
có thể dùng thêm connection và commit độc lập. Chỉ chọn nếu atomicity contract cho
phép.

## Solution 2 — Manual coordinator loop ngoài transaction

Khi không dùng Spring Retry hoặc cần một operation deadline chung:

```java
@Service
public class ManualReservationRetryCoordinator {
    private final ReservationAttemptService attempts;
    private final RetryPolicy retryPolicy;
    private final RetryBackoff retryBackoff;

    public ReservationResult reserve(
        UUID commandId,
        String sku,
        int quantity,
        Instant deadline
    ) {
        for (int attempt = 1; ; attempt++) {
            try {
                return attempts.reserveOnce(
                    commandId,
                    sku,
                    quantity
                );
            } catch (ObjectOptimisticLockingFailureException conflict) {
                RetryDecision decision = retryPolicy.decide(
                    attempt,
                    deadline,
                    conflict
                );

                if (decision == RetryDecision.STOP) {
                    throw conflict;
                }

                retryBackoff.pauseInterruptibly(
                    attempt,
                    deadline
                );
            }
        }
    }
}
```

Catch chạy sau `reserveOnce()` proxy đã rollback. Backoff không giữ database
connection/transaction. `pauseInterruptibly` phải tôn trọng cancellation và không
vượt operation deadline.

Loop này correct vì transaction nằm trong called bean, không nằm quanh loop.

## Solution 3 — `TransactionTemplate` cho explicit attempt

```java
@Service
public class ProgrammaticReservationRetry {
    private final TransactionTemplate attemptTransaction;
    private final ReservationWork work;
    private final RetryPolicy retryPolicy;
    private final RetryBackoff retryBackoff;

    public ProgrammaticReservationRetry(
        PlatformTransactionManager transactionManager,
        ReservationWork work,
        RetryPolicy retryPolicy,
        RetryBackoff retryBackoff
    ) {
        this.attemptTransaction =
            new TransactionTemplate(transactionManager);
        this.attemptTransaction.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW
        );
        this.work = work;
        this.retryPolicy = retryPolicy;
        this.retryBackoff = retryBackoff;
    }

    public ReservationResult reserve(
        ReservationCommand command,
        Instant deadline
    ) {
        for (int attempt = 1; ; attempt++) {
            try {
                return attemptTransaction.execute(status ->
                    work.reserve(command)
                );
            } catch (ObjectOptimisticLockingFailureException conflict) {
                if (!retryPolicy.canRetry(attempt, deadline, conflict)) {
                    throw conflict;
                }
                retryBackoff.pauseInterruptibly(attempt, deadline);
            }
        }
    }
}
```

`execute()` rollback/close hoàn tất trước khi exception tới catch. `REQUIRES_NEW`
làm semantics explicit cả khi caller có transaction, nhưng independent commit và
connection cost phải được chấp nhận.

Nếu method chắc chắn là root và không có outer transaction, `REQUIRED` cũng tạo
new transaction cho mỗi separate `execute()` call sau previous completion.

## Solution 4 — Explicit advisor order

Có thể cấu hình retry advice ngoài transaction advice. Tuy nhiên đây là lựa chọn
kém self-evident hơn separate beans:

```text
Retry advisor order < Transaction advisor order
=> retry interceptor is outer
```

Giá trị/order cụ thể phụ thuộc configuration và framework setup. Nếu chọn:

- inspect actual advisor chain trong integration test;
- assert transaction identity thay đổi giữa attempts;
- assert previous completion là rollback trước next begin;
- khóa configuration bằng test khi nâng version.

Không chỉ unit test annotations hoặc assume lower/higher number mà không inspect
runtime proxy.

## Failure classification

Allowlist:

```java
boolean isRetryable(Throwable failure) {
    Throwable specific = mostSpecificCause(failure);

    return failure instanceof ObjectOptimisticLockingFailureException
        || isSqlState(specific, "40001")
        || isSqlState(specific, "40P01");
}
```

SQLSTATE classification cần traverse cause chain một cách có kiểm soát. Không
retry `InsufficientStockException`, duplicate command có stored result, invalid
input, mapping bug hoặc cancellation.

Một deadlock/serialization retry cũng phải chạy toàn bộ database unit lại trong
new transaction. Domain-specific external side-effect safety vẫn cần idempotency/
outbox.

## Business revalidation

Attempt 2 không replay `available = 2` decision. Nó reload:

```java
InventoryItem current = inventory.findById(command.sku())
    .orElseThrow();
current.reserve(command.quantity());
```

Nếu current stock không đủ, `InsufficientStockException` là final domain outcome,
không phải optimistic conflict cần retry tiếp.

## Conflict outcome

| Actor | Detector | Immediate behavior | Final behavior |
| --- | --- | --- | --- |
| Winner A | Version predicate matches | Update/commit | Success |
| Loser B attempt 1 | Affected rows = 0 | Fail and rollback | Eligible for retry |
| B attempt 2 | Reload v8, predicate matches | Update/commit | Success |
| Exhausted loser | Retry budget/deadline | Stop | Explicit contention failure |
| Business loser | Fresh stock insufficient | No retry | Domain rejection |

## Alternatives khi hot-key contention cao

Retry không phải giải pháp throughput cho mọi workload:

- atomic conditional decrement có thể tránh read/write retry;
- `PESSIMISTIC_WRITE` chuyển conflict thành blocking queue;
- serialize commands theo aggregate key;
- shard/partition hot aggregate;
- admission control/backpressure;
- return contention failure thay vì retry server-side.

Mỗi lựa chọn đổi latency, lock duration, database load và horizontal scaling.
Không có benchmark phổ quát; đo conflict rate và final throughput trên workload
đại diện.

## So sánh

| Lựa chọn | Boundary clarity | Outer Tx safety | Complexity | Main trade-off |
| --- | --- | --- | --- | --- |
| Separate coordinator/worker | Cao | Cần guard caller | Thấp-vừa | Khuyến nghị mặc định |
| Manual outer loop | Cao | Cần guard caller | Vừa | Deadline/classification linh hoạt |
| `TransactionTemplate` NEW | Rất cao | Cao | Vừa | Independent commit, connection cost |
| Advisor ordering | Thấp-vừa | Phụ thuộc chain | Vừa-cao | Configuration dễ regression |
| Same-Tx loop + clear | Sai | Sai | Có vẻ thấp | Không thể tạo clean retry |

## Production checklist

### Boundary

- [ ] Retry catch nằm ngoài failed transaction interceptor.
- [ ] Mỗi attempt có transaction/persistence context identity mới.
- [ ] Previous rollback hoàn tất trước backoff.
- [ ] Backoff không giữ database connection.
- [ ] Outer transaction assumptions được enforce/test.

### Policy

- [ ] Retryable failures dùng allowlist/type/SQLSTATE.
- [ ] Attempts và overall deadline đều bounded.
- [ ] Backoff có jitter và tôn trọng interruption.
- [ ] Fresh attempt revalidates business state.
- [ ] Exhaustion trả explicit non-success outcome.

### Data và operations

- [ ] `@Version` column/predicate được integration test.
- [ ] Command ID uniqueness xử lý duplicate delivery riêng.
- [ ] External side effects nằm sau commit qua outbox khi phù hợp.
- [ ] Conflict/attempt/final-outcome metrics có aggregate key dimension an toàn.
- [ ] Hot keys có fallback ngoài optimistic retry.
