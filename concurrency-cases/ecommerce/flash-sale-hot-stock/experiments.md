# Thực nghiệm tranh chấp tồn kho nóng

## 1. Mục tiêu

Bộ kiểm thử không cố tạo một benchmark dùng chung. Nó chứng minh các quan hệ
nhân quả sau:

1. Câu cập nhật có điều kiện vẫn bảo vệ tồn kho dưới tải đồng thời.
2. Nhiều giao dịch chờ cùng một dòng có thể chiếm hết vùng kết nối.
3. Khi vùng kết nối đầy, một truy vấn không liên quan cũng phải chờ.
4. Cổng điều tiết từ chối yêu cầu trước khi chúng lấy kết nối.
5. Hết thời gian chờ khóa được trả như lỗi kỹ thuật, không phải hết hàng.
6. Giới hạn cục bộ của hai máy chủ không tạo một giới hạn chung.
7. Gợi ý hết hàng phải gắn với phiên bản chiến dịch.

Các bài về khóa chạy với PostgreSQL thật qua Testcontainers. Kích thước vùng kết
nối nhỏ trong fixture chỉ nhằm tạo trạng thái có thể quan sát một cách tất định;
nó không phải cấu hình khuyến nghị cho môi trường thật.

## 2. Cấu hình kiểm thử

```java
@Testcontainers(disabledWithoutDocker = true)
@SpringBootTest
class FlashSaleHotStockIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17-alpine");

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add(
                "spring.datasource.url",
                POSTGRES::getJdbcUrl
        );
        registry.add(
                "spring.datasource.username",
                POSTGRES::getUsername
        );
        registry.add(
                "spring.datasource.password",
                POSTGRES::getPassword
        );

        // Giá trị nhỏ chỉ phục vụ việc tái hiện trạng thái cạn vùng kết nối.
        registry.add(
                "spring.datasource.hikari.maximum-pool-size",
                () -> "4"
        );
        registry.add(
                "spring.datasource.hikari.minimum-idle",
                () -> "4"
        );
        registry.add(
                "spring.datasource.hikari.connection-timeout",
                () -> "1000"
        );
        registry.add(
                "spring.jpa.hibernate.ddl-auto",
                () -> "validate"
        );

        registry.add(
                "app.flash-sale.concurrent-requests-by-product[77]",
                () -> "1"
        );
        registry.add(
                "app.flash-sale.lock-timeout",
                () -> "200ms"
        );
        registry.add(
                "app.flash-sale.statement-timeout",
                () -> "1s"
        );
        registry.add(
                "app.flash-sale.request-budget",
                () -> "2s"
        );
    }

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Autowired
    private NamedParameterJdbcTemplate jdbc;

    @Autowired
    private DataSource dataSource;

    @Autowired
    private InventoryStockDao stock;

    @Autowired
    private FlashSaleReservationTx worker;

    @Autowired
    private FlashSaleReservationFacade facade;

    @Autowired
    private HotProductAdmissionGate admission;

    @Autowired
    private SoldOutHint soldOutHint;

    private HikariDataSource hikari;
    private JdbcTemplate observer;
    private ExecutorService executor;

    @BeforeEach
    void reset() throws SQLException {
        hikari = dataSource.unwrap(HikariDataSource.class);
        executor = Executors.newFixedThreadPool(20);

        DriverManagerDataSource observerDataSource =
                new DriverManagerDataSource(
                        POSTGRES.getJdbcUrl(),
                        POSTGRES.getUsername(),
                        POSTGRES.getPassword()
                );
        observer = new JdbcTemplate(observerDataSource);

        jdbc.update(
                "DELETE FROM inventory_reservation",
                Map.of()
        );
        jdbc.update("""
                INSERT INTO inventory_item (
                    product_id,
                    on_hand_quantity,
                    available_quantity,
                    reserved_quantity,
                    version
                ) VALUES (77, 10, 10, 0, 0)
                ON CONFLICT (product_id) DO UPDATE
                SET on_hand_quantity = EXCLUDED.on_hand_quantity,
                    available_quantity = EXCLUDED.available_quantity,
                    reserved_quantity = EXCLUDED.reserved_quantity,
                    version = EXCLUDED.version
                """, Map.of());

        soldOutHint.openCampaign(new FlashSaleKey(77L, 1L));
    }

    @AfterEach
    void shutdown() throws InterruptedException {
        executor.shutdownNow();
        assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Kết nối quan sát không đi qua HikariCP của ứng dụng. Nhờ đó bài kiểm thử vẫn đọc
được `pg_stat_activity` ngay cả khi toàn bộ kết nối ứng dụng đang bị chiếm.

## 3. Hàm hỗ trợ

```java
private <T> T inReadCommitted(Supplier<T> work) {
    TransactionTemplate template =
            new TransactionTemplate(transactionManager);
    template.setIsolationLevel(
            TransactionDefinition.ISOLATION_READ_COMMITTED
    );
    return template.execute(status -> work.get());
}

private void setApplicationName(String value) {
    jdbc.queryForObject("""
            SELECT set_config('application_name', :value, true)
            """, Map.of("value", value), String.class);
}

private void await(CountDownLatch latch) {
    try {
        assertTrue(latch.await(5, TimeUnit.SECONDS));
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError(interrupted);
    }
}

private void awaitCondition(
        BooleanSupplier condition,
        String failureMessage
) {
    long deadline = System.nanoTime()
            + Duration.ofSeconds(5).toNanos();

    while (System.nanoTime() < deadline) {
        if (condition.getAsBoolean()) {
            return;
        }
        LockSupport.parkNanos(
                Duration.ofMillis(10).toNanos()
        );
        if (Thread.currentThread().isInterrupted()) {
            throw new AssertionError(
                    "interrupted while waiting for condition"
            );
        }
    }
    fail(failureMessage);
}

private int lockWaiters(String prefix) {
    Integer count = observer.queryForObject("""
            SELECT count(*)
            FROM pg_stat_activity
            WHERE application_name LIKE ?
              AND wait_event_type = 'Lock'
              AND cardinality(pg_blocking_pids(pid)) > 0
            """, Integer.class, prefix + "%");
    return Objects.requireNonNull(count);
}

private String findSqlState(Throwable failure) {
    for (Throwable current = failure;
         current != null;
         current = current.getCause()) {
        if (current instanceof SQLException sql) {
            return sql.getSQLState();
        }
    }
    return null;
}

private ReserveStock command(String label, int quantity) {
    UUID orderId = UUID.nameUUIDFromBytes(
            label.getBytes(StandardCharsets.UTF_8)
    );
    return new ReserveStock(
            orderId,
            new FlashSaleKey(77L, 1L),
            quantity
    );
}

private int available() {
    return jdbc.queryForObject("""
            SELECT available_quantity
            FROM inventory_item
            WHERE product_id = 77
            """, Map.of(), Integer.class);
}

private int reservedCounter() {
    return jdbc.queryForObject("""
            SELECT reserved_quantity
            FROM inventory_item
            WHERE product_id = 77
            """, Map.of(), Integer.class);
}

private int reservedRowsQuantity() {
    return jdbc.queryForObject("""
            SELECT COALESCE(SUM(quantity), 0)
            FROM inventory_reservation
            WHERE product_id = 77
              AND status = 'RESERVED'
            """, Map.of(), Integer.class);
}

private FlashSaleLimits testLimits(Map<Long, Integer> limits) {
    return new FlashSaleLimits(
            limits,
            Duration.ofMillis(100),
            Duration.ofMillis(500),
            Duration.ofSeconds(2)
    );
}

private <T> T executeTechnicalRetry(Supplier<T> operation) {
    for (int attempt = 1; attempt <= 2; attempt++) {
        try {
            return operation.get();
        } catch (CannotAcquireLockException failure) {
            if (attempt == 2) {
                throw failure;
            }
        }
    }
    throw new IllegalStateException("unreachable");
}

private PoolSaturation saturatePoolWithHotProduct()
        throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() ->
            inReadCommitted(() -> {
                setApplicationName("ecom002-pool-holder");
                stock.tryReserve(77, 1).orElseThrow();
                holderUpdated.countDown();
                await(releaseHolder);
                return null;
            })
    );
    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    List<Future<Optional<StockAfterReserve>>> waiters =
            new ArrayList<>();
    for (int index = 0; index < 3; index++) {
        int waiterIndex = index;
        waiters.add(executor.submit(() ->
                inReadCommitted(() -> {
                    setApplicationName(
                            "ecom002-pool-waiter-" + waiterIndex
                    );
                    return stock.tryReserve(77, 1);
                })
        ));
    }

    try {
        awaitCondition(
                () -> lockWaiters("ecom002-pool-waiter-") == 3,
                "pool was not saturated by lock waiters"
        );
        return new PoolSaturation(
                releaseHolder,
                holder,
                List.copyOf(waiters)
        );
    } catch (RuntimeException | Error failure) {
        releaseHolder.countDown();
        throw failure;
    }
}

private record PoolSaturation(
        CountDownLatch releaseHolder,
        Future<Void> holder,
        List<Future<Optional<StockAfterReserve>>> waiters
) {
    void release() {
        releaseHolder.countDown();
    }

    void awaitCompletion() throws Exception {
        assertNull(holder.get(5, TimeUnit.SECONDS));
        for (Future<Optional<StockAfterReserve>> waiter : waiters) {
            assertTrue(waiter.get(5, TimeUnit.SECONDS).isPresent());
        }
    }
}
```

Không dùng `Thread.sleep()` để quyết định một giao dịch đã lấy khóa hoặc một luồng
đã chờ kết nối. Bài kiểm thử quan sát trạng thái thật và mọi vòng chờ đều có thời
hạn.

## 4. Cập nhật có điều kiện vẫn giữ đúng bất biến

```java
@Test
void atomicWorkerNeverReservesMoreThanAvailable() throws Exception {
    int actors = 16;
    CountDownLatch ready = new CountDownLatch(actors);
    CountDownLatch start = new CountDownLatch(1);
    List<Future<ReserveResult>> futures = new ArrayList<>();

    for (int index = 0; index < actors; index++) {
        ReserveStock command = command("buyer-" + index, 1);
        futures.add(executor.submit(() -> {
            ready.countDown();
            assertTrue(start.await(5, TimeUnit.SECONDS));
            return worker.reserve(command);
        }));
    }

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    List<ReserveResult> results = new ArrayList<>();
    for (Future<ReserveResult> future : futures) {
        results.add(future.get(10, TimeUnit.SECONDS));
    }

    long reserved = results.stream()
            .filter(result -> result.outcome()
                    == ReserveOutcome.RESERVED)
            .count();
    long outOfStock = results.stream()
            .filter(result -> result.outcome()
                    == ReserveOutcome.OUT_OF_STOCK)
            .count();

    assertEquals(10, reserved);
    assertEquals(actors - reserved, outOfStock);
    assertEquals(0, available());
    assertEquals(10, reservedCounter());
    assertEquals(10, reservedRowsQuantity());
}
```

Bài này xác nhận lớp đúng đắn trước khi kiểm tra hành vi tài nguyên. Nó không kết
luận hệ thống có thể phục vụ một lượng yêu cầu sản xuất cụ thể.

## 5. Hàng đợi khóa chiếm toàn bộ vùng kết nối

Fixture có bốn kết nối. Một giao dịch giữ khóa và ba giao dịch khác chờ đúng dòng
đó sẽ chiếm hết vùng kết nối:

```java
@Test
void hotRowLockQueueOccupiesApplicationPool() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() ->
            inReadCommitted(() -> {
                setApplicationName("ecom002-holder");
                stock.tryReserve(77, 1).orElseThrow();
                holderUpdated.countDown();
                await(releaseHolder);
                return null;
            })
    );

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    List<Future<Optional<StockAfterReserve>>> waiters =
            new ArrayList<>();
    for (int index = 0; index < 3; index++) {
        int waiterIndex = index;
        waiters.add(executor.submit(() ->
                inReadCommitted(() -> {
                    setApplicationName(
                            "ecom002-waiter-" + waiterIndex
                    );
                    return stock.tryReserve(77, 1);
                })
        ));
    }

    try {
        awaitCondition(
                () -> lockWaiters("ecom002-waiter-") == 3,
                "three database lock waiters were not observed"
        );
        assertEquals(
                4,
                hikari.getHikariPoolMXBean()
                        .getActiveConnections()
        );
    } finally {
        releaseHolder.countDown();
    }

    assertNull(holder.get(5, TimeUnit.SECONDS));
    for (Future<Optional<StockAfterReserve>> waiter : waiters) {
        assertTrue(waiter.get(5, TimeUnit.SECONDS).isPresent());
    }
}
```

Giá trị cần chứng minh là trạng thái tài nguyên: một giao dịch làm việc, ba giao
dịch chờ và bốn kết nối đang bị chiếm.

## 6. Truy vấn không liên quan cũng phải chờ kết nối

Bài sau mở lại cùng bố trí nhưng gửi thêm một truy vấn không truy cập sản phẩm 77:

```java
@Test
void unrelatedQueryWaitsWhenHotStockConsumesPool() throws Exception {
    PoolSaturation saturation = saturatePoolWithHotProduct();

    Future<Integer> unrelated = executor.submit(() ->
            jdbc.queryForObject(
                    "SELECT 1",
                    Map.of(),
                    Integer.class
            )
    );

    try {
        awaitCondition(
                () -> hikari.getHikariPoolMXBean()
                        .getThreadsAwaitingConnection() > 0,
                "unrelated query did not wait for a connection"
        );
        assertThrows(
                TimeoutException.class,
                () -> unrelated.get(100, TimeUnit.MILLISECONDS)
        );
    } finally {
        saturation.release();
    }

    assertEquals(1, unrelated.get(5, TimeUnit.SECONDS));
    saturation.awaitCompletion();
}
```

`PoolSaturation` là lớp hỗ trợ đóng gói đúng bố trí của bài trước và luôn giải phóng
latch trong `finally`. Bài kiểm thử không khẳng định thời gian phản hồi cụ thể;
nó chỉ chứng minh truy vấn độc lập bị buộc phải chờ cùng vùng kết nối.

## 7. Cổng điều tiết từ chối trước khi tạo hàng đợi khóa

Giữ quyền duy nhất của sản phẩm 77 và một giao dịch cơ sở dữ liệu. Các yêu cầu
khác đi qua facade phải nhận `BUSY` mà không lấy kết nối:

```java
@Test
void admissionGateRejectsBeforeDatabasePool() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch releaseHolder = new CountDownLatch(1);

    Future<Void> admitted = executor.submit(() -> {
        try (HotProductAdmissionGate.Permit ignored =
                     admission.tryEnter(77).orElseThrow()) {
            return inReadCommitted(() -> {
                setApplicationName("ecom002-admitted");
                stock.tryReserve(77, 1).orElseThrow();
                holderUpdated.countDown();
                await(releaseHolder);
                return null;
            });
        }
    });

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    List<Future<ReserveResult>> rejected = new ArrayList<>();
    for (int index = 0; index < 8; index++) {
        ReserveStock command = command("rejected-" + index, 1);
        rejected.add(executor.submit(() -> facade.reserve(command)));
    }

    try {
        for (Future<ReserveResult> result : rejected) {
            assertEquals(
                    ReserveOutcome.BUSY,
                    result.get(5, TimeUnit.SECONDS).outcome()
            );
        }
        assertEquals(1, hikari.getHikariPoolMXBean()
                .getActiveConnections());
        assertEquals(0, lockWaiters("ecom002-%"));
    } finally {
        releaseHolder.countDown();
    }

    assertNull(admitted.get(5, TimeUnit.SECONDS));
}
```

Case này cố ý trả bận thay vì xếp hàng trong ứng dụng. Nếu sản phẩm yêu cầu chờ,
cần dùng một hàng đợi có giới hạn và hợp đồng phản hồi khác.

## 8. Hết thời gian chờ khóa là lỗi kỹ thuật

```java
@Test
void lockTimeoutIsNotMappedToOutOfStock() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() ->
            inReadCommitted(() -> {
                stock.tryReserve(77, 1).orElseThrow();
                holderUpdated.countDown();
                await(releaseHolder);
                return null;
            })
    );

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    Future<Optional<StockAfterReserve>> contender =
            executor.submit(() -> inReadCommitted(() -> {
                jdbc.queryForObject("""
                        SELECT set_config(
                            'lock_timeout',
                            '100ms',
                            true
                        )
                        """, Map.of(), String.class);
                return stock.tryReserve(77, 1);
            }));

    try {
        ExecutionException failure = assertThrows(
                ExecutionException.class,
                () -> contender.get(5, TimeUnit.SECONDS)
        );
        assertEquals("55P03", findSqlState(failure));
    } finally {
        releaseHolder.countDown();
    }

    assertNull(holder.get(5, TimeUnit.SECONDS));
}
```

Nếu lớp API đổi ngoại lệ này thành `OUT_OF_STOCK`, bài kiểm tra hợp đồng phải thất
bại. Hết thời gian chờ cho biết hệ thống không lấy được khóa trong ngân sách, chứ
không cho biết số lượng hiện tại.

## 9. Giới hạn cục bộ không phải giới hạn toàn cụm

```java
@Test
void twoApplicationNodesEachOwnTheirPermit() {
    FlashSaleLimits limits = testLimits(Map.of(77L, 1));
    HotProductAdmissionGate nodeA =
            new HotProductAdmissionGate(limits);
    HotProductAdmissionGate nodeB =
            new HotProductAdmissionGate(limits);

    try (HotProductAdmissionGate.Permit permitA =
                 nodeA.tryEnter(77).orElseThrow();
         HotProductAdmissionGate.Permit permitB =
                 nodeB.tryEnter(77).orElseThrow()) {

        assertTrue(nodeA.tryEnter(77).isEmpty());
        assertTrue(nodeB.tryEnter(77).isEmpty());
        // Hai node vẫn đang có tổng cộng hai quyền đã cấp.
    }
}
```

Bài này ngăn tài liệu hoặc tên lớp mô tả `Semaphore` như một giới hạn toàn cụm.
Khi tăng số máy chủ, ngân sách tổng phải được tính lại.

## 10. Gợi ý hết hàng phải phân biệt chiến dịch

```java
@Test
void soldOutHintDoesNotLeakIntoNewCampaign() {
    FlashSaleKey firstCampaign = new FlashSaleKey(77L, 1L);
    FlashSaleKey nextCampaign = new FlashSaleKey(77L, 2L);

    soldOutHint.markSoldOut(firstCampaign);

    assertTrue(soldOutHint.isSoldOut(firstCampaign));
    assertFalse(soldOutHint.isSoldOut(nextCampaign));

    soldOutHint.openCampaign(firstCampaign);
    assertFalse(soldOutHint.isSoldOut(firstCampaign));
}
```

Gợi ý chỉ giảm tải. Bài kiểm thử tồn kho vẫn phải chạy qua PostgreSQL để chứng
minh bất biến; không kiểm tra tính đúng đắn bằng bản đồ trong bộ nhớ.

## 11. Không thử lại kết quả hết hàng

```java
@Test
void outOfStockIsReturnedWithoutRetry() {
    jdbc.update("""
            UPDATE inventory_item
            SET on_hand_quantity = 0,
                available_quantity = 0,
                reserved_quantity = 0
            WHERE product_id = 77
            """, Map.of());

    AtomicInteger attempts = new AtomicInteger();
    Supplier<ReserveResult> operation = () -> {
        attempts.incrementAndGet();
        return worker.reserve(command("sold-out", 1));
    };

    ReserveResult result = executeTechnicalRetry(operation);

    assertEquals(ReserveOutcome.OUT_OF_STOCK, result.outcome());
    assertEquals(1, attempts.get());
}
```

Hàm hỗ trợ trong test chỉ thử lại ngoại lệ kỹ thuật đã phân loại. Một kết quả
nghiệp vụ bình thường được trả ngay.

## 12. Không dùng khẳng định hiệu năng tuyệt đối

Những khẳng định sau không phù hợp với kiểm thử tích hợp thông thường:

```java
// Không ổn định giữa máy cá nhân và CI:
assertTrue(duration.toMillis() < 50);

// Không có cơ sở dùng chung cho mọi hạ tầng:
assertEquals(10_000, requestsPerSecond);
```

Thay vào đó, kiểm tra trạng thái và quan hệ:

- có bao nhiêu kết nối đang hoạt động;
- có luồng chờ lấy kết nối hay không;
- có phiên PostgreSQL chờ khóa hay không;
- yêu cầu bị từ chối trước hay sau khi lấy kết nối;
- số lần thực thi có lớn hơn số yêu cầu gốc hay không;
- bất biến tồn kho có còn đúng hay không.

Phép đo độ trễ và thông lượng vẫn cần thiết, nhưng phải chạy như bài kiểm thử tải
riêng trên hạ tầng đã biết và chỉ dùng kết quả của chính môi trường đó.

## 13. Ma trận độ phủ

| Thực nghiệm | Điều được chứng minh |
| --- | --- |
| Cập nhật đồng thời | Tồn kho vẫn đúng trước khi tối ưu tải |
| Lấp đầy vùng kết nối | Giao dịch chờ khóa vẫn chiếm kết nối |
| Truy vấn độc lập | Tải nóng lan sang chức năng dùng chung vùng kết nối |
| Cổng điều tiết | Yêu cầu vượt giới hạn bị chặn trước PostgreSQL |
| Hết thời gian chờ khóa | `55P03` khác với hết hàng |
| Hai cổng cục bộ | Giới hạn mỗi máy không phải giới hạn toàn cụm |
| Phiên bản chiến dịch | Gợi ý hết hàng không rò sang lần mở bán mới |
| Chính sách thử lại | Hết hàng không tạo thêm lần thực thi |

## 14. Quan sát trong môi trường thật

Truy vấn sau cho biết phiên nào đang chờ khóa và ai đang chặn chúng:

```sql
SELECT application_name,
       pid,
       state,
       wait_event_type,
       wait_event,
       pg_blocking_pids(pid) AS blocking_pids,
       clock_timestamp() - xact_start AS transaction_age,
       query
FROM pg_stat_activity
WHERE datname = current_database()
  AND state <> 'idle';
```

Cần ghép dữ liệu PostgreSQL với các chỉ số ứng dụng:

- yêu cầu vào, được nhận, bị báo bận, hết hàng và thành công;
- kết nối đang dùng, nhàn rỗi và luồng chờ kết nối;
- thời gian lấy kết nối, chờ khóa, chạy câu lệnh và toàn bộ giao dịch;
- số lần thử trên mỗi mã yêu cầu;
- độ trễ của API không thuộc đợt bán;
- lượng tồn đọng và tuổi yêu cầu lâu nhất nếu dùng hàng đợi.

## 15. Quy tắc tránh kiểm thử chập chờn

- Dùng latch để giữ giao dịch tại đúng điểm sau khi đã cập nhật.
- Dùng kết nối quan sát riêng để không phụ thuộc vùng kết nối đang kiểm tra.
- Xác nhận `pg_stat_activity` trước khi kiểm tra số kết nối.
- Mọi latch, future và vòng thăm dò đều có thời hạn.
- Giải phóng giao dịch giữ khóa trong `finally`.
- Không so sánh độ trễ tuyệt đối trong bài tích hợp.
- Đặt tên `application_name` để truy vết đúng phiên kiểm thử.
- Kiểm tra cả tài nguyên lẫn bất biến dữ liệu cuối cùng.

Nguyên tắc nền được trình bày tại
[tài liệu kiểm thử đồng thời](../../concepts/concurrency-testing.md).
