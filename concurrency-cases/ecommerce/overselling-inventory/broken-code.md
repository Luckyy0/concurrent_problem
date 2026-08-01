# Mã nguồn gây bán vượt tồn kho

## 1. Cấu trúc dữ liệu

```sql
CREATE TABLE inventory_item (
    product_id BIGINT PRIMARY KEY,
    on_hand_quantity INTEGER NOT NULL,
    available_quantity INTEGER NOT NULL,
    reserved_quantity INTEGER NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    CONSTRAINT ck_inventory_non_negative CHECK (
        on_hand_quantity >= 0
        AND available_quantity >= 0
        AND reserved_quantity >= 0
    ),
    CONSTRAINT ck_inventory_conservation CHECK (
        available_quantity + reserved_quantity = on_hand_quantity
    )
);

CREATE TABLE inventory_reservation (
    reservation_id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    product_id BIGINT NOT NULL REFERENCES inventory_item(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    status VARCHAR(24) NOT NULL CHECK (status = 'RESERVED'),
    created_at TIMESTAMPTZ NOT NULL
);
```

Dữ liệu ban đầu:

```text
product_id = 77
on_hand_quantity = 5
available_quantity = 5
reserved_quantity = 0
version = 0
```

Hai khách hàng gửi hai yêu cầu độc lập, mỗi yêu cầu mua `4` sản phẩm.

## 2. Thực thể không có `@Version`

```java
@Entity
@Table(name = "inventory_item")
public class InventoryItem {

    @Id
    @Column(name = "product_id")
    private Long productId;

    @Column(name = "on_hand_quantity", nullable = false)
    private int onHandQuantity;

    @Column(name = "available_quantity", nullable = false)
    private int availableQuantity;

    @Column(name = "reserved_quantity", nullable = false)
    private int reservedQuantity;

    @Column(name = "version", nullable = false)
    private long version; // Không có @Version

    protected InventoryItem() {
    }

    public boolean hasEnough(int quantity) {
        return quantity > 0 && availableQuantity >= quantity;
    }

    public void reserve(int quantity) {
        if (!hasEnough(quantity)) {
            throw new IllegalStateException("insufficient stock");
        }
        availableQuantity -= quantity;
        reservedQuantity += quantity;
        version++;
    }

    public int getAvailableQuantity() {
        return availableQuantity;
    }

    public int getReservedQuantity() {
        return reservedQuantity;
    }
}
```

Cột `version` trong ví dụ này chỉ là số đếm thông thường. Vì không có
`@Version`, Hibernate không thêm điều kiện `version = ?` vào câu `UPDATE` và
không phát hiện giao dịch khác đã sửa dòng trước đó.

## 3. Kho dữ liệu và bản ghi giữ hàng

```java
public interface InventoryItemRepository
        extends JpaRepository<InventoryItem, Long> {
}

@Entity
@Table(name = "inventory_reservation")
public class InventoryReservation {

    @Id
    @Column(name = "reservation_id")
    private UUID reservationId;

    @Column(name = "order_id", nullable = false)
    private UUID orderId;

    @Column(name = "product_id", nullable = false)
    private long productId;

    @Column(name = "quantity", nullable = false)
    private int quantity;

    @Column(name = "status", nullable = false)
    private String status;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    protected InventoryReservation() {
    }

    public static InventoryReservation reserved(
            UUID orderId,
            long productId,
            int quantity,
            Instant now
    ) {
        InventoryReservation value = new InventoryReservation();
        value.reservationId = UUID.randomUUID();
        value.orderId = orderId;
        value.productId = productId;
        value.quantity = quantity;
        value.status = "RESERVED";
        value.createdAt = now;
        return value;
    }

    public UUID getReservationId() {
        return reservationId;
    }
}

public interface InventoryReservationRepository
        extends JpaRepository<InventoryReservation, UUID> {
}
```

## 4. Dịch vụ chứa lỗi

```java
@Service
public class BrokenInventoryService {

    private final InventoryItemRepository items;
    private final InventoryReservationRepository reservations;
    private final Clock clock;

    public BrokenInventoryService(
            InventoryItemRepository items,
            InventoryReservationRepository reservations,
            Clock clock
    ) {
        this.items = items;
        this.reservations = reservations;
        this.clock = clock;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReserveResult reserve(ReserveStock command) {
        InventoryItem item = items.findById(command.productId())
                .orElseThrow(ProductNotFoundException::new);

        // Lỗi: quyết định dựa trên dữ liệu đã đọc vào bộ nhớ Java.
        if (!item.hasEnough(command.quantity())) {
            return ReserveResult.outOfStock(command.orderId());
        }

        // Lỗi: Hibernate sẽ ghi lại giá trị tuyệt đối đã tính từ dữ liệu cũ.
        item.reserve(command.quantity());

        reservations.save(InventoryReservation.reserved(
                command.orderId(),
                command.productId(),
                command.quantity(),
                clock.instant()
        ));

        return ReserveResult.reserved(command.orderId());
    }
}

public record ReserveStock(
        UUID orderId,
        long productId,
        int quantity
) {
    public ReserveStock {
        Objects.requireNonNull(orderId);
        if (quantity <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }
    }
}
```

Đây là mã nguồn thường gặp trong dự án thật: có giao dịch, có kiểm tra đầu vào,
có ràng buộc dữ liệu và không cố ý viết sai cú pháp. Lỗi nằm ở khoảng hở giữa
`SELECT` và `UPDATE`.

## 5. SQL được thực thi

Hai giao dịch có thể cùng chạy câu đọc sau:

```sql
SELECT product_id,
       on_hand_quantity,
       available_quantity,
       reserved_quantity,
       version
FROM inventory_item
WHERE product_id = 77;
```

Cả hai cùng nhận `available_quantity = 5`. Khi Hibernate đẩy thay đổi xuống cơ
sở dữ liệu, mỗi giao dịch đều gửi một câu ghi giá trị tuyệt đối tương đương:

```sql
UPDATE inventory_item
SET on_hand_quantity = 5,
    available_quantity = 1,
    reserved_quantity = 4,
    version = 1
WHERE product_id = 77;
```

Câu `UPDATE` thứ hai vẫn phải chờ khóa dòng của câu thứ nhất. Tuy nhiên, sau khi
được chạy, nó không có điều kiện nào cho biết quyết định mua hàng đã được đưa ra
từ phiên bản cũ. Nó chỉ ghi đè cùng giá trị `1/4` và báo đã cập nhật một dòng.

## 6. Điều kiện tái hiện

- PostgreSQL chạy ở `READ COMMITTED`.
- Hai yêu cầu dùng hai kết nối và hai giao dịch riêng.
- Cả hai hoàn tất `findById()` trước khi một bên đẩy câu `UPDATE`.
- Thực thể không có `@Version`.
- Truy vấn không dùng `FOR UPDATE`.
- Điều kiện đủ hàng không nằm trong câu `UPDATE`.

## 7. Dòng thời gian lỗi

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | Đọc số lượng `5` | |
| 2 | | Đọc số lượng `5` |
| 3 | Kiểm tra `5 >= 4`, chấp nhận | Kiểm tra `5 >= 4`, chấp nhận |
| 4 | Tạo bản ghi giữ `4` | Tạo bản ghi giữ `4` |
| 5 | Ghi tồn kho thành `1/4` | Chờ khóa dòng |
| 6 | Chốt giao dịch | Được đánh thức |
| 7 | | Ghi đè tồn kho thành `1/4` và chốt |

Trạng thái cuối:

```text
inventory_item.available_quantity = 1
inventory_item.reserved_quantity = 4

tổng quantity trong inventory_reservation = 8
```

Các ràng buộc trên `inventory_item` vẫn đúng nên PostgreSQL không phát sinh lỗi.

## 8. Những cách sửa chưa đủ

### Chỉ thêm `@Transactional`

Giao dịch bảo vệ việc cùng chốt hoặc cùng hoàn tác, không tự biến chuỗi
`SELECT → kiểm tra trong Java → UPDATE` thành một thao tác nguyên tử.

### Chỉ dùng `synchronized`

```java
public synchronized ReserveResult reserve(ReserveStock command) {
    return transactionalWorker.reserve(command);
}
```

Cách này chỉ tuần tự hóa các luồng đi qua cùng một đối tượng trong một JVM. Máy
chủ ứng dụng khác vẫn có thể cập nhật cùng sản phẩm.

### Chỉ đổi sang phép trừ tương đối

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity
WHERE product_id = :productId;
```

Phép trừ tương đối tránh ghi đè mất một lần trừ, nhưng thiếu điều kiện đủ hàng
nên có thể làm số lượng âm.

### Kiểm tra trước rồi mới chạy câu trừ

```java
if (items.currentAvailable(productId) >= quantity) {
    items.decrement(productId, quantity);
}
```

Khoảng hở giữa hai câu SQL vẫn tồn tại. Hai giao dịch vẫn có thể cùng vượt qua
bước kiểm tra.

### Bỏ qua số dòng bị ảnh hưởng

```java
items.reserveIfEnough(productId, quantity);
reservations.save(reservedRecord);
return ReserveResult.reserved(orderId);
```

Ngay cả khi câu cập nhật đã an toàn, mã nguồn trên vẫn báo thành công khi câu
lệnh trả về `0` dòng.

### Đọc từ bản sao chỉ đọc

Bản sao PostgreSQL có thể chậm hơn máy chủ chính. Số lượng trên bản sao chỉ phù
hợp để hiển thị gần đúng, không phải căn cứ để chấp nhận một lần mua hàng.

## 9. Dấu hiệu trong môi trường thật

- Tổng số lượng giữ hàng lớn hơn `reserved_quantity`.
- Nhiều đơn đều báo thành công trong khi chỉ một đơn có thể được giao.
- Không có lỗi `CHECK` hoặc số lượng âm để cảnh báo sớm.
- Lỗi xuất hiện nhiều hơn khi tăng số máy chủ hoặc lượng truy cập.
- Kiểm thử tuần tự luôn thành công nhưng kiểm thử có rào chắn lại thất bại.

Phần sửa đúng nằm trong [solutions.md](solutions.md); cách chứng minh lỗi bằng
hai giao dịch thật nằm trong [experiments.md](experiments.md).
