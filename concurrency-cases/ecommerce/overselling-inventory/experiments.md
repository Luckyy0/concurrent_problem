# Thực nghiệm bán vượt tồn kho

## 1. Mục tiêu

Bộ kiểm thử cần chứng minh các hành vi sau:

1. Luồng đọc, kiểm tra rồi ghi có thể tạo hai lần giữ hàng từ cùng một số lượng.
2. Cập nhật có điều kiện chỉ cho phép đúng một bên thắng khi hàng không đủ cho cả
   hai.
3. Giao dịch đang chờ kiểm tra lại điều kiện sau khi bên giữ khóa chốt.
4. Nếu bên giữ khóa hoàn tác, bên chờ có thể tiếp tục trên dữ liệu ban đầu.
5. Lỗi ở bước chèn bản ghi giữ hàng làm hoàn tác thay đổi tồn kho.
6. Hết thời gian chờ khóa là lỗi kỹ thuật, không phải hết hàng.
7. Dưới nhiều yêu cầu đồng thời, các bất biến cuối cùng vẫn đúng.

Các bài liên quan đến MVCC và khóa phải chạy với PostgreSQL thật qua
Testcontainers. H2 không mô phỏng đủ hành vi cần kiểm chứng.

## 2. Cấu hình Testcontainers

```java
@Testcontainers(disabledWithoutDocker = true)
@SpringBootTest
class OversellingInventoryIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17-alpine");

    @DynamicPropertySource
    static void datasource(DynamicPropertyRegistry registry) {
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
        registry.add(
                "spring.datasource.hikari.maximum-pool-size",
                () -> "12"
        );
        registry.add(
                "spring.jpa.hibernate.ddl-auto",
                () -> "validate"
        );
    }

    @Autowired
    private PlatformTransactionManager transactionManager;

    @Autowired
    private NamedParameterJdbcTemplate jdbc;

    @Autowired
    private InventoryStockDao stock;

    @Autowired
    private InventoryReservationTx reservationTx;

    @Autowired
    private BrokenInventoryWithBarrier broken;

    @PersistenceContext
    private EntityManager entityManager;

    private ExecutorService executor;

    @BeforeEach
    void resetData() {
        executor = Executors.newFixedThreadPool(20);

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
                ) VALUES (77, 5, 5, 0, 0)
                ON CONFLICT (product_id) DO UPDATE
                SET on_hand_quantity = EXCLUDED.on_hand_quantity,
                    available_quantity = EXCLUDED.available_quantity,
                    reserved_quantity = EXCLUDED.reserved_quantity,
                    version = EXCLUDED.version
                """, Map.of());
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Mỗi bài kiểm thử xóa lịch sử và đưa sản phẩm 77 về trạng thái cố định. Tất cả thao
tác chờ đều có thời hạn để một lỗi bế tắc không làm treo bộ kiểm thử.

## 3. Hàm hỗ trợ giao dịch và quan sát khóa

```java
private <T> T inReadCommitted(Supplier<T> work) {
    TransactionTemplate template =
            new TransactionTemplate(transactionManager);
    template.setIsolationLevel(
            TransactionDefinition.ISOLATION_READ_COMMITTED
    );
    return template.execute(status -> work.get());
}

private void await(CountDownLatch latch) {
    try {
        assertTrue(latch.await(5, TimeUnit.SECONDS));
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError(interrupted);
    }
}

private void setApplicationName(String value) {
    jdbc.queryForObject("""
            SELECT set_config('application_name', :value, true)
            """, Map.of("value", value), String.class);
}

private void awaitDatabaseBlock(String applicationName) {
    long deadline = System.nanoTime()
            + Duration.ofSeconds(5).toNanos();

    while (System.nanoTime() < deadline) {
        Boolean blocked = jdbc.queryForObject("""
                SELECT EXISTS (
                    SELECT 1
                    FROM pg_stat_activity
                    WHERE application_name = :applicationName
                      AND wait_event_type = 'Lock'
                      AND cardinality(pg_blocking_pids(pid)) > 0
                )
                """, Map.of(
                "applicationName", applicationName
        ), Boolean.class);

        if (Boolean.TRUE.equals(blocked)) {
            return;
        }

        LockSupport.parkNanos(
                Duration.ofMillis(10).toNanos()
        );
        if (Thread.currentThread().isInterrupted()) {
            throw new AssertionError(
                    "interrupted while observing lock wait"
            );
        }
    }
    fail("database lock wait was not observed");
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
    return new ReserveStock(orderId, 77L, quantity);
}

private int available(long productId) {
    return jdbc.queryForObject("""
            SELECT available_quantity
            FROM inventory_item
            WHERE product_id = :productId
            """, Map.of("productId", productId), Integer.class);
}

private int reservedCounter(long productId) {
    return jdbc.queryForObject("""
            SELECT reserved_quantity
            FROM inventory_item
            WHERE product_id = :productId
            """, Map.of("productId", productId), Integer.class);
}

private long version(long productId) {
    return jdbc.queryForObject("""
            SELECT version
            FROM inventory_item
            WHERE product_id = :productId
            """, Map.of("productId", productId), Long.class);
}

private int reservedRowsQuantity(long productId) {
    return jdbc.queryForObject("""
            SELECT COALESCE(SUM(quantity), 0)
            FROM inventory_reservation
            WHERE product_id = :productId
              AND status = 'RESERVED'
            """, Map.of("productId", productId), Integer.class);
}

private boolean inventoryMatchesReservations(long productId) {
    return reservedCounter(productId)
            == reservedRowsQuantity(productId);
}

private void setInventory(
        long productId,
        int onHand,
        int available,
        int reserved,
        long version
) {
    int changed = jdbc.update("""
            UPDATE inventory_item
            SET on_hand_quantity = :onHand,
                available_quantity = :available,
                reserved_quantity = :reserved,
                version = :version
            WHERE product_id = :productId
            """, new MapSqlParameterSource()
            .addValue("productId", productId)
            .addValue("onHand", onHand)
            .addValue("available", available)
            .addValue("reserved", reserved)
            .addValue("version", version));
    assertEquals(1, changed);
}

private void seedReservation(UUID reservationId) {
    jdbc.update("""
            INSERT INTO inventory_item (
                product_id,
                on_hand_quantity,
                available_quantity,
                reserved_quantity,
                version
            ) VALUES (88, 1, 0, 1, 0)
            ON CONFLICT (product_id) DO UPDATE
            SET on_hand_quantity = 1,
                available_quantity = 0,
                reserved_quantity = 1,
                version = 0
            """, Map.of());

    jdbc.update("""
            INSERT INTO inventory_reservation (
                reservation_id,
                order_id,
                product_id,
                quantity,
                status,
                created_at
            ) VALUES (
                :reservationId,
                :orderId,
                88,
                1,
                'RESERVED',
                now()
            )
            """, Map.of(
            "reservationId", reservationId,
            "orderId", UUID.randomUUID()
    ));
}
```

`LockSupport.parkNanos()` chỉ giảm nhịp truy vấn quan sát; nó không quyết định
thứ tự chạy của hai giao dịch. Thứ tự quan trọng được điều phối bằng latch và
được xác nhận qua `pg_stat_activity`.

## 4. Tái hiện lỗi bằng rào chắn

Lớp dưới đây chỉ dùng trong kiểm thử. Rào chắn buộc cả hai giao dịch tải xong
thực thể trước khi bên nào được phép thay đổi dữ liệu:

```java
@Service
public class BrokenInventoryWithBarrier {

    private final InventoryItemRepository items;
    private final InventoryReservationRepository reservations;
    private final CyclicBarrier bothLoaded = new CyclicBarrier(2);
    private final Clock clock;

    // Constructor injection omitted.

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReserveResult reserve(ReserveStock command) {
        InventoryItem item = items.findById(command.productId())
                .orElseThrow(ProductNotFoundException::new);

        awaitBarrier(bothLoaded);

        if (!item.hasEnough(command.quantity())) {
            return ReserveResult.notAvailable(command.orderId());
        }

        item.reserve(command.quantity());
        InventoryReservation reservation =
                InventoryReservation.reserved(command.orderId(), command.productId(), command.quantity(), clock.instant());
        reservations.save(reservation);
        items.flush();

        return ReserveResult.reserved(
                command.orderId(),
                reservation.getReservationId(),
                item.getAvailableQuantity()
        );
    }

    private static void awaitBarrier(CyclicBarrier barrier) {
        try {
            barrier.await(5, TimeUnit.SECONDS);
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(interrupted);
        } catch (BrokenBarrierException | TimeoutException failure) {
            throw new IllegalStateException(failure);
        }
    }
}
```

```java
@Test
void readCheckWriteCreatesReconciliationDrift() throws Exception {
    Future<ReserveResult> a = executor.submit(() ->
            broken.reserve(command("order-a", 4))
    );
    Future<ReserveResult> b = executor.submit(() ->
            broken.reserve(command("order-b", 4))
    );

    assertEquals(
            ReserveOutcome.RESERVED,
            a.get(10, TimeUnit.SECONDS).outcome()
    );
    assertEquals(
            ReserveOutcome.RESERVED,
            b.get(10, TimeUnit.SECONDS).outcome()
    );

    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(8, reservedRowsQuantity(77));
    assertFalse(inventoryMatchesReservations(77));
}
```

Bài kiểm thử không chỉ chờ một ngoại lệ. Nó chứng minh trạng thái cuối vi phạm
đối soát dù hai giao dịch đều báo thành công.

## 5. Hai yêu cầu cạnh tranh chỉ có một bên thắng

```java
@Test
void conditionalUpdateAcceptsExactlyOneBuyer() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Callable<ReserveResult> buyerA =
            buyer(command("order-a", 4), ready, start);
    Callable<ReserveResult> buyerB =
            buyer(command("order-b", 4), ready, start);

    Future<ReserveResult> a = executor.submit(buyerA);
    Future<ReserveResult> b = executor.submit(buyerB);

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    List<ReserveResult> results = List.of(
            a.get(10, TimeUnit.SECONDS),
            b.get(10, TimeUnit.SECONDS)
    );

    assertEquals(1, results.stream()
            .filter(result -> result.outcome()
                    == ReserveOutcome.RESERVED)
            .count());
    assertEquals(1, results.stream()
            .filter(result -> result.outcome()
                    == ReserveOutcome.NOT_AVAILABLE)
            .count());

    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(4, reservedRowsQuantity(77));
    assertTrue(inventoryMatchesReservations(77));
}

private Callable<ReserveResult> buyer(
        ReserveStock command,
        CountDownLatch ready,
        CountDownLatch start
) {
    return () -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return reservationTx.reserve(command);
    };
}
```

Việc A hay B thắng không được cố định. Bất biến cần kiểm tra là số bên thắng,
trạng thái tồn kho và tổng lượng giữ hàng.

## 6. Bên chờ kiểm tra lại điều kiện sau khi bên giữ khóa chốt

```java
@Test
void waiterReturnsEmptyAfterHolderCommits() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch allowCommit = new CountDownLatch(1);

    Future<Optional<StockAfterReserve>> holder =
            executor.submit(() -> inReadCommitted(() -> {
                setApplicationName("ecom001-holder");
                Optional<StockAfterReserve> result =
                        stock.tryReserve(77, 4);
                assertTrue(result.isPresent());
                holderUpdated.countDown();
                await(allowCommit);
                return result;
            }));

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    Future<Optional<StockAfterReserve>> waiter =
            executor.submit(() -> inReadCommitted(() -> {
                setApplicationName("ecom001-waiter");
                return stock.tryReserve(77, 4);
            }));

    try {
        awaitDatabaseBlock("ecom001-waiter");
        assertThrows(
                TimeoutException.class,
                () -> waiter.get(100, TimeUnit.MILLISECONDS)
        );
    } finally {
        allowCommit.countDown();
    }

    assertTrue(holder.get(5, TimeUnit.SECONDS).isPresent());
    assertTrue(waiter.get(5, TimeUnit.SECONDS).isEmpty());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```

Kết quả rỗng của bên chờ là bằng chứng PostgreSQL đã đánh giá lại điều kiện
`available_quantity >= 4` trên giá trị mới là `1`.

## 7. Bên chờ thành công nếu bên giữ khóa hoàn tác

```java
@Test
void waiterCanReserveAfterHolderRollsBack() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch forceRollback = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() ->
            inReadCommitted(() -> {
                setApplicationName("ecom001-rollback-holder");
                stock.tryReserve(77, 4).orElseThrow();
                holderUpdated.countDown();
                await(forceRollback);
                throw new RollbackForTest();
            })
    );

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    Future<Optional<StockAfterReserve>> waiter =
            executor.submit(() -> inReadCommitted(() -> {
                setApplicationName("ecom001-rollback-waiter");
                return stock.tryReserve(77, 4);
            }));

    awaitDatabaseBlock("ecom001-rollback-waiter");
    forceRollback.countDown();

    ExecutionException failure = assertThrows(
            ExecutionException.class,
            () -> holder.get(5, TimeUnit.SECONDS)
    );
    assertInstanceOf(RollbackForTest.class, failure.getCause());

    StockAfterReserve winner =
            waiter.get(5, TimeUnit.SECONDS).orElseThrow();
    assertEquals(1, winner.availableQuantity());
    assertEquals(4, winner.reservedQuantity());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```

Chỉ thay đổi của bên chờ được giữ lại. Phần cập nhật chưa chốt của bên giữ khóa
biến mất hoàn toàn.

## 8. Lỗi ở bước sau làm hoàn tác tồn kho

```java
@Test
void reservationInsertFailureRollsBackStockUpdate() {
    UUID duplicateId = UUID.randomUUID();
    seedReservation(duplicateId);

    assertThrows(DuplicateKeyException.class, () ->
            inReadCommitted(() -> {
                stock.tryReserve(77, 4).orElseThrow();
                jdbc.update("""
                        INSERT INTO inventory_reservation (
                            reservation_id,
                            order_id,
                            product_id,
                            quantity,
                            status,
                            created_at
                        ) VALUES (
                            :reservationId,
                            :orderId,
                            77,
                            4,
                            'RESERVED',
                            now()
                        )
                        """, Map.of(
                        "reservationId", duplicateId,
                        "orderId", UUID.randomUUID()
                ));
                return null;
            })
    );

    assertEquals(5, available(77));
    assertEquals(0, reservedCounter(77));
    assertEquals(0, version(77));
}
```

Mục tiêu là kiểm tra dữ liệu sau hoàn tác, không chỉ kiểm tra loại ngoại lệ.

## 9. Hết thời gian chờ khóa không phải hết hàng

```java
@Test
void lockTimeoutIsReportedAsTechnicalFailure() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() ->
            inReadCommitted(() -> {
                stock.tryReserve(77, 4).orElseThrow();
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
                            '150ms',
                            true
                        )
                        """, Map.of(), String.class);
                return stock.tryReserve(77, 4);
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
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```

Nếu lớp dịch vụ bắt mọi ngoại lệ rồi trả `NOT_AVAILABLE`, bài kiểm thử này phải
thất bại vì hệ thống đã che giấu tình trạng quá tải thành lỗi nghiệp vụ.

## 10. SQL trực tiếp không làm mới thực thể đã tải

```java
@Test
void directUpdateLeavesManagedEntityStale() {
    inReadCommitted(() -> {
        InventoryItem managed = entityManager.find(
                InventoryItem.class,
                77L
        );
        assertEquals(5, managed.getAvailableQuantity());

        stock.tryReserve(77, 4).orElseThrow();

        assertTrue(entityManager.contains(managed));
        assertEquals(5, managed.getAvailableQuantity());

        entityManager.clear();
        InventoryItem reloaded = entityManager.find(
                InventoryItem.class,
                77L
        );
        assertEquals(1, reloaded.getAvailableQuantity());
        assertEquals(4, reloaded.getReservedQuantity());
        return null;
    });
}
```

Bài kiểm thử này bảo vệ quy ước không trộn thực thể tồn kho đã tải với câu cập
nhật trực tiếp trong cùng ngữ cảnh Hibernate.

## 11. Nhiều yêu cầu không được giữ quá số lượng có sẵn

```java
@Test
void manyBuyersNeverReserveMoreThanOnHand() throws Exception {
    setInventory(77, 10, 10, 0, 0);

    int actors = 16;
    CountDownLatch ready = new CountDownLatch(actors);
    CountDownLatch start = new CountDownLatch(1);
    List<Future<ReserveResult>> futures = new ArrayList<>();

    for (int index = 0; index < actors; index++) {
        ReserveStock command = command(
                "order-" + index,
                1
        );
        futures.add(executor.submit(() -> {
            ready.countDown();
            assertTrue(start.await(5, TimeUnit.SECONDS));
            return reservationTx.reserve(command);
        }));
    }

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    List<ReserveResult> results = new ArrayList<>();
    long deadline = System.nanoTime()
            + Duration.ofSeconds(20).toNanos();

    for (Future<ReserveResult> future : futures) {
        long remaining = deadline - System.nanoTime();
        assertTrue(remaining > 0, "global deadline exceeded");
        results.add(future.get(
                remaining,
                TimeUnit.NANOSECONDS
        ));
    }

    assertEquals(10, results.stream()
            .filter(result -> result.outcome()
                    == ReserveOutcome.RESERVED)
            .count());
    assertEquals(6, results.stream()
            .filter(result -> result.outcome()
                    == ReserveOutcome.NOT_AVAILABLE)
            .count());
    assertEquals(0, available(77));
    assertEquals(10, reservedCounter(77));
    assertEquals(10, reservedRowsQuantity(77));
    assertTrue(inventoryMatchesReservations(77));
}
```

Đây là kiểm thử tải có giới hạn, không thay thế bài tái hiện lỗi bằng rào chắn.
Nó kiểm tra thêm rằng mã nguồn thật vẫn giữ đúng bất biến khi có nhiều bên tranh
chấp.

## 12. Ma trận độ phủ

| Thực nghiệm | Cơ chế được kiểm tra | Bất biến được bảo vệ |
| --- | --- | --- |
| Đọc–kiểm tra–ghi | Ghi đè từ dữ liệu cũ | Phát hiện lệch giữa bộ đếm và lịch sử |
| Hai người mua | Cập nhật có điều kiện | Chỉ một bên thắng |
| Bên giữ khóa chốt | Kiểm tra lại `WHERE` | Không bán quá số lượng còn lại |
| Bên giữ khóa hoàn tác | Khôi phục phiên bản cũ | Một bên khác vẫn có thể tiến lên |
| Chèn bản ghi giữ hàng lỗi | Hoàn tác toàn giao dịch | Không trừ hàng thiếu lịch sử |
| Hết thời gian chờ | Phân loại SQLSTATE | Không đổi lỗi kỹ thuật thành hết hàng |
| Thực thể đã tải | Ngữ cảnh Hibernate cũ | Không dùng trạng thái Java sai |
| Nhiều người mua | Tranh chấp thực tế | Tổng giữ hàng không vượt tồn kho |

## 13. Quan sát trong môi trường thật

Có thể quan sát giao dịch đang chờ khóa bằng truy vấn:

```sql
SELECT application_name,
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

Các chỉ số cần theo dõi:

- tỷ lệ một dòng và không dòng nào được cập nhật theo `product_id`;
- thời gian chờ khóa và thời lượng giao dịch;
- số lỗi `55P03`, `40P01`, `40001`;
- số kết nối hoạt động và số luồng chờ kết nối;
- chênh lệch giữa `reserved_quantity` và tổng các bản ghi `RESERVED`;
- số giao dịch bị hoàn tác sau khi câu cập nhật tồn kho đã chạy.

Không ghi toàn bộ câu lệnh kèm dữ liệu khách hàng nhạy cảm vào nhật ký.

## 14. Quy tắc tránh kiểm thử chập chờn

- Dùng `CountDownLatch` hoặc `CyclicBarrier` tại đúng điểm tranh chấp.
- Không dùng `Thread.sleep()` để đoán một giao dịch đã lấy khóa.
- Xác nhận trạng thái chờ bằng `pg_stat_activity` khi thứ tự khóa là điều cần
  chứng minh.
- Mọi `await()` và `Future.get()` đều có thời hạn.
- Giải phóng latch trong `finally` để lỗi khẳng định không làm kẹt luồng khác.
- Tắt `ExecutorService` sau mỗi bài kiểm thử.
- Khi bắt `InterruptedException`, khôi phục cờ ngắt của luồng.
- Kiểm tra trạng thái dữ liệu cuối cùng, không chỉ kiểm tra ngoại lệ.

Nguyên tắc tổng quát được trình bày trong
[tài liệu kiểm thử đồng thời](../../concepts/concurrency-testing.md).
