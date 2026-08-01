# Môi trường thực nghiệm: Tái hiện xung đột khóa lạc quan bằng Testcontainers

## 1. Mục tiêu thực nghiệm

Mục đích của bộ kiểm thử này nhằm chứng minh các luận điểm kỹ thuật sau:

1. Hệ quả trực tiếp của việc thiếu cơ chế `@Version` (gây ra hiện tượng ghi đè dữ liệu).
2. Kiểm soát đồng bộ hai luồng truy xuất cùng một phiên bản dữ liệu, và chỉ định luồng hoàn tất transaction trước.
3. Xác nhận luồng chậm hơn (luồng thua cuộc) sẽ kích hoạt ngoại lệ khóa lạc quan, thay đổi `0` dòng và transaction bị hủy bỏ (rollback).
4. Kiểm chứng tính toàn vẹn của dữ liệu cuối cùng trong database thuộc về luồng chiến thắng, thay vì chỉ kiểm tra sự hiện diện của ngoại lệ.
5. Kiểm tra cấu trúc câu lệnh SQL do Hibernate sinh ra để xác nhận sự xuất hiện của mệnh đề `version = ?`.
6. Xác thực cơ chế phòng vệ tại tầng API khi phía gọi truyền lên thông số version không hợp lệ (dữ liệu lỗi thời).

> **Ghi chú kỹ thuật:** Việc kiểm thử cơ chế khóa lạc quan yêu cầu sử dụng rào chắn luồng có độ chính xác cao nhằm thiết lập kịch bản đụng độ định danh, thay vì phụ thuộc vào các bài kiểm thử tải (load test) mang tính ngẫu nhiên.

## 2. Thiết lập môi trường bằng PostgreSQL Testcontainers

Môi trường kiểm thử yêu cầu hệ quản trị database vật lý thay thế cho các database in-memory (như H2) nhằm bảo đảm tính chính xác của cơ chế transaction và MVCC:

```java
@Testcontainers
@SpringBootTest
class OptimisticOfferIT {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private OfferAttemptService attempts;

    @Autowired
    private JdbcTemplate jdbc;

    // Cấp phát ExecutorService với 2 luồng đại diện cho 2 phía gọi
    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    @BeforeEach
    void seed() {
        // Thiết lập trạng thái dữ liệu nguyên bản
        jdbc.update("delete from product_offer");
        // Khởi tạo bản ghi: Giá 100, Phiên bản 7
        jdbc.update("""
                insert into product_offer(offer_id, price, title, version)
                values (42, 100.00, 'Launch offer', 7)
                """);
    }

    @AfterEach
    void shutdown() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Lưu ý: Phương thức test tuyệt đối không khai báo `@Transactional` để cho phép hệ thống tự động xử lý ranh giới transaction.

## 3. Kiến trúc điều phối rào chắn

Sử dụng `CountDownLatch` để kiểm soát thứ tự thực thi của các luồng:

```java
@TestConfiguration
class GateConfiguration {

    @Bean
    @Primary
    OfferEditGate offerEditGate() {
        return new TwoEditorGate();
    }
}

final class TwoEditorGate implements OfferEditGate {

    private final CountDownLatch bothLoaded = new CountDownLatch(2);
    private final CountDownLatch allowA = new CountDownLatch(1);
    private final CountDownLatch aCommitted = new CountDownLatch(1);

    @Override
    public void afterLoad(String actor, long version) {
        bothLoaded.countDown();
        await(bothLoaded, "Bắt buộc hai luồng hoàn tất nạp v" + version);
        
        if ("A".equals(actor)) {
            await(allowA, "Đợi tín hiệu tiếp tục cho tiến trình A");
        } else {
            await(aCommitted, "Tiến trình B bị đình chỉ cho đến khi A hoàn tất commit");
        }
    }

    void allowA() {
        allowA.countDown();
    }

    void aCommitted() {
        aCommitted.countDown();
    }
}

static void await(CountDownLatch latch, String description) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Vượt quá thời gian chờ (timeout): " + description);
        }
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Tiến trình bị gián đoạn: " + description, interrupted);
    }
}
```

Trong cấu hình môi trường thực tế, interface `OfferEditGate` sẽ triển khai không hoạt động (no-op). Ở môi trường kiểm thử, `aCommitted()` chỉ được gọi khi transaction A đã kết thúc pha commit.

## 4. Dịch vụ cập nhật qua Spring proxy

```java
@Service
public class OfferAttemptService {

    private final ProductOfferRepository offers;
    private final OfferEditGate gate;

    public OfferAttemptService(
            ProductOfferRepository offers,
            OfferEditGate gate
    ) {
        this.offers = offers;
        this.gate = gate;
    }

    // Yêu cầu phân định transaction độc lập cho từng luồng
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public long edit(String actor, BigDecimal price) {
        ProductOffer offer = offers.findById(42L).orElseThrow();
        
        // Luồng thực thi tiến nhập vào rào chắn kiểm thử
        gate.afterLoad(actor, offer.version());
        
        offer.changePrice(price);
        offers.flush();
        return offer.version();
    }
}
```

## 5. Thử nghiệm 1: Kịch bản ghi đè khi thiếu `@Version`

Trường hợp mô phỏng loại bỏ thuộc tính `version`:

```java
Future<Void> a = executor.submit(() -> {
    brokenAttempts.edit("A", new BigDecimal("90.00"));
    return null;
});
Future<Void> b = executor.submit(() -> {
    brokenAttempts.edit("B", new BigDecimal("80.00"));
    return null;
});

gate.allowA();
a.get(5, TimeUnit.SECONDS);
gate.aCommitted();
b.get(5, TimeUnit.SECONDS);

// Kết quả của B đã ghi đè hoàn toàn thay đổi của A
assertThat(price()).isEqualByComparingTo("80.00");
// Cả 2 luồng đều nhận phản hồi thành công từ hệ thống
assertThat(successfulCalls()).isEqualTo(2);
```

Bài kiểm thử này minh họa rủi ro mất bản cập nhật nếu hệ thống chỉ tập trung kiểm tra luồng trả về mà không đối chiếu trực tiếp dữ liệu sau cùng tại database.

## 6. Thử nghiệm 2: Kích hoạt xung đột bằng khóa lạc quan

```java
@Test
void secondEditorCannotOverwriteCommittedVersion() throws Exception {
    Future<Long> a = executor.submit(() ->
            attempts.edit("A", new BigDecimal("90.00"))
    );
    Future<Throwable> b = executor.submit(() -> {
        try {
            attempts.edit("B", new BigDecimal("80.00"));
            return null;
        } catch (Throwable failure) {
            return failure; // Lưu trữ ngoại lệ phát sinh
        }
    });

    gate.allowA();
    assertThat(a.get(5, TimeUnit.SECONDS)).isEqualTo(8); // A hoàn tất, version = 8
    gate.aCommitted();

    Throwable loser = b.get(5, TimeUnit.SECONDS); // Luồng B hoàn tất pha thực thi
    // Xác minh ngoại lệ khóa lạc quan được ném ra
    assertThat(loser)
            .isInstanceOf(ObjectOptimisticLockingFailureException.class);
            
    // Xác minh tính toàn vẹn của dữ liệu thuộc về transaction A
    assertThat(price()).isEqualByComparingTo("90.00");
    assertThat(version()).isEqualTo(8);
}
```

## 7. Thử nghiệm 3: Vòng đời của transaction hỏng

Một khi luồng B vướng lỗi xung đột, trạng thái của transaction bị chuyển thành `rollback-only`. Không có khả năng bắt lỗi và thử lại (catch and retry) trong cùng một transaction. Kế hoạch xử lý đòi hỏi khởi tạo transaction mới và tải lại thực thể.

```java
// Truy xuất lại thực thể từ một transaction hoàn toàn mới
OfferView reloaded = readService.get(42);
assertThat(reloaded.price()).isEqualByComparingTo("90.00");
assertThat(reloaded.version()).isEqualTo(8);
// Định danh của các transaction không được phép trùng lặp
assertThat(attemptProbe.transactionIds()).doesNotHaveDuplicates();
```

Khẳng định nguyên lý: Thao tác tiếp nối bắt buộc phải hoạt động trên nền tảng dữ liệu mới.

## 8. Thử nghiệm 4: Kiểm soát tính nhất quán tại biên ứng dụng (xác thực version phía gọi)

```java
ChangeOfferPrice stale = new ChangeOfferPrice(
        42,
        7, // Phiên bản giả định từ phía gọi (database đã là 8)
        new BigDecimal("80.00"),
        UUID.randomUUID()
);

// Hệ thống từ chối lập tức trước khi truy xuất database cập nhật
assertThatThrownBy(() -> editor.changePrice(stale))
        .isInstanceOf(StaleOfferEditException.class);
        
// Dữ liệu hợp lệ được bảo toàn
assertThat(price()).isEqualByComparingTo("90.00");
assertThat(version()).isEqualTo(8);
```

Trên kiến trúc REST API, lỗi `StaleOfferEditException` cần được ánh xạ thành `412 Precondition Failed` nhằm phản hồi trạng thái dữ liệu lỗi thời tới phía gọi.

## 9. Thử nghiệm 5: Phân tích cấu trúc lệnh SQL

Sử dụng cơ chế `StatementInspector` của Hibernate nhằm phân tích luồng truy vấn trước khi thực thi:

```java
assertThat(sqlCapture.updatesFor("product_offer"))
        .anySatisfy(sql -> {
            assertThat(sql).contains("version=?");
            assertThat(sql).contains("where offer_id=?");
            assertThat(sql).contains("and version=?"); // Mệnh đề chốt chặn bảo mật
        });
```

Tiêu chuẩn đối sánh chỉ tìm kiếm từ khóa thay vì đối chiếu chuỗi SQL hoàn chỉnh, giúp mã kiểm thử tránh tình trạng kiểm thử thiếu ổn định do sự thay đổi trong định dạng / thứ tự cột.

## 10. Thử nghiệm 6: Yêu cầu bắt buộc đối với truy vấn SQL thuần

Nếu dự án áp dụng SQL truy vấn thuần:

```sql
update product_offer
set price = 95.00,
    version = version + 1   -- Bắt buộc tích hợp gia tăng biến số
where offer_id = 42
  and version = 7;
```

Viết kiểm thử nhằm mô phỏng kết quả `affected rows = 0` (khi gán version không khớp) là nguyên tắc bắt buộc để rà soát các lỗ hổng của cơ chế tối ưu hóa không qua ORM.

## 11. Thử nghiệm 7: Xác thực khả năng mở rộng đa tiến trình

Sự hiện diện của mệnh đề `version = ?` trong database đóng vai trò cốt lõi. Bất luận kiến trúc được xử lý qua đa luồng cục bộ hay triển khai nhiều instance trên các node độc lập, trạng thái xung đột vẫn được nhận diện y hệt. Yếu tố này phủ định hoàn toàn khả năng sử dụng khóa nội bộ JVM (như `synchronized`) cho mô hình bảo vệ database phân tán.

## 12. Thử nghiệm 8: Xung đột giữa cập nhật và xóa dữ liệu

- A truy xuất v7. B thực thi lệnh XÓA (Delete).
- A hoàn tất quá trình cập nhật bộ nhớ, thực thi `flush`... và nhận về `affected rows = 0`.
- Hệ thống có thể chuyển hóa lỗi này thành xung đột lạc quan hoặc ngoại lệ không tìm thấy (tùy thuộc nghiệp vụ). Khuyến nghị không được áp dụng các biện pháp phục hồi giả lập để khởi tạo lại bản ghi.

## 13. Ma trận bao phủ trạng thái

| Kịch bản | Trạng thái Luồng A (chiến thắng) | Trạng thái Luồng B (thua cuộc) | Giá trị database sau cùng |
| --- | --- | --- | --- |
| Không cấu hình Version | Thông báo thành công | Không báo lỗi (ghi đè ngầm) | `80` (Dữ liệu luồng A bị hủy bỏ) |
| **Áp dụng `@Version`** | Cập nhật 1 bản ghi | Trả về `0` bản ghi / Ném ngoại lệ | `90 / v8` |
| Phía gọi gửi version cũ | Tích hợp thành `v8` | Từ chối ngay khi xác thực | `90 / v8` |
| Sử dụng SQL thuần | Cập nhật 1 bản ghi | Trả về `0` bản ghi | Tự động gia tăng lên 1 phiên bản |
| Sửa đổi gặp xóa bỏ | Thực thi xóa dữ liệu thành công | Xảy ra lỗi do `affected rows = 0` | Không tồn tại bản ghi |

## 14. Tiêu chuẩn chống kiểm thử thiếu ổn định

- Vận dụng `CountDownLatch` để kiểm soát nhịp đồng bộ của luồng, nghiêm cấm sử dụng `Thread.sleep` phụ thuộc vào tốc độ phần cứng.
- Áp dụng các cấu hình giới hạn thời gian (timeout) chặt chẽ cho toàn bộ các rào chắn (latch, future).
- Thiết lập kịch bản làm sạch trạng thái database vật lý (`delete from product_offer`) trong các pha khởi tạo.
- Khẳng định tính pháp lý thông qua PostgreSQL Testcontainers thay vì H2 in-memory DB.

## 15. Tiêu chuẩn cho môi trường thực tế

Khi triển khai lên hệ thống thực, hệ thống giám sát cần thu thập chỉ số tần suất xung đột, cùng số lượng trả về mã HTTP `412/409`. Một sự gia tăng đột biến về tỷ lệ lỗi khóa lạc quan sau quá trình triển khai là dấu hiệu cảnh báo nghiêm trọng về các tệp lệnh SQL thuần / migration đã vi phạm cấu trúc quản trị version.
