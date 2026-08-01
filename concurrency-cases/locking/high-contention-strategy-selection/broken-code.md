# Phân Tích Mã Nguồn Lỗi: Chiến Lược Lock Sụp Đổ Dưới Tải Cao

## 1. Cấu Trúc Bảng (Schema)

```sql
CREATE TABLE inventory_item (
    product_id BIGINT PRIMARY KEY,
    available_quantity INTEGER NOT NULL,
    reserved_quantity INTEGER NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    CHECK (available_quantity >= 0),
    CHECK (reserved_quantity >= 0)
);

CREATE TABLE order_reservation (
    command_id UUID PRIMARY KEY,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    outcome VARCHAR(24) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
);
```

Giả định: Sản phẩm `SKU-2024` có `available_quantity = 500`, `reserved_quantity = 0`. Hệ thống nhận `800` yêu cầu đồng thời, mỗi yêu cầu mua `1` sản phẩm.

## 2. Entity

```java
@Entity
@Table(name = "inventory_item")
public class InventoryItem {
    @Id
    private Long productId;

    private int availableQuantity;
    private int reservedQuantity;

    @Version
    private long version;

    protected InventoryItem() {}

    public boolean canReserve(int quantity) {
        return quantity > 0 && availableQuantity >= quantity;
    }

    public void reserve(int quantity) {
        if (!canReserve(quantity)) {
            throw new IllegalStateException("insufficient stock");
        }
        availableQuantity -= quantity;
        reservedQuantity += quantity;
    }

    // getters omitted
}
```

## 3. Chiến lược 1 — Lock lạc quan (@Version) trên bản ghi nóng

```java
@Service
public class OptimisticReservationService {
    private final InventoryItemRepository inventory;
    private final OrderReservationRepository reservations;
    private final Clock clock;

    // constructor injection omitted

    @Transactional
    public ReservationResult reserve(ReserveStockCommand command) {
        InventoryItem item = inventory.findById(command.productId())
                .orElseThrow(ProductNotFoundException::new);

        if (!item.canReserve(command.quantity())) {
            return ReservationResult.outOfStock();
        }

        item.reserve(command.quantity());
        reservations.save(OrderReservation.reserved(command, clock.instant()));

        return ReservationResult.reserved();
        // Hibernate flush tại đây → UPDATE ... WHERE version = ?
    }
}
```

Khi Hibernate thực thi flush, câu lệnh SQL có dạng:

```sql
UPDATE inventory_item
SET available_quantity = 496,
    reserved_quantity = 4,
    version = 11
WHERE product_id = 2024
  AND version = 10;
```

Vấn đề: Nếu 50 transaction đồng thời đọc cùng `version = 10`, chỉ 1 transaction commit thành công. 49 transaction còn lại nhận `OptimisticLockException` và phải thử lại. Ở lần thử thứ hai, 49 transaction lại tranh chấp `version = 11`, và lại chỉ 1 thành công. Quá trình này lặp lại liên tục.

### Bộ thử lại sai cách (Broken retry)

```java
@Component
public class RetryableReservationFacade {
    private final OptimisticReservationService service;

    // constructor injection omitted

    public ReservationResult reserveWithRetry(ReserveStockCommand command) {
        int maxAttempts = 5;
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                return service.reserve(command);
            } catch (OptimisticLockException e) {
                if (attempt == maxAttempts) {
                    throw new ContentionException("max retries exhausted");
                }
                // LỖI 1: Không có backoff/jitter
                // LỖI 2: Không giới hạn thời gian tổng (deadline)
                // LỖI 3: Không phân biệt lỗi tương tranh với lỗi hệ thống khác
            }
        }
        throw new IllegalStateException("unreachable");
    }
}
```

Phản mẫu:
- Thử lại ngay lập tức (immediate retry) khiến tất cả các transaction thất bại đồng loạt quay lại tranh chấp trong cùng thời điểm.
- 5 lần thử lại trên 800 yêu cầu tạo ra tổng cộng tới `4000` lượt truy vấn database thay vì `800`.
- Không có thời gian trễ ngẫu nhiên (jitter) nên các luồng đồng bộ với nhau, duy trì xung đột.

## 4. Chiến lược 2 — Lock bi quan (FOR UPDATE) trên bản ghi nóng

```java
public interface InventoryItemRepository extends JpaRepository<InventoryItem, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT i FROM InventoryItem i WHERE i.productId = :productId")
    Optional<InventoryItem> findByIdForUpdate(@Param("productId") long productId);
}
```

```java
@Service
public class PessimisticReservationService {
    private final InventoryItemRepository inventory;
    private final OrderReservationRepository reservations;
    private final Clock clock;

    // constructor injection omitted

    @Transactional
    public ReservationResult reserve(ReserveStockCommand command) {
        InventoryItem item = inventory.findByIdForUpdate(command.productId())
                .orElseThrow(ProductNotFoundException::new);

        if (!item.canReserve(command.quantity())) {
            return ReservationResult.outOfStock();
        }

        item.reserve(command.quantity());
        reservations.save(OrderReservation.reserved(command, clock.instant()));

        return ReservationResult.reserved();
    }
}
```

Câu lệnh SQL phát sinh:

```sql
SELECT * FROM inventory_item
WHERE product_id = 2024
FOR UPDATE;
```

Vấn đề: Transaction đầu tiên chiếm lock thành công. 799 transaction còn lại xếp hàng chờ trên cùng bản ghi. Mỗi transaction giữ kết nối database trong suốt thời gian chờ. Khi connection pool chỉ có 20 kết nối và 799 yêu cầu đang chờ, hệ thống rơi vào trạng thái nghẽn.

### Thiếu lock timeout

```java
// Không thiết lập lock_timeout
// Không thiết lập statement_timeout
// → Transaction chờ vô thời hạn cho đến khi PostgreSQL phát hiện hoặc hệ thống can thiệp
```

Phản mẫu:
- Không thiết lập `javax.persistence.lock.timeout` hoặc `SET LOCAL lock_timeout`.
- Connection pool cạn kiệt: 20 kết nối bị giữ chờ lock, các yêu cầu mới (kể cả không liên quan đến sản phẩm nóng) phải chờ kết nối.
- Cascading timeout: API Gateway timeout → Client retry → Tải tăng thêm.

## 5. Chiến lược 3 — Atomic UPDATE nhưng thiếu kiểm soát tải

```java
public interface InventoryItemRepository extends JpaRepository<InventoryItem, Long> {
    @Modifying
    @Query(value = """
        UPDATE inventory_item
        SET available_quantity = available_quantity - :quantity,
            reserved_quantity = reserved_quantity + :quantity,
            version = version + 1
        WHERE product_id = :productId
          AND available_quantity >= :quantity
        """, nativeQuery = true)
    int decrementIfEnough(@Param("productId") long productId,
                          @Param("quantity") int quantity);
}
```

```java
@Service
public class AtomicReservationService {
    private final InventoryItemRepository inventory;
    private final OrderReservationRepository reservations;
    private final Clock clock;

    // constructor injection omitted

    @Transactional
    public ReservationResult reserve(ReserveStockCommand command) {
        int changed = inventory.decrementIfEnough(
                command.productId(), command.quantity());

        if (changed == 0) {
            reservations.save(
                    OrderReservation.outOfStock(command, clock.instant()));
            return ReservationResult.outOfStock();
        }

        reservations.save(
                OrderReservation.reserved(command, clock.instant()));
        return ReservationResult.reserved();
    }
}
```

Chiến lược này là chính xác nhất trong ba phương pháp trên: phía thua nhận `0` dòng thay vì ngoại lệ, không cần thử lại. Tuy nhiên, tại mức tương tranh cực cao, vẫn tồn tại vấn đề:

- `UPDATE` vẫn cấp lock bản ghi; các transaction khác phải chờ cho đến khi lock được giải phóng.
- Lock giữ cho đến khi toàn bộ transaction commit, bao gồm cả lệnh `INSERT` vào `order_reservation` và `outbox_event`.
- Nếu transaction chứa thêm logic xử lý phức tạp sau `UPDATE`, thời gian giữ lock kéo dài.

## 6. Phản mẫu tổng hợp (Common anti-patterns)

### Phản mẫu 1 — Dùng synchronized thay cho chiến lược database

```java
private final Map<Long, Object> productLocks = new ConcurrentHashMap<>();

public ReservationResult reserve(ReserveStockCommand command) {
    synchronized (productLocks.computeIfAbsent(command.productId(), k -> new Object())) {
        return transactionalService.reserve(command);
    }
}
```

`synchronized` chỉ bảo vệ một JVM. Trong kiến trúc phân tán, nhiều instance vẫn thực thi song song, gây ra đúng lỗi ban đầu.

### Phản mẫu 2 — Nâng cấp mức cô lập thay vì chọn chiến lược phù hợp

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public ReservationResult reserve(ReserveStockCommand command) {
    // ...
}
```

Mức cô lập `SERIALIZABLE` giúp PostgreSQL phát hiện xung đột tuần tự, nhưng trên bản ghi nóng, tỷ lệ abort (`40001`) cũng tăng tương tự như lock lạc quan. Hệ thống vẫn cần thử lại, và tải thử lại vẫn bị nhân bản.

### Phản mẫu 3 — Thử lại bên trong transaction bị hỏng

```java
@Transactional
public ReservationResult reserve(ReserveStockCommand command) {
    for (int i = 0; i < 3; i++) {
        try {
            return doReserve(command);
        } catch (OptimisticLockException e) {
            entityManager.clear();
            // LỖI: Transaction đã bị đánh dấu rollback-only
            // Lệnh thử lại trong cùng transaction sẽ thất bại với lỗi 25P02
        }
    }
    return ReservationResult.error();
}
```

Mỗi lần thử lại phải mở một transaction vật lý mới. Vòng lặp retry phải nằm BÊN NGOÀI ranh giới `@Transactional`.

### Phản mẫu 4 — Tăng connection pool để giải quyết nghẽn

Tăng `maximumPoolSize` từ `20` lên `200` không giải quyết vấn đề gốc. Nó chỉ cho phép thêm 180 kết nối xếp hàng chờ lock trên cùng bản ghi. PostgreSQL phải quản lý thêm backend process, tiêu thụ RAM và CPU. Giải pháp đúng là giảm thời gian giữ lock và kiểm soát tải đầu vào.

### Phản mẫu 5 — Bỏ qua kết quả 0 dòng

```java
inventory.decrementIfEnough(productId, quantity);
// Tiếp tục xử lý mà không kiểm tra affected rows
return ReservationResult.reserved();
```

Phản mẫu này đã được phân tích trong LOCK-004. Kết quả `0` dòng không phải là thành công.

## 7. Dấu hiệu nhận biết (Observability symptoms)

- Tỷ lệ `OptimisticLockException` tăng đột biến khi có sự kiện flash-sale.
- Thời gian phản hồi trung bình (p95/p99) tăng từ 50ms lên hàng giây.
- Số kết nối active trong connection pool chạm giới hạn tối đa.
- Lỗi `HikariPool: Connection is not available` xuất hiện trong log.
- Tỷ lệ lỗi `55P03` (lock timeout) tăng mạnh.
- Tải database (CPU, IOPS) tăng gấp bội do retry nhưng thông lượng thành công không tăng tương ứng.
- Các API không liên quan đến flash-sale cũng bị chậm do thiếu kết nối.

## 8. Tổng kết

Ba chiến lược cơ bản (lock lạc quan, lock bi quan, atomic UPDATE) đều bảo đảm tính đúng đắn. Vấn đề nằm ở hành vi phi chức năng khi mức tương tranh vượt ngưỡng thiết kế. Bài phân tích tiếp theo sẽ trình bày chi tiết timeline xung đột, so sánh định tính, và đề xuất khung lựa chọn chiến lược dựa trên đặc điểm thực tế.
