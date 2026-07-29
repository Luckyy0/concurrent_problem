# Broken retry placement

## Versioned aggregate

```java
@Entity
@Table(name = "inventory_item")
public class InventoryItem {
    @Id
    private String sku;

    private int available;

    @Version
    private long version;

    protected InventoryItem() {
    }

    public void reserve(int quantity) {
        if (quantity <= 0 || available < quantity) {
            throw new InsufficientStockException(sku, quantity, available);
        }
        available -= quantity;
    }

    public long getVersion() {
        return version;
    }
}
```

Mỗi distinct command có một durable record. Unique command ID phục vụ duplicate
command detection; nó không thay `@Version` trong việc bảo vệ stock mutation:

```java
@Entity
@Table(
    name = "reservation_record",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_reservation_command",
        columnNames = "command_id"
    )
)
public class ReservationRecord {
    @Id
    private UUID id;

    private UUID commandId;
    private String sku;
    private int quantity;

    public static ReservationRecord accepted(
        UUID commandId,
        String sku,
        int quantity
    ) {
        return new ReservationRecord(
            UUID.randomUUID(),
            commandId,
            sku,
            quantity
        );
    }
}
```

Non-essential getters/constructors được lược bỏ.

## Broken loop trong cùng transaction

```java
@Service
public class BrokenInventoryReservationService {
    private final InventoryItemRepository inventory;
    private final ReservationRecordRepository reservations;
    private final EntityManager entityManager;
    private final RetryBackoff retryBackoff;

    public BrokenInventoryReservationService(
        InventoryItemRepository inventory,
        ReservationRecordRepository reservations,
        EntityManager entityManager,
        RetryBackoff retryBackoff
    ) {
        this.inventory = inventory;
        this.reservations = reservations;
        this.entityManager = entityManager;
        this.retryBackoff = retryBackoff;
    }

    @Transactional
    public ReservationResult reserveWithRetry(
        UUID commandId,
        String sku,
        int quantity
    ) {
        int maxAttempts = 3;

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                InventoryItem item = inventory.findById(sku)
                    .orElseThrow();
                long loadedVersion = item.getVersion();

                item.reserve(quantity);
                reservations.save(ReservationRecord.accepted(
                    commandId,
                    sku,
                    quantity
                ));

                inventory.flush();
                return ReservationResult.accepted(
                    commandId,
                    loadedVersion,
                    attempt
                );
            } catch (ObjectOptimisticLockingFailureException conflict) {
                entityManager.clear();

                if (attempt == maxAttempts) {
                    throw conflict;
                }
                retryBackoff.pause(attempt);
            }
        }

        throw new IllegalStateException("Unreachable");
    }
}
```

Loop có attempt limit nhưng boundary sai. Cả ba iterations nằm trong một physical
transaction do outer proxy tạo. Sau failed flush:

- transaction/persistence provider có failed state hoặc rollback-only;
- managed entities vừa tham gia stale write;
- reservation insert của attempt cũ thuộc cùng doomed transaction;
- `clear()` chỉ detach entities;
- backoff giữ connection/transaction lâu hơn;
- iteration sau không phải independent attempt.

> **Nói ngắn gọn:** `for` tạo execution repetition, không tạo transaction mới.

## Nếu bỏ explicit flush

Variant sau còn không catch được optimistic conflict:

```java
@Transactional
public ReservationResult reserveWithRetry(...) {
    for (...) {
        try {
            InventoryItem item = inventory.findById(sku).orElseThrow();
            item.reserve(quantity);
            return ReservationResult.accepted(...);
        } catch (ObjectOptimisticLockingFailureException conflict) {
            // never reached when conflict is detected during proxy commit
        }
    }
}
```

Hibernate có thể flush sau target method return, trong transaction interceptor
commit phase. Lúc đó call stack đã rời loop; exception chỉ xuất hiện ở caller.

Explicit flush giúp conflict xuất hiện trong attempt body, nhưng không tự tạo clean
retry boundary.

## Self-invoked `REQUIRES_NEW` không tạo transaction mới

```java
@Transactional
public ReservationResult reserveWithRetry(...) {
    for (...) {
        try {
            return this.reserveOnce(...);
        } catch (ObjectOptimisticLockingFailureException conflict) {
            // ...
        }
    }
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public ReservationResult reserveOnce(...) {
    // ...
}
```

`this.reserveOnce()` không đi qua Spring proxy. `REQUIRES_NEW` annotation không
được intercept; attempt vẫn dùng outer transaction.

## Retry interceptor nằm bên trong transaction interceptor

Đặt hai annotations trên cùng method không tự chứng minh ordering:

```java
@Retryable(
    retryFor = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 3
)
@Transactional
public ReservationResult reserve(...) {
    // load, mutate, flush
}
```

Broken advisor chain:

```text
caller
  -> TransactionInterceptor begins Tx
       -> RetryInterceptor catches conflict and invokes target again
            -> attempts share the same Tx/context
       -> outer commit fails or rolls back
```

Correct chain phải là retry outside transaction, nhưng relying on implicit advisor
order làm correctness khó review và dễ đổi theo configuration. Tách coordinator
và attempt worker thành hai beans làm boundary hiển thị bằng object graph.

## Catch quá rộng

```java
catch (RuntimeException failure) {
    retryBackoff.pause(attempt);
}
```

Cách này retry cả `InsufficientStockException`, unique idempotency conflict,
validation bug và programming error. Retry policy phải classify allowlist failure
types; domain rejection không trở thành transient chỉ vì nó là exception.

## Preconditions tái hiện

- Hai physical transactions load cùng `version = 7`.
- Command A commit trước, row thành version 8.
- Command B flush `UPDATE ... WHERE version = 7`.
- Hibernate nhận affected-row count `0`.
- Retry loop/interceptor catch conflict trước khi transaction boundary rollback.
- Attempt tiếp theo reuse cùng thread-bound transaction/persistence context.

## Những cách sửa chưa đủ

- Gọi `EntityManager.clear()` sau conflict.
- Gọi `refresh()` trong rollback-only transaction.
- Thêm nhiều attempts hoặc backoff dài hơn.
- Đặt `REQUIRES_NEW` trên self-invoked method.
- Chỉ move `flush()` vào loop.
- Retry mọi `RuntimeException`.
- Assume unique command key tự bảo vệ stock update.
