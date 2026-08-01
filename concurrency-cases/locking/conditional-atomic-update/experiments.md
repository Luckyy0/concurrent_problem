# Thực Nghiệm (Experiments): Khởi Tạo Lock Nguyên Tử Có Điều Kiện

## 1. Mục Tiêu Kiểm Thử

Bộ kiểm thử (Test suite) được thiết kế nhằm mục đích chứng minh các hành vi sau:

1. Kịch bản Đọc-Kiểm tra-Ghi (Read-Check-Write) cơ bản sẽ gây ra sự sai lệch số liệu đối soát (Reconciliation failure).
2. Khi thực thi câu lệnh UPDATE nguyên tử đa luồng (Concurrent atomic UPDATE), nếu chỉ đủ số lượng tồn kho cho 1 yêu cầu, hệ thống đảm bảo chỉ có đúng 1 yêu cầu thành công.
3. Transaction đang chờ (Waiter) sẽ tự động kích hoạt đánh giá lại điều kiện `WHERE` (Predicate recheck) sau khi transaction giữ lock (Holder) hoàn tất (Commit).
4. Nếu transaction giữ lock bị hủy bỏ (Rollback), transaction chờ sẽ tiếp nhận trạng thái ban đầu của tồn kho và xử lý tiếp.
5. Hiện tượng quá thời gian chờ lock (Lock timeout) là một lỗi kỹ thuật của hệ thống, hoàn toàn khác biệt với kết quả trả về `0` dòng (Hết hàng).
6. Khi xử lý một mã lệnh trùng lặp liên tục (Duplicate command), hệ thống chỉ thực hiện trừ số lượng tồn kho duy nhất một lần.
7. Nếu phát sinh lỗi ở các khâu sau khi thực thi lệnh UPDATE, hệ thống phải hoàn tác (Rollback) toàn bộ dữ liệu tồn kho.
8. Gọi Bulk DML trực tiếp sẽ vượt qua Hibernate, có khả năng gây ảnh hưởng đến bộ đệm (Persistence context).
9. Trong các bài toán tải cao (Stress test), hệ thống phải luôn bảo toàn tính đúng đắn của dữ liệu tổng.

> **Lưu ý quan trọng:** Do tính chất mô phỏng các cơ chế chuyên sâu của PostgreSQL, việc kiểm thử **BẮT BUỘC** phải được thực hiện thông qua Testcontainers với một database thật, không sử dụng H2 in-memory.

## 2. Cấu Hình Testcontainers (Fixture)

```java
@Testcontainers
@SpringBootTest
class ConditionalInventoryUpdateIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17-alpine");

    @DynamicPropertySource
    static void datasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.datasource.hikari.maximum-pool-size", () -> "8");
        registry.add("app.inventory.database.lock-timeout", () -> "5s");
        registry.add("app.inventory.database.statement-timeout", () -> "10s");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
    }

    @Autowired PlatformTransactionManager transactionManager;
    @Autowired NamedParameterJdbcTemplate jdbc;
    @Autowired InventoryMutationDao inventory;
    @Autowired InventoryReservationCoordinator coordinator;
    @PersistenceContext EntityManager entityManager;

    private ExecutorService executor;

    @BeforeEach
    void reset() {
        executor = Executors.newFixedThreadPool(8);
        jdbc.update("delete from outbox_event", Map.of());
        jdbc.update("delete from inventory_reservation", Map.of());
        jdbc.update("""
                update inventory_item
                set on_hand_quantity = 5,
                    available_quantity = 5,
                    reserved_quantity = 0,
                    revision = 10
                where product_id = 77
                """, Map.of());
    }

    @AfterEach
    void shutdown() throws InterruptedException {
        executor.shutdownNow();
        assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Kịch bản khởi tạo cung cấp sẵn dữ liệu sản phẩm `77`. Phương thức `reset()` đảm nhận việc làm sạch bảng lịch sử và phục hồi lại số lượng, tránh thao tác xóa và chèn lại làm xáo trộn định danh dòng (Row identity).

## 3. Các Phương Thức Hỗ Trợ Điều Phối (Coordination Helpers)

```java
@FunctionalInterface
interface TransactionWork<T> {
    T run();
}

private <T> T inTransaction(TransactionWork<T> work) {
    TransactionTemplate template =
            new TransactionTemplate(transactionManager);
    template.setIsolationLevel(
            TransactionDefinition.ISOLATION_READ_COMMITTED
    );
    return template.execute(status -> work.run());
}

private void awaitLatch(CountDownLatch latch) {
    try {
        assertTrue(latch.await(5, TimeUnit.SECONDS));
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError(interrupted);
    }
}

private void setLocalApplicationName(String name) {
    jdbc.queryForObject("""
            select set_config('application_name', :name, true)
            """, Map.of("name", name), String.class);
}

private void awaitDatabaseBlock(String applicationName) {
    long deadline = System.nanoTime() + Duration.ofSeconds(5).toNanos();

    while (System.nanoTime() < deadline) {
        Boolean blocked = jdbc.queryForObject("""
                select exists (
                    select 1
                    from pg_stat_activity
                    where application_name = :name
                      and wait_event_type = 'Lock'
                      and cardinality(pg_blocking_pids(pid)) > 0
                )
                """, Map.of("name", applicationName), Boolean.class);
        if (Boolean.TRUE.equals(blocked)) {
            return;
        }

        LockSupport.parkNanos(Duration.ofMillis(10).toNanos());
        if (Thread.currentThread().isInterrupted()) {
            throw new AssertionError("interrupted while observing lock wait");
        }
    }
    fail("database lock wait was not observed");
}

private String sqlState(Throwable failure) {
    for (Throwable current = failure;
         current != null;
         current = current.getCause()) {
        if (current instanceof SQLException sql) {
            return sql.getSQLState();
        }
    }
    return null;
}
```

Việc đồng bộ hóa giữa các luồng sử dụng `CountDownLatch`. Phương thức `awaitDatabaseBlock` kiểm tra View `pg_stat_activity` của PostgreSQL nhằm xác nhận trạng thái cấp lock (Lock state). Đảm bảo đặt thời gian chờ tối đa (Timeout) để ngăn ngừa hiện tượng kiểm thử treo không thời hạn (Hanging test).

## 4. Thực Nghiệm 1 — Tái Hiện Lỗi Ghi Đè Dữ Liệu (Lost Update)

Thiết kế một lớp mô phỏng luồng lỗi (Broken service), dùng `CyclicBarrier` để đồng bộ hóa hai luồng transaction ngay sau thao tác `SELECT`:

```java
@Service
class BarrierBrokenReservationTx {
    private final InventoryItemRepository items;
    private final InventoryReservationRepository reservations;
    private final CyclicBarrier bothLoaded = new CyclicBarrier(2);

    // Constructor injection omitted.

    @Transactional
    public UUID reserve(ReserveStockCommand command) {
        InventoryItem item =
                items.findById(command.productId()).orElseThrow();
        assertEquals(5, item.availableQuantity());

        try {
            bothLoaded.await(5, TimeUnit.SECONDS); // Đồng bộ hai luồng
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(interrupted);
        } catch (BrokenBarrierException | TimeoutException failure) {
            throw new IllegalStateException(failure);
        }

        // Bắt đầu thực thi thao tác cập nhật (Ghi đè tuyệt đối)
        item.reserve(command.quantity());
        UUID reservationId = UUID.randomUUID();
        reservations.save(InventoryReservation.reserved(
                command,
                reservationId
        ));
        return reservationId;
    }
}
```

```java
@Test
void plainReadCheckWriteBreaksReconciliation() throws Exception {
    Future<UUID> a = executor.submit(
            () -> broken.reserve(command("A", 4))
    );
    Future<UUID> b = executor.submit(
            () -> broken.reserve(command("B", 4))
    );

    a.get(10, TimeUnit.SECONDS);
    b.get(10, TimeUnit.SECONDS);

    // Kiểm chứng: Sự bất đồng bộ (Reconciliation drift)
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(8, reservedAuditQuantity(77));
    assertEquals(2, reservedOutcomeCount(77));
    assertFalse(projectionMatchesAudit(77)); // Sổ đối soát lệch
}
```
Kết quả cho thấy mặc dù các ràng buộc database (Constraints) không bị vi phạm, tính đồng nhất của hệ thống đã bị mất, xác nhận nguy cơ của việc Đọc-Kiểm tra-Ghi độc lập.

## 5. Thực Nghiệm 2 — Tính Chất Tự Đánh Giá Lại (Predicate Recheck)

```java
@Test
void waiterReturnsNoRowAfterHolderCommits() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch allowCommit = new CountDownLatch(1);

    // Luồng A: Thực hiện cập nhật nhưng chưa Commit
    Future<Optional<InventoryAfterReserve>> holder =
            executor.submit(() -> inTransaction(() -> {
                setLocalApplicationName("lock004-holder");
                Optional<InventoryAfterReserve> changed =
                        inventory.tryReserve(77, 4);
                assertTrue(changed.isPresent());
                holderUpdated.countDown();
                awaitLatch(allowCommit);
                return changed;
            }));

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    // Luồng B: Yêu cầu cùng tài nguyên, rơi vào trạng thái chờ (Wait)
    Future<Optional<InventoryAfterReserve>> waiter =
            executor.submit(() -> inTransaction(() -> {
                setLocalApplicationName("lock004-waiter");
                return inventory.tryReserve(77, 4);
            }));

    awaitDatabaseBlock("lock004-waiter"); // Xác thực trạng thái Blocking
    assertThrows(
            TimeoutException.class,
            () -> waiter.get(100, TimeUnit.MILLISECONDS)
    );

    allowCommit.countDown(); // Cho phép luồng A Commit

    // Kiểm định kết quả
    assertTrue(holder.get(5, TimeUnit.SECONDS).isPresent()); // Luồng A thành công
    assertTrue(waiter.get(5, TimeUnit.SECONDS).isEmpty()); // Luồng B không có dữ liệu trả về
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(11, revision(77));
}
```
Việc luồng B nhận được `Optional.empty()` là kết quả từ tính năng Predicate Recheck của PostgreSQL. Database sử dụng trạng thái dữ liệu đã cập nhật bởi luồng A (với `available = 1`) để đánh giá lại điều kiện `1 >= 4`, và từ chối thao tác do điều kiện không thỏa mãn.

## 6. Thực Nghiệm 3 — Kịch Bản Hoàn Tác Của Transaction Giữ Lock (Holder Rollback)

```java
@Test
void waiterCanApplyAfterHolderRollsBack() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch forceRollback = new CountDownLatch(1);

    // Luồng A: Cập nhật thành công nhưng cưỡng chế lỗi để Rollback
    Future<Void> holder = executor.submit(() -> inTransaction(() -> {
        setLocalApplicationName("lock004-rollback-holder");
        assertTrue(inventory.tryReserve(77, 4).isPresent());
        holderUpdated.countDown();
        awaitLatch(forceRollback);
        throw new RollbackForTest();
    }));

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    // Luồng B: Vào trạng thái chờ (Wait)
    Future<Optional<InventoryAfterReserve>> waiter =
            executor.submit(() -> inTransaction(() -> {
                setLocalApplicationName("lock004-rollback-waiter");
                return inventory.tryReserve(77, 4);
            }));

    awaitDatabaseBlock("lock004-rollback-waiter");
    forceRollback.countDown(); // Kích hoạt luồng A Rollback

    ExecutionException rolledBack =
            assertThrows(ExecutionException.class,
                    () -> holder.get(5, TimeUnit.SECONDS));
    assertInstanceOf(RollbackForTest.class, rolledBack.getCause());

    // Luồng B sau khi được đánh thức, tiến hành thao tác trên dữ liệu gốc
    InventoryAfterReserve winner =
            waiter.get(5, TimeUnit.SECONDS).orElseThrow();
    assertEquals(1, winner.available());
    assertEquals(4, winner.reserved());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(11, revision(77)); // Revision chỉ tăng 1 lần
}
```
Luồng B thành công và phiên bản (Revision) cũng nhất quán do toàn bộ hoạt động của A đã được loại bỏ.

## 7. Thực Nghiệm 4 — Phân Biệt Lỗi Lock Timeout

```java
@Test
void lockTimeoutIsTechnicalFailureNotOutOfStock() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> inTransaction(() -> {
        inventory.tryReserve(77, 4).orElseThrow();
        holderUpdated.countDown();
        awaitLatch(releaseHolder);
        return null;
    }));

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    Future<Optional<InventoryAfterReserve>> contender =
            executor.submit(() -> inTransaction(() -> {
                // Áp đặt giới hạn chờ (150ms)
                jdbc.queryForObject("""
                        select set_config(
                            'lock_timeout',
                            '150ms',
                            true
                        )
                        """, Map.of(), String.class);
                return inventory.tryReserve(77, 4);
            }));

    // Bắt ngoại lệ lỗi kỹ thuật hệ thống
    ExecutionException failed =
            assertThrows(ExecutionException.class,
                    () -> contender.get(5, TimeUnit.SECONDS));
    assertEquals("55P03", sqlState(failed)); // Mã lỗi đặc trưng của Timeout

    releaseHolder.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```
Mã lỗi `55P03` khẳng định việc quá tải hệ thống, tầng ứng dụng phải phân biệt rạch ròi với trạng thái nghiệp vụ `OUT_OF_STOCK` (0 dòng bị ảnh hưởng). Latch release luôn được đề xuất đặt trong khối `finally` cho các bài kiểm thử thực tế.

## 8. Thực Nghiệm 5 — Tính Nguyên Tử Của Cạnh Tranh (Concurrency Resolution)

```java
@Test
void twoCommandsCommitOneReserveAndOneOutOfStock() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Callable<ReservationResult> actorA = () -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return coordinator.reserve(command("A", 4));
    };
    Callable<ReservationResult> actorB = () -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return coordinator.reserve(command("B", 4));
    };

    Future<ReservationResult> a = executor.submit(actorA);
    Future<ReservationResult> b = executor.submit(actorB);

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    List<ReservationResult> results = List.of(
            a.get(10, TimeUnit.SECONDS),
            b.get(10, TimeUnit.SECONDS)
    );

    // Xác minh kết quả không có tính mập mờ
    assertEquals(1, results.stream()
            .filter(ReservationResult::reserved).count());
    assertEquals(1, results.stream()
            .filter(ReservationResult::outOfStock).count());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(4, reservedAuditQuantity(77));
    assertEquals(1, outboxEventCount("InventoryReserved"));
    assertTrue(projectionMatchesAudit(77));
}
```
Sổ sách đối soát đảm bảo đồng nhất, một transaction thành công và một transaction chuyển trạng thái từ chối (`OUT_OF_STOCK`).

## 9. Thực Nghiệm 6 — Chống Trùng Lặp Yêu Cầu (Idempotent Replay)

```java
@Test
void sameCommandDecrementsOnceAndReplaysOutcome() throws Exception {
    ReserveStockCommand command = command("SAME", 4);
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Callable<ReservationResult> duplicate = () -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return coordinator.reserve(command);
    };

    Future<ReservationResult> first = executor.submit(duplicate);
    Future<ReservationResult> second = executor.submit(duplicate);

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    ReservationResult one = first.get(10, TimeUnit.SECONDS);
    ReservationResult two = second.get(10, TimeUnit.SECONDS);

    // Xác nhận việc tái phát lại kết quả
    assertEquals(one.reservationId(), two.reservationId());
    assertTrue(one.reserved());
    assertTrue(two.reserved());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77)); // Chỉ xử lý 1 lần
    assertEquals(1, commandRowCount(command.commandId()));
    assertEquals(1, outboxEventCount("InventoryReserved"));
}

@Test
void reusedCommandWithDifferentPayloadIsRejected() {
    UUID commandId = UUID.randomUUID();
    coordinator.reserve(command(commandId, "order-A", 4));

    // Thử thao tác với mã Payload khác nhau
    assertThrows(
            IdempotencyMismatchException.class,
            () -> coordinator.reserve(
                    command(commandId, "order-B", 1)
            )
    );
    // Tính nguyên vẹn kho hàng
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```
Bảo vệ hệ thống khỏi việc khách hàng lặp lại (Retry) lệnh bị trễ mạng, tính chất lũy đẳng (Idempotency) là một yêu cầu bắt buộc bên cạnh transaction nguyên tử.

## 10. Thực Nghiệm 7 — Lỗi Hoàn Tác Do Vi Phạm Ràng Buộc (Rollback Recovery)

```java
@Test
void downstreamConstraintFailureRollsBackAtomicUpdate() {
    UUID duplicateEventId = seedOutboxEvent();

    assertThrows(DuplicateKeyException.class, () ->
            inTransaction(() -> {
                inventory.tryReserve(77, 4).orElseThrow();
                
                // Vi phạm khóa ngoại tại khâu ghi nhận Outbox
                jdbc.update("""
                        insert into outbox_event (
                            event_id,
                            aggregate_type,
                            aggregate_id,
                            event_type,
                            payload,
                            created_at
                        ) values (
                            :eventId,
                            'InventoryReservation',
                            'test',
                            'InventoryReserved',
                            '{}'::jsonb,
                            now()
                        )
                        """, Map.of("eventId", duplicateEventId));
                return null;
            })
    );

    // Xác minh transaction đã hoàn tác (Rollback) tồn kho một cách nguyên vẹn
    assertEquals(5, available(77));
    assertEquals(0, reservedCounter(77));
    assertEquals(10, revision(77));
    assertEquals(0, reservedOutcomeCount(77));
}
```

## 11. Thực Nghiệm 8 — Cập Nhật Bulk DML và Bộ Đệm Hibernate (Persistence Context)

```java
@Test
void directUpdateDoesNotRefreshManagedEntity() {
    inTransaction(() -> {
        // Tải Entity lên bộ đệm RAM
        InventoryItem managed =
                entityManager.find(InventoryItem.class, 77L);
        assertEquals(5, managed.availableQuantity());

        // Cập nhật thông qua Bulk DML (Bỏ qua JPA)
        inventory.tryReserve(77, 4).orElseThrow();

        // Đối tượng JPA không tự nhận diện sự thay đổi
        assertTrue(entityManager.contains(managed));
        assertEquals(5, managed.availableQuantity());

        // Làm sạch (Clear) bộ đệm và truy vấn lại
        entityManager.clear();
        InventoryItem reloaded =
                entityManager.find(InventoryItem.class, 77L);
        assertEquals(1, reloaded.availableQuantity());
        assertEquals(4, reloaded.reservedQuantity());
        return null;
    });
}
```
Bài kiểm tra cảnh báo nguy cơ thao tác trộn lẫn giữa Bulk DML và EntityManager, có thể dẫn đến việc Entity (State) không đồng bộ và lỗi cập nhật đè khi có sự kiện Flush.

## 12. Thực Nghiệm 9 — Kiểm Thử Tải Trọng Mức Cao (Stress Test)

```java
@Test
void manyCommandsNeverReserveBeyondOnHand() throws Exception {
    setInventory(77, 20, 0, 20); // Thiết lập 20 đơn vị hàng

    int actors = 32; // Số lượng thread truy cập
    CountDownLatch ready = new CountDownLatch(actors);
    CountDownLatch start = new CountDownLatch(1);
    List<Future<ReservationResult>> futures = new ArrayList<>();

    for (int i = 0; i < actors; i++) {
        String label = "actor-" + i;
        futures.add(executor.submit(() -> {
            ready.countDown();
            assertTrue(start.await(5, TimeUnit.SECONDS));
            return coordinator.reserve(command(label, 1));
        }));
    }

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown(); // Khởi tạo thao tác truy cập đồng thời

    List<ReservationResult> results = new ArrayList<>();
    long deadline = System.nanoTime() + Duration.ofSeconds(20).toNanos();
    for (Future<ReservationResult> future : futures) {
        long remaining = deadline - System.nanoTime();
        assertTrue(remaining > 0, "global future deadline exceeded");
        results.add(future.get(remaining, TimeUnit.NANOSECONDS));
    }

    // Đảm bảo dữ liệu tổng không vi phạm tính toàn vẹn (Conservation Invariant)
    assertEquals(20, results.stream()
            .filter(ReservationResult::reserved).count()); 
    assertEquals(12, results.stream()
            .filter(ReservationResult::outOfStock).count()); 
    assertEquals(0, available(77)); 
    assertEquals(20, reservedCounter(77));
    assertEquals(20, reservedAuditQuantity(77));
    assertEquals(20, outboxEventCount("InventoryReserved"));
    assertTrue(projectionMatchesAudit(77)); 
}
```

## 13. Thực Nghiệm 10 — Hợp Đồng Nghiệp Vụ Với Kết Quả 0 Dòng (Zero Rows)

```java
@Test
void invalidQuantityIsRejectedBeforeSql() {
    assertThrows(
            IllegalArgumentException.class,
            () -> command("invalid", 0) // Thông số không hợp lệ
    );
    assertEquals(5, available(77));
}

@Test
void stableExistingProductMapsZeroRowToOutOfStock() {
    setInventory(77, 2, 3, 5);

    ReservationResult result =
            coordinator.reserve(command("too-large", 4)); // Vượt quá khả năng cung cấp

    assertTrue(result.outOfStock()); // Chuyển đổi thành trạng thái nghiệp vụ
    assertEquals(2, available(77));
    assertEquals(3, reservedCounter(77));
    assertEquals(1, outcomeCount("OUT_OF_STOCK"));
    assertEquals(0, outboxEventCount("InventoryReserved"));
}
```

## 14. Tổng Hợp Độ Phủ Mức Độ (Coverage Matrix)

| Thí Nghiệm (Experiment) | Tính Chất Kỹ Thuật (Technical Validation) | Ràng Buộc Nghiệp Vụ (Business Invariant) |
| --- | --- | --- |
| 1 | Bất đồng bộ trong truy cập Read-Write riêng lẻ | Sự lệch lạc giữa báo cáo và dữ liệu cơ sở |
| 2 | Chức năng tự kiểm tra lại (Predicate Recheck) | Đồng bộ trạng thái theo thiết kế hệ thống |
| 3 | Khôi phục từ transaction bị hủy bỏ (Holder Rollback) | Tiếp diễn transaction trên dữ liệu gốc |
| 4 | Hiện tượng Time-out chờ cấp lock (Lock Timeout) | Phát hiện và xử lý phân nhánh lỗi `55P03` |
| 5 | Chấp nhận giới hạn đồng bộ (Concurrent resolution) | Không gây sập hệ thống (deadlock hay Race condition) |
| 6 | Nhận biết truy cập từ cùng một mã lệnh (Idempotency) | Bảo toàn số lượng và kết xuất kết quả trước đó |
| 7 | Hủy Rollback do nguyên nhân ngoại vi (Downstream failure) | Phục hồi hoàn toàn dữ liệu |
| 8 | Sự đồng bộ giữa Bulk DML và Context Hibernate | Tránh rủi ro ghi đè dữ liệu sai |
| 9 | Giám sát giới hạn tương tranh qua số lượng thread cao | Tránh tình trạng tranh chấp sai dữ liệu |
| 10 | Từ chối truy cập qua lỗi logic (Bad arguments/Zero Rows) | Giữ nguyên quy trình quản trị mã lỗi nghiệp vụ |

## 15. Quan Sát Thông Số Production (Production Observability)

Có thể kiểm tra tình trạng lock của database với query:
```sql
SELECT application_name,
       state,
       wait_event_type,
       wait_event,
       pg_blocking_pids(pid) AS blockers,
       clock_timestamp() - xact_start AS transaction_age,
       query
FROM pg_stat_activity
WHERE datname = current_database()
  AND state <> 'idle';
```

Nên thiết lập giám sát các thông số (Metrics):
- Tỷ lệ hoàn tất cập nhật (1 dòng) so với thất bại/từ chối (0 dòng).
- Sự xuất hiện của lỗi quá tải hệ thống `55P03` (lock timeout) hay `40P01` (deadlock).
- Thời lượng chờ đợi (Lock wait duration) tại Layer database.
- Trạng thái Connection Pool (như HikariCP: Pending, Active, Timeout threads).
- Dữ liệu thu thập từ Job đối soát tính đồng bộ (Reconciliation check).

## 16. Tiêu Chuẩn Cho Độ Tin Cậy Của Bài Test (Anti-Flaky Guidelines)

- Sử dụng cơ chế đồng bộ (Barrier/Latch) ngay tại các điểm tạo ra điều kiện tương tranh (Race conditions).
- Khẳng định tính chính xác bằng hàm kiểm tra Block (ví dụ như `awaitDatabaseBlock`) trước khi phát lệnh chạy các luồng khác.
- Yêu cầu cấu hình thời gian chờ tối đa (Bound) ở các hàm đợi (Future/Latch).
- Xây dựng lượng luồng xử lý đủ lớn trong Thread Pool để tránh tình trạng kẹt kết nối luồng.
- Luôn tắt bộ luồng ngầm (Shutdown executors) tại các khối `finally` hoặc `tearDown`.
- Sử dụng UUID độc lập cho các trường khóa chính (Primary key) giữa các bài kiểm thử nhằm hạn chế tác dụng phụ.
