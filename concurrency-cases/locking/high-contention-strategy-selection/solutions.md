# Giải Pháp — Khung Lựa Chọn Chiến Lược Dựa Trên Mức Tương Tranh

## 1. Nguyên tắc lựa chọn

Không tồn tại chiến lược lock tối ưu cho mọi trường hợp. Lựa chọn phải dựa trên đặc điểm tương tranh thực tế:

```text
Mức tương tranh thấp  → Lock lạc quan (@Version)
Mức tương tranh trung bình → Lock bi quan (FOR UPDATE) hoặc Atomic UPDATE
Mức tương tranh cao → Atomic UPDATE
Mức tương tranh cực cao (hot key) → Atomic UPDATE + Kiểm soát đầu vào (Admission control)
```

Tính đúng đắn (correctness) là điều kiện tiên quyết, không phải tiêu chí so sánh. Cả bốn chiến lược đều bảo đảm tính đúng đắn. Điểm khác biệt nằm ở hành vi phi chức năng: thông lượng, độ trễ, tải thử lại, và mức tiêu thụ kết nối.

## 2. Chiến lược 1 — Lock lạc quan (@Version)

### Khi nào phù hợp

- Tương tranh thấp: ít hơn 5–10 transaction đồng thời trên cùng bản ghi.
- Xung đột hiếm khi xảy ra: hệ thống hoạt động bình thường, không có sự kiện đặc biệt.
- Đọc nhiều hơn ghi: phần lớn yêu cầu chỉ đọc, xung đột ghi không thường xuyên.
- Cần giảm thiểu thời gian giữ lock: không muốn các transaction chờ đợi nhau.

### Triển khai chuẩn

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

    public void reserve(int quantity) {
        if (availableQuantity < quantity) {
            throw new InsufficientStockException();
        }
        availableQuantity -= quantity;
        reservedQuantity += quantity;
    }
}
```

### Bộ thử lại đúng cách

```java
@Component
public class OptimisticReservationFacade {
    private final OptimisticReservationService service;

    // constructor injection omitted

    @Retryable(
        retryFor = OptimisticLockException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 50, multiplier = 2, random = true)
    )
    public ReservationResult reserve(ReserveStockCommand command) {
        return service.reserve(command);
    }
}
```

Yêu cầu:
- Retry nằm BÊN NGOÀI `@Transactional` để mỗi lần thử lại mở transaction mới.
- Backoff với jitter để phân tán thời điểm thử lại.
- Giới hạn số lần thử lại (bounded retry).
- Giới hạn thời gian tổng (deadline) nếu có SLA.

### Giới hạn

Khi tỷ lệ xung đột vượt 50%, hệ thống tiêu thụ phần lớn tài nguyên cho các transaction thất bại. Chuyển sang atomic UPDATE hoặc lock bi quan.

## 3. Chiến lược 2 — Lock bi quan (FOR UPDATE)

### Khi nào phù hợp

- Tương tranh trung bình: 10–50 transaction đồng thời.
- Logic nghiệp vụ cần đọc dữ liệu trước khi quyết định: phải tải entity, tính toán trên nhiều trường, rồi cập nhật.
- Quy trình phức tạp sau khi đọc: logic nghiệp vụ không thể gói gọn trong một mệnh đề `WHERE`.
- Chấp nhận xếp hàng: thời gian chờ nằm trong ngưỡng chấp nhận được.

### Triển khai chuẩn

```java
public interface InventoryItemRepository extends JpaRepository<InventoryItem, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "javax.persistence.lock.timeout", value = "500"))
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

Yêu cầu:
- Thiết lập `lock_timeout` để tránh chờ vô hạn. Giá trị phụ thuộc SLA và đặc điểm hệ thống.
- Transaction ngắn: không thực hiện remote I/O hoặc logic phức tạp sau khi chiếm lock.
- Xử lý `LockTimeoutException` (`55P03`): trả về trạng thái `BUSY` hoặc thử lại với backoff.
- Xử lý deadlock (`40P01`): thử lại toàn bộ transaction.

### Giới hạn

Khi lock queue dài hơn connection pool, hệ thống nghẽn toàn bộ. Cần giám sát chiều dài lock queue và chuyển sang atomic UPDATE nếu queue thường xuyên vượt ngưỡng.

## 4. Chiến lược 3 — Cập nhật nguyên tử (Conditional UPDATE)

### Khi nào phù hợp

- Tương tranh trung bình đến cao: 10–200 transaction đồng thời.
- Logic kiểm tra có thể gói gọn trong mệnh đề `WHERE`: điều kiện đơn giản trên cùng bản ghi.
- Phía thua không cần thử lại: kết quả `0` dòng là câu trả lời nghiệp vụ hợp lệ (hết hàng).
- Ưu tiên thông lượng: giảm thiểu thời gian giữ lock bằng cách loại bỏ bước đọc-tính toán trên RAM.

### Triển khai chuẩn

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
        RETURNING available_quantity, reserved_quantity, version
        """, nativeQuery = true)
    List<Object[]> decrementReturning(@Param("productId") long productId,
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
        List<Object[]> rows = inventory.decrementReturning(
                command.productId(), command.quantity());

        if (rows.isEmpty()) {
            // 0 dòng: không đủ hàng hoặc sản phẩm không tồn tại
            reservations.save(
                    OrderReservation.outOfStock(command, clock.instant()));
            return ReservationResult.outOfStock();
        }

        Object[] row = rows.get(0);
        int remainingAvailable = ((Number) row[0]).intValue();
        int remainingReserved = ((Number) row[1]).intValue();
        long newVersion = ((Number) row[2]).longValue();

        reservations.save(OrderReservation.reserved(
                command, remainingAvailable, remainingReserved, clock.instant()));

        return ReservationResult.reserved(remainingAvailable);
    }
}
```

Yêu cầu:
- Kiểm tra đầu vào (input validation) trước khi gửi xuống database: `quantity > 0`.
- Không tải entity bằng JPA nếu chỉ cần thực thi atomic UPDATE. Tránh xung đột bộ đệm (persistence context).
- Phân biệt nguyên nhân 0 dòng: thiếu hàng, sản phẩm không tồn tại, hoặc tham số không hợp lệ.
- Sử dụng `RETURNING` để lấy trạng thái sau cập nhật trong cùng câu lệnh.

### Giới hạn

Khi tải vượt xa khả năng xử lý tuần tự (hàng nghìn yêu cầu trong vài giây), lock queue vẫn có thể cạn kiệt connection pool. Cần kết hợp kiểm soát đầu vào.

## 5. Chiến lược 4 — Kiểm soát đầu vào (Admission control)

### Khi nào phù hợp

- Tương tranh cực cao: hàng nghìn yêu cầu trong vài giây trên cùng bản ghi.
- Flash-sale, viral event, hoặc bất kỳ kịch bản hot-key nào.
- Cần bảo vệ database khỏi bị quá tải bởi lưu lượng đột biến.

### Ý tưởng

Giới hạn số lượng yêu cầu được phép truy cập database tại một thời điểm. Các yêu cầu vượt ngưỡng bị từ chối ngay tại tầng ứng dụng.

### Triển khai với Semaphore cấp ứng dụng

```java
@Component
public class AdmissionControlledReservationFacade {
    private final AtomicReservationService service;
    private final Semaphore permit;

    public AdmissionControlledReservationFacade(
            AtomicReservationService service,
            @Value("${reservation.max-concurrent:10}") int maxConcurrent
    ) {
        this.service = service;
        this.permit = new Semaphore(maxConcurrent);
    }

    public ReservationResult reserve(ReserveStockCommand command) {
        if (!permit.tryAcquire()) {
            return ReservationResult.busy();
        }
        try {
            return service.reserve(command);
        } finally {
            permit.release();
        }
    }
}
```

Giới hạn: `Semaphore` chỉ bảo vệ một JVM. Trong kiến trúc đa máy chủ, mỗi instance có semaphore riêng, tổng tải vẫn bằng `maxConcurrent × số_instance`. Giá trị `maxConcurrent` cần được tính toán dựa trên connection pool và số instance.

### Triển khai với hàng đợi (Message queue)

```text
API nhận yêu cầu → Kafka topic (partition theo product_id) → Consumer tuần tự
  ↓ (phản hồi ngay)
202 Accepted
  ↓ (sau khi consumer xử lý)
Webhook/SSE thông báo kết quả
```

Ưu điểm:
- Database nhận tải ổn định, bất kể lưu lượng đầu vào.
- Không có lock queue, không cạn kiệt kết nối.
- Có thể mở rộng bằng cách thêm partition (nếu có nhiều sản phẩm nóng).

Nhược điểm:
- Chuyển từ mô hình đồng bộ (synchronous) sang bất đồng bộ (asynchronous).
- Phía gọi (caller) cần cơ chế nhận kết quả sau (webhook, polling, SSE).
- Hạ tầng phức tạp hơn: cần Kafka, consumer group, error handling.
- Phải giải quyết idempotency ở cả tầng producer và consumer.

## 6. Khung quyết định (Decision framework)

### Bước 1 — Xác định mức tương tranh

```text
Đo lường: Số transaction đồng thời trên cùng bản ghi trong khoảng thời gian cao điểm.
Phương pháp: Giám sát pg_stat_activity, lock queue length, retry rate.
```

### Bước 2 — Đánh giá đặc điểm logic nghiệp vụ

| Câu hỏi | Nếu CÓ | Nếu KHÔNG |
| --- | --- | --- |
| Logic kiểm tra gói gọn trong WHERE? | Atomic UPDATE là ứng viên chính | Cần FOR UPDATE hoặc SERIALIZABLE |
| Phía thua cần thử lại? | Lock bi quan hoặc hàng đợi | Atomic UPDATE (0 dòng = kết quả) |
| Cần đọc nhiều trường trước khi quyết định? | FOR UPDATE | Atomic UPDATE |
| Xung đột ghi hiếm khi xảy ra? | @Version | Atomic UPDATE hoặc FOR UPDATE |

### Bước 3 — Chọn chiến lược cơ sở

```text
if (tương_tranh_thấp AND xung_đột_hiếm)
    → @Version + bounded retry + backoff/jitter

else if (logic_phức_tạp OR cần_đọc_trước_khi_quyết_định)
    → FOR UPDATE + lock_timeout + transaction ngắn

else if (logic_đơn_giản AND có_thể_gói_trong_WHERE)
    → Atomic UPDATE + kiểm tra affected rows

else if (tương_tranh_cực_cao OR hot_key)
    → Atomic UPDATE + admission control (semaphore/queue)
```

### Bước 4 — Thiết kế lớp bảo vệ bổ sung

Bất kể chiến lược nào, luôn kết hợp:

| Lớp bảo vệ | Mục đích |
| --- | --- |
| `CHECK (available_quantity >= 0)` | Phòng thủ sâu (defense in depth) tại database |
| Idempotency key (command_id) | Ngăn xử lý trùng lặp khi client thử lại |
| Lock timeout | Giới hạn thời gian chờ, tránh giữ kết nối vô hạn |
| Giám sát (monitoring) | Phát hiện sớm khi mức tương tranh vượt ngưỡng thiết kế |
| Circuit breaker | Ngắt mạch khi tỷ lệ lỗi vượt ngưỡng, bảo vệ database |

## 7. Chiến lược kết hợp (Hybrid approach)

Trong hệ thống thực tế, mức tương tranh thay đổi theo thời gian. Có thể kết hợp chiến lược dựa trên trạng thái hệ thống:

```java
@Service
public class AdaptiveReservationService {
    private final AtomicReservationService atomicService;
    private final Semaphore hotKeyPermit;
    private final HotKeyDetector hotKeyDetector;

    // constructor injection omitted

    public ReservationResult reserve(ReserveStockCommand command) {
        if (hotKeyDetector.isHotKey(command.productId())) {
            // Sản phẩm nóng: áp dụng kiểm soát đầu vào
            if (!hotKeyPermit.tryAcquire()) {
                return ReservationResult.busy();
            }
            try {
                return atomicService.reserve(command);
            } finally {
                hotKeyPermit.release();
            }
        }

        // Sản phẩm bình thường: atomic UPDATE không cần giới hạn
        return atomicService.reserve(command);
    }
}
```

`HotKeyDetector` có thể dựa trên:
- Cấu hình tĩnh: đánh dấu sản phẩm flash-sale trước sự kiện.
- Giám sát động: theo dõi tần suất truy vấn và tự động phát hiện hot key.

## 8. So sánh tổng hợp (Summary comparison)

| Chiến lược | Mức tương tranh phù hợp | Ưu điểm chính | Nhược điểm chính |
| --- | --- | --- | --- |
| @Version | Thấp (< 10) | Đơn giản, không chờ lock | Retry storm khi tương tranh cao |
| FOR UPDATE | Trung bình (10–50) | Phù hợp logic phức tạp | Giữ kết nối khi chờ |
| Atomic UPDATE | Trung bình–cao (10–200) | Thời gian giữ lock ngắn nhất | Chỉ phù hợp logic đơn giản |
| Admission control | Cực cao (> 200) | Bảo vệ database khỏi quá tải | Phức tạp hạ tầng, mô hình bất đồng bộ |

## 9. Xử lý lỗi và phục hồi (Error handling)

| Mã lỗi | Chiến lược xử lý |
| --- | --- |
| `OptimisticLockException` | Thử lại với transaction mới, backoff/jitter, tối đa 3 lần |
| `55P03` (lock timeout) | Trả về `BUSY`, thử lại với backoff, hoặc chuyển sang hàng đợi |
| `40P01` (deadlock) | Thử lại toàn bộ transaction, tối đa 2 lần |
| `40001` (serialization failure) | Thử lại toàn bộ transaction, backoff/jitter |
| Affected rows = 0 | Phản hồi `OUT_OF_STOCK`, KHÔNG thử lại |
| Connection pool timeout | Trả về `SERVICE_UNAVAILABLE`, kích hoạt circuit breaker |

## 10. Các yêu cầu giám sát (Observability requirements)

Để vận hành chiến lược hiệu quả, cần giám sát:

| Chỉ số | Mục đích |
| --- | --- |
| Tỷ lệ retry (retry rate) | Phát hiện sớm tình trạng retry storm |
| Lock wait time (p95, p99) | Đo chiều dài lock queue |
| Active connections / Pool size | Phát hiện cạn kiệt connection pool |
| Tỷ lệ lỗi 55P03 / 40P01 | Phát hiện lock timeout và deadlock |
| Thông lượng thành công / giây | So sánh với tải đầu vào |
| Thời gian phản hồi (p95, p99) | Đo ảnh hưởng đến trải nghiệm phía gọi |
| Số sản phẩm có tỷ lệ xung đột > 50% | Phát hiện hot key mới |

## 11. Phạm vi tài liệu (Scope boundary)

Bài toán này cung cấp khung lựa chọn chiến lược định tính. Không đưa ra con số hiệu năng cụ thể hoặc khuyến nghị kích thước connection pool.

Quyết định cụ thể thuộc phạm vi bài toán nghiệp vụ:
- Tồn kho flash-sale: `ECOM-001`, `ECOM-002`.
- Rút tiền đồng thời: `BANK-001`.
- Giữ chỗ sự kiện: `BOOK-001`.

Cơ chế phân tán (Redis lock, Kafka partition) thuộc phạm vi `REDIS-*` và `DIST-*`.
