# Thí nghiệm tích hợp với PostgreSQL (PostgreSQL integration experiments)

## Mục tiêu

Bộ chiến dịch kiểm thử (test) này phải lôi ra ánh sáng hai tầng sự thật:

1. **Sự thật trần trụi về mặt kỹ thuật (technical fact):** lột mặt nạ mức cô lập thực tế (effective isolation) của đích thân cái luồng giao dịch đọc (reader transaction);
2. **Sự thật sứt mẻ về mặt nghiệp vụ (business fact):** việc hai lần đọc nhả ra con số chói lọi `100/120` ở cái ranh giới rách nát (broken boundary) và ngoan ngoãn ói về cặp `100/100` khi đã vào khuôn khổ ranh giới chuẩn (fixed boundary).

Chỉ chăm chăm soi mói cái vỏ (inspect annotation) hay lập bàn giả vờ (mock transaction manager) thì phỏng có ích gì. Bài test phải cưỡi trên mình con ngựa chiến PostgreSQL thật (dùng PostgreSQL thật qua Testcontainers) bởi cái đặc trưng ngữ nghĩa bản ảnh (snapshot semantics) vốn dĩ là đặc sản riêng của từng dòng họ cơ sở dữ liệu (database-specific).

> **Nói ngắn gọn:** bàn tay thao túng phải điều khiển luồng ghi (writer) ụp cái chốt (commit) vào chuẩn xác ở giữa hai khe lệnh SELECT, rà soát lại thông số cài đặt (đo setting) trên ngay chính cái đường dây (connection) của tay đọc, rồi cuối cùng mới giáng búa phán quyết kết quả (assert outcome).

## Cấu trúc bài kiểm thử (Test topology)

```text
luồng kiểm thử (test thread)
  ├─ luồng lính đọc (reader executor) -> màn chắn facade (facade proxy) -> Giao dịch Tx-R -> Lệnh SELECT #1 -> cổng chặn (gate) -> Lệnh SELECT #2
  └─ luồng kiểm thử (test thread)     -> màn chắn writer (writer proxy) -> Giao dịch Tx-W -> Lệnh UPDATE -> Lệnh COMMIT -> giật tung cổng chặn (release gate)
```

Nghiêm cấm cắm chiếc cọc `@Transactional` trên đầu phương thức kiểm thử (Test method). Nếu ngoan cố cắm bừa, lũ reader/writer sẽ tự động dắt tay nhau leo lên con thuyền giao dịch do test tạo ra (test-managed transaction) hoặc tồi tệ hơn là màn dọn dẹp kết liễu (cleanup rollback) sẽ nhẫn tâm xóa sạch sành sanh mấy dấu vết commit thật.

## Con ngựa chiến PostgreSQL (PostgreSQL container) và lược đồ (schema)

```java
@Testcontainers
@SpringBootTest
class IsolationBoundaryIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Bản vẽ dữ liệu cốt lõi nhất (Schema tối thiểu):

```sql
create table product_price (
    product_id bigint primary key,
    price bigint not null
);
```

Mỗi bài test xông xáo (Mỗi test) dập lại cơ sở dữ liệu (database) về nguyên trạng thái đã chốt sổ (committed state) trong veo:

```sql
delete from product_price;
insert into product_price(product_id, price) values (1, 100);
```

Mớ thao tác dọn dẹp/khởi tạo (Cleanup/setup) bị đuổi ra rìa ngoài giao dịch đọc (reader transaction). Rủi như có cả một bầy test chạy hối hả (parallel), hãy khéo léo phết ID nhận diện riêng cho từng đứa (từng test) hoặc tắt bẵng lệnh đua đòi chạy song song (disable parallel execution) cho ngay chính lớp này (class này).

## Cổng chặn giật dây thao túng (Gate điều phối deterministic)

Cổng chặn (Gate) bóp nắn ra một thứ tự nhào lộn điệu nghệ (partial order): `lần đọc một (first read) < luồng ghi chốt hạ (writer commit) < lần đọc hai (second read)`:

```java
@Component
final class SnapshotProbe {
    private final AtomicReference<Gate> current = new AtomicReference<>();
    private final AtomicReference<String> observedIsolation =
        new AtomicReference<>();

    Gate install() {
        Gate gate = new Gate(
            new CountDownLatch(1),
            new CountDownLatch(1)
        );
        if (!current.compareAndSet(null, gate)) {
            throw new IllegalStateException("A scenario is already active");
        }
        observedIsolation.set(null);
        return gate;
    }

    void afterFirstRead(JdbcTemplate jdbc) {
        Gate gate = requireGate();
        observedIsolation.set(jdbc.queryForObject(
            "select current_setting('transaction_isolation')",
            String.class
        ));
        gate.firstReadDone().countDown();
        awaitOrFail(gate.allowSecondRead(), Duration.ofSeconds(5));
    }

    String observedIsolation() {
        return Objects.requireNonNull(observedIsolation.get());
    }

    void clear(Gate expected) {
        if (!current.compareAndSet(expected, null)) {
            throw new IllegalStateException("Unexpected active scenario");
        }
    }

    private Gate requireGate() {
        return Objects.requireNonNull(
            current.get(),
            "No scenario installed"
        );
    }

    static void awaitOrFail(CountDownLatch latch, Duration timeout) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for scenario gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }

    record Gate(
        CountDownLatch firstReadDone,
        CountDownLatch allowSecondRead
    ) {
        void awaitFirstRead() {
            SnapshotProbe.awaitOrFail(firstReadDone, Duration.ofSeconds(5));
        }

        void releaseSecondRead() {
            allowSecondRead.countDown();
        }
    }
}
```

Mọi khoảng chờ đọa đày (Mọi wait) đều kè kè lưỡi hái (timeout). Khối lệnh `finally` dũng cảm gánh vác sứ mệnh thả phanh cổng chặn (release gate) để lũ reader không phải chịu cảnh chết trân treo lơ lửng khi tên writer ngỏm củ tỏi hay lúc lưới kiểm tra (assertion) trên luồng test khóc thét báo tử (thất bại).

## Các pháo đài dịch vụ mồi (Fixture services)

Dùng lưới JDBC truy vấn (query) đánh trực diện (trực tiếp) giúp đôi SELECT xáp lá cà thẳng thừng cạp cơ sở dữ liệu (chạm database), phá tan cái mưu hèn kế bẩn của JPA khi vác bộ đệm hạng nhất (first-level cache) nhè ra ném trả cái thực thể bị giam cầm (managed entity) cũ rích hòng che lấp (che) vỡ lở (anomaly):

```java
@Repository
class PriceQueries {
    private final JdbcTemplate jdbc;

    long getPrice(long productId) {
        return jdbc.queryForObject(
            "select price from product_price where product_id = ?",
            Long.class,
            productId
        );
    }

    long getCommittedPrice(long productId) {
        return getPrice(productId);
    }
}
```

Dịch vụ vòng trong (Inner service) hồn nhiên khoác trên mình cái vỏ bọc lừa lọc (declaration đang gây hiểu nhầm):

```java
@Service
class SnapshotQueryService {
    private final PriceQueries prices;
    private final SnapshotProbe probe;
    private final JdbcTemplate jdbc;

    @Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true)
    public PriceReport readTwice(long productId) {
        long first = prices.getPrice(productId);
        probe.afterFirstRead(jdbc);
        long second = prices.getPrice(productId);
        return new PriceReport(first, second);
    }
}
```

Đôi song sinh facade chỉ trái khuấy (khác) nhau ở cái chốt phân cấp cô lập lúc mở cửa (creation-point isolation):

```java
@Service
class BrokenReportFacade {
    private final SnapshotQueryService snapshots;

    @Transactional
    public PriceReport generate(long productId) {
        return snapshots.readTwice(productId);
    }
}
```

```java
@Service
class FixedReportFacade {
    private final SnapshotQueryService snapshots;

    @Transactional(
        isolation = Isolation.REPEATABLE_READ,
        readOnly = true
    )
    public PriceReport generate(long productId) {
        return snapshots.readTwice(productId);
    }
}
```

Kẻ nã đạn (Writer) bắt buộc phải khoác áo bean proxy tách biệt (riêng) để vung tay cập nhật (update) và chốt sổ (commit) trước khi pháo đài test rủ lòng thương phóng thích reader:

```java
@Service
class PriceWriter {
    private final JdbcTemplate jdbc;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void update(long productId, long price) {
        int changed = jdbc.update(
            "update product_price set price = ? where product_id = ?",
            price,
            productId
        );
        if (changed != 1) {
            throw new IllegalStateException("Expected exactly one product");
        }
    }
}
```

Bùa chú `REQUIRES_NEW` cài cắm ở writer chỉ sắm vai trò tuyên cáo rằng giao dịch Tx-W này hất cẳng không chung đụng (độc lập với) kẻ phát lệnh test (test caller). Đừng có nhầm tưởng đây là kế sách sửa sang lại thân thế (phương án sửa) cho reader.

## Thí nghiệm 1 — Ngõ cụt (Broken path) đích thị là hang ổ READ COMMITTED

```java
@Test
void innerRepeatableReadDoesNotUpgradeExistingDefaultTransaction()
        throws Exception {
    SnapshotProbe.Gate gate = probe.install();
    ExecutorService readerPool = Executors.newSingleThreadExecutor();

    try {
        Future<PriceReport> reader =
            readerPool.submit(() -> broken.generate(1L));

        gate.awaitFirstRead();
        writer.update(1L, 120L); // ngúng nguẩy chỉ thèm return sau khi Tx-W chịu commit
        gate.releaseSecondRead();

        PriceReport report = reader.get(5, TimeUnit.SECONDS);

        assertThat(probe.observedIsolation()).isEqualTo("read committed");
        assertThat(report.firstPrice()).isEqualTo(100L);
        assertThat(report.secondPrice()).isEqualTo(120L);
    } finally {
        gate.releaseSecondRead();
        probe.clear(gate);
        readerPool.shutdownNow();
        assertThat(readerPool.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Bài test thảm hại sụp đổ (thất bại) nếu như gã thợ implementation trót dại “nhẹ dạ cả tin” vào cái bùa chú vòng trong (inner annotation). Vệt máu `100/120` là bằng chứng đanh thép (business evidence) tố cáo gã PostgreSQL đã hì hục rặn ra (tạo) cái bản ảnh rác (statement snapshot mới) ngay lần ngoạm (đọc) thứ hai.

## Thí nghiệm 2 — Mức cô lập cắm ở ranh giới vòng ngoài (outer boundary) sẽ ban phát bản ảnh tĩnh (stable snapshot)

Vuốt lại giá trị về mức `100`, rồi lại xua quân dập lại y nguyên kịch bản giao tranh (cùng interleaving):

```java
@Test
void outerRepeatableReadKeepsBothReadsOnOneSnapshot() throws Exception {
    SnapshotProbe.Gate gate = probe.install();
    ExecutorService readerPool = Executors.newSingleThreadExecutor();

    try {
        Future<PriceReport> reader =
            readerPool.submit(() -> fixed.generate(1L));

        gate.awaitFirstRead();
        writer.update(1L, 120L);
        gate.releaseSecondRead();

        PriceReport report = reader.get(5, TimeUnit.SECONDS);

        assertThat(probe.observedIsolation()).isEqualTo("repeatable read");
        assertThat(report.firstPrice()).isEqualTo(100L);
        assertThat(report.secondPrice()).isEqualTo(100L);
        assertThat(prices.getCommittedPrice(1L)).isEqualTo(120L);
    } finally {
        gate.releaseSecondRead();
        probe.clear(gate);
        readerPool.shutdownNow();
        assertThat(readerPool.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Lưới đánh chặn cuối cùng (Assertion cuối) ung dung chạy lót đường sau khi reader thở phào (hoàn tất), nằm chễm chệ trong vùng giao dịch/câu truy vấn riêng rẽ (độc lập). Nó dõng dạc thanh minh (chứng minh) rằng cái kết quả `100/100` phơi bày ra đó hoàn toàn không phải do gã writer ngủ quên chưa commit: mà thực chất trạng thái đã an bài (committed state thật) đã kịp biến hình thành `120`, có chăng chỉ là Tx-R của chúng ta bị che mắt thiêng cứ mải mê đắm đuối vớt vát lại cái ảnh cũ rích (snapshot cũ).

## Thí nghiệm 3 — Đòn kiểm tra (Validation) hô biến sai trái (mismatch) thành trái đắng tức thời (fail-fast)

Lôi ra hẳn một góc chiến trường riêng biệt (test application context riêng) để cho gã quản lý bật lò xo soi mói (validation):

```java
@TestConfiguration
class StrictTransactionTestConfiguration {

    @Bean
    @Primary
    JpaTransactionManager strictTransactionManager(
        EntityManagerFactory entityManagerFactory
    ) {
        JpaTransactionManager manager =
            new JpaTransactionManager(entityManagerFactory);
        manager.setValidateExistingTransaction(true);
        return manager;
    }
}
```

Chiến binh test bắt buộc phải dựng lều ở một ngọn núi riêng (class/context riêng) để tuyệt đối không đầu độc (mutate) cái lò ấp (singleton) transaction manager của thiên hạ, trong khi bao trận chiến (test khác) đang nảy lửa:

```java
@Test
void incompatibleInnerIsolationIsRejectedBeforeQueryRuns() {
    assertThatThrownBy(() -> broken.generate(1L))
        .isInstanceOf(IllegalTransactionStateException.class)
        .hasMessageContaining("isolation");

    assertThat(queryCounter.value()).isZero();
}
```

Đồng hồ đếm nhịp `queryCounter` bẽn lẽn tăng giá ở bước đầu `readTwice()`. Nếu cái tên cản móp (transaction interceptor) kiên quyết cự tuyệt hất cẳng (từ chối join) từ ngoài ngõ rào (trước method body), bộ đếm kia sẽ muôn đời chôn chân ở số không (zero). Tùy lứa Spring (version), mấy dòng càm ràm chi tiết (message) có thể múa lượn (thay đổi); thế nhưng cái chủng loại oán thán (type) và cái thực tại phũ phàng thân xác (body) không được ban ơn (chạy) mới là tấm khiên bất hoại (assertions bền hơn).

## Thí nghiệm 4 — Quái chiêu tự kỷ (Self-invocation) bất tài trong việc dựng ranh giới mong ước (intended boundary)

Bày thêm một đạo bùa chú (fixture riêng) rình mò đoạt lấy trạng thái giao dịch lẩn khuất trong ngõ ngách tự ru ngủ (self-invoked helper):

```java
@Service
class SelfInvokingReport {

    PriceReport generate(long productId) {
        return this.readWithStableSnapshot(productId);
    }

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    PriceReport readWithStableSnapshot(long productId) {
        assertThat(TransactionSynchronizationManager
            .isActualTransactionActive()).isFalse();
        // rình mò (probe) hai lệnh repository hì hục cày cuốc trong trận thử nghiệm tổng (full test)
        return null;
    }
}
```

Chớ dại cài cắm bẫy rập trên bãi mìn thật (Không đặt assertion production) như trong cái vở diễn trên; ở bãi chiến trường thực (trong fixture thật), cứ tống khứ cái cục nợ trạng thái đó thông qua đám đầu dò (probe) rồi hẵng vác gậy ra phán xét (assert) từ phía chiến tuyến (test). Bài diễn (Experiment) này vớt vát thêm sắc màu cho kịch bản quái thai SPR-001 và không ôm mộng soán ngôi (không thay thế) hai màn phô diễn kết quả (outcome tests) của ngài PostgreSQL.

## Thí nghiệm 5 — Khái niệm DEFAULT phải cong đuôi đo đạc theo nhịp thở môi trường (environment)

Khơi mào một ngọn đuốc soi sáng công khai (public probe service):

```java
@Transactional
public String defaultIsolation() {
    return jdbc.queryForObject(
        "select current_setting('transaction_isolation')",
        String.class
    );
}
```

Cầm gậy đập móp (Assert) cái vỏ thùng chứa (container profile hiện tại) phải nôn ra đúng chữ `read committed`. Nhược bằng bãi chiến trường (production) có trót dại lén nhồi cấu hình hắc ám (default khác), lo mà tạc hình nặn tượng (tạo profile/container config) cho tương xứng và đập búa khẳng định đinh ninh (assert explicit). Cấm tiệt chuyện lén lút tráo đổi (Không đổi) cái ruột (global/session default) qua mặt giữa đám chiến binh đánh chéo cánh (concurrent tests) vì rắp tâm thay áo của bọn hồ mượn chứa (connection pool) có thể xảo quyệt luồn lách phô bày (làm kết quả) kết tủa lệ thuộc hèn mọn bám vào (phụ thuộc) từng cọng đường truyền mong manh (connection) được xí phần (mượn).

## Tấm bản đồ giăng bẫy (Coverage matrix)

| Sân khấu (Scenario) | Lời sấm truyền cô lập thực (Expected effective isolation) | Quả báo mong đợi (Expected outcome) |
| --- | --- | --- |
| Vòng ngoài DEFAULT, lồng vòng trong REQUIRED RR | `read committed` | `100/120` |
| Vòng ngoài RR, lồng vòng trong REQUIRED RR | `repeatable read` | `100/100` |
| Vòng ngoài RC, lồng vòng trong RR, mắt thần (validation) mở | Không dám ho he query (chạy query) | Hộc máu đứt bóng sớm (Fail-fast) |
| Trống trơn không có outer Tx, thọc sườn inner RR qua proxy | `repeatable read` | `100/100` |
| Trống trơn không có outer Tx, lén lút (self-invoked) inner RR | Chẳng bói đâu ra (Không có) service Tx theo đúng ý đồ (intended) | Sập rào chắn vỡ mộng (Boundary test fails) |
| Vòng ngoài RC, lồng vòng trong vỗ ngực REQUIRES_NEW RR | Lõi trong cùng (Inner) lên `repeatable read` | Ngai vàng riêng biệt (Independent) `100/100` |

## Mẹo trừ tà "hên xui vỡ lở" (Chống flaky)

- Nhất nhất bọn dây thòng lọng (latch), bóng ma (future) hay đám đao phủ (executor termination) đều phải giắt theo bùa đếm ngược mốc (timeout).
- Lời thề `finally` phải nhắm mắt thả rông (release) người đọc (reader) trước khi dội bom (shutdown executor).
- Võ công của kẻ ghi (Writer method) chỉ được thu tay (return) lúc đã niêm phong chốt hạ (sau commit).
- Tuyệt đối cấm tiệt bám víu thời gian rùa bò (Không dùng delay theo thời gian) hòng thầy bói xem voi trò đan lồng ghép nhịp (đoán interleaving).
- Không rỗi hơi tống (chạy) phương thức kiểm tra vào lồng giam (giao dịch).
- Lột sạch ảo mộng vay mượn H2 hòng che lấp (đại diện) sự thâm thúy của cỗ xe tăng PostgreSQL MVCC.
- Vứt bỏ thói quen xài ké (Không reuse) mớ cổng chắn (gate) dơ dáy giữa hai cuộc chiến (hai tests).
- Trang bị áo giáp xịn dọn mâm sạch (Dùng committed fixture setup) cùng số hiệu hàng hóa mã khóa (product ID riêng) khi rượt đánh giáp lá cà (chạy parallel).
- Rủi thay bóng ma quá lố (future timeout), nhanh tay tốc ký (log) cái mặt mốc của cửa chặn (gate state) và hình hài mức độ cô lập bắt tại trận (effective isolation đã quan sát).

## Chùy sắt phán xét sống còn (Assertions bắt buộc)

Đem một bài test lèo tèo ra phán (chỉ assert) `read committed` coi bộ sẽ sập bẫy lọt khe rớt lề tàn tích dơ bẩn (query/cache behavior). Cái kiểu chọc ngoáy (chỉ assert) mỗi bãi mìn `100/120` thì có khi hân hoan ăn mừng (pass) bởi cái ranh giới ất ơ (boundary khác) nào đó đã vô phúc đứt gánh (tình cờ bị tách). Binh đoàn thép trọn vẹn (Bộ test hoàn chỉnh) phải giáng đòn chí mạng (assert):

```text
vỡ nát (broken): cốt lõi (effective) = read committed, thành quả (values) = 100/120
tái thiết (fixed): cốt lõi (effective) = repeatable read, thành quả (values) = 100/100
hậu trường (after): thành quả biệt lập an bài (independently committed value) = 120
lưới thép (strict): bức tường nội tại ngang phè phè (incompatible inner boundary) đứt bóng tức thì trước khi nã súng (rejected before method body)
```

Tấm lá chắn (contract) đầy uy quyền này đủ sức gông cổ bóp chết thói ngạo mạn (bắt regression) mỗi khi mấy lá bùa (annotation), chiêu truyền máu (propagation), khuôn khổ (datasource default) hay não bộ quan tòa (transaction manager configuration) dở chứng (thay đổi).
