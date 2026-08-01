# Mã nguồn làm sản phẩm nóng kéo sập tài nguyên chung

## 1. Cấu trúc dữ liệu

Case này giữ nguyên mô hình tồn kho an toàn của ECOM-001:

```sql
CREATE TABLE inventory_item (
    product_id BIGINT PRIMARY KEY,
    on_hand_quantity INTEGER NOT NULL,
    available_quantity INTEGER NOT NULL,
    reserved_quantity INTEGER NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    CHECK (available_quantity >= 0),
    CHECK (reserved_quantity >= 0),
    CHECK (available_quantity + reserved_quantity = on_hand_quantity)
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

Một sản phẩm được công bố trước là sản phẩm bán giới hạn. Trong khoảng thời gian
ngắn, số yêu cầu cùng nhắm tới dòng của sản phẩm này lớn hơn nhiều số giao dịch
có thể đồng thời đi qua vùng kết nối.

## 2. Câu cập nhật đúng nhưng thiếu bảo vệ tải

```java
@Repository
public class InventoryStockDao {

    private final NamedParameterJdbcTemplate jdbc;

    public InventoryStockDao(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<StockAfterReserve> tryReserve(
            long productId,
            int quantity
    ) {
        return jdbc.query("""
                UPDATE inventory_item
                SET available_quantity = available_quantity - :quantity,
                    reserved_quantity = reserved_quantity + :quantity,
                    version = version + 1
                WHERE product_id = :productId
                  AND available_quantity >= :quantity
                RETURNING product_id,
                          available_quantity,
                          reserved_quantity,
                          version
                """, Map.of(
                "productId", productId,
                "quantity", quantity
        ), STOCK_MAPPER).stream().findFirst();
    }
}
```

```java
@Service
public class DirectFlashSaleTx {

    private final InventoryStockDao stock;
    private final InventoryReservationDao reservations;
    private final Clock clock;

    // Constructor injection omitted.

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReserveResult reserve(ReserveStock command) {
        Optional<StockAfterReserve> changed = stock.tryReserve(
                command.productId(),
                command.quantity()
        );

        if (changed.isEmpty()) {
            return ReserveResult.outOfStock(command.orderId());
        }

        UUID reservationId = UUID.randomUUID();
        reservations.insertReserved(
                reservationId,
                command,
                clock.instant()
        );
        return ReserveResult.reserved(
                command.orderId(),
                reservationId,
                changed.orElseThrow().availableQuantity()
        );
    }
}
```

Mã trên bảo vệ tồn kho đúng. Điểm thiếu là mọi yêu cầu HTTP đều được phép mở giao
dịch và đi thẳng vào vùng kết nối. Khi cùng một dòng trở thành điểm nóng, các
giao dịch xếp hàng tại PostgreSQL trong khi vẫn giữ kết nối.

## 3. Bộ điều phối không giới hạn đầu vào

```java
@RestController
@RequestMapping("/flash-sale/reservations")
public class BrokenFlashSaleController {

    private final DirectFlashSaleTx worker;

    public BrokenFlashSaleController(DirectFlashSaleTx worker) {
        this.worker = worker;
    }

    @PostMapping
    public ReserveResult reserve(@RequestBody ReserveStock command) {
        return worker.reserve(command);
    }
}
```

Hàng đợi thực tế bị đẩy xuống các tầng không được thiết kế làm nơi hấp thụ đợt
tăng tải: luồng HTTP, vùng kết nối rồi hàng đợi khóa của PostgreSQL.

## 4. Thử lại ngay mọi lỗi tạm thời

```java
@Component
public class BrokenRetryingFlashSaleFacade {

    private final DirectFlashSaleTx worker;

    public BrokenRetryingFlashSaleFacade(DirectFlashSaleTx worker) {
        this.worker = worker;
    }

    public ReserveResult reserve(ReserveStock command) {
        RuntimeException lastFailure = null;

        for (int attempt = 0; attempt < 5; attempt++) {
            try {
                return worker.reserve(command);
            } catch (CannotAcquireLockException
                     | CannotGetJdbcConnectionException failure) {
                lastFailure = failure;
                // Lỗi: thử lại ngay, không xét thời hạn phản hồi còn lại.
            }
        }
        throw lastFailure;
    }
}
```

Mỗi lần gọi `worker.reserve()` mở một giao dịch mới nên cấu trúc giao dịch không
sai. Vấn đề là chính sách thử lại không có khoảng lùi, không có độ lệch ngẫu
nhiên và không biết hệ thống đang quá tải. Các yêu cầu thất bại cùng quay lại,
tạo thêm giao dịch đúng lúc hàng đợi đã dài.

Không thử lại kết quả `OUT_OF_STOCK`. Đó là quyết định nghiệp vụ, không phải lỗi
tạm thời.

## 5. Giữ khóa trong khi gọi dịch vụ ngoài

```java
@Transactional
public ReserveResult reserve(ReserveStock command) {
    InventoryItem item = items.findForUpdate(command.productId())
            .orElseThrow(ProductNotFoundException::new);

    if (!item.hasEnough(command.quantity())) {
        return ReserveResult.outOfStock(command.orderId());
    }

    PriceQuote quote = pricingClient.confirmPrice(
            command.productId()
    ); // Lỗi: lời gọi mạng khi đang giữ khóa dòng.

    item.reserve(command.quantity());
    return saveReservation(command, quote);
}
```

`FOR UPDATE` lấy khóa trước lời gọi mạng. Độ trễ hoặc lỗi của dịch vụ giá kéo dài
thời gian giữ khóa và kết nối. Tất cả người mua cùng sản phẩm phải chờ công việc
không thuộc trách nhiệm của PostgreSQL.

## 6. Hàng đợi trong bộ nhớ không giới hạn

```java
@Bean
Executor flashSaleExecutor() {
    return Executors.newFixedThreadPool(8);
}

public CompletableFuture<ReserveResult> reserveAsync(
        ReserveStock command
) {
    return CompletableFuture.supplyAsync(
            () -> worker.reserve(command),
            flashSaleExecutor
    );
}
```

`Executors.newFixedThreadPool()` dùng hàng đợi không giới hạn. Nó chuyển nơi tích
tụ yêu cầu từ vùng kết nối sang bộ nhớ JVM nhưng không tạo giới hạn tồn đọng,
không có hợp đồng từ chối và mất toàn bộ công việc chưa xử lý khi tiến trình dừng.

## 7. Cổng cục bộ bị hiểu nhầm là giới hạn toàn cụm

```java
@Component
public class MisleadingGlobalGate {

    private final Semaphore permits = new Semaphore(10);

    public boolean tryEnter() {
        return permits.tryAcquire();
    }

    public void leave() {
        permits.release();
    }
}
```

Nếu mỗi máy chủ có một đối tượng trên, mỗi máy tự cấp số giấy phép riêng. Khi số
máy chủ tăng, tổng số giao dịch được nhận cũng tăng. `Semaphore` vẫn hữu ích để
bảo vệ từng vùng kết nối cục bộ, nhưng tên và tài liệu trên khiến đội vận hành
tưởng rằng nó tạo một giới hạn chung cho toàn cụm.

Một lỗi khác là gọi `leave()` khi `tryEnter()` trả `false`, làm số giấy phép tăng
dần. Quyền xử lý phải được biểu diễn bằng một đối tượng chỉ được đóng sau khi đã
cấp thành công.

## 8. Dùng cờ hết hàng làm nguồn sự thật

```java
private final ConcurrentHashMap<Long, Boolean> soldOut =
        new ConcurrentHashMap<>();

public ReserveResult reserve(ReserveStock command) {
    if (soldOut.getOrDefault(command.productId(), false)) {
        return ReserveResult.outOfStock(command.orderId());
    }

    ReserveResult result = worker.reserve(command);
    if (result.outOfStock()) {
        soldOut.put(command.productId(), true);
    }
    return result;
}
```

Cờ cục bộ có thể giảm truy cập sau khi hàng đã hết, nhưng đoạn mã không gắn cờ
với một chiến dịch hoặc phiên bản tồn kho và không có đường xóa khi nhập thêm
hàng. Sau khi tồn kho được bổ sung, máy chủ vẫn tiếp tục từ chối sai.

Cờ này chỉ được dùng như gợi ý từ chối sớm trong một vòng đời được kiểm soát.
Nó không thay thế câu `UPDATE` có điều kiện.

## 9. Tăng vùng kết nối như một cách chữa chính

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 200
```

Không có phép tính ngân sách hay kiểm tra giới hạn PostgreSQL đi kèm cấu hình.
Thay đổi này chỉ cho phép thêm giao dịch cùng chờ một dòng, đồng thời tăng số kết
nối mà PostgreSQL phải quản lý. Tốc độ ghi tuần tự của dòng nóng không tăng theo
số kết nối.

## 10. Nâng mức cô lập để xử lý quá tải

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public ReserveResult reserve(ReserveStock command) {
    return doReserve(command);
}
```

`SERIALIZABLE` có thể phát sinh thêm lỗi `40001` cần thử lại. Nó bảo vệ các bất
biến phức tạp, không phải cơ chế điều tiết lưu lượng cho một dòng đã được bảo vệ
bằng câu cập nhật có điều kiện.

## 11. Dòng thời gian nghẽn

Gọi `P` là số kết nối có thể được đường xử lý này sử dụng và `N` là số yêu cầu
cùng đến, với `N > P`:

| Bước | Yêu cầu đang giữ kết nối | Yêu cầu còn lại | Trạng thái |
| --- | --- | --- | --- |
| 1 | Yêu cầu đầu lấy khóa dòng | Các yêu cầu khác bắt đầu vào | Một giao dịch tiến lên |
| 2 | Tối đa `P - 1` giao dịch chờ khóa | `N - P` yêu cầu chờ kết nối | Vùng kết nối đã đầy |
| 3 | Giao dịch đầu chốt | Một bên chờ khóa được đánh thức | Hàng đợi dịch chuyển từng vị trí |
| 4 | Hàng có thể đã hết | Nhiều bên vẫn phải lấy kết nối để nhận `0` dòng | Công việc vô ích tiếp tục |
| 5 | Một số phản hồi hết thời hạn | Phía gọi gửi lại yêu cầu | Tải mới chồng lên hàng đợi cũ |

Tồn kho vẫn không âm. Lỗi quan sát được là chậm, hết thời gian, báo bận và ảnh
hưởng dây chuyền sang chức năng khác.

## 12. Dấu hiệu trong môi trường thật

- Số kết nối đang dùng chạm giới hạn trong lúc nhiều phiên chờ cùng một dòng.
- `pg_stat_activity` cho thấy nhiều giao dịch có `wait_event_type = 'Lock'`.
- Luồng chờ lấy kết nối tăng cùng thời điểm sản phẩm nóng được mở bán.
- Lỗi `55P03` và lỗi lấy kết nối tăng nhưng số lần giữ hàng thành công không tăng
  tương ứng.
- API không liên quan chậm dù không truy cập sản phẩm nóng.
- Số lần thử lại lớn hơn số yêu cầu gốc.
- Bộ nhớ tăng do hàng đợi trong ứng dụng không có giới hạn.

Phần sửa đúng nằm trong [solutions.md](solutions.md). Các phép kiểm tra định tính
về khóa, vùng kết nối và điều tiết nằm trong [experiments.md](experiments.md).
