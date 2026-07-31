# Cấu hình Thực nghiệm (Experiments) cho Cập nhật Nguyên tử có điều kiện

## Mục tiêu

Bộ test (Test suite) của chúng ta phải chứng minh được những điều sau đây bằng thực tế:

1. Kịch bản ngây thơ "đọc-kiểm-tra-ghi" chắc chắn làm lệch sổ đối soát.
2. Nếu chạy câu lệnh UPDATE nguyên tử đa luồng, khi chỉ còn đủ hàng cho 1 người, thì chỉ có đúng 1 người thắng.
3. Kẻ đứng chờ (waiter) sẽ tự động chạy lại cục điều kiện `WHERE` sau khi người giữ khóa (holder) chốt sổ thành công.
4. Nếu người giữ khóa hủy bỏ (rollback), kẻ đứng chờ được lấy lại kho hàng ban đầu.
5. Hết giờ chờ khóa (Lock timeout) là lỗi hệ thống, hoàn toàn khác với việc trả về `0` dòng (hết hàng).
6. Bấm nút liên tục với cùng 1 mã đơn (duplicate command) thì cũng chỉ bị trừ hàng đúng 1 lần.
7. Nếu gặp lỗi ở các bước sau câu lệnh UPDATE, kho hàng phải được hoàn tác (rollback).
8. Gọi SQL thẳng xuống DB sẽ qua mặt Hibernate và làm bộ đệm bị ôi thiu.
9. Dù có ném hàng chục luồng vào cùng lúc, tổng kho vẫn luôn được bảo toàn tuyệt đối (conservation invariant).

Chú ý quan trọng: Vì chúng ta đang test các cơ chế đặc thù của PostgreSQL, nên **BẮT BUỘC** phải dùng thư viện Testcontainers chạy database thật. Chạy bằng H2 in-memory sẽ không chứng minh được gì cả!

## Cấu trúc Testcontainers (Fixture)

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

Script khởi tạo đã tạo sẵn sản phẩm `77`. Trong hàm `reset()`, ta chỉ đơn giản là dọn dẹp bảng lịch sử và reset lại số đếm, chứ không cần xóa rồi insert lại (tránh làm thay đổi định danh dòng).

## Các hàm Hỗ trợ Điều phối (Coordination helpers)

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

Chúng ta dùng `CountDownLatch` để chặn các luồng nhằm tạo ra thứ tự chạy chính xác. Hàm `awaitDatabaseBlock` sẽ chọc thẳng vào bảng theo dõi `pg_stat_activity` của Postgres để chắc chắn rằng: "À, luồng này đang thực sự bị PostgreSQL khóa mõm rồi". Luôn nhớ đặt thời gian chờ tối đa (timeout) cho các hàm đợi để test không bị treo cứng ngắc.

## Thực nghiệm 1 — Tái hiện thảm họa ghi đè mất dữ liệu

Ta viết riêng một class hỏng để test, dùng `CyclicBarrier` chặn 2 luồng lại ngay sau khi vừa `SELECT` xong:

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
        assertEquals(5, item.availableQuantity()); // Đọc lên 5

        try {
            bothLoaded.await(5, TimeUnit.SECONDS); // Chặn lại, đợi đủ 2 thằng cùng tới đây
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(interrupted);
        } catch (BrokenBarrierException | TimeoutException failure) {
            throw new IllegalStateException(failure);
        }

        // Bắt đầu nhắm mắt ghi đè
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

    // Assert: Sổ sách lệch bét nhè!
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(8, reservedAuditQuantity(77));
    assertEquals(2, reservedOutcomeCount(77));
    assertFalse(projectionMatchesAudit(77)); // ĐỐI SOÁT LỆCH
}
```

Tất cả các rào chắn (constraints) của dòng dữ liệu đều cho qua, nhưng hàm chạy đối soát (reconciliation) thì gào thét vì số liệu lệch nhau.

## Thực nghiệm 2 — Khi luồng chốt sổ, luồng chờ tự "nhìn lại bản thân" (recheck)

```java
@Test
void waiterReturnsNoRowAfterHolderCommits() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch allowCommit = new CountDownLatch(1);

    // Luồng A: Chạy vào UPDATE và giữ nguyên khóa ở đó, chưa chốt sổ
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

    // Luồng B: Húc đầu vào và bị PostgreSQL cho đứng chờ
    Future<Optional<InventoryAfterReserve>> waiter =
            executor.submit(() -> inTransaction(() -> {
                setLocalApplicationName("lock004-waiter");
                return inventory.tryReserve(77, 4);
            }));

    awaitDatabaseBlock("lock004-waiter"); // Xác nhận B đang bị khóa mỏ
    assertThrows(
            TimeoutException.class,
            () -> waiter.get(100, TimeUnit.MILLISECONDS)
    );

    allowCommit.countDown(); // Bật đèn xanh cho A chốt sổ

    // Kiểm tra kết quả
    assertTrue(holder.get(5, TimeUnit.SECONDS).isPresent()); // A thắng
    assertTrue(waiter.get(5, TimeUnit.SECONDS).isEmpty()); // B tay trắng ra về
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(11, revision(77));
}
```

> **Nói ngắn gọn:** Kết quả `Optional.empty()` của B có được là nhờ PostgreSQL sau khi thức dậy khỏi chế độ chờ, đã nhìn thấy con số kho `available=1` mới tinh, tự động đem đi xét lại cụm `WHERE 1 >= 4`, thấy sai nên báo về 0 dòng. Cực kỳ thông minh!

## Thực nghiệm 3 — Luồng đang giữ khóa lật lọng (Rollback) thì luồng chờ thắng

```java
@Test
void waiterCanApplyAfterHolderRollsBack() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch forceRollback = new CountDownLatch(1);

    // Luồng A: UPDATE thành công nhưng cuối cùng quậy tung chảo ném lỗi Rollback
    Future<Void> holder = executor.submit(() -> inTransaction(() -> {
        setLocalApplicationName("lock004-rollback-holder");
        assertTrue(inventory.tryReserve(77, 4).isPresent());
        holderUpdated.countDown();
        awaitLatch(forceRollback);
        throw new RollbackForTest();
    }));

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    // Luồng B: Đứng chờ ngoan ngoãn
    Future<Optional<InventoryAfterReserve>> waiter =
            executor.submit(() -> inTransaction(() -> {
                setLocalApplicationName("lock004-rollback-waiter");
                return inventory.tryReserve(77, 4);
            }));

    awaitDatabaseBlock("lock004-rollback-waiter");
    forceRollback.countDown(); // Ra lệnh cho A tự hủy (Rollback)

    ExecutionException rolledBack =
            assertThrows(ExecutionException.class,
                    () -> holder.get(5, TimeUnit.SECONDS));
    assertInstanceOf(RollbackForTest.class, rolledBack.getCause());

    // Luồng B tỉnh dậy, thấy A như chưa từng tồn tại, trừ kho cái vèo!
    InventoryAfterReserve winner =
            waiter.get(5, TimeUnit.SECONDS).orElseThrow();
    assertEquals(1, winner.available());
    assertEquals(4, winner.reserved());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(11, revision(77)); // Chỉ lên 1 phiên bản
}
```

Kết quả trừ và phiên bản (revision) của A bay màu sạch sẽ; luồng B xông vào và làm chủ sân khấu.

## Thực nghiệm 4 — Quá giờ chờ khóa (Lock timeout) là lỗi hệ thống, không phải hết hàng

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
                // Ép luồng này chỉ được chờ khóa tối đa 150ms
                jdbc.queryForObject("""
                        select set_config(
                            'lock_timeout',
                            '150ms',
                            true
                        )
                        """, Map.of(), String.class);
                return inventory.tryReserve(77, 4);
            }));

    // Bụp! Văng lỗi hệ thống
    ExecutionException failed =
            assertThrows(ExecutionException.class,
                    () -> contender.get(5, TimeUnit.SECONDS));
    assertEquals("55P03", sqlState(failed)); // Đúng chuẩn mã lỗi timeout

    releaseHolder.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```

Hãy tập thói quen nhả chốt khóa (release latch) ở cụm `finally` khi code thật để test khỏi bị treo. Nhắc lại: Bị văng lỗi này thì đừng có ghi xuống DB là `OUT_OF_STOCK`!

## Thực nghiệm 5 — Cùng lao vào, chỉ một người được vinh danh

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
    start.countDown(); // Mở cổng cho 2 con ngựa cùng chạy

    List<ReservationResult> results = List.of(
            a.get(10, TimeUnit.SECONDS),
            b.get(10, TimeUnit.SECONDS)
    );

    // Chắc chắn 1 ăn, 1 tịt
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

Người thắng có thể là A, có thể là B tùy duyên hệ điều hành. Nhưng sổ sách thì 100% không bao giờ sai.

## Thực nghiệm 6 — Bấm nút điên cuồng (Concurrent duplicate replay)

```java
@Test
void sameCommandDecrementsOnceAndReplaysOutcome() throws Exception {
    ReserveStockCommand command = command("SAME", 4); // Cùng 1 mã đơn
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

    // Cả 2 lần đều trả về thành công nhưng ID lịch sử chỉ là một!
    assertEquals(one.reservationId(), two.reservationId());
    assertTrue(one.reserved());
    assertTrue(two.reserved());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77)); // Chỉ trừ 4 chiếc, không trừ 8 chiếc
    assertEquals(1, commandRowCount(command.commandId()));
    assertEquals(1, outboxEventCount("InventoryReserved"));
}

@Test
void reusedCommandWithDifferentPayloadIsRejected() {
    UUID commandId = UUID.randomUUID();
    coordinator.reserve(command(commandId, "order-A", 4));

    // Thử lấy mã cũ đi mua cái order khác xem?
    assertThrows(
            IdempotencyMismatchException.class,
            () -> coordinator.reserve(
                    command(commandId, "order-B", 1)
            )
    );
    // Bị chửi văng ra ngoài, kho giữ nguyên
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```

Test chống trùng lặp (Idempotency) là để bảo vệ khách hàng khỏi bị trừ lố, nó khác hoàn toàn với test cập nhật nguyên tử (bảo vệ kho). Ta phải test cả hai!

## Thực nghiệm 7 — Lỗi râu ria kéo theo kho hàng bị Hoàn tác

Cài sẵn một tin nhắn hỏng trong hộp thư, để cố tình gây lỗi sập database sau khi lệnh trừ kho đã chạy:

```java
@Test
void downstreamConstraintFailureRollsBackAtomicUpdate() {
    UUID duplicateEventId = seedOutboxEvent();

    assertThrows(DuplicateKeyException.class, () ->
            inTransaction(() -> {
                inventory.tryReserve(77, 4).orElseThrow(); // Trừ kho ngon lành
                
                // Cố tình nhét trùng khóa ngoại để gây lỗi sập hầm
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

    // Assert: Sập rồi thì kho phải còn y nguyên, cấm trừ bậy!
    assertEquals(5, available(77));
    assertEquals(0, reservedCounter(77));
    assertEquals(10, revision(77));
    assertEquals(0, reservedOutcomeCount(77));
}
```

Kiểm tra số tồn kho trả lại nguyên vẹn mới là ý chính của bài test này, chứ không phải ngồi ngắm xem nó văng ra cái exception gì.

## Thực nghiệm 8 — Bulk DML bắn lén làm JPA ngớ ngẩn

```java
@Test
void directUpdateDoesNotRefreshManagedEntity() {
    inTransaction(() -> {
        // Lôi lên RAM
        InventoryItem managed =
                entityManager.find(InventoryItem.class, 77L);
        assertEquals(5, managed.availableQuantity());

        // Bắn SQL đâm lén sau lưng
        inventory.tryReserve(77, 4).orElseThrow();

        // Object trên RAM chả biết cái khỉ gì, vẫn cứ ngỡ là 5
        assertTrue(entityManager.contains(managed));
        assertEquals(5, managed.availableQuantity());

        // Đuổi nó ra khỏi bộ đệm, bắt tải lại
        entityManager.clear();
        InventoryItem reloaded =
                entityManager.find(InventoryItem.class, 77L);
        assertEquals(1, reloaded.availableQuantity());
        assertEquals(4, reloaded.reservedQuantity());
        return null;
    });
}
```

Bài test này để nhắc nhở những tâm hồn bé bỏng: Trong cái Class xử lý lõi, ĐỪNG CÓ GỌI lệnh `findById` lấy Entity lên làm gì!

## Thực nghiệm 9 — Thử thách cực đại (Stress test) giữ vững tổng kho

```java
@Test
void manyCommandsNeverReserveBeyondOnHand() throws Exception {
    setInventory(77, 20, 0, 20); // Có 20 chiếc

    int actors = 32; // Cho hẳn 32 ông khách giành giật
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
    start.countDown(); // Xông lên!!!

    List<ReservationResult> results = new ArrayList<>();
    long deadline = System.nanoTime() + Duration.ofSeconds(20).toNanos();
    for (Future<ReservationResult> future : futures) {
        long remaining = deadline - System.nanoTime();
        assertTrue(remaining > 0, "global future deadline exceeded");
        results.add(future.get(remaining, TimeUnit.NANOSECONDS));
    }

    // Khốc liệt nhưng kết quả phải chuẩn mực
    assertEquals(20, results.stream()
            .filter(ReservationResult::reserved).count()); // 20 ông mua được
    assertEquals(12, results.stream()
            .filter(ReservationResult::outOfStock).count()); // 12 ông ngậm ngùi về tay trắng
    assertEquals(0, available(77)); // Kho sạch sẽ
    assertEquals(20, reservedCounter(77));
    assertEquals(20, reservedAuditQuantity(77));
    assertEquals(20, outboxEventCount("InventoryReserved"));
    assertTrue(projectionMatchesAudit(77)); // Sổ sách khớp bưng
}
```

Mở cổng `latch` để các ông khách lao vào cắn xé nhau, và test phải sống sót chạy xong, không bị đứng hình.

## Thực nghiệm 10 — Hợp đồng cho trả về "0 dòng"

```java
@Test
void invalidQuantityIsRejectedBeforeSql() {
    assertThrows(
            IllegalArgumentException.class,
            () -> command("invalid", 0) // Mua 0 chiếc
    );
    assertEquals(5, available(77));
}

@Test
void stableExistingProductMapsZeroRowToOutOfStock() {
    setInventory(77, 2, 3, 5);

    ReservationResult result =
            coordinator.reserve(command("too-large", 4)); // Mua lố

    assertTrue(result.outOfStock()); // Chắc chắn bị phán là Hết hàng
    assertEquals(2, available(77));
    assertEquals(3, reservedCounter(77));
    assertEquals(1, outcomeCount("OUT_OF_STOCK"));
    assertEquals(0, outboxEventCount("InventoryReserved"));
}
```

Nếu hệ thống cho phép "sản phẩm bị xóa", hãy test kỹ đoạn bắt mã lỗi `NOT_FOUND`, chứ đừng ném bừa về chung 1 rọ `OUT_OF_STOCK` rồi đi lừa dối người dùng nhé.

## Tổng kết độ phủ sóng của Test (Coverage matrix)

| Bài Thực nghiệm | Kỹ thuật được kiểm chứng | Luật kinh doanh phải giữ |
| --- | --- | --- |
| 1 | Hai luồng đọc phải số liệu cũ | Kho thì đúng mà sổ sách lệch tan nát |
| 2 | Luồng giữ khóa chạy xong, luồng chờ tự xét lại điều kiện | 1 người có dữ liệu trả về, 1 người trả về empty |
| 3 | Luồng giữ khóa lật lọng (hủy bỏ) | Luồng chờ vẫn lấy được hàng như bình thường |
| 4 | Cấu hình `lock_timeout` | Ném lỗi `55P03`, không được lầm với hết hàng |
| 5 | Chạy song song từ đầu đến cuối | 1 thành công, 1 hết hàng |
| 6 | Gửi liên tục trùng mã lệnh | Chỉ trừ kho 1 lần/lưu hộp thư 1 lần, báo về kết quả cũ |
| 7 | Hư bột hư đường khúc cuối | Hoàn tác toàn bộ kho về số ban đầu |
| 8 | JPA vs Cập nhật thẳng (DML) | Object JPA bị ngáo đá cho tới khi xóa (clear) |
| 9 | 32 luồng lao vào tranh 20 món | Đúng 20 ông mua được, sổ kho không mẻ 1 đồng |
| 10 | Đầu vào láo lếu / Hết hàng trả 0 dòng | Kết quả báo ra phải rõ ràng theo hợp đồng |

## Xác minh thực tế trên Production

Hãy xài câu query chọc vào tim PostgreSQL này để theo dõi:

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

Giám sát bằng các chỉ số sương máu:
- Tỷ lệ update bị về `0` dòng / `RETURNING` rỗng tuếch.
- Mã lỗi kẹt khóa `55P03`, `40P01`, sập tuần tự hóa `40001`.
- Thời gian chờ giao dịch / chờ khóa dòng.
- Số luồng Active, Pending và Timeout của hồ bơi kết nối HikariCP.
- Số lượng phát hiện trùng mã lệnh.
- Tốc độ đẻ đơn hàng (throughput).
- Tỷ lệ lỗi lệch sổ đối soát.

Luật thép: Không bao giờ in log thông tin khách hàng ra ngoài. Nếu bị timeout, việc đầu tiên là đổ log database và thread dump ra xem thằng nào đang bị chặn, chứ đừng có hở chút là đi tăng số giây timeout lên một cách ngu xuẩn.

## Bí kíp viết Test không bị chớp giật (Chống flaky)

- Luôn xài các cái chặn (barrier/latch) ngay đúng cái khe nứt dễ dính lỗi nhất.
- Bắt buộc phải xác nhận database "Ok, tao đang khóa thằng kia kìa" rồi mới thả cho luồng khác chạy tiếp.
- Mọi hàm đợi (latch/future) đều PHẢI CÓ thời gian chờ tối đa (bound bound bound!).
- Tạo thread pool bự bự một chút để không bị kẹt vì hết luồng.
- Luôn dọn dẹp tắt hết luồng ngầm trong khối `finally`.
- Dùng ID sinh ngẫu nhiên cho mỗi unit test để tránh rác từ test này chạy lọt qua test kia.
- Chạy test giả lập (stress test) thì tốt, nhưng không được bỏ qua các test canh đúng thời điểm (causal test).

