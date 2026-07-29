# PostgreSQL experiments cho pessimistic write lock

## Mục tiêu

Test suite phải chứng minh cả safety lẫn progress:

1. plain `SELECT` cho phép hai decision cùng thấy `AVAILABLE`;
2. `FOR UPDATE` làm competitor block rồi revalidate committed state;
3. rollback release lock và cho waiter tiếp tục;
4. `lock_timeout` tạo bounded failure, không làm partial write;
5. plain reader vẫn đọc committed version khi holder chưa commit;
6. multi-row operation dùng stable order;
7. Spring/JPA path giữ invariant trên PostgreSQL thật.

Không dùng H2 để suy luận row-lock behavior.

## Testcontainers fixture

```java
@Testcontainers
@SpringBootTest
class PessimisticSeatHoldIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17-alpine");

    @DynamicPropertySource
    static void datasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
    }

    @Autowired JdbcTemplate jdbc;
    @Autowired PlatformTransactionManager transactionManager;
    @Autowired SeatHoldCoordinator coordinator;

    private ExecutorService executor;

    @BeforeEach
    void reset() {
        executor = Executors.newFixedThreadPool(4);
        jdbc.update("delete from seat_hold");
        jdbc.update("""
                update show_seat
                set state = 'AVAILABLE',
                    hold_id = null,
                    holder_customer_id = null,
                    hold_until = null
                where show_id = 42 and seat_no in ('A-10', 'A-11')
                """);
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Migration thật tạo hai seat rows cố định và các constraints trong
`solutions.md`. Mỗi test dùng command IDs mới để không vô tình đi vào replay
path.

## Helper transaction và bounded wait

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

private void useApplicationName(String name) {
    jdbc.queryForObject(
            "select set_config('application_name', ?, true)",
            String.class,
            name
    );
}

private void awaitDatabaseBlock(String applicationName) {
    long deadline = System.nanoTime() + Duration.ofSeconds(5).toNanos();

    while (System.nanoTime() < deadline) {
        Boolean blocked = jdbc.queryForObject("""
                select exists (
                    select 1
                    from pg_stat_activity
                    where application_name = ?
                      and wait_event_type = 'Lock'
                      and cardinality(pg_blocking_pids(pid)) > 0
                )
                """, Boolean.class, applicationName);
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

`CountDownLatch` tạo event order; PostgreSQL views xác nhận contender đang block
thật. Mọi `Future.get`, latch wait và executor shutdown đều có timeout.

## Experiment 1 — Tái hiện stale decision

Test-only broken worker dừng hai actors ngay sau plain load:

```java
@Service
class BarrierBrokenSeatHoldTx {
    private final ShowSeatRepository seats;
    private final SeatHoldRepository holds;
    private final CyclicBarrier bothLoaded = new CyclicBarrier(2);

    // Constructor injection omitted.

    @Transactional
    public UUID hold(HoldSeatCommand command) {
        ShowSeat seat = seats.findById(command.seatId()).orElseThrow();
        assertEquals(SeatState.AVAILABLE, seat.state());

        try {
            bothLoaded.await(5, TimeUnit.SECONDS);
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(interrupted);
        } catch (BrokenBarrierException | TimeoutException failure) {
            throw new IllegalStateException(failure);
        }

        UUID holdId = UUID.randomUUID();
        seat.hold(holdId, command.customerId(), Instant.now().plusSeconds(120));
        holds.save(SeatHold.active(holdId, command));
        return holdId;
    }
}
```

```java
@Test
void plainLoadLetsTwoBusinessDecisionsPass() throws Exception {
    // Experiment này chạy legacy schema chưa có uq_seat_hold_one_active.
    Future<UUID> a = executor.submit(
            () -> brokenWorker.hold(command("hold-a", 501))
    );
    Future<UUID> b = executor.submit(
            () -> brokenWorker.hold(command("hold-b", 902))
    );

    UUID holdA = a.get(10, TimeUnit.SECONDS);
    UUID holdB = b.get(10, TimeUnit.SECONDS);

    assertNotEquals(holdA, holdB);
    assertEquals(2, activeHoldCount(42, "A-10"));
    assertEquals(1, seatProjectionCount(42, "A-10"));
    assertTrue(currentSeatHoldId(42, "A-10").equals(holdA)
            || currentSeatHoldId(42, "A-10").equals(holdB));
}
```

Assertion chính là business invariant bị phá, không phải chỉ “hai threads đã
chạy”.

## Experiment 2 — Holder commit, waiter revalidate

Holder cập nhật row nhưng dừng trước commit. Contender dùng cùng locking SQL:

```java
@Test
void waiterBlocksThenRejectsCommittedHold() throws Exception {
    CountDownLatch holderHasLock = new CountDownLatch(1);
    CountDownLatch allowHolderCommit = new CountDownLatch(1);

    Future<HoldOutcome> holder = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-holder");
        lockSeat(42, "A-10");
        createHold("hold-a", 501);
        holderHasLock.countDown();
        awaitLatch(allowHolderCommit);
        return HoldOutcome.HELD;
    }));

    assertTrue(holderHasLock.await(5, TimeUnit.SECONDS));

    Future<HoldOutcome> waiter = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-waiter");
        ShowSeatRow seat = lockSeat(42, "A-10");
        return seat.state().equals("AVAILABLE")
                ? createHoldAndReturn("hold-b", 902)
                : HoldOutcome.ALREADY_HELD;
    }));

    awaitDatabaseBlock("lock003-waiter");
    assertThrows(
            TimeoutException.class,
            () -> waiter.get(100, TimeUnit.MILLISECONDS)
    );

    allowHolderCommit.countDown();

    assertEquals(HoldOutcome.HELD,
            holder.get(5, TimeUnit.SECONDS));
    assertEquals(HoldOutcome.ALREADY_HELD,
            waiter.get(5, TimeUnit.SECONDS));
    assertEquals(1, activeHoldCount(42, "A-10"));
    assertEquals(501L, currentSeatCustomer(42, "A-10"));
}
```

> **Nói ngắn gọn:** test không chỉ thấy waiter chậm; nó chứng minh waiter bị
> PostgreSQL lock chặn, sau đó đọc outcome mới và không tạo side effect thứ hai.

## Experiment 3 — Rollback release lock

```java
@Test
void waiterCanWinAfterHolderRollsBack() throws Exception {
    CountDownLatch holderHasLock = new CountDownLatch(1);
    CountDownLatch forceRollback = new CountDownLatch(1);

    Future<HoldOutcome> holder = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-rollback-holder");
        lockSeat(42, "A-10");
        createHold("hold-a", 501);
        holderHasLock.countDown();
        awaitLatch(forceRollback);
        throw new RollbackForTest();
    }));

    assertTrue(holderHasLock.await(5, TimeUnit.SECONDS));

    Future<HoldOutcome> waiter = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-rollback-waiter");
        ShowSeatRow seat = lockSeat(42, "A-10");
        assertEquals("AVAILABLE", seat.state());
        return createHoldAndReturn("hold-b", 902);
    }));

    awaitDatabaseBlock("lock003-rollback-waiter");
    forceRollback.countDown();

    ExecutionException rolledBack =
            assertThrows(ExecutionException.class,
                    () -> holder.get(5, TimeUnit.SECONDS));
    assertInstanceOf(RollbackForTest.class, rolledBack.getCause());

    assertEquals(HoldOutcome.HELD,
            waiter.get(5, TimeUnit.SECONDS));
    assertEquals(1, activeHoldCount(42, "A-10"));
    assertEquals(902L, currentSeatCustomer(42, "A-10"));
    assertEquals(0, holdCountByCommand("hold-a"));
}
```

Assertions chứng minh cả uncommitted projection lẫn audit insert của holder đã
rollback.

## Experiment 4 — `lock_timeout` tạo bounded failure

```java
@Test
void lockTimeoutRollsBackWithoutPartialHold() throws Exception {
    CountDownLatch holderHasLock = new CountDownLatch(1);
    CountDownLatch releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-timeout-holder");
        lockSeat(42, "A-10");
        holderHasLock.countDown();
        awaitLatch(releaseHolder);
        return null;
    }));

    assertTrue(holderHasLock.await(5, TimeUnit.SECONDS));

    Future<HoldOutcome> contender = executor.submit(() ->
            inTransaction(() -> {
                useApplicationName("lock003-timeout-contender");
                jdbc.queryForObject(
                        "select set_config('lock_timeout', '150ms', true)",
                        String.class
                );
                lockSeat(42, "A-10");
                return createHoldAndReturn("hold-b", 902);
            })
    );

    ExecutionException failed =
            assertThrows(ExecutionException.class,
                    () -> contender.get(5, TimeUnit.SECONDS));
    assertEquals("55P03", sqlState(failed));
    assertEquals(0, holdCountByCommand("hold-b"));
    assertEquals("AVAILABLE", currentSeatState(42, "A-10"));

    releaseHolder.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
}
```

Test phải release holder trong `finally` ở implementation đầy đủ để failure của
một assertion không để task treo. Sau timeout, framework rollback transaction
trước một retry/mapping mới.

## Experiment 5 — Plain reader không bị row lock chặn

```java
@Test
void plainReaderSeesLastCommittedVersion() throws Exception {
    CountDownLatch uncommittedUpdateReady = new CountDownLatch(1);
    CountDownLatch allowCommit = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> inTransaction(() -> {
        lockSeat(42, "A-10");
        createHold("hold-a", 501);
        uncommittedUpdateReady.countDown();
        awaitLatch(allowCommit);
        return null;
    }));

    assertTrue(uncommittedUpdateReady.await(5, TimeUnit.SECONDS));

    Future<String> reader = executor.submit(
            () -> currentSeatState(42, "A-10")
    );
    assertEquals("AVAILABLE", reader.get(2, TimeUnit.SECONDS));

    allowCommit.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
    assertEquals("HELD", currentSeatState(42, "A-10"));
    assertEquals(1, activeHoldCount(42, "A-10"));
}
```

Nếu dashboard chỉ cần last committed value, plain SELECT tránh biến read traffic
thành lock waiters.

## Experiment 6 — Spring/JPA end-to-end invariant

```java
@Test
void twoJpaRequestsProduceOneHoldAndOneRejection() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Callable<HoldResult> actorA = () -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return coordinator.hold(command("hold-a", 501));
    };
    Callable<HoldResult> actorB = () -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return coordinator.hold(command("hold-b", 902));
    };

    Future<HoldResult> a = executor.submit(actorA);
    Future<HoldResult> b = executor.submit(actorB);

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    List<HoldResult> results = List.of(
            a.get(10, TimeUnit.SECONDS),
            b.get(10, TimeUnit.SECONDS)
    );

    assertEquals(1, results.stream().filter(HoldResult::held).count());
    assertEquals(1,
            results.stream().filter(HoldResult::alreadyHeld).count());
    assertEquals(1, activeHoldCount(42, "A-10"));
    assertTrue(projectionMatchesActiveHold(42, "A-10"));
}
```

Test chạy nhiều lần có thể thay winner nhưng invariant và outcome cardinality
không đổi. Deterministic raw-SQL experiments phía trên giải thích exact blocking
mechanism; test này bảo vệ wiring Spring proxy/Hibernate.

## Experiment 7 — Stable order cho group hold

```java
@Test
void reverseInputUsesSameDatabaseLockOrder() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Future<GroupHoldResult> x = executor.submit(() -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return groupService.hold(List.of("A-10", "A-11"), commandId());
    });
    Future<GroupHoldResult> y = executor.submit(() -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return groupService.hold(List.of("A-11", "A-10"), commandId());
    });

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    List<GroupHoldResult> results = List.of(
            x.get(10, TimeUnit.SECONDS),
            y.get(10, TimeUnit.SECONDS)
    );

    assertEquals(1, results.stream().filter(GroupHoldResult::held).count());
    assertEquals(1,
            results.stream().filter(GroupHoldResult::unavailable).count());
    assertEquals(2, activeHoldCountForSeats("A-10", "A-11"));
    assertEquals(0, deadlockCountForTestRun());
}
```

Ngoài assertions, SQL capture phải cho thấy cả calls dùng
`ORDER BY show_id, seat_no FOR UPDATE`. Không biến “không thấy deadlock trong một
run” thành bằng chứng duy nhất; audit query/order là primary guarantee.

## Experiment 8 — Duplicate replay

```java
@Test
void sameCommandReplaysCommittedHold() {
    UUID commandId = UUID.randomUUID();
    HoldSeatCommand first = command(commandId, 501, "A-10");

    HoldResult accepted = coordinator.hold(first);
    HoldResult replayed = coordinator.hold(first);

    assertTrue(accepted.held());
    assertTrue(replayed.replayed());
    assertEquals(accepted.holdId(), replayed.holdId());
    assertEquals(1, holdCountByCommand(commandId));
    assertEquals(1, activeHoldCount(42, "A-10"));
}

@Test
void reusedCommandWithDifferentCustomerIsRejected() {
    UUID commandId = UUID.randomUUID();
    coordinator.hold(command(commandId, 501, "A-10"));

    assertThrows(
            IdempotencyMismatchException.class,
            () -> coordinator.hold(command(commandId, 902, "A-10"))
    );
    assertEquals(1, holdCountByCommand(commandId));
}
```

Replay protection không thay assertions same-seat contention.

## Core JDBC helper

```java
private ShowSeatRow lockSeat(long showId, String seatNo) {
    return jdbc.queryForObject("""
            select show_id, seat_no, state, hold_id, holder_customer_id
            from show_seat
            where show_id = ? and seat_no = ?
            for update
            """, SHOW_SEAT_ROW_MAPPER, showId, seatNo);
}
```

Test helper phải fail nếu row không tồn tại; zero-row query không chứng minh lock
đã được acquire.

## Coverage matrix

| Experiment | Interleaving được ép | Business assertion |
| --- | --- | --- |
| 1 | Hai plain loads trước flush | Hai ACTIVE holds tái hiện invariant violation |
| 2 | B wait khi A giữ lock | Một HELD, một ALREADY_HELD |
| 3 | A rollback trước B acquire | Chỉ hold B tồn tại |
| 4 | B vượt `lock_timeout` | `55P03`, không partial hold |
| 5 | Plain reader khi A uncommitted | Đọc old committed rồi thấy new committed |
| 6 | Hai JPA calls cùng bắt đầu | Đúng một accepted hold |
| 7 | Reverse group input | Stable order, một group winner |
| 8 | Duplicate command | Replay cùng response, không duplicate |

## Production verification

Theo dõi đồng thời:

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

- Hibernate SQL/`StatementInspector`: locking query có clause phù hợp.
- SQLSTATE counters: `55P03` riêng với `40P01`.
- Lock wait và transaction duration histograms.
- HikariCP active, idle, pending và acquisition timeout.
- Domain outcomes theo hot seat/show.
- Reconciliation query giữa `show_seat.hold_id` và ACTIVE `seat_hold`.

Không log customer/payment data hoặc raw bind values. Khi test timeout, thu
`pg_stat_activity`, `pg_locks`, `pg_blocking_pids` và thread dump trước khi chỉ
tăng timeout.

## Chống flaky

- Dùng latch/barrier để tạo causal order, không đoán scheduler timing.
- Xác nhận `wait_event_type = 'Lock'` trước khi release holder.
- Mọi wait/future đều có upper bound.
- Release latches và shutdown executor trong `finally`.
- Mỗi test reset rows và dùng unique command IDs.
- Giữ raw lock-semantic test cạnh Spring/JPA integration test.
- Không dùng một stress loop pass làm bằng chứng correctness.
