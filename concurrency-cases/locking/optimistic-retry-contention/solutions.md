# Giải pháp bounded retry với fresh attempt

## One-attempt worker

```java
@Service
public class RewardCreditAttempt {

    private final RewardWalletRepository wallets;
    private final RewardCreditRepository credits;
    private final OutboxRepository outbox;

    public RewardCreditAttempt(
            RewardWalletRepository wallets,
            RewardCreditRepository credits,
            OutboxRepository outbox
    ) {
        this.wallets = wallets;
        this.credits = credits;
        this.outbox = outbox;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public CreditResult creditOnce(CreditCommand command) {
        var existing = credits.findById(command.commandId());
        if (existing.isPresent()) {
            return CreditResult.replayed(existing.orElseThrow());
        }

        RewardWallet wallet = wallets.findById(command.walletId())
                .orElseThrow();
        wallet.requireActive();
        wallet.credit(command.points());

        credits.save(RewardCredit.from(command));
        outbox.save(RewardCreditedEvent.from(command));
        wallets.flush();
        credits.flush();
        outbox.flush();
        return CreditResult.applied(command.commandId(), wallet.points());
    }
}
```

`REQUIRES_NEW` phù hợp vì coordinator là root không có outer transaction.
Independent commit/extra connection cost phải được chấp nhận; guard architecture
ngăn transactional caller bọc coordinator.

## Retry policy

```java
public record RetryBudget(
        int maxAttempts,
        Duration initialBackoff,
        Duration maxBackoff
) {
    public RetryBudget {
        if (maxAttempts < 1 || initialBackoff.isNegative()
                || maxBackoff.compareTo(initialBackoff) < 0) {
            throw new IllegalArgumentException("invalid retry budget");
        }
    }
}

@Component
public class OptimisticRetryPolicy {

    private final Clock clock;
    private final RetryBudget budget;

    public boolean canRetry(int completedAttempts, Instant deadline) {
        return completedAttempts < budget.maxAttempts()
                && clock.instant().isBefore(deadline);
    }
}
```

> **Nói ngắn gọn:** attempts giới hạn database work; deadline giới hạn user-visible
> time. Không cái nào thay thế cái kia.

## Backoff có jitter và testable

Tách calculation khỏi waiting:

```java
public interface JitterSource {
    long nextLong(long exclusiveUpperBound);
}

public final class BackoffPlan {

    private final RetryBudget budget;
    private final JitterSource jitter;

    public Duration delayAfter(int completedAttempt) {
        long factor = 1L << Math.min(completedAttempt - 1, 10);
        long base = Math.min(
                budget.maxBackoff().toMillis(),
                Math.multiplyExact(
                        budget.initialBackoff().toMillis(),
                        factor
                )
        );
        long extra = jitter.nextLong(Math.max(1, base / 2 + 1));
        return Duration.ofMillis(
                Math.min(budget.maxBackoff().toMillis(), base + extra)
        );
    }
}
```

Waiter abstraction tôn trọng deadline/interrupt:

```java
public interface RetryWaiter {
    void waitFor(Duration delay, Instant deadline);
}
```

Production có thể dùng scheduler hoặc interruptible parking; test dùng recording
waiter, không fixed sleep.

## Non-transactional coordinator

```java
@Service
public class RewardCreditCoordinator {

    private final RewardCreditAttempt attempts;
    private final OptimisticRetryPolicy policy;
    private final BackoffPlan backoff;
    private final RetryWaiter waiter;
    private final Clock clock;

    public CreditResult credit(CreditCommand command, Instant deadline) {
        if (TransactionSynchronizationManager
                .isActualTransactionActive()) {
            throw new IllegalStateException(
                    "retry coordinator cannot join outer transaction"
            );
        }

        for (int attempt = 1; ; attempt++) {
            try {
                return attempts.creditOnce(command);
            } catch (ObjectOptimisticLockingFailureException conflict) {
                if (!policy.canRetry(attempt, deadline)) {
                    throw new WalletContentionException(
                            command.commandId(),
                            attempt,
                            conflict
                    );
                }
                Duration remaining = Duration.between(
                        clock.instant(),
                        deadline
                );
                Duration delay = min(backoff.delayAfter(attempt), remaining);
                waiter.waitFor(delay, deadline);
            }
        }
    }
}
```

Catch chạy sau attempt proxy rollback. `min` chọn duration nhỏ hơn. Production
code inject dependencies/validated config bằng constructor.

## Duplicate command race

Hai requests cùng command ID có thể cùng thấy record absent. Unique primary key
chọn winner; loser rollback. Coordinator chỉ replay `23505` nếu constraint chính
xác là `reward_credit_pkey`, bằng fresh read transaction. Unique failure khác
propagate.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
public CreditResult requireCommitted(UUID commandId) {
    return credits.findById(commandId)
            .map(CreditResult::replayed)
            .orElseThrow();
}
```

Không phân loại duplicate-command race là optimistic retry.

## Spring Retry alternative

`@Retryable` dùng được khi retry advisor nằm ngoài transactional worker bean,
exception type allowlisted, max attempts/backoff/exhaustion rõ. Overall deadline
và transaction identity vẫn cần integration test. Separate beans dễ audit hơn
đặt `@Retryable` + `@Transactional` cùng method.

## TransactionTemplate alternative

```java
var tx = new TransactionTemplate(transactionManager);
tx.setPropagationBehavior(
        TransactionDefinition.PROPAGATION_REQUIRES_NEW
);

for (int attempt = 1; ; attempt++) {
    try {
        return tx.execute(status -> work.credit(command));
    } catch (ObjectOptimisticLockingFailureException conflict) {
        // policy/deadline/backoff ngoài completed transaction
    }
}
```

Không cache entity ngoài `execute`.

## Exhaustion contract

Exhaustion là explicit technical contention outcome:

- synchronous API: `409/503` tùy contract, `Retry-After` chỉ khi có ý nghĩa;
- async durable command: giữ pending/requeue bằng workflow riêng;
- không trả success nếu không có `reward_credit`;
- caller query status bằng command ID khi outcome ambiguous.

## Alternatives khi contention tăng

### Atomic delta

```sql
update reward_wallet
set points = points + :delta
where wallet_id = :id;
```

Commutative delta có thể tránh read/version retry; idempotency record vẫn phải
cùng transaction. Đây thường là lựa chọn nhỏ hơn cho simple counter.

### Pessimistic lock

Serialize trước calculation, giảm wasted retries nhưng tăng wait/connection
occupancy và deadlock/timeout policy.

### Per-key queue/ownership

Giảm concurrent writers nhưng thêm ordering, delivery, crash và operational
complexity. Xem selection framework `LOCK-005`.

## Failure matrix

| Outcome | Retry? | State |
| --- | --- | --- |
| Optimistic conflict | Có nếu budget/deadline | Old attempt rollback |
| Duplicate command | Fresh replay | Một credit |
| Wallet suspended/invalid | Không | Domain rejection |
| Deadline/attempt exhausted | Không | Explicit contention failure |
| Interrupt/cancel | Không | Propagate cancellation |
| Commit response loss | Query same command ID | Có thể đã commit |
| External publish | Outbox only | Sau commit |

## Trade-off

| Strategy | Correctness | Load under contention | Latency | Complexity |
| --- | --- | --- | --- | --- |
| Bounded optimistic retry | Mạnh + idempotency | Attempts amplify | Backoff/tail | Trung bình |
| Atomic delta | Mạnh cho simple delta | Ít round trips | Thấp | Thấp |
| Pessimistic | Mạnh | Blocking wait | Predictable/timeout | Trung bình |
| Queue | Phụ thuộc delivery protocol | Serialized per key | Queueing | Cao |

## Checklist

- [ ] Coordinator không transaction; attempt qua proxy.
- [ ] Previous rollback hoàn tất trước backoff.
- [ ] Fresh attempt reload entity/command state.
- [ ] Attempt cap và overall deadline.
- [ ] Exponential backoff có jitter, interrupt/cancel aware.
- [ ] Command ID stable và unique record cùng transaction.
- [ ] Chỉ allowlist optimistic conflict.
- [ ] Exhaustion không báo success.
- [ ] Attempts/success, conflicts, backoff, exhaustion và pool pressure có metric.
- [ ] Hot-key bền vững trigger strategy review, không chỉ tăng cap.
