# Code lỗi — retry vô hạn hoặc trong transaction cũ

## Entity versioned

```java
@Entity
@Table(name = "reward_wallet")
public class RewardWallet {
    @Id
    private Long walletId;

    @Column(nullable = false)
    private long points;

    @Version
    private long version;

    protected RewardWallet() {
    }

    public void credit(long delta) {
        if (delta <= 0) {
            throw new IllegalArgumentException("delta must be positive");
        }
        points = Math.addExact(points, delta);
    }
}
```

```sql
create table reward_credit (
    command_id uuid primary key,
    wallet_id bigint not null,
    points bigint not null check (points > 0)
);
```

## Lỗi 1 — loop trong cùng `@Transactional`

```java
@Transactional
public CreditResult credit(CreditCommand command) {
    for (;;) {
        try {
            RewardWallet wallet = wallets.findById(command.walletId())
                    .orElseThrow();
            wallet.credit(command.points());
            credits.save(RewardCredit.from(command));
            wallets.flush();
            return CreditResult.applied(command.commandId());
        } catch (ObjectOptimisticLockingFailureException conflict) {
            entityManager.clear();
        }
    }
}
```

Optimistic conflict đánh dấu transaction rollback. `clear()` không rollback hay
tạo snapshot mới. Loop còn vô hạn, không deadline/cancellation và có thể nhận
`UnexpectedRollbackException`/failure tiếp theo.

> **Nói ngắn gọn:** retry loop nằm trong transaction thì transaction sở hữu loop;
> muốn fresh attempts, transaction phải nằm bên trong loop.

## Lỗi 2 — fresh transaction nhưng retry ngay vô hạn

```java
public CreditResult credit(CreditCommand command) {
    while (true) {
        try {
            return attempts.creditOnce(command);
        } catch (ObjectOptimisticLockingFailureException conflict) {
            // no cap, no deadline, no backoff
        }
    }
}
```

Boundary có thể mới nhưng hot actors cùng retry ngay, tạo lockstep conflicts.
Caller cancel không dừng loop; pool/database tiếp tục chịu work.

## Lỗi 3 — `@Retryable` quá rộng

```java
@Retryable(retryFor = RuntimeException.class)
public CreditResult credit(CreditCommand command) {
    return attempts.creditOnce(command);
}
```

Validation, mapping bug, duplicate constraint, cancellation và database outage
đều bị retry. Không có stable classification/exhaustion contract.

## Lỗi 4 — tạo command ID mới mỗi attempt

```java
CreditCommand next = new CreditCommand(
        UUID.randomUUID(),
        original.walletId(),
        original.points()
);
```

Nếu attempt trước commit nhưng response mất, retry có ID mới cộng điểm lần hai.
Command ID là identity nghiệp vụ, không phải attempt ID.

## Lỗi 5 — backoff trong transaction

```java
@Transactional
public void creditOnce(...) {
    // conflict-prone work
    retryBackoff.pause(...); // giữ connection/transaction
}
```

Delay chỉ được chạy sau rollback. Backoff trong transaction tăng pool pressure và
transaction duration.

## Lỗi 6 — reuse entity giữa attempts

```java
RewardWallet stale = wallets.findById(id).orElseThrow();
for (...) {
    transactionTemplate.execute(status -> {
        stale.credit(delta);
        return wallets.save(stale);
    });
}
```

Entity thuộc/đã rời persistence context cũ và mang version/state stale. Fresh
attempt phải load lại bên trong boundary, không truyền managed entity qua loop.

## Lỗi 7 — attempt cap nhưng không overall deadline

```java
for (int attempt = 1; attempt <= 10; attempt++) {
    backoff.pause(attempt);
}
```

Mỗi query/lock/connection acquisition có thể chậm; 10 attempts không chứng minh
operation nằm trong request SLO. Deadline phải bao attempts, waits, backoff và
cleanup.

## Lỗi 8 — retry business rejection

Wallet suspended hoặc command invalid không thay đổi khi reload. Retry chỉ đốt
budget. Allowlist conflict transient, không retry mọi loser/outcome.

## Lỗi 9 — external call trong attempt

```java
wallet.credit(points);
notificationClient.send(command.commandId());
wallets.flush();
```

Conflict/rollback không thu hồi notification. Retry gửi lại; dùng outbox trong
successful transaction và downstream idempotency.

## Lỗi 10 — local serialization

JVM mutex giảm conflict trong một pod nhưng App-2 vẫn update cùng wallet. Nó che
contention ở test đơn node và không thay PostgreSQL version predicate.

## Điều kiện tái hiện

- Một wallet committed, version cố định.
- N actors load cùng version bằng barrier.
- Cho flush đồng thời hoặc winner commit trước losers.
- Đo attempts/transactions/queries, không chỉ final points.
- Mọi Future/latch bounded; PostgreSQL Testcontainers.
- Test method không có outer transaction.

## Các cách sửa chưa đủ

- Chỉ tăng `maxAttempts`.
- Delay cố định không jitter.
- Catch generic `DataAccessException`.
- `entityManager.clear()` trong same transaction.
- `REQUIRES_NEW` nhưng gọi bằng self-invocation.
- Retry giữ entity/result cũ.
- Không replay command đã commit.
- Exhaustion trả success/pending mà không durable workflow.
- Dùng invented throughput numbers thay conflict metrics thật.
