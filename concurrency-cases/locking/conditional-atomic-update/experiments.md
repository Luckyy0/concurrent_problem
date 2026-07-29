# PostgreSQL experiments cho conditional atomic UPDATE

## Mục tiêu

Test suite cần chứng minh:

1. plain read–check–write làm projection lệch reservation audit;
2. concurrent guarded UPDATE chỉ có một actor thắng khi stock không đủ cho cả hai;
3. waiter re-evaluate predicate sau holder commit;
4. holder rollback cho waiter dùng original stock;
5. lock timeout là technical failure, không phải affected rows `0`;
6. duplicate command decrement đúng một lần;
7. lỗi sau UPDATE rollback counters;
8. bulk DML có stale persistence-context behavior;
9. nhiều actors vẫn giữ conservation invariant.

PostgreSQL semantics phải chạy bằng Testcontainers, không suy luận từ H2.

## Testcontainers fixture

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

Production migration tạo product row `77`. Tests đổi counters trong setup nhưng
không xóa/reinsert target row, nên known-row assumption ổn định.

## Coordination helpers

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

Latches tạo causal order. `pg_stat_activity` xác nhận exact database wait. Mọi
latch, future và executor wait đều có timeout.

## Experiment 1 — Tái hiện lost update và audit mismatch

Test-only broken worker dùng barrier ngay sau plain load:

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
            bothLoaded.await(5, TimeUnit.SECONDS);
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(interrupted);
        } catch (BrokenBarrierException | TimeoutException failure) {
            throw new IllegalStateException(failure);
        }

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

    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(8, reservedAuditQuantity(77));
    assertEquals(2, reservedOutcomeCount(77));
    assertFalse(projectionMatchesAudit(77));
}
```

Row constraints pass; business reconciliation mới phát hiện lỗi.

## Experiment 2 — Commit làm waiter re-evaluate predicate

```java
@Test
void waiterReturnsNoRowAfterHolderCommits() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch allowCommit = new CountDownLatch(1);

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

    Future<Optional<InventoryAfterReserve>> waiter =
            executor.submit(() -> inTransaction(() -> {
                setLocalApplicationName("lock004-waiter");
                return inventory.tryReserve(77, 4);
            }));

    awaitDatabaseBlock("lock004-waiter");
    assertThrows(
            TimeoutException.class,
            () -> waiter.get(100, TimeUnit.MILLISECONDS)
    );

    allowCommit.countDown();

    assertTrue(holder.get(5, TimeUnit.SECONDS).isPresent());
    assertTrue(waiter.get(5, TimeUnit.SECONDS).isEmpty());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(11, revision(77));
}
```

> **Nói ngắn gọn:** `Optional.empty()` chỉ xuất hiện sau khi PostgreSQL chờ,
> nhìn current row `available=1` và đánh giá guard lần nữa.

## Experiment 3 — Rollback cho waiter thắng

```java
@Test
void waiterCanApplyAfterHolderRollsBack() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch forceRollback = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> inTransaction(() -> {
        setLocalApplicationName("lock004-rollback-holder");
        assertTrue(inventory.tryReserve(77, 4).isPresent());
        holderUpdated.countDown();
        awaitLatch(forceRollback);
        throw new RollbackForTest();
    }));

    assertTrue(holderUpdated.await(5, TimeUnit.SECONDS));

    Future<Optional<InventoryAfterReserve>> waiter =
            executor.submit(() -> inTransaction(() -> {
                setLocalApplicationName("lock004-rollback-waiter");
                return inventory.tryReserve(77, 4);
            }));

    awaitDatabaseBlock("lock004-rollback-waiter");
    forceRollback.countDown();

    ExecutionException rolledBack =
            assertThrows(ExecutionException.class,
                    () -> holder.get(5, TimeUnit.SECONDS));
    assertInstanceOf(RollbackForTest.class, rolledBack.getCause());

    InventoryAfterReserve winner =
            waiter.get(5, TimeUnit.SECONDS).orElseThrow();
    assertEquals(1, winner.available());
    assertEquals(4, winner.reserved());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(11, revision(77));
}
```

A uncommitted decrement/revision biến mất; B chỉ increment revision một lần.

## Experiment 4 — Lock timeout khác affected rows `0`

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
                jdbc.queryForObject("""
                        select set_config(
                            'lock_timeout',
                            '150ms',
                            true
                        )
                        """, Map.of(), String.class);
                return inventory.tryReserve(77, 4);
            }));

    ExecutionException failed =
            assertThrows(ExecutionException.class,
                    () -> contender.get(5, TimeUnit.SECONDS));
    assertEquals("55P03", sqlState(failed));

    releaseHolder.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```

Implementation đầy đủ release latch trong `finally` để assertion failure không
giữ task tới timeout. Technical failure không tạo `OUT_OF_STOCK` command row.

## Experiment 5 — End-to-end: một reserve, một rejection

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

Winner có thể là A hoặc B; invariant và outcome cardinality không phụ thuộc
scheduler.

## Experiment 6 — Concurrent duplicate replay

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

    assertEquals(one.reservationId(), two.reservationId());
    assertTrue(one.reserved());
    assertTrue(two.reserved());
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
    assertEquals(1, commandRowCount(command.commandId()));
    assertEquals(1, outboxEventCount("InventoryReserved"));
}

@Test
void reusedCommandWithDifferentPayloadIsRejected() {
    UUID commandId = UUID.randomUUID();
    coordinator.reserve(command(commandId, "order-A", 4));

    assertThrows(
            IdempotencyMismatchException.class,
            () -> coordinator.reserve(
                    command(commandId, "order-B", 1)
            )
    );
    assertEquals(1, available(77));
    assertEquals(4, reservedCounter(77));
}
```

Duplicate test không thay same-product contention test; hai invariants khác nhau.

## Experiment 7 — Lỗi sau UPDATE rollback counter

Pre-seed một outbox `event_id`, rồi cố insert lại ID đó sau mutation:

```java
@Test
void downstreamConstraintFailureRollsBackAtomicUpdate() {
    UUID duplicateEventId = seedOutboxEvent();

    assertThrows(DuplicateKeyException.class, () ->
            inTransaction(() -> {
                inventory.tryReserve(77, 4).orElseThrow();
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

    assertEquals(5, available(77));
    assertEquals(0, reservedCounter(77));
    assertEquals(10, revision(77));
    assertEquals(0, reservedOutcomeCount(77));
}
```

Assertion quan trọng là counter rollback, không chỉ exception type.

## Experiment 8 — Bulk DML làm managed entity stale

```java
@Test
void directUpdateDoesNotRefreshManagedEntity() {
    inTransaction(() -> {
        InventoryItem managed =
                entityManager.find(InventoryItem.class, 77L);
        assertEquals(5, managed.availableQuantity());

        inventory.tryReserve(77, 4).orElseThrow();

        assertTrue(entityManager.contains(managed));
        assertEquals(5, managed.availableQuantity());

        entityManager.clear();
        InventoryItem reloaded =
                entityManager.find(InventoryItem.class, 77L);
        assertEquals(1, reloaded.availableQuantity());
        assertEquals(4, reloaded.reservedQuantity());
        return null;
    });
}
```

Test này không khuyến khích load trước; nó khóa regression cho persistence-context
assumption. Primary service không load inventory entity.

## Experiment 9 — Nhiều actors giữ conservation

```java
@Test
void manyCommandsNeverReserveBeyondOnHand() throws Exception {
    setInventory(77, 20, 0, 20);

    int actors = 32;
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
    start.countDown();

    List<ReservationResult> results = new ArrayList<>();
    long deadline = System.nanoTime() + Duration.ofSeconds(20).toNanos();
    for (Future<ReservationResult> future : futures) {
        long remaining = deadline - System.nanoTime();
        assertTrue(remaining > 0, "global future deadline exceeded");
        results.add(future.get(remaining, TimeUnit.NANOSECONDS));
    }

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

Ready latch chỉ chờ initial wave bằng executor capacity. Sau khi start gate mở,
các queued tasks tiếp tục chạy mà không làm test harness tự starvation.

## Experiment 10 — Zero-row contract

```java
@Test
void invalidQuantityIsRejectedBeforeSql() {
    assertThrows(
            IllegalArgumentException.class,
            () -> command("invalid", 0)
    );
    assertEquals(5, available(77));
}

@Test
void stableExistingProductMapsZeroRowToOutOfStock() {
    setInventory(77, 2, 3, 5);

    ReservationResult result =
            coordinator.reserve(command("too-large", 4));

    assertTrue(result.outOfStock());
    assertEquals(2, available(77));
    assertEquals(3, reservedCounter(77));
    assertEquals(1, outcomeCount("OUT_OF_STOCK"));
    assertEquals(0, outboxEventCount("InventoryReserved"));
}
```

Nếu product có thể missing, thêm test cho documented `NOT_AVAILABLE`/`NOT_FOUND`
contract; không silently reuse out-of-stock assertion.

## Coverage matrix

| Experiment | Cơ chế | Business assertion |
| --- | --- | --- |
| 1 | Hai stale entity loads | Projection lệch audit |
| 2 | Holder commit, waiter recheck | Một returned row, một empty |
| 3 | Holder rollback | Waiter thắng trên original stock |
| 4 | `lock_timeout` | `55P03`, không bị map no-op |
| 5 | Hai end-to-end commands | Một RESERVED, một OUT_OF_STOCK |
| 6 | Concurrent same command | Một decrement/outbox, replay same result |
| 7 | Failure sau UPDATE | Counters rollback |
| 8 | Managed entity + direct DML | Object stale cho tới clear/reload |
| 9 | 32 commands, stock 20 | Đúng 20 accepted, conservation giữ |
| 10 | Input/zero row | Outcome contract rõ |

## Production verification

Theo dõi:

```sql
select application_name,
       state,
       wait_event_type,
       wait_event,
       pg_blocking_pids(pid) as blockers,
       clock_timestamp() - xact_start as transaction_age,
       query
from pg_stat_activity
where datname = current_database()
  and state <> 'idle';
```

- affected rows `1/0` và `RETURNING` empty rate;
- SQLSTATE `55P03`, `40P01`, `40001`;
- transaction/row-lock wait duration;
- HikariCP active, pending và acquisition timeout;
- duplicate replay/fingerprint mismatch;
- reservation/outbox throughput;
- scheduled reconciliation mismatch.

Không log raw customer/order payload hoặc bind values. Khi test timeout, thu
database activity và thread dump trước khi chỉ tăng timeout.

## Chống flaky

- Dùng barrier/latch tại exact race window.
- Xác nhận database lock wait trước khi release holder.
- Mọi latch/future/executor wait có upper bound.
- Executor capacity phải đủ cho ready-barrier semantics.
- Release latches và shutdown executors trong `finally`.
- Reset state và dùng unique command IDs mỗi test.
- Giữ deterministic SQL tests cạnh end-to-end Spring test.
- Stress test bổ sung, không thay thế causal test.
