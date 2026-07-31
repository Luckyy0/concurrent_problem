# Giải phẫu những đoạn Code "có mùi" — Lỗi Đọc, Kiểm tra rồi Ghi đè số cứng

## Cấu trúc bảng ban đầu (Schema)

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

Giả sử Sản phẩm `77` đang có Tổng=`5`, Sẵn sàng=`5`, Đã giữ=`0`.

Lưu ý: Các rào chắn `CHECK` ở cấp độ dòng (row-level) vẫn bị vượt mặt dễ dàng khi xảy ra lỗi ghi đè (lost update): Luồng đi sau cùng chỉ ghi lại trạng thái Sẵn sàng=`1`, Đã giữ=`4`, trong khi dưới bảng Lịch sử (inventory_reservation) đã có tận 2 cái lịch sử tổng cộng tới `8` chiếc. Database vẫn vui vẻ chập nhận vì `1 + 4 = 5` và `>= 0`.

## Thực thể (Entity) thiếu thẻ `@Version`

```java
@Entity
@Table(name = "inventory_item")
public class InventoryItem {
    @Id
    private Long productId;

    private int onHandQuantity;
    private int availableQuantity;
    private int reservedQuantity;
    private long revision; // <-- Thiếu @Version ở đây

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

Trường `revision` ở đây chỉ là một biến đếm số nguyên bình thường. Do bạn không gắn thẻ `@Version`, Hibernate sẽ **không** tự động biến nó thành lớp khóa bảo vệ (optimistic lock) khi chạy lệnh `UPDATE`.

## Dịch vụ viết sai (Broken service)

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

        // LỖI: Kiểm tra nghiệp vụ trên RAM bằng đồ "thiu"
        if (!item.canReserve(command.quantity())) {
            return ReservationResult.outOfStock();
        }

        // LỖI: Cập nhật biến trên RAM
        item.reserve(command.quantity());
        
        InventoryReservation reservation =
                InventoryReservation.reserved(command, clock.instant());
        reservations.save(reservation);
        outbox.save(OutboxEvent.inventoryReserved(reservation));

        return ReservationResult.reserved(reservation.getReservationId());
    }
}
```

Hàm này được gắn `@Transactional` đàng hoàng, mọi thứ được lưu cùng nhau. Nhưng chết ở chỗ: Logic kiểm tra "còn hàng không" lại nằm trên code Java dựa trên cái Ảnh chụp cũ rích lúc mới `findById`, nó hoàn toàn đứt liên kết với câu lệnh `UPDATE` lát nữa mới đẩy xuống DB.

Lúc này Hibernate ngây thơ xuất ra câu SQL ghi đè con số chết (absolute value):

```sql
UPDATE inventory_item
SET available_quantity = 1,
    reserved_quantity = 4,
    revision = 1
WHERE product_id = 77;
```

Nếu 2 luồng cùng lúc chạy, cả 2 đối tượng `item` trên RAM đều bị cũ, cùng tự trừ đi 4 và cùng phát ra y chang cái lệnh Update này. Và Database lại báo sửa thành công 1 dòng cho cả 2 bên.

## Điều kiện để tai nạn này xảy ra

- Database chạy ở mức cách ly `READ COMMITTED` (mặc định của Postgres).
- Sản phẩm `77` có sẵn `5` cái.
- 2 Khách hàng cùng mua `4` cái với 2 Mã đơn khác nhau.
- Cả 2 luồng cùng chạy qua lệnh `SELECT` (findById) xong xuôi hết rồi mới có 1 luồng kịp đẩy (flush) xuống DB.
- Không hề dùng `@Version`, không dùng khóa bi quan `FOR UPDATE` và cũng không dùng Cập nhật có điều kiện.
- ID của lịch sử và ID của hộp thư (outbox) sinh ra ngẫu nhiên nên không đụng độ nhau.

## Dòng thời gian của Tai nạn

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | `SELECT` → Thấy có sẵn `5` | |
| 2 | | `SELECT` → Thấy có sẵn `5` |
| 3 | Java phán: `5 >= 4`, DUYỆT | Java phán: `5 >= 4`, DUYỆT |
| 4 | Lưu lịch sử A / Hộp thư A | Lưu lịch sử B / Hộp thư B |
| 5 | `UPDATE` gửi con số chết → `1/4` | |
| 6 | Chốt sổ (commit) | |
| 7 | | `UPDATE` mù quáng ghi đè số cũ → `1/4`, chốt sổ |

Hậu quả thảm khốc:
```text
Trong bảng kho: available=1, reserved=4
Trong bảng lịch sử: Đơn A giữ 4 chiếc, Đơn B giữ 4 chiếc, TỔNG = 8 chiếc
```
Không có cái `CHECK constraint` nào của DB bị vi phạm, nhưng cái sổ đối soát giữa 2 bảng thì lệch nhau 1 dặm.

> **Nói ngắn gọn:** Rào chắn (Constraint) chỉ bảo vệ đúng hình hài của 1 dòng dữ liệu; nó không thông minh tới mức biết bạn vừa sinh ra 2 cái lịch sử sai sự thật ở cái bảng bên cạnh.

## 12 Lỗi kinh điển của lập trình viên mới

### Lỗi 1 — Tin mù quáng rằng `@Transactional` sẽ tự khóa dữ liệu

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
Cái chữ `@Transactional` chỉ vẽ ra một "vùng an toàn" cho việc Commit hoặc Rollback chung. Còn bên trong đó, cái lệnh `SELECT` đầu tiên và cái `UPDATE` cuối cùng vẫn là 2 câu lệnh rời rạc, thừa sức cho luồng khác chạy chen ngang vào giữa.

### Lỗi 2 — Viết UPDATE có điều kiện nhưng... bỏ quên kết quả trả về

```java
@Transactional
public ReservationResult reserve(ReserveStockCommand command) {
    // Gọi lệnh atomic update xịn nhưng quên lấy kết quả trả về
    inventory.decrementIfEnough(command.productId(), command.quantity());
    reservations.save(InventoryReservation.reserved(command));
    return ReservationResult.reserved(); // Lúc nào cũng báo thành công!
}
```
Nếu hết hàng, DB không update dòng nào (trả về `0`). Code của bạn chả thèm đoái hoài kiểm tra số `0` đó mà cứ nhắm mắt báo là "Giữ chỗ thành công". Lệnh SQL thông minh mà người dùng nó không kiểm tra thì cũng bằng thừa.

### Lỗi 3 — Viết SQL thiếu đi cái "khiên bảo vệ" (business guard)

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE product_id = :productId; -- Đợi đã, thiếu điều kiện kiểm tra tồn kho!!
```
Bạn trừ tương đối (Atomic arithmetic) thì chống được chuyện ghi đè mất kết quả, nhưng không chống được chuyện kho bị ÂM. Phải nhét cái điều kiện `AND available_quantity >= :quantity` vào cụm WHERE nữa mới đủ.

### Lỗi 4 — Đọc lên RAM kiểm tra rồi xuống DB nhắm mắt trừ bừa

```java
if (inventory.currentAvailable(productId) >= quantity) {
    // Đã thỏa điều kiện ở RAM, tao trừ dưới DB luôn bất chấp
    inventory.decrementUnconditionally(productId, quantity); 
}
```
Check trên RAM là check đồ "thiu". Cụm điều kiện (Predicate) và cụm sửa dữ liệu (Mutation) phải luôn dính chặt vào nhau trong 1 câu lệnh dưới Database.

### Lỗi 5 — Gửi Bulk DML thẳng xuống DB nhưng RAM vẫn ngậm đồ cũ

```java
@Transactional
public void reserve(long productId, int quantity) {
    // Bước 1: JPA tự ngậm dữ liệu cũ vào RAM (Bộ đệm)
    InventoryItem stale = inventory.findById(productId).orElseThrow();

    // Bước 2: Bắn SQL trực tiếp trừ kho lách qua mặt JPA
    int changed = inventory.decrementIfEnough(productId, quantity);
    if (changed == 1) {
        // Bước 3: Sửa nhẹ 1 field khác trên object cũ mèm trên RAM
        stale.renameForAudit("reserved"); 
    }
    // Cuối hàm, JPA tự động flush object cũ này xuống DB và... BÙM! 
    // Thành quả của lệnh SQL ở Bước 2 bị ghi đè sạch.
}
```
Đừng bao giờ trộn lẫn việc bắn SQL trực tiếp (Bulk DML) với việc lôi Thực thể (Entity) lên RAM sửa nếu bạn chưa rành việc dọn dẹp bộ đệm (`clear/refresh`).

### Lỗi 6 — Gọi `save()` dư thừa sau lệnh Atomic UPDATE

```java
int changed = inventory.decrementIfEnough(productId, quantity);
InventoryItem detachedSnapshot = mapper.fromEarlierRequest(request);
inventory.save(detachedSnapshot); // LỖI
```
Hàm `save()` (hoặc `merge`) của Spring Data sẽ cố gắng ghi lại toàn bộ cái object vào DB. Việc này vô tình phá hỏng hoàn toàn kết quả nguyên tử mà lệnh UPDATE ở trên vừa cực khổ làm được. Ai thắng lệnh Atomic UPDATE thì coi như xong việc, KHÔNG cần phải `save` cái entity nào nữa!

### Lỗi 7 — Nhắm mắt phán số "0 dòng" luôn là `OUT_OF_STOCK`

Nếu Database trả về là sửa `0` dòng (affected rows = 0), nó có thể vì:
- Mã sản phẩm đó bị sai, không hề tồn tại.
- Bạn truyền số lượng mua bằng `0` (sai logic).
- Sản phẩm đó đang bị Khóa vô hiệu hóa.
- Hoặc đúng là Hết hàng thật.

Đừng thiết kế API theo kiểu "ngồi đoán mò". Phải validate input tử tế, đảm bảo sản phẩm có tồn tại rồi mới chạy lệnh, hoặc phải trả ra các mã lỗi nghiệp vụ rõ ràng cho Frontend.

### Lỗi 8 — Đánh tráo khái niệm: Lỗi quá tải = Lỗi hết hàng

```java
try {
    return inventory.decrementIfEnough(productId, quantity) == 1;
} catch (CannotAcquireLockException ignored) {
    return false; // Code bẩn: Ém nhẹm lỗi kẹt khóa (55P03) thành lỗi Hết hàng (OUT_OF_STOCK)
}
```
Bị cập nhật `0` dòng là một kết quả hợp lệ của quy tắc kinh doanh. Nhưng việc văng lỗi `55P03` là do hệ thống đang quá tải không chen chân vào lấy khóa nổi. Bạn phải ném lỗi này ra để hệ thống báo động hoặc Tự thử lại (retry), chứ không được lừa user là "Bạn ơi hết hàng rồi".

### Lỗi 9 — Khách bấm liên tục, sinh mã đơn mới để thử lại

Khi mạng bị lag mất kết quả trả về, nếu bạn cứ sinh ra cái mã đơn (command ID) mới toanh rồi đập vô hệ thống, DB sẽ coi đó là 2 đơn khác nhau và sẽ trừ kho 2 lần. Phải tái sử dụng lại cái "Mã lệnh cũ" và có bảng chống trùng lặp (durable claim) để tự vệ. Lệnh SQL Cập nhật có điều kiện chỉ chống bán âm kho, nó KHÔNG CHỐNG ĐƯỢC CHUYỆN KHÁCH BỊ TRỪ HÀNG NHIỀU LẦN.

### Lỗi 10 — Hý hửng báo tin trước khi chốt sổ (Publish trước commit)

```java
int changed = inventory.decrementIfEnough(productId, quantity);
if (changed == 1) {
    kafkaTemplate.send("inventory-reserved", event); // Gửi tin nhắn ầm ĩ lên mạng
    reservations.save(reservation);
}
// Nếu bị lỗi văng exception ở dòng dưới, Giao dịch Rollback, DB hủy bỏ lệnh trừ kho
// Nhưng tin nhắn thì đã bay qua máy khác mất rồi!
```
Muốn gửi tin nhắn ra ngoài, hãy xài mô hình Hộp thư gửi đi (Outbox). Lưu tin nhắn vào DB chung 1 Giao dịch với lệnh sửa kho. Sau khi Commit xong xuôi mới cho tiến trình ngầm chui vào DB đọc Hộp thư ra mà gửi đi.

### Lỗi 11 — Dùng từ khóa Java (JVM lock) trong môi trường nhiều máy chủ

```java
synchronized (localProductMutex(productId)) {
    return transactionalWorker.reserve(command);
}
```
Lệnh `synchronized` của Java chỉ khóa được trong đúng 1 máy tính của bạn. Mở 2 máy chủ lên là nó mù tịt. Điểm phân xử tranh chấp chéo (cross-node) chuẩn mực nhất chính là câu lệnh UPDATE dưới Database.

### Lỗi 12 — Dùng Database "chép" (Replica) để kiểm tra nghiệp vụ

Đọc tồn kho từ cục DB Replica (DB đọc) sẽ rất nhanh, nhưng dữ liệu của nó luôn trễ hơn DB chính (Primary). Nếu bạn dại dột tin vào con số lấy từ Replica để quyết định nghiệp vụ (bỏ qua điều kiện kiểm tra ở cụm WHERE khi Ghi), bạn sẽ rước lại lỗi bán lố hàng. Replica chỉ sinh ra để phục vụ cho các nút "Hiển thị" (Display) trên màn hình mà thôi.

## Những triệu chứng bệnh (Dấu hiệu quan sát được)

- Tính tống hàm SUM() trong bảng Lịch sử thì ra một số, nhưng field `reserved_quantity` trong bảng Kho lại là một số khác.
- Lệnh UPDATE bắn về `0` dòng (hết hàng) nhưng API chóp bu vẫn trả về chữ `RESERVED` xanh lè.
- Hibernate nhả ra con số cũ mèm dù bạn vừa gọi câu lệnh SQL thuần cập nhật.
- 1 cái Mã lệnh (Duplicate command) bị trừ kho tận 2-3 lần vì Client spam nút "Thử lại".
- Mã lỗi kẹt xe `55P03` bị log nhầm thành lỗi Nghiệp vụ Khách hàng.
- Bug biến mất khi chạy trên máy Dev, nhưng lên Production (chạy nhiều máy chủ) thì lỗi lòi ra.

## Trình tự "Độc Hại" cần phải vứt bỏ ngay lập tức

```text
BƯỚC 1: Lấy số lượng từ Database lên
→ BƯỚC 2: Code Java ngồi xét duyệt `có sẵn >= số lượng mua`
→ BƯỚC 3: Code Java ngồi bấm máy tính trừ trừ cộng cộng ra con số tuyệt đối
→ BƯỚC 4: Xuống Database gọi UPDATE ghi đè vào bằng ID
```

**Cách trị bệnh duy nhất:** Hãy ném cụm kiểm tra và cụm ý định trừ hàng thành một khối duy nhất (`WHERE` làm màng lọc, `SET` làm hành động), và nhớ LẤY KẾT QUẢ SỐ DÒNG (affected-row) về để rẽ nhánh nghiệp vụ.

