# Thiết kế bảo vệ hệ thống khi một sản phẩm trở nên quá nóng

## 1. Nguyên tắc thiết kế

Không thay cơ chế bảo vệ dữ liệu chỉ vì tải tăng. Thiết kế được khuyến nghị dùng
hai lớp độc lập:

```text
Lớp 1 — Tính đúng đắn:
PostgreSQL + UPDATE có điều kiện + giao dịch ngắn

Lớp 2 — Khả năng chịu tải:
điều tiết đầu vào + giới hạn chờ + cô lập tài nguyên
```

Cổng điều tiết có thể từ chối nhầm một yêu cầu vẫn còn cơ hội mua hàng nếu hệ
thống đang bận. Nó không được phép chấp nhận nhầm và bỏ qua câu SQL. PostgreSQL
vẫn là nơi quyết định có đủ hàng hay không.

## 2. Câu cập nhật có thẩm quyền

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity,
    version = version + 1
WHERE product_id = :productId
  AND available_quantity >= :quantity
RETURNING product_id,
          available_quantity,
          reserved_quantity,
          version;
```

Không chuyển về `@Version` chỉ để xử lý đợt bán cao điểm. Với dòng có xung đột
liên tục, khóa lạc quan làm nhiều giao dịch thất bại sau khi đã đọc và làm việc.
Câu cập nhật có điều kiện để bên thua nhận kết quả rỗng mà không cần thử lại vì
hết hàng.

## 3. Hợp đồng lệnh và kết quả

```java
public record FlashSaleKey(
        long productId,
        long campaignVersion
) {
}

public record ReserveStock(
        UUID orderId,
        FlashSaleKey sale,
        int quantity
) {
    public ReserveStock {
        Objects.requireNonNull(orderId, "orderId");
        Objects.requireNonNull(sale, "sale");
        if (quantity <= 0) {
            throw new IllegalArgumentException(
                    "quantity must be positive"
            );
        }
    }
}

public enum ReserveOutcome {
    RESERVED,
    OUT_OF_STOCK,
    BUSY
}

public record ReserveResult(
        UUID orderId,
        ReserveOutcome outcome,
        UUID reservationId,
        Integer remainingAvailable
) {
    public static ReserveResult busy(UUID orderId) {
        return new ReserveResult(
                orderId,
                ReserveOutcome.BUSY,
                null,
                null
        );
    }

    public static ReserveResult outOfStock(UUID orderId) {
        return new ReserveResult(
                orderId,
                ReserveOutcome.OUT_OF_STOCK,
                null,
                null
        );
    }
}
```

`campaignVersion` giúp phân biệt một đợt bán đã hết với lần bổ sung hàng hoặc
chiến dịch mới. Nó đặc biệt quan trọng nếu dùng gợi ý hết hàng trong bộ nhớ.

## 4. Cấu hình ngân sách theo sản phẩm nóng

```java
@ConfigurationProperties("app.flash-sale")
public record FlashSaleLimits(
        Map<Long, Integer> concurrentRequestsByProduct,
        Duration lockTimeout,
        Duration statementTimeout,
        Duration requestBudget
) {
    public FlashSaleLimits {
        concurrentRequestsByProduct = Map.copyOf(
                concurrentRequestsByProduct
        );
        concurrentRequestsByProduct.forEach((productId, limit) -> {
            if (productId <= 0 || limit <= 0) {
                throw new IllegalArgumentException(
                        "product id and limit must be positive"
                );
            }
        });
        if (lockTimeout.isNegative()
                || lockTimeout.isZero()
                || statementTimeout.compareTo(lockTimeout) <= 0
                || requestBudget.compareTo(statementTimeout) <= 0) {
            throw new IllegalArgumentException(
                    "timeouts must satisfy lock < statement < request"
            );
        }
    }
}
```

Không đặt giá trị mặc định trong tài liệu. Giới hạn phải được suy ra từ ngân sách
kết nối, số máy chủ tối đa, công việc khác dùng chung PostgreSQL và kết quả kiểm
thử tải.

## 5. Cổng điều tiết cục bộ an toàn

```java
@Component
public class HotProductAdmissionGate {

    private final Map<Long, Semaphore> gates;

    public HotProductAdmissionGate(FlashSaleLimits limits) {
        Map<Long, Semaphore> configured = new HashMap<>();
        limits.concurrentRequestsByProduct().forEach(
                (productId, count) -> configured.put(
                        productId,
                        new Semaphore(count)
                )
        );
        this.gates = Map.copyOf(configured);
    }

    public Optional<Permit> tryEnter(long productId) {
        Semaphore gate = gates.get(productId);

        if (gate == null) {
            return Optional.of(Permit.noop());
        }
        if (!gate.tryAcquire()) {
            return Optional.empty();
        }
        return Optional.of(Permit.acquired(gate));
    }

    public static final class Permit implements AutoCloseable {

        private final Semaphore gate;
        private final AtomicBoolean closed = new AtomicBoolean();

        private Permit(Semaphore gate) {
            this.gate = gate;
        }

        static Permit acquired(Semaphore gate) {
            return new Permit(gate);
        }

        static Permit noop() {
            return new Permit(null);
        }

        @Override
        public void close() {
            if (closed.compareAndSet(false, true) && gate != null) {
                gate.release();
            }
        }
    }
}
```

Đối tượng `Permit` ngăn trả quyền hai lần và bảo đảm chỉ quyền đã cấp mới được
hoàn lại. Danh sách sản phẩm được cấu hình hữu hạn nên không tạo bản đồ khóa tăng
vô hạn theo dữ liệu đầu vào.

`Semaphore` không công bằng theo mặc định. Bật chế độ công bằng có thể thay đổi
thông lượng nhưng vẫn không tạo hợp đồng “đến trước mua trước” trên toàn cụm.

## 6. Cổng điều phối nằm ngoài giao dịch

```java
@Component
public class FlashSaleReservationFacade {

    private final HotProductAdmissionGate admission;
    private final FlashSaleReservationTx worker;
    private final SoldOutHint soldOutHint;

    public FlashSaleReservationFacade(
            HotProductAdmissionGate admission,
            FlashSaleReservationTx worker,
            SoldOutHint soldOutHint
    ) {
        this.admission = admission;
        this.worker = worker;
        this.soldOutHint = soldOutHint;
    }

    public ReserveResult reserve(ReserveStock command) {
        if (soldOutHint.isSoldOut(command.sale())) {
            return ReserveResult.outOfStock(command.orderId());
        }

        Optional<HotProductAdmissionGate.Permit> granted =
                admission.tryEnter(command.sale().productId());

        if (granted.isEmpty()) {
            return ReserveResult.busy(command.orderId());
        }

        try (HotProductAdmissionGate.Permit ignored =
                     granted.orElseThrow()) {
            ReserveResult result = worker.reserve(command);
            if (result.outcome() == ReserveOutcome.OUT_OF_STOCK) {
                soldOutHint.markSoldOut(command.sale());
            }
            return result;
        }
    }
}
```

Cổng điều phối không có `@Transactional`. Quyền được lấy trước khi mở giao dịch
và được trả sau khi lớp đại diện giao dịch đã chốt hoặc ném lỗi. Nếu đặt vòng
ngoài này trong cùng giao dịch, yêu cầu có thể lấy kết nối trước khi xin quyền,
làm mất mục đích điều tiết.

## 7. Giao dịch ngắn

```java
@Service
public class FlashSaleReservationTx {

    private final InventoryStockDao stock;
    private final InventoryReservationDao reservations;
    private final PostgreSqlTimeoutPolicy timeoutPolicy;
    private final Clock clock;

    // Constructor injection omitted.

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReserveResult reserve(ReserveStock command) {
        timeoutPolicy.apply();

        Optional<StockAfterReserve> changed = stock.tryReserve(
                command.sale().productId(),
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

        return new ReserveResult(
                command.orderId(),
                ReserveOutcome.RESERVED,
                reservationId,
                changed.orElseThrow().availableQuantity()
        );
    }
}
```

Không gọi dịch vụ giá, thanh toán, Kafka hoặc API khác trong phương thức này.
Những dữ liệu cần kiểm tra trước phải được chuẩn bị trước khi xin quyền điều tiết,
miễn là quyết định cuối vẫn được bảo vệ bởi câu SQL.

## 8. Giới hạn chờ trong PostgreSQL

```java
@Component
public class PostgreSqlTimeoutPolicy {

    private final NamedParameterJdbcTemplate jdbc;
    private final FlashSaleLimits limits;

    public PostgreSqlTimeoutPolicy(
            NamedParameterJdbcTemplate jdbc,
            FlashSaleLimits limits
    ) {
        this.jdbc = jdbc;
        this.limits = limits;
    }

    public void apply() {
        setLocal("lock_timeout", limits.lockTimeout());
        setLocal("statement_timeout", limits.statementTimeout());
    }

    private void setLocal(String name, Duration value) {
        jdbc.queryForObject("""
                SELECT set_config(:name, :value, true)
                """, Map.of(
                "name", name,
                "value", value.toMillis() + "ms"
        ), String.class);
    }
}
```

`true` giới hạn cấu hình trong giao dịch hiện tại. Thời gian lấy kết nối của
HikariCP phải được cấu hình riêng và cũng phải nằm trong thời hạn phản hồi tổng.

Không dùng giới hạn chờ để biến lỗi kỹ thuật thành `OUT_OF_STOCK`. SQLSTATE
`55P03` được ánh xạ thành lỗi bận hoặc không khả dụng theo hợp đồng API.

## 9. Gợi ý hết hàng có phiên bản

```java
@Component
public class SoldOutHint {

    private final Set<FlashSaleKey> soldOut =
            ConcurrentHashMap.newKeySet();

    public boolean isSoldOut(FlashSaleKey key) {
        return soldOut.contains(key);
    }

    public void markSoldOut(FlashSaleKey key) {
        soldOut.add(key);
    }

    public void openCampaign(FlashSaleKey key) {
        soldOut.remove(key);
    }
}
```

Đây là tối ưu cục bộ sau khi PostgreSQL đã xác nhận không còn dòng đủ hàng. Một
máy chưa có gợi ý vẫn chạy câu SQL an toàn. Khi bổ sung hàng, quy trình vận hành
phải dùng `campaignVersion` mới hoặc xóa gợi ý trên mọi máy.

Không dùng gợi ý “còn hàng” để báo thành công. Mọi yêu cầu được nhận vẫn phải đi
qua PostgreSQL.

## 10. Ánh xạ phản hồi quá tải

```java
@RestController
@RequestMapping("/flash-sale/reservations")
public class FlashSaleController {

    private final FlashSaleReservationFacade facade;

    public FlashSaleController(FlashSaleReservationFacade facade) {
        this.facade = facade;
    }

    @PostMapping
    public ResponseEntity<ReserveResult> reserve(
            @RequestBody ReserveStock command
    ) {
        ReserveResult result = facade.reserve(command);
        return switch (result.outcome()) {
            case RESERVED -> ResponseEntity.ok(result);
            case OUT_OF_STOCK -> ResponseEntity
                    .status(HttpStatus.CONFLICT)
                    .body(result);
            case BUSY -> ResponseEntity
                    .status(HttpStatus.TOO_MANY_REQUESTS)
                    .body(result);
        };
    }
}
```

Chính sách gửi lại phải được công bố riêng. Không thêm `Retry-After` tùy ý nếu hệ
thống chưa có quy tắc xác định thời điểm hợp lệ. Khi cho phép gửi lại, phía gọi
cần khoảng lùi, độ lệch ngẫu nhiên và cùng mã yêu cầu để tránh tạo ý định mới.

## 11. Cô lập ngân sách kết nối

Cổng cục bộ phải dùng ít quyền hơn ngân sách kết nối còn lại sau khi dành chỗ cho
API thông thường, tác vụ nền và công việc quản trị. Khi các đường tải có đặc điểm
rất khác nhau, cân nhắc:

- triển khai đường flash sale thành nhóm máy chủ riêng;
- dùng vùng kết nối riêng nhưng vẫn trỏ cùng PostgreSQL;
- đặt giới hạn tổng kết nối ở PostgreSQL và giám sát theo tên ứng dụng;
- giới hạn số quyền theo số máy chủ tối đa, không chỉ số máy đang chạy hiện tại.

Tách vùng kết nối chỉ cô lập ảnh hưởng. Tổng kết nối của mọi vùng vẫn phải nằm
trong khả năng PostgreSQL.

## 12. Chính sách thử lại

```java
public record RetryBudget(int maxAttempts) {

    public RetryBudget {
        if (maxAttempts <= 0) {
            throw new IllegalArgumentException(
                    "maxAttempts must be positive"
            );
        }
    }

    public boolean mayRetry(
            DatabaseFailure failure,
            Deadline deadline,
            int attempt
    ) {
        return failure.retryable()
                && deadline.hasTimeForAnotherAttempt()
                && attempt < maxAttempts;
    }
}
```

Vòng thử lại nằm ngoài `FlashSaleReservationTx` để mỗi lần có giao dịch mới. Tuy
nhiên, đường đồng bộ trong đợt tải nóng nên ưu tiên trả `BUSY` thay vì tự lặp nếu
việc lặp chỉ đưa yêu cầu trở lại cùng hàng đợi.

Không thử lại:

- `OUT_OF_STOCK`;
- `BUSY` trong cùng lời gọi;
- lỗi kiểm tra đầu vào;
- yêu cầu đã hết thời hạn tổng.

## 13. Phương án hàng đợi bền vững

Khi nghiệp vụ muốn nhận yêu cầu thay vì từ chối, hợp đồng thay đổi thành bất đồng
bộ:

```text
POST /flash-sale/requests
→ ghi yêu cầu bền vững với mã yêu cầu duy nhất
→ trả 202 Accepted và requestId

bộ xử lý phân vùng theo productId
→ đọc từng yêu cầu
→ chạy UPDATE có điều kiện
→ lưu RESERVED hoặc OUT_OF_STOCK

GET /flash-sale/requests/{requestId}
→ trả kết quả đã lưu
```

Yêu cầu cùng `productId` phải tới cùng phân vùng. Hàng đợi cần giới hạn tồn đọng
hoặc chính sách dừng tiếp nhận; nếu không, nó chỉ trì hoãn quá tải.

Phương án này bắt buộc giải quyết xử lý trùng, gửi lại, lỗi giữa cập nhật kho và
ghi nhận kết quả, cân bằng lại bộ xử lý và việc hủy yêu cầu. Không triển khai một
hàng đợi trong bộ nhớ rồi trả `202`, vì tiến trình dừng sẽ làm mất yêu cầu đã xác
nhận.

## 14. Các phương án không nên dùng làm lớp chính

| Phương án | Vấn đề dưới tải nóng |
| --- | --- |
| `@Version` và thử lại rộng | Tạo nhiều giao dịch thất bại trên dòng có xung đột thường xuyên |
| `FOR UPDATE` cho phép gọi mạng | Giữ khóa và kết nối lâu hơn cần thiết |
| Tăng vùng kết nối | Thêm bên chờ cùng một nút thắt tuần tự |
| `SERIALIZABLE` | Có thể tạo thêm lỗi `40001` cần thử lại |
| Khóa phân tán thay PostgreSQL | Thêm hệ thống điều phối nhưng không bỏ được giao dịch tồn kho |
| Cờ hết hàng không phiên bản | Có thể từ chối sai sau khi bổ sung hàng |
| Hàng đợi JVM không giới hạn | Tăng bộ nhớ và mất công việc khi tiến trình dừng |

## 15. So sánh hai hợp đồng phục vụ

| Tiêu chí | Đồng bộ và từ chối nhanh | Hàng đợi bền vững |
| --- | --- | --- |
| Phản hồi ban đầu | Kết quả mua, hết hàng hoặc bận | Đã tiếp nhận cùng mã theo dõi |
| Nơi chờ | Phía gọi hoặc không chờ | Hàng đợi bền vững |
| Sử dụng kết nối khi chờ | Không | Không |
| Độ phức tạp | Thấp hơn | Cao hơn |
| Xử lý trùng | Vẫn cần cho lần gửi lại | Bắt buộc ở cả tiếp nhận và xử lý |
| Thứ tự theo sản phẩm | Không cam kết | Có thể định nghĩa qua phân vùng |
| Phục hồi tiến trình | Yêu cầu chưa nhận có thể gửi lại | Bộ xử lý tiếp tục từ yêu cầu đã lưu |

Không có phương án nào tự tăng thông lượng của một dòng PostgreSQL. Khác biệt là
cách kiểm soát lượng công việc đang chờ và hợp đồng với phía gọi.

## 16. Danh sách kiểm tra triển khai

- [ ] Câu `UPDATE` có điều kiện vẫn là lớp bảo vệ tồn kho.
- [ ] Điều tiết diễn ra trước khi mở giao dịch.
- [ ] Quyền xử lý luôn được trả trong `finally` hoặc `try`-with-resources.
- [ ] Giới hạn cục bộ được tính cùng số máy chủ tối đa.
- [ ] Có ngân sách kết nối dành cho chức năng không thuộc đợt bán.
- [ ] Thời gian lấy kết nối, chờ khóa và chạy câu lệnh nằm trong thời hạn tổng.
- [ ] `BUSY`, `OUT_OF_STOCK` và lỗi kỹ thuật được phân biệt.
- [ ] Không tự động thử lại kết quả nghiệp vụ.
- [ ] Không có lời gọi mạng trong giao dịch giữ khóa.
- [ ] Gợi ý hết hàng gắn với phiên bản chiến dịch và có đường làm mới.
- [ ] Hàng đợi, nếu có, bền vững và có giới hạn tồn đọng.
- [ ] Ngưỡng cấu hình được lấy từ phép đo, không sao chép từ tài liệu.
