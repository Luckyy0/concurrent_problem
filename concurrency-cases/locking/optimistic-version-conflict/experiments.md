# Vào Phòng Thí Nghiệm: Ép Đụng Độ Khóa Lạc Quan Bằng Testcontainers

## 1. Mục tiêu Thí nghiệm

Trong bài này, chúng ta sẽ bắt tay viết code Test để chứng minh:

1. Không xài `@Version` thì DB bị ghi đè thê thảm thế nào.
2. Ép hai luồng cùng bốc lên một version, rồi bắt luồng A phải chốt sổ trước luồng B.
3. Kẻ thua cuộc (luồng B) sẽ bị văng lỗi Optimistic, sửa được `0` dòng và Giao dịch bị vứt xó.
4. Kiểm tra Dữ liệu cuối cùng dưới DB có đúng của luồng A không, chứ không phải chỉ test xem có văng lỗi là xong.
5. Soi thử xem Hibernate đẻ ra câu SQL có đuôi `version = ?` thật không.
6. Chặn đứng Client gửi cái Version ôi thiu từ ngoài cửa.

> **Nói ngắn gọn:** Đã test Khóa Lạc Quan thì phải tạo ra được cái Bẫy Chặn (gate) chính xác đến từng mi-li-giây, chứ không phải cứ ném bừa (load test) hàng trăm luồng rồi cầu trời cho nó đụng nhau đâu.

## 2. Dựng sân khấu thật với PostgreSQL Testcontainers

Đừng xài DB ảo (H2), hãy dùng đồ thật:

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

    // Dùng 2 thread để đóng vai nhân viên A và B
    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    @BeforeEach
    void seed() {
        // Dọn dẹp sân bãi
        jdbc.update("delete from product_offer");
        // Nhét dữ liệu gốc vào: Giá 100, version 7
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

Lưu ý: Không đặt `@Transactional` ở hàm Test nhé; bắt 2 luồng phải nhào xuống DB commit/rollback thật sự.

## 3. Tạo Bẫy Chặn (Gate) điều phối kịch bản

Chúng ta dùng `CountDownLatch` để bắt các luồng phải "xếp hàng" đúng ý đồ của đạo diễn:

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
        await(bothLoaded, "Bắt cả hai phải tải xong v" + version);
        
        if ("A".equals(actor)) {
            await(allowA, "Chờ lệnh mới cho A chạy tiếp");
        } else {
            await(aCommitted, "B phải há mồm chờ A chốt sổ xong mới được chạy");
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
            throw new AssertionError("Đợi quá lâu rồi (Timeout): " + description);
        }
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Bị cắt ngang: " + description, interrupted);
    }
}
```

Trên Production cái Gate này chẳng làm gì cả (no-op). Ở đây, Test chỉ gọi hàm `aCommitted()` khi và chỉ khi Lệnh A đã chốt sổ thành công xuống Database.

## 4. Dịch vụ Sửa giá thông qua Spring Proxy

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

    // Mở Giao Dịch mới tinh cho mỗi Luồng
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public long edit(String actor, BigDecimal price) {
        ProductOffer offer = offers.findById(42L).orElseThrow();
        
        // Bước chân vào cái bẫy chặn của đạo diễn
        gate.afterLoad(actor, offer.version());
        
        offer.changePrice(price);
        offers.flush();
        return offer.version();
    }
}
```

## 5. Thí nghiệm 1: Ghi đè rành rành (Không có `@Version`)

Nếu xóa cột `version` đi, chuyện gì xảy ra?

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

// B thản nhiên đạp bay công sức của A!
assertThat(price()).isEqualByComparingTo("80.00");
// Mà cả 2 lệnh lại đều báo thành công mới cay chứ!
assertThat(successfulCalls()).isEqualTo(2);
```

Đấy, nếu chỉ check lỗi mà không moi database lên soi tận mắt giá trị cuối cùng, thì bạn đã bỏ lọt Lỗi mất dữ liệu (Lost Update) rồi!

## 6. Thí nghiệm 2: Gắn `@Version` vào là Xung đột ngay

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
            return failure; // Bắt gọn cái Lỗi ném ra
        }
    });

    gate.allowA();
    assertThat(a.get(5, TimeUnit.SECONDS)).isEqualTo(8); // A sửa xong tăng lên v8
    gate.aCommitted();

    Throwable loser = b.get(5, TimeUnit.SECONDS); // Luồng B xong
    // Luồng B ăn ngay quả Lỗi mồm thúi
    assertThat(loser)
            .isInstanceOf(ObjectOptimisticLockingFailureException.class);
            
    // Và quan trọng nhất: Dữ liệu cuối cùng được bảo toàn là của A
    assertThat(price()).isEqualByComparingTo("90.00");
    assertThat(version()).isEqualTo(8);
}
```

## 7. Thí nghiệm 3: Đã Sập là Sập Luôn (Failed transaction)

Khi B vướng lỗi đụng độ, cái Giao Dịch của B coi như "chết lâm sàng" (rollback-only). Bạn không thể "hô hấp nhân tạo" cho nó bằng cách `catch` lỗi rồi gọi lệnh khác đè vào.
Muốn cứu? Mở Giao dịch MỚI, bốc dữ liệu mới từ DB lên lại.

```java
// Bốc lại dữ liệu từ một Giao dịch MỚI TINH
OfferView reloaded = readService.get(42);
assertThat(reloaded.price()).isEqualByComparingTo("90.00");
assertThat(reloaded.version()).isEqualTo(8);
// Id của các Giao dịch không được trùng nhau!
assertThat(attemptProbe.transactionIds()).doesNotHaveDuplicates();
```

Test này chỉ để khẳng định: Muốn chơi lại thì phải làm ván mới, cấm ăn gian.

## 8. Thí nghiệm 4: Sút ngay từ cổng nếu Client rinh Version thiu

```java
ChangeOfferPrice stale = new ChangeOfferPrice(
        42,
        7, // Version của Client nè, giờ DB nó là 8 cơ!
        new BigDecimal("80.00"),
        UUID.randomUUID()
);

// Sút thẳng cẳng không thương tiếc
assertThatThrownBy(() -> editor.changePrice(stale))
        .isInstanceOf(StaleOfferEditException.class);
        
// Data của A vẫn bình yên vô sự
assertThat(price()).isEqualByComparingTo("90.00");
assertThat(version()).isEqualTo(8);
```

Khi làm Controller HTTP, nhớ ánh xạ lỗi này thành `412 Precondition Failed` (Mày không đủ tư cách vì data ôi thiu) thay vì báo lỗi server nhé.

## 9. Thí nghiệm 5: Mổ xẻ câu SQL (SQL inspection)

Dùng đồ nghề `StatementInspector` của Hibernate để tóm lấy câu SQL trước khi nó chui xuống DB:

```java
assertThat(sqlCapture.updatesFor("product_offer"))
        .anySatisfy(sql -> {
            assertThat(sql).contains("version=?");
            assertThat(sql).contains("where offer_id=?");
            assertThat(sql).contains("and version=?"); // Quan trọng nhất là khúc này!
        });
```

Chỉ tìm từ khóa thôi, đừng dại mà assert nguyên xi nguyên chuỗi SQL (nhỡ Hibernate nó đổi cách thụt lề hay thứ tự cột là test lăn ra đỏ quạch).

## 10. Thí nghiệm 6: Viết SQL chay cũng phải có tình người

Nếu ông nào thích xài SQL thuần:

```sql
update product_offer
set price = 95.00,
    version = version + 1   -- Bắt buộc phải có dòng này nha mấy cha!
where offer_id = 42
  and version = 7;
```

Viết test để assert xem dòng đó trả về `1` (thành công) và mớm cái v7 thêm lần nữa để ép nó trả về `0`. Cái Test này sinh ra để trị mấy ông Tối Ưu Hóa làm hỏng hợp đồng (contract) của hệ thống.

## 11. Thí nghiệm 7: 2 Luồng = 2 Server (Multi-instance)

Code bạn chạy 2 Threads trên 1 máy, hay chạy 2 Container trên 2 máy tính khác nhau thì Kết Quả Đụng Độ Vẫn Y Chang. Lý do? Kẻ phân xử cuối cùng là cái Database bự chảng nằm dưới, nó nắm cái đuôi `version = ?`. Đó là lý do bạn ĐỪNG BAO GIỜ dùng `synchronized` để giải quyết bài toán này.

## 12. Thí nghiệm 8: Thằng thì Sửa, Thằng thì Xóa

A tải v7. Thằng B bay xuống DB gọi lệnh Xóa Mất Tích.
A tải xong loay hoay sửa rồi xả SQL xuống... Bùm! Trả về `0` dòng (affected rows `0`).
Lúc này bạn văng lỗi Khóa Lạc Quan hay văng lỗi "Data Not Found" (Không tìm thấy) đều được, tùy luật của Công ty bạn. Nhưng tuyệt đối không tự phục hồi (cứu sống) dòng bị xóa.

## 13. Bảng Tổng Kết Số Phận (Ma trận bao phủ)

| Kịch bản | Kẻ thắng (A) | Kẻ thua (B) | Giá trị DB sau cùng |
| --- | --- | --- | --- |
| Không cột Version | Cả 2 đều báo Thành Công | Chả báo lỗi gì sất | `80`, A khóc hận vì mất tích |
| **Có `@Version`** | Đổi được 1 dòng | Sửa được `0` dòng / Văng Exception | `90 / v8` |
| Bơm Client thiu | DB lưu `v8` | Đuổi về từ vòng giữ xe | `90 / v8` |
| SQL Chay thuần | Đổi được 1 dòng | Sửa được `0` dòng | Version tự nhích lên 1 |
| Sửa đụng Xóa | Lệnh Xóa bay cái roẹt | Cắn lưỡi vì lỗi | Bốc hơi khỏi DB |

## 14. Bí kíp dập tắt Flaky Test (Test lúc xanh lúc đỏ)

- Dùng `CountDownLatch` ép các luồng xếp hàng, đừng bao giờ dùng `Thread.sleep(1000)` ngu ngốc.
- Phải có Timeout cho mọi hành động đợi chờ, hết giờ là giựt sập.
- Phải nhổ cỏ dọn rác DB (`delete from product_offer`) gọn gàng sạch sẽ.
- Chơi đồ thật (PostgreSQL Testcontainers) chứ đừng tin thằng lươn lẹo H2 DB.

## 15. Áp dụng lên Đời Thật (Production)

Đưa lên môi trường thật là phải soi log. Log ra số đụng độ, trả 412/409 bao nhiêu lần. Trò đụng độ này mà bỗng nhiên tăng vọt ngay sau đợt Đẩy Code (Deploy), thì khoan chửi Client, coi chừng mấy "pháp sư" viết SQL Chay lười cộng dồn Version rồi đó!
