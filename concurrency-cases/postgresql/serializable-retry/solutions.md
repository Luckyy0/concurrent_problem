# Giải pháp retry toàn transaction trong boundary mới

## Mục tiêu thiết kế

```text
non-transactional coordinator
  → transactional SERIALIZABLE attempt
  → commit success: trả durable decision
  → 40001: rollback hoàn tất
  → bounded backoff bên ngoài transaction
  → fresh attempt reload toàn bộ state
```

Command ID không đổi giữa attempts. Chỉ successful transaction được ghi decision,
reservation và outbox intent.

## Giải pháp 1 — Tách coordinator và attempt worker

### Command và outcome

```java
package example.limit;

import java.math.BigDecimal;
import java.util.UUID;

public record ReserveLimitCommand(
        UUID commandId,
        long merchantId,
        BigDecimal amount
) {
    public ReserveLimitCommand {
        if (commandId == null) {
            throw new IllegalArgumentException("commandId is required");
        }
        if (amount == null || amount.signum() <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }
    }
}

public record LimitDecision(
        UUID commandId,
        Outcome outcome
) {
    public enum Outcome {
        ACCEPTED,
        REJECTED
    }
}
```

### Một attempt trong transaction riêng

```java
package example.limit;

import java.math.BigDecimal;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class SerializableLimitAttempt {

    private final MerchantLimitRepository limits;
    private final CreditReservationRepository reservations;
    private final LimitCommandDecisionRepository decisions;
    private final OutboxRepository outbox;

    public SerializableLimitAttempt(
            MerchantLimitRepository limits,
            CreditReservationRepository reservations,
            LimitCommandDecisionRepository decisions,
            OutboxRepository outbox
    ) {
        this.limits = limits;
        this.reservations = reservations;
        this.decisions = decisions;
        this.outbox = outbox;
    }

    @Transactional(
            propagation = Propagation.REQUIRES_NEW,
            isolation = Isolation.SERIALIZABLE
    )
    public LimitDecision execute(ReserveLimitCommand command) {
        var existing = decisions.findById(command.commandId());
        if (existing.isPresent()) {
            return existing.orElseThrow().toResult();
        }

        BigDecimal limit = limits.findById(command.merchantId())
                .orElseThrow(() ->
                        new MerchantLimitNotFoundException(command.merchantId())
                )
                .limitAmount();
        BigDecimal active = reservations.activeTotal(command.merchantId());

        LimitDecision result;
        if (active.add(command.amount()).compareTo(limit) > 0) {
            var decision = LimitCommandDecision.rejected(command);
            decisions.save(decision);
            result = decision.toResult();
        } else {
            reservations.save(CreditReservation.accepted(command));
            var decision = LimitCommandDecision.accepted(command);
            decisions.save(decision);
            outbox.save(OutboxEvent.limitReservationAccepted(command));
            result = decision.toResult();
        }

        reservations.flush();
        decisions.flush();
        outbox.flush();
        return result;
    }
}
```

`REQUIRES_NEW` được dùng vì coordinator là root use case không có transaction và
mỗi attempt phải độc lập. Nếu method này bị gọi từ một outer business transaction,
independent commit có thể phá outer atomicity và tốn thêm connection khi suspend;
kiến trúc phải cấm hoặc thiết kế rõ trường hợp đó.

`execute()` có thể tạo result trước commit, nhưng caller chỉ nhận result sau khi
transaction proxy commit thành công. Nếu commit trả `40001`, exception đi ra
coordinator và result không được sử dụng.

> **Nói ngắn gọn:** worker sở hữu đúng một attempt; coordinator sở hữu policy
> retry. Không component nào vừa giữ transaction vừa chờ backoff.

## Phân loại PostgreSQL failure

```java
package example.limit;

import java.sql.SQLException;
import java.util.Set;
import org.postgresql.util.PSQLException;

public final class PostgreSqlFailures {

    private static final Set<String> COMMAND_CONSTRAINTS = Set.of(
            "limit_command_decision_pkey",
            "credit_reservation_command_id_key"
    );

    private PostgreSqlFailures() {
    }

    public static boolean isSerializationFailure(Throwable failure) {
        return hasSqlState(failure, "40001");
    }

    public static boolean isDeadlock(Throwable failure) {
        return hasSqlState(failure, "40P01");
    }

    public static boolean isDuplicateCommand(Throwable failure) {
        for (Throwable current = failure;
                current != null;
                current = current.getCause()) {
            if (current instanceof PSQLException postgres
                    && "23505".equals(postgres.getSQLState())
                    && postgres.getServerErrorMessage() != null
                    && COMMAND_CONSTRAINTS.contains(
                            postgres.getServerErrorMessage().getConstraint()
                    )) {
                return true;
            }
        }
        return false;
    }

    private static boolean hasSqlState(
            Throwable failure,
            String expected
    ) {
        for (Throwable current = failure;
                current != null;
                current = current.getCause()) {
            if (current instanceof SQLException sql) {
                for (SQLException chained = sql;
                        chained != null;
                        chained = chained.getNextException()) {
                    if (expected.equals(chained.getSQLState())) {
                        return true;
                    }
                }
            }
        }
        return false;
    }
}
```

Constraint names là database contract nên migration không được đổi im lặng.
`23505` trên constraint khác phải propagate; không phải mọi uniqueness failure là
duplicate command.

## Retry policy có deadline

```java
package example.limit;

import java.time.Clock;
import java.time.Instant;
import org.springframework.stereotype.Component;

@Component
public final class SerializableRetryPolicy {

    private static final int MAX_ATTEMPTS = 4;
    private final Clock clock;

    public SerializableRetryPolicy(Clock clock) {
        this.clock = clock;
    }

    public boolean canRetry(int completedAttempt, Instant deadline) {
        return completedAttempt < MAX_ATTEMPTS
                && clock.instant().isBefore(deadline);
    }
}
```

Production values phải lấy từ configuration đã validate; `4` chỉ làm sample
boundary dễ đọc, không phải benchmark hay universal recommendation.

### Backoff có jitter, chạy ngoài transaction

```java
package example.limit;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.locks.LockSupport;
import org.springframework.stereotype.Component;

@Component
public final class RetryBackoff {

    private static final Duration INITIAL = Duration.ofMillis(20);
    private static final Duration MAXIMUM = Duration.ofMillis(200);
    private final Clock clock;

    public RetryBackoff(Clock clock) {
        this.clock = clock;
    }

    public void pause(int completedAttempt, Instant deadline) {
        long factor = 1L << Math.min(completedAttempt - 1, 10);
        long baseMillis = Math.min(
                MAXIMUM.toMillis(),
                INITIAL.toMillis() * factor
        );
        long jitterMillis = ThreadLocalRandom.current()
                .nextLong(Math.max(1, baseMillis / 2));
        long remainingMillis = Math.max(
                0,
                Duration.between(clock.instant(), deadline).toMillis()
        );
        long pauseMillis = Math.min(
                baseMillis + jitterMillis,
                remainingMillis
        );

        LockSupport.parkNanos(Duration.ofMillis(pauseMillis).toNanos());
        if (Thread.currentThread().isInterrupted()) {
            throw new RetryInterruptedException();
        }
    }
}
```

Application dùng virtual threads/async scheduling có thể thay blocking backoff
bằng scheduler phù hợp. Contract không đổi: không giữ transaction/connection,
tôn trọng cancellation và deadline.

## Coordinator không có transaction

```java
package example.limit;

import java.time.Duration;
import java.time.Instant;
import org.springframework.stereotype.Service;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Service
public class LimitReservationCoordinator {

    private final SerializableLimitAttempt attempts;
    private final LimitDecisionReplay replay;
    private final SerializableRetryPolicy retryPolicy;
    private final RetryBackoff backoff;

    public LimitReservationCoordinator(
            SerializableLimitAttempt attempts,
            LimitDecisionReplay replay,
            SerializableRetryPolicy retryPolicy,
            RetryBackoff backoff
    ) {
        this.attempts = attempts;
        this.replay = replay;
        this.retryPolicy = retryPolicy;
        this.backoff = backoff;
    }

    public LimitDecision reserve(ReserveLimitCommand command) {
        if (TransactionSynchronizationManager
                .isActualTransactionActive()) {
            throw new IllegalStateException(
                    "Retry coordinator must not join an outer transaction"
            );
        }

        Instant deadline = Instant.now().plus(Duration.ofSeconds(2));
        for (int attempt = 1; ; attempt++) {
            try {
                return attempts.execute(command);
            } catch (RuntimeException failure) {
                if (PostgreSqlFailures.isDuplicateCommand(failure)) {
                    return replay.requireCommitted(command.commandId());
                }
                if (!PostgreSqlFailures.isSerializationFailure(failure)) {
                    throw failure;
                }
                if (!retryPolicy.canRetry(attempt, deadline)) {
                    throw new LimitContentionException(
                            command.commandId(),
                            attempt,
                            failure
                    );
                }
                backoff.pause(attempt, deadline);
            }
        }
    }
}
```

Nên inject `Clock` và configurable operation deadline thay vì gọi
`Instant.now()` trực tiếp trong production implementation; đoạn code giữ phần
orchestration ngắn. `attempts.execute()` return/throw sau transaction interceptor,
nên catch chạy khi rollback/commit đã kết thúc.

Replay dùng read transaction mới:

```java
@Service
public class LimitDecisionReplay {

    private final LimitCommandDecisionRepository decisions;

    public LimitDecisionReplay(
            LimitCommandDecisionRepository decisions
    ) {
        this.decisions = decisions;
    }

    @Transactional(
            propagation = Propagation.REQUIRES_NEW,
            readOnly = true
    )
    public LimitDecision requireCommitted(UUID commandId) {
        return decisions.findById(commandId)
                .orElseThrow(() ->
                        new DuplicateCommandOutcomeUnavailable(commandId)
                )
                .toResult();
    }
}
```

Nếu connection mất với ambiguous commit, cùng command ID gọi coordinator lại và
đi qua replay check đầu attempt. API không tự tạo ID mới.

## Spring Retry là phương án tương đương

Có thể dùng `RetryTemplate`/`@Retryable` khi:

- exception được translate thành type riêng chỉ cho `40001`;
- retry advice chắc chắn nằm ngoài transaction advice;
- backoff/deadline/exhaustion được cấu hình và integration-test;
- attempt worker vẫn là bean riêng.

Không annotate cùng một method rồi assume advisor order. `SPR-006` phân tích
runtime proxy chain chi tiết hơn.

## Giải pháp 2 — `TransactionTemplate`

Khi muốn boundary programmatic:

```java
var template = new TransactionTemplate(transactionManager);
template.setPropagationBehavior(
        TransactionDefinition.PROPAGATION_REQUIRES_NEW
);
template.setIsolationLevel(
        TransactionDefinition.ISOLATION_SERIALIZABLE
);

for (int attempt = 1; ; attempt++) {
    try {
        return template.execute(status -> work.execute(command));
    } catch (RuntimeException failure) {
        if (!PostgreSqlFailures.isSerializationFailure(failure)
                || !policy.canRetry(attempt, deadline)) {
            throw failure;
        }
        backoff.pause(attempt, deadline);
    }
}
```

Mỗi `execute` hoàn tất commit/rollback trước khi catch chạy. Cần giữ work object
không cache entity/result giữa attempts.

## Giải pháp 3 — Guard row hoặc conditional counter

Nếu merchant là hot key và retry rate cao, serialize decision trực tiếp:

```sql
select limit_amount
from merchant_limit
where merchant_id = :merchantId
for update;
```

Sau khi giữ guard row, đọc total và insert. Actor thứ hai block rồi thấy total
mới. Cách này dễ dự đoán conflict nhưng tăng lock wait/connection occupancy; cần
`lock_timeout`, transaction ngắn và deterministic order khi chạm nhiều merchants.

Một conditional counter còn nhỏ hơn:

```sql
update merchant_limit
set reserved_amount = reserved_amount + :amount
where merchant_id = :merchantId
  and reserved_amount + :amount <= limit_amount;
```

Affected-row `1` là accepted; `0` là rejected. Reservation/decision insert nằm
cùng transaction, và counter cần reconciliation discipline.

## Giải pháp 4 — `SERIALIZABLE READ ONLY DEFERRABLE`

Dành cho report dài chỉ đọc:

```sql
begin isolation level serializable read only deferrable;
```

Transaction có thể chờ safe snapshot rồi chạy mà không bị serialization abort.
Không áp dụng cho read-write reservation command.

## Hành vi khi lỗi

| Failure | Transaction outcome | Retry? |
| --- | --- | --- |
| `40001` | Known abort | Có, whole transaction, bounded |
| `40P01` | Deadlock victim abort | Có nếu safe; đồng thời sửa ordering |
| `55P03` | Lock acquisition timeout | Theo deadline/policy riêng |
| `23505` command constraint | Current duplicate attempt rollback | Fresh read/replay same command |
| Business `REJECTED` | Durable decision commit | Không |
| Invalid input | Không bắt đầu attempt | Không |
| Connection loss at commit | Ambiguous | Query/replay by command ID |
| Attempts exhausted | Không active attempt | Trả retry-later/technical outcome |

## So sánh trade-off

| Cách | Correctness boundary | Contention/latency | Retry | Database load | Vận hành | Multi-instance |
| --- | --- | --- | --- | --- | --- | --- |
| SSI + bounded retry | Predicate dependencies | Không block vì SIREAD; abort under overlap | Bắt buộc | Tăng theo attempts | Theo dõi `40001`/predicate locks | Có |
| Guard `FOR UPDATE` | Merchant row | Block tuần tự trên hot merchant | Timeout/deadlock | Wait giữ connection | Lock observability | Có |
| Conditional counter | Atomic merchant row update | Hot-row serialization | Thường không retry business reject | Ít queries, counter writes | Reconciliation | Có |
| Queue/owner | Partition ownership | Queue latency/backpressure | Redelivery/idempotency | Giảm concurrency tại DB | Cao | Có |
| JVM mutex | Process memory | Serialize cục bộ | Không sửa cross-node | Vẫn conflict | Dễ nhưng sai scope | Không |

## Checklist trước production

### Isolation và boundary

- [ ] Effective isolation được assert là `serializable`.
- [ ] Coordinator không có outer transaction.
- [ ] Mỗi attempt qua proxy/template và dùng fresh persistence context.
- [ ] Catch bao trùm cả commit exception.

### Retry policy

- [ ] Chỉ allowlist `40001` và failures được phê duyệt.
- [ ] Attempt cap, jitter/backoff, deadline và cancellation đều có.
- [ ] Mỗi attempt reload limit, total và durable decision.
- [ ] Exhaustion không được báo success.

### Idempotency và side effects

- [ ] Command ID giữ nguyên qua retry/client replay.
- [ ] Unique constraint names được coi là contract.
- [ ] Decision và outbox commit cùng business change.
- [ ] Không remote I/O trong transaction.
- [ ] Ambiguous outcome có query/replay path.

### Vận hành

- [ ] Metric `40001`, attempts, exhaustion, latency và pool pressure tồn tại.
- [ ] Query plan/index/predicate-lock granularity được theo dõi.
- [ ] Hot merchant có admission control hoặc alternative strategy.
- [ ] Testcontainers regression kiểm tra final total và decisions.
