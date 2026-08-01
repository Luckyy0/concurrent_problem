# Vị trí đặt retry bị lỗi

## Aggregate có đánh version

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

Mỗi command riêng biệt có một bản ghi bền vững (durable record). ID duy nhất của command phục vụ cho việc phát hiện các command trùng lặp; nó không thay thế cho `@Version` trong việc bảo vệ cập nhật stock:

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

Các getter/constructor không thiết yếu đã được lược bỏ.

## Vòng lặp lỗi bên trong cùng một transaction

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

Vòng lặp có giới hạn số lần thử nhưng ranh giới (boundary) lại sai. Cả ba lần lặp đều nằm trong cùng một physical transaction do proxy bên ngoài tạo ra. Sau khi flush thất bại:

- transaction/persistence provider có trạng thái lỗi (failed state) hoặc rollback-only;
- các thực thể đang được quản lý (managed entities) vừa tham gia vào một lần ghi lỗi thời (stale write);
- thao tác insert reservation của lần thử cũ thuộc cùng một transaction đã hỏng (doomed transaction);
- lệnh `clear()` chỉ ngắt kết nối (detach) các thực thể;
- backoff giữ lại connection/transaction lâu hơn;
- lần lặp tiếp theo không phải là một lần thử độc lập.

> **Nói ngắn gọn:** vòng lặp `for` tạo ra sự lặp lại về mặt thực thi mã nguồn, không tạo ra transaction mới.

## Nếu bỏ lệnh flush tường minh

Biến thể sau đây thậm chí còn không bắt (catch) được lỗi optimistic conflict:

```java
@Transactional
public ReservationResult reserveWithRetry(...) {
    for (...) {
        try {
            InventoryItem item = inventory.findById(sku).orElseThrow();
            item.reserve(quantity);
            return ReservationResult.accepted(...);
        } catch (ObjectOptimisticLockingFailureException conflict) {
            // không bao giờ chạy đến đây khi xung đột được phát hiện trong quá trình commit của proxy
        }
    }
}
```

Hibernate có thể flush sau khi phương thức đích đã trả về, cụ thể là trong giai đoạn commit của transaction interceptor. Lúc đó call stack đã rời khỏi vòng lặp; ngoại lệ chỉ xuất hiện ở phía gọi (caller).

Lệnh flush tường minh giúp xung đột xuất hiện ngay trong phần thân của lần thử, nhưng không tự tạo ra ranh giới retry sạch.

## Lệnh gọi nội bộ (Self-invoked) `REQUIRES_NEW` không tạo transaction mới

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

Lời gọi `this.reserveOnce()` không đi qua Spring proxy. Annotation `REQUIRES_NEW` không bị chặn (intercept) bởi Spring proxy; lần thử vẫn sử dụng transaction bên ngoài.

## Retry interceptor nằm bên trong transaction interceptor

Việc đặt cả hai annotation trên cùng một phương thức không tự chứng minh được thứ tự thực thi (ordering):

```java
@Retryable(
    retryFor = ObjectOptimisticLockingFailureException.class,
    maxAttempts = 3
)
@Transactional
public ReservationResult reserve(...) {
    // tải, sửa đổi, flush
}
```

Chuỗi advisor bị lỗi:

```text
caller
  -> TransactionInterceptor bắt đầu Tx
       -> RetryInterceptor bắt lỗi xung đột và gọi lại đích
            -> các lần thử dùng chung Tx/context
       -> commit bên ngoài thất bại hoặc rollback
```

Chuỗi đúng phải là quá trình retry ở bên ngoài transaction, nhưng việc phụ thuộc vào thứ tự ngầm định của các advisor làm cho tính đúng đắn khó bị đánh giá (review) và dễ thay đổi theo cấu hình. Tách biệt bộ điều phối (coordinator) và bộ xử lý (worker) thành hai bean riêng biệt làm cho ranh giới hiển thị rõ ràng thông qua cấu trúc đối tượng (object graph).

## Catch quá rộng

```java
catch (RuntimeException failure) {
    retryBackoff.pause(attempt);
}
```

Cách này retry cả ngoại lệ `InsufficientStockException`, các xung đột tính lũy đẳng (unique idempotency), lỗi xác thực (validation bug) và lỗi lập trình. Chính sách retry phải phân loại các dạng lỗi được phép (allowlist failure types); việc từ chối theo logic nghiệp vụ (domain rejection) không trở thành lỗi tạm thời (transient) chỉ vì nó là một ngoại lệ.

## Điều kiện tiên quyết để tái hiện (Preconditions)

- Hai physical transaction cùng đọc `version = 7`.
- Command A commit trước, biến dòng dữ liệu thành version 8.
- Command B flush `UPDATE ... WHERE version = 7`.
- Hibernate nhận kết quả số dòng bị ảnh hưởng (affected-row count) là `0`.
- Vòng lặp retry/interceptor bắt (catch) xung đột trước khi ranh giới transaction tiến hành rollback.
- Lần thử tiếp theo tái sử dụng lại transaction/persistence context đã liên kết với luồng (thread-bound) đó.

## Những cách sửa chưa đủ

- Gọi `EntityManager.clear()` sau xung đột.
- Gọi `refresh()` trong transaction đã ở trạng thái rollback-only.
- Thêm số lượng lần thử hoặc thời gian chờ dài hơn.
- Đặt `REQUIRES_NEW` trên phương thức được gọi nội bộ (self-invoked method).
- Chỉ di chuyển lệnh `flush()` vào trong vòng lặp.
- Retry đối với mọi `RuntimeException`.
- Giả định rằng khóa duy nhất (unique command key) có thể tự bảo vệ việc cập nhật stock.
