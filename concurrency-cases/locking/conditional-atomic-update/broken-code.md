# Phân Tích Mã Nguồn Lỗi: Rủi Ro Trong Luồng Đọc-Kiểm Tra-Ghi (Read-Check-Write)

## 1. Cấu Trúc Bảng (Schema) Ban Đầu

```sql
CREATE TABLE inventory_item (
    product_id BIGINT PRIMARY KEY,
    on_hand_quantity INTEGER NOT NULL,
    available_quantity INTEGER NOT NULL,
    reserved_quantity INTEGER NOT NULL,
    revision BIGINT NOT NULL DEFAULT 0,
    CHECK (on_hand_quantity >= 0),
    CHECK (available_quantity >= 0),
    CHECK (reserved_quantity >= 0),
    CHECK (available_quantity + reserved_quantity = on_hand_quantity)
);

CREATE TABLE inventory_reservation (
    reservation_id UUID PRIMARY KEY,
    command_id UUID NOT NULL UNIQUE,
    order_id UUID NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    outcome VARCHAR(24) NOT NULL,
    request_fingerprint VARCHAR(64) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
);
```

Giả định: Sản phẩm `77` hiện tại có Tổng `on_hand_quantity=5`, Số lượng sẵn có `available_quantity=5`, Đã giữ `reserved_quantity=0`.

Lưu ý: Các ràng buộc (`CHECK`) cấp độ dòng (Row-level) không ngăn chặn được lỗi mất cập nhật (Lost update). Nếu hai transaction đồng thời trừ số lượng trên giá trị cũ, bản ghi kho hàng có thể kết thúc ở trạng thái hợp lệ (`1 + 4 = 5` và `>= 0`), nhưng bảng lịch sử (`inventory_reservation`) lại ghi nhận tổng số lượng giữ chỗ là `8`. Dữ liệu sẽ mất đồng bộ.

## 2. Lỗi Thiếu Cơ Chế Lock Lạc Quan (`@Version`)

```java
@Entity
@Table(name = "inventory_item")
public class InventoryItem {
    @Id
    private Long productId;

    private int onHandQuantity;
    private int availableQuantity;
    private int reservedQuantity;
    private long revision; // <-- Thiếu annotation @Version

    protected InventoryItem() {}

    boolean canReserve(int quantity) {
        return quantity > 0 && availableQuantity >= quantity;
    }

    void reserve(int quantity) {
        if (!canReserve(quantity)) {
            throw new IllegalStateException("insufficient stock");
        }
        availableQuantity -= quantity;
        reservedQuantity += quantity;
        revision++;
    }
}
```

Trường `revision` trong thực thể này chỉ là một biến đếm số nguyên cơ bản. Do thiếu annotation `@Version`, Hibernate sẽ không tự động áp dụng cơ chế lock lạc quan (Optimistic locking) khi sinh mã `UPDATE`. Điều này khiến ứng dụng mất đi lớp bảo vệ trước lỗi dữ liệu đồng thời.

## 3. Thiết Kế Service Chứa Lỗi Logic (Broken Service)

```java
@Service
public class BrokenInventoryReservationService {
    private final InventoryItemRepository inventory;
    private final InventoryReservationRepository reservations;
    private final OutboxRepository outbox;
    private final Clock clock;

    public BrokenInventoryReservationService(
            InventoryItemRepository inventory,
            InventoryReservationRepository reservations,
            OutboxRepository outbox,
            Clock clock
    ) {
        this.inventory = inventory;
        this.reservations = reservations;
        this.outbox = outbox;
        this.clock = clock;
    }

    @Transactional
    public ReservationResult reserve(ReserveStockCommand command) {
        InventoryItem item = inventory.findById(command.productId())
                .orElseThrow(ProductNotFoundException::new);

        // LỖI: Kiểm tra điều kiện nghiệp vụ dựa trên trạng thái cũ (Stale state) trong RAM
        if (!item.canReserve(command.quantity())) {
            return ReservationResult.outOfStock();
        }

        // LỖI: Cập nhật trực tiếp số lượng tuyệt đối trên RAM
        item.reserve(command.quantity());
        
        InventoryReservation reservation =
                InventoryReservation.reserved(command, clock.instant());
        reservations.save(reservation);
        outbox.save(OutboxEvent.inventoryReserved(reservation));

        return ReservationResult.reserved(reservation.getReservationId());
    }
}
```

Service trên sử dụng `@Transactional` để đảm bảo lưu dữ liệu trọn vẹn. Tuy nhiên, logic kiểm tra `canReserve` lại đánh giá trên bản ghi đã tải lên RAM từ phương thức `findById()`. Nó không truyền điều kiện ràng buộc xuống câu lệnh `UPDATE`.

Hibernate sẽ phát sinh câu lệnh SQL ghi đè số lượng tĩnh (Absolute update):

```sql
UPDATE inventory_item
SET available_quantity = 1,
    reserved_quantity = 4,
    revision = 1
WHERE product_id = 77;
```

Trong tình huống đồng thời, cả hai tiến trình đều áp dụng câu lệnh ghi đè y hệt nhau thay vì giảm trừ tương đối (Relative decrement), dẫn đến việc PostgreSQL ghi nhận thao tác hợp lệ (Affected rows = 1) cho cả hai, nhưng số liệu thực tế bị sai lệch nghiêm trọng.

## 4. Điều Kiện Phát Sinh Sự Cố

- Database hoạt động ở mức cô lập `READ COMMITTED` (Mặc định).
- Sản phẩm `77` hiện có `5` sản phẩm.
- Hai transaction đồng thời yêu cầu mua `4` sản phẩm, sử dụng Command ID khác biệt.
- Cả hai transaction cùng thực hiện thao tác `SELECT` và hoàn tất đánh giá điều kiện trên RAM trước khi bất kỳ lệnh cập nhật (Flush) nào được gửi xuống database.
- Hệ thống không sử dụng Lock lạc quan (`@Version`), Lock bi quan (`FOR UPDATE`), hoặc Cập nhật có điều kiện.

## 5. Dòng Thời Gian Xảy Ra Lỗi (Timeline)

| Bước | Transaction A | Transaction B |
| --- | --- | --- |
| 1 | `SELECT` → Nhận giá trị `5` | |
| 2 | | `SELECT` → Nhận giá trị `5` |
| 3 | Đánh giá nghiệp vụ: `5 >= 4`, CHẤP NHẬN | Đánh giá nghiệp vụ: `5 >= 4`, CHẤP NHẬN |
| 4 | Lưu lịch sử A / Hộp thư A | Lưu lịch sử B / Hộp thư B |
| 5 | Gửi lệnh `UPDATE` ghi đè giá trị tĩnh `1/4` | |
| 6 | Xác nhận (Commit) | |
| 7 | | Gửi lệnh `UPDATE` ghi đè giá trị tĩnh `1/4` và Xác nhận (Commit) |

Hậu quả: 
```text
Bảng kho hàng (`inventory_item`): available=1, reserved=4
Bảng lịch sử (`inventory_reservation`): Đơn A giữ 4, Đơn B giữ 4, Tổng cộng = 8 sản phẩm
```
Không có ràng buộc (Constraint) nào bị vi phạm, nhưng tiến trình đối soát (Reconciliation) sẽ ghi nhận sự chênh lệch dữ liệu bất thường. Ràng buộc cấp độ dòng không thể nhận thức được số lượng đơn đặt hàng đã được ghi trong bảng khác.

## 6. Các Phản Mẫu Kinh Điển (Anti-Patterns) Cần Tránh

### Phản Mẫu 1 — Phụ thuộc quá mức vào `@Transactional`

```java
@Transactional
public ReservationResult reserve(ReserveStockCommand command) {
    InventoryItem item = inventory.findById(command.productId()).orElseThrow();
    if (item.getAvailableQuantity() < command.quantity()) {
        return ReservationResult.outOfStock();
    }
    item.reserve(command.quantity());
    return ReservationResult.reserved();
}
```
Annotation `@Transactional` chỉ tạo ranh giới cho Commit/Rollback. Nó không tự động gom nhóm `SELECT` và `UPDATE` thành một khối nguyên tử (Atomic block) nếu thiếu cơ chế lock (Locking mechanism). Các transaction khác vẫn có quyền can thiệp vào giữa quá trình.

### Phản Mẫu 2 — Bỏ qua kết quả của câu lệnh UPDATE (Ignoring affected rows)

```java
@Transactional
public ReservationResult reserve(ReserveStockCommand command) {
    // Thực thi atomic update nhưng phớt lờ số lượng dòng trả về
    inventory.decrementIfEnough(command.productId(), command.quantity());
    reservations.save(InventoryReservation.reserved(command));
    return ReservationResult.reserved();
}
```
Khi thao tác bị từ chối (trả về `0` dòng), ứng dụng không kiểm tra kết quả mà vẫn tiếp tục tạo lịch sử thành công. Hệ thống cần sử dụng kết quả dòng để phân nhánh luồng nghiệp vụ.

### Phản Mẫu 3 — Cập nhật tương đối thiếu điều kiện (Missing predicate guard)

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE product_id = :productId; -- THIẾU KIỂM TRA NGHIỆP VỤ
```
Giảm trừ tương đối (Relative update) giải quyết được lỗi ghi đè tĩnh, nhưng lại cho phép giá trị kho hàng giảm xuống số âm nếu thiếu điều kiện `AND available_quantity >= :quantity`.

### Phản Mẫu 4 — Đánh giá nghiệp vụ tách rời Cập nhật dữ liệu

```java
if (inventory.currentAvailable(productId) >= quantity) {
    // Điều kiện đánh giá trên RAM, rủi ro sai sót cao
    inventory.decrementUnconditionally(productId, quantity); 
}
```
Điều kiện kiểm tra (Predicate) và tác vụ thay đổi (Mutation) phải được tích hợp trong cùng một câu lệnh SQL duy nhất tại database.

### Phản Mẫu 5 — Gây lỗi bộ đệm JPA do Bulk DML (Stale persistence context)

```java
@Transactional
public void reserve(long productId, int quantity) {
    // Entity được tải vào bộ đệm của JPA
    InventoryItem stale = inventory.findById(productId).orElseThrow();

    // Thực thi cập nhật SQL trực tiếp (Bỏ qua JPA)
    int changed = inventory.decrementIfEnough(productId, quantity);
    if (changed == 1) {
        // Thay đổi thuộc tính trên entity cũ
        stale.renameForAudit("reserved"); 
    }
    // Lỗi: JPA sẽ tự động flush entity (chứa dữ liệu cũ) xuống database, phá vỡ hiệu ứng cập nhật của lệnh Bulk DML.
}
```
Tránh sử dụng Entity cùng với Bulk DML trong cùng một transaction nếu không quản lý vòng đời bộ đệm một cách rõ ràng thông qua `EntityManager.clear()` và `refresh()`.

### Phản Mẫu 6 — Gọi `save()` dư thừa sau khi thực thi SQL nguyên tử

```java
int changed = inventory.decrementIfEnough(productId, quantity);
InventoryItem detachedSnapshot = mapper.fromEarlierRequest(request);
inventory.save(detachedSnapshot); // LỖI
```
Gọi phương thức `save()` hoặc `merge()` sẽ ép Spring Data ghi lại toàn bộ thuộc tính đối tượng xuống database. Điều này phá hỏng cấu trúc nguyên tử mà lệnh `UPDATE` trước đó vừa thiết lập.

### Phản Mẫu 7 — Đánh đồng 0 dòng với lỗi Hết Hàng (`OUT_OF_STOCK`)

Giá trị trả về `0` có thể do nhiều nguyên nhân:
- Sản phẩm không tồn tại (ID không hợp lệ).
- Số lượng mua không hợp lệ.
- Trạng thái vô hiệu hóa.
- Thiếu tồn kho (Hết hàng).

Ứng dụng cần có hệ thống kiểm duyệt đầu vào (Input validation) và tách biệt logic tìm kiếm bản ghi để định hướng mã lỗi chính xác.

### Phản Mẫu 8 — Xử lý lỗi hệ thống như lỗi nghiệp vụ (Masking system exceptions)

```java
try {
    return inventory.decrementIfEnough(productId, quantity) == 1;
} catch (CannotAcquireLockException ignored) {
    return false; // Phản mẫu: Biến lỗi quá tải cấp lock (55P03) thành lỗi Hết hàng
}
```
Lỗi hết thời gian chờ lock (Lock timeout) là dấu hiệu hệ thống quá tải (Contention). Cần lan truyền ngoại lệ này để kích hoạt luồng Retry hoặc thông báo lỗi hạ tầng, tuyệt đối không ẩn giấu dưới danh nghĩa lỗi nghiệp vụ.

### Phản Mẫu 9 — Vi phạm tính Lũy Đẳng (Broken Idempotency)

Khi hệ thống mạng bị trễ, người dùng có thể gửi lại yêu cầu bằng việc sinh một mã định danh (Command ID) mới. Do lệnh khác nhau, database sẽ trừ tồn kho nhiều lần. Cần duy trì tính duy nhất của mã lệnh và triển khai bảng chống trùng lặp (Durable claim) để tái tạo trạng thái phản hồi mà không lặp lại tác vụ nghiệp vụ.

### Phản Mẫu 10 — Lan truyền hiệu ứng phụ (Side effect) trước khi Commit

```java
int changed = inventory.decrementIfEnough(productId, quantity);
if (changed == 1) {
    kafkaTemplate.send("inventory-reserved", event); // Lỗi kiến trúc: Giao tiếp mạng không hoàn tác được
    reservations.save(reservation);
}
```
Nếu transaction phát sinh lỗi và bị hủy bỏ (Rollback), hệ thống ngoại vi vẫn nhận được thông báo không chính xác. Phải áp dụng mô hình Transactional Outbox.

### Phản Mẫu 11 — Sử dụng lock phân giải cấp JVM trong hệ thống phân tán

```java
synchronized (localProductMutex(productId)) {
    return transactionalWorker.reserve(command);
}
```
Từ khóa `synchronized` chỉ có hiệu lực trên một máy chủ đơn lẻ (Single instance). Đối với hệ thống có nhiều máy chủ, bắt buộc phải sử dụng cơ chế xử lý đồng thời tại database.

### Phản Mẫu 12 — Kiểm tra nghiệp vụ trên Node bản sao (Read Replica)

Dữ liệu trên Node bản sao (Replica) thường có độ trễ đồng bộ so với Node chính (Primary). Phụ thuộc vào dữ liệu này để thực hiện kiểm tra nghiệp vụ có thể dẫn đến việc vượt rào số lượng tồn kho. Replica chỉ nên phục vụ các thao tác hiển thị, không dùng cho xử lý logic ghi.

## 7. Dấu Hiệu Nhận Biết Lỗi (Observability Symptoms)

- Kết quả báo cáo (SUM) trên bảng lịch sử không khớp với cột lưu giá trị tổng trong bảng tồn kho.
- Hệ thống trừ không thành công (0 dòng) nhưng API vẫn ghi nhận lịch sử và báo `RESERVED`.
- Khung JPA/Hibernate duy trì và lưu các dữ liệu không tương thích.
- Bị trùng lặp xử lý do mã Command bị thay đổi khi Client gọi thử lại.
- Báo cáo nhầm mã lỗi kỹ thuật hệ thống (`55P03`) thành lỗi hết hàng nghiệp vụ.
- Tình trạng lỗi xuất hiện ngẫu nhiên trên môi trường phân tán nhưng không xảy ra trên môi trường cục bộ.

## 8. Chuỗi Logic Xử Lý Khuyến Nghị (Correct Workflow)

Tuyệt đối từ bỏ luồng làm việc tuyến tính rủi ro:
`Tải cấu trúc -> Kiểm tra nghiệp vụ bằng bộ đệm -> Tính toán số đếm -> Cập nhật ghi đè`.

Hãy chuyển đổi thành: **Đóng gói điều kiện kiểm tra (WHERE) và ý định thay đổi (SET) thành một transaction SQL nguyên tử duy nhất. Xử lý logic nghiệp vụ và rẽ nhánh dựa vào số lượng bản ghi trả về.**
