# Deterministic predicate-capacity experiments

## Mục tiêu

Test suite phải chứng minh:

1. repeated COUNT ở `READ COMMITTED` thấy committed phantom;
2. hai count-then-insert attempts làm final active count `11`;
3. `REPEATABLE READ` giữ snapshot nhưng vẫn không bảo vệ capacity;
4. `SERIALIZABLE` abort một actor và giữ final count `10`;
5. conditional counter chỉ có một winner;
6. parent row lock có bounded loser behavior;
7. `SKIP LOCKED` claim finite slot không cấp cùng unit cho hai actors;
8. rollback không làm counter drift.

Dùng latch/barrier thay cho delay theo wall clock. Mọi `await` và `Future.get`
đều có timeout.

> **Nói ngắn gọn:** regression test phải assert `active <= capacity`, số winners
> và side effects sau commit; chỉ assert exception type là chưa đủ.

## PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
class PhantomCapacityIntegrationTest {

    private static final UUID POOL_ID =
        UUID.fromString("10000000-0000-0000-0000-000000000042");
    private static final UUID EXISTING_REQUEST_ID =
        UUID.fromString("20000000-0000-0000-0000-000000000042");

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("concurrency")
            .withUsername("test")
            .withPassword("test");

    private static HikariDataSource dataSource;
    private ExecutorService executor;

    @BeforeAll
    static void createSchema() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(POSTGRES.getJdbcUrl());
        config.setUsername(POSTGRES.getUsername());
        config.setPassword(POSTGRES.getPassword());
        config.setMaximumPoolSize(8);
        dataSource = new HikariDataSource(config);

        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        jdbc.execute("""
            create table processing_pool (
                pool_id uuid primary key,
                capacity integer not null,
                used_slots integer not null,
                check (
                    capacity >= 0
                    and used_slots >= 0
                    and used_slots <= capacity
                )
            )
            """);
        jdbc.execute("""
            create table slot_allocation (
                allocation_id uuid primary key,
                pool_id uuid not null references processing_pool(pool_id),
                request_id uuid not null,
                status varchar(16) not null,
                unique (pool_id, request_id)
            )
            """);
        jdbc.execute("""
            create table processing_slot (
                pool_id uuid not null references processing_pool(pool_id),
                slot_no integer not null,
                allocation_id uuid,
                primary key (pool_id, slot_no),
                unique (allocation_id)
            )
            """);
    }

    @BeforeEach
    void resetState() {
        executor = Executors.newFixedThreadPool(2);
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        jdbc.update("delete from processing_slot");
        jdbc.update("delete from slot_allocation");
        jdbc.update("delete from processing_pool");
        jdbc.update(
            """
            insert into processing_pool(pool_id, capacity, used_slots)
            values (?, 10, 9)
            """,
            POOL_ID
        );
        for (int number = 1; number <= 9; number++) {
            jdbc.update(
                """
                insert into slot_allocation(
                    allocation_id, pool_id, request_id, status
                ) values (?, ?, ?, 'ACTIVE')
                """,
                UUID.randomUUID(),
                POOL_ID,
                number == 1
                    ? EXISTING_REQUEST_ID
                    : UUID.randomUUID()
            );
        }
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }

    @AfterAll
    static void closePool() {
        dataSource.close();
    }
}
```

Test methods không có outer transaction. Mỗi actor dùng connection riêng.

## Coordination helpers

```java
final class BothCountedGate {
    private final CountDownLatch counted = new CountDownLatch(2);
    private final CountDownLatch mayInsert = new CountDownLatch(1);

    void counted() {
        counted.countDown();
        await(counted, "both predicate reads");
    }

    void releaseInserts() {
        mayInsert.countDown();
    }

    void awaitInsertPermission() {
        await(mayInsert, "insert permission");
    }
}

record AttemptOutcome(
    long observedCount,
    boolean committed,
    String sqlState
) {
}

private static void await(CountDownLatch latch, String step) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Timed out waiting for " + step);
        }
    } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Interrupted while waiting for " + step, ex);
    }
}
```

## JDBC helpers

```java
private static long activeCount(Connection connection) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select count(*)
        from slot_allocation
        where pool_id = ?
          and status = 'ACTIVE'
        """)) {
        statement.setObject(1, POOL_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
            return rs.getLong(1);
        }
    }
}

private static void insertActive(
    Connection connection,
    UUID requestId
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        insert into slot_allocation(
            allocation_id, pool_id, request_id, status
        ) values (?, ?, ?, 'ACTIVE')
        """)) {
        statement.setObject(1, UUID.randomUUID());
        statement.setObject(2, POOL_ID);
        statement.setObject(3, requestId);
        assertThat(statement.executeUpdate()).isEqualTo(1);
    }
}

private static AttemptOutcome brokenAttempt(
    int isolation,
    UUID requestId,
    BothCountedGate gate
) throws SQLException {
    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        connection.setTransactionIsolation(isolation);
        long observed = activeCount(connection);
        gate.counted();
        gate.awaitInsertPermission();

        try {
            insertActive(connection, requestId);
            connection.commit();
            return new AttemptOutcome(observed, true, null);
        } catch (SQLException ex) {
            connection.rollback();
            return new AttemptOutcome(observed, false, ex.getSQLState());
        }
    }
}
```

## Experiment 1 — `READ COMMITTED` thấy committed phantom

```java
@Test
void laterReadCommittedStatementSeesNewMatchingRow() throws Exception {
    CountDownLatch firstCountDone = new CountDownLatch(1);
    CountDownLatch writerCommitted = new CountDownLatch(1);

    Future<List<Long>> reader = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            connection.setTransactionIsolation(
                Connection.TRANSACTION_READ_COMMITTED
            );
            long first = activeCount(connection);
            firstCountDone.countDown();
            await(writerCommitted, "writer commit");
            long second = activeCount(connection);
            connection.commit();
            return List.of(first, second);
        }
    });

    Future<Void> writer = executor.submit(() -> {
        await(firstCountDone, "first count");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            insertActive(connection, UUID.randomUUID());
            connection.commit();
        }
        writerCommitted.countDown();
        return null;
    });

    assertThat(reader.get(5, TimeUnit.SECONDS)).containsExactly(9L, 10L);
    writer.get(5, TimeUnit.SECONDS);
}
```

## Experiment 2 — Broken race commits `11`

```java
@Test
void readCommittedCountThenInsertExceedsCapacity() throws Exception {
    BothCountedGate gate = new BothCountedGate();

    Future<AttemptOutcome> a = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_READ_COMMITTED,
            UUID.randomUUID(),
            gate
        )
    );
    Future<AttemptOutcome> b = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_READ_COMMITTED,
            UUID.randomUUID(),
            gate
        )
    );

    gate.releaseInserts();
    AttemptOutcome resultA = a.get(5, TimeUnit.SECONDS);
    AttemptOutcome resultB = b.get(5, TimeUnit.SECONDS);

    assertThat(resultA.observedCount()).isEqualTo(9);
    assertThat(resultB.observedCount()).isEqualTo(9);
    assertThat(resultA.committed()).isTrue();
    assertThat(resultB.committed()).isTrue();
    assertThat(inspector().activeCount()).isEqualTo(11);
    assertThat(inspector().activeCount())
        .isGreaterThan(inspector().capacity());
}
```

Broken regression cố ý assert violation để reproduction không phụ thuộc xác suất.

## Experiment 3 — `REPEATABLE READ` vẫn vỡ capacity

```java
@Test
void repeatableReadStableSnapshotsDoNotEnforceCapacity() throws Exception {
    BothCountedGate gate = new BothCountedGate();

    Future<AttemptOutcome> a = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_REPEATABLE_READ,
            UUID.randomUUID(),
            gate
        )
    );
    Future<AttemptOutcome> b = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_REPEATABLE_READ,
            UUID.randomUUID(),
            gate
        )
    );

    gate.releaseInserts();
    List<AttemptOutcome> outcomes = List.of(
        a.get(5, TimeUnit.SECONDS),
        b.get(5, TimeUnit.SECONDS)
    );

    assertThat(outcomes).allSatisfy(outcome -> {
        assertThat(outcome.observedCount()).isEqualTo(9);
        assertThat(outcome.committed()).isTrue();
    });
    assertThat(inspector().activeCount()).isEqualTo(11);
}
```

Test này không gọi kết quả là visible phantom; mỗi transaction không thấy row mới
của actor kia trong stable snapshot. Assertion tập trung vào final invariant.

## Experiment 4 — `SERIALIZABLE` abort một actor

```java
@Test
void serializableTurnsPredicateRaceIntoRetryableAbort() throws Exception {
    BothCountedGate gate = new BothCountedGate();

    Future<AttemptOutcome> a = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_SERIALIZABLE,
            UUID.randomUUID(),
            gate
        )
    );
    Future<AttemptOutcome> b = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_SERIALIZABLE,
            UUID.randomUUID(),
            gate
        )
    );

    gate.releaseInserts();
    List<AttemptOutcome> outcomes = List.of(
        a.get(5, TimeUnit.SECONDS),
        b.get(5, TimeUnit.SECONDS)
    );

    assertThat(outcomes.stream().filter(AttemptOutcome::committed).count())
        .isEqualTo(1);
    assertThat(outcomes.stream()
        .filter(outcome -> "40001".equals(outcome.sqlState()))
        .count()).isEqualTo(1);
    assertThat(inspector().activeCount()).isEqualTo(10);
    assertThat(inspector().activeCount())
        .isLessThanOrEqualTo(inspector().capacity());
}
```

Không assert A hay B là loser. Production retry bắt đầu transaction mới; test này
chỉ kiểm tra database conflict contract.

## Experiment 5 — Atomic counter có đúng một winner

```java
record ClaimOutcome(boolean accepted, int affectedRows) {
}

private static ClaimOutcome atomicClaim(
    UUID requestId,
    CountDownLatch ready,
    CountDownLatch start
) throws SQLException {
    ready.countDown();
    await(start, "atomic claim start");

    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        try (PreparedStatement claim = connection.prepareStatement("""
            update processing_pool
               set used_slots = used_slots + 1
             where pool_id = ?
               and used_slots < capacity
            """)) {
            claim.setObject(1, POOL_ID);
            int affected = claim.executeUpdate();
            if (affected == 1) {
                insertActive(connection, requestId);
            }
            connection.commit();
            return new ClaimOutcome(affected == 1, affected);
        } catch (SQLException ex) {
            connection.rollback();
            throw ex;
        }
    }
}

@Test
void conditionalCounterAcceptsOnlyOneConcurrentRequest() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Future<ClaimOutcome> a = executor.submit(
        () -> atomicClaim(UUID.randomUUID(), ready, start)
    );
    Future<ClaimOutcome> b = executor.submit(
        () -> atomicClaim(UUID.randomUUID(), ready, start)
    );

    await(ready, "both claimers ready");
    start.countDown();
    List<ClaimOutcome> outcomes = List.of(
        a.get(5, TimeUnit.SECONDS),
        b.get(5, TimeUnit.SECONDS)
    );

    assertThat(outcomes.stream().filter(ClaimOutcome::accepted).count())
        .isEqualTo(1);
    assertThat(outcomes.stream().mapToInt(ClaimOutcome::affectedRows).sum())
        .isEqualTo(1);
    assertThat(inspector().usedSlots()).isEqualTo(10);
    assertThat(inspector().activeCount()).isEqualTo(10);
}
```

## Experiment 6 — Parent lock có bounded loser

```java
@Test
void parentRowLockMakesSecondAllocatorWaitOrTimeout() throws Exception {
    CountDownLatch parentLocked = new CountDownLatch(1);
    CountDownLatch contenderFinished = new CountDownLatch(1);

    Future<Void> owner = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            lockParent(connection);
            parentLocked.countDown();
            await(contenderFinished, "contender timeout");
            connection.commit();
        }
        return null;
    });

    Future<String> contender = executor.submit(() -> {
        await(parentLocked, "parent lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (Statement statement = connection.createStatement()) {
                statement.execute("set local lock_timeout = '300ms'");
            }
            try {
                lockParent(connection);
                connection.commit();
                return "unexpected-lock";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                contenderFinished.countDown();
            }
        }
    });

    assertThat(contender.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    owner.get(5, TimeUnit.SECONDS);
}
```

Một test service-level khác cho owner commit allocation rồi contender retry/recount
phải assert first `ACCEPTED`, second `FULL`, final active `10`.

## Experiment 7 — `SKIP LOCKED` không cấp cùng finite slot

```java
@Test
void skipLockedReturnsNoCandidateWhileOnlySlotIsClaimed() throws Exception {
    JdbcTemplate jdbc = new JdbcTemplate(dataSource);
    jdbc.update(
        """
        insert into processing_slot(pool_id, slot_no, allocation_id)
        values (?, 1, null)
        """,
        POOL_ID
    );

    CountDownLatch slotLocked = new CountDownLatch(1);
    CountDownLatch contenderDone = new CountDownLatch(1);
    UUID allocationId = UUID.randomUUID();

    Future<OptionalInt> owner = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            OptionalInt slot = selectFreeSlotSkipLocked(connection);
            slotLocked.countDown();
            await(contenderDone, "slot contender");
            assignSlot(connection, slot.orElseThrow(), allocationId);
            connection.commit();
            return slot;
        }
    });

    Future<OptionalInt> contender = executor.submit(() -> {
        await(slotLocked, "slot lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            OptionalInt slot = selectFreeSlotSkipLocked(connection);
            connection.commit();
            contenderDone.countDown();
            return slot;
        }
    });

    assertThat(contender.get(5, TimeUnit.SECONDS)).isEmpty();
    assertThat(owner.get(5, TimeUnit.SECONDS)).hasValue(1);
    assertThat(inspector().assignedSlotCount()).isEqualTo(1);
}
```

`selectFreeSlotSkipLocked()` dùng đúng query `FOR UPDATE SKIP LOCKED LIMIT 1` từ
solutions; row lock sống tới connection commit/rollback.

## Experiment 8 — Claim và INSERT rollback cùng nhau

```java
@Test
void failedAllocationRollsBackCounterClaim() {
    JdbcTemplate jdbc = new JdbcTemplate(dataSource);
    TransactionTemplate transactionTemplate = new TransactionTemplate(
        new DataSourceTransactionManager(dataSource)
    );

    assertThatThrownBy(() -> transactionTemplate.executeWithoutResult(status -> {
        int claimed = jdbc.update(
            """
            update processing_pool
               set used_slots = used_slots + 1
             where pool_id = ?
               and used_slots < capacity
            """,
            POOL_ID
        );
        assertThat(claimed).isEqualTo(1);

        jdbc.update(
            """
            insert into slot_allocation(
                allocation_id, pool_id, request_id, status
            ) values (?, ?, ?, 'ACTIVE')
            """,
            UUID.randomUUID(),
            POOL_ID,
            EXISTING_REQUEST_ID
        );
    })).isInstanceOf(DataIntegrityViolationException.class);

    assertThat(inspector().usedSlots()).isEqualTo(9);
    assertThat(inspector().activeCount()).isEqualTo(9);
}
```

`EXISTING_REQUEST_ID` được seed trước test để unique violation xảy ra trong cùng
transaction. Không catch exception bên trong transaction rồi commit.

## Helpers còn lại

```java
private static void lockParent(Connection connection) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select pool_id
        from processing_pool
        where pool_id = ?
        for update
        """)) {
        statement.setObject(1, POOL_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
        }
    }
}

private static OptionalInt selectFreeSlotSkipLocked(Connection connection)
    throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select slot_no
        from processing_slot
        where pool_id = ?
          and allocation_id is null
        order by slot_no
        for update skip locked
        limit 1
        """)) {
        statement.setObject(1, POOL_ID);
        try (ResultSet rs = statement.executeQuery()) {
            return rs.next()
                ? OptionalInt.of(rs.getInt(1))
                : OptionalInt.empty();
        }
    }
}

private static void assignSlot(
    Connection connection,
    int slotNo,
    UUID allocationId
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        update processing_slot
           set allocation_id = ?
         where pool_id = ?
           and slot_no = ?
           and allocation_id is null
        """)) {
        statement.setObject(1, allocationId);
        statement.setObject(2, POOL_ID);
        statement.setInt(3, slotNo);
        assertThat(statement.executeUpdate()).isEqualTo(1);
    }
}
```

`inspector()` dùng `JdbcTemplate` mới ngoài actor transactions để chỉ đọc state đã
commit: `capacity`, `used_slots`, active count và assigned slot count.

## Coverage matrix

| Experiment | Mechanism | Controlled order | Invariant assertion |
| --- | --- | --- | --- |
| 1 | RC statement snapshots | B commit giữa COUNTs | `9 -> 10` |
| 2 | RC broken race | both count before insert | final `11` |
| 3 | RR stable snapshots | both count before insert | both commit, final `11` |
| 4 | SERIALIZABLE SSI | both predicate-read before insert | one `40001`, final `10` |
| 5 | Conditional counter | simultaneous UPDATE | one affected row, final `10` |
| 6 | Parent `FOR UPDATE` | owner holds row lock | contender `55P03` |
| 7 | `SKIP LOCKED` slots | owner holds only slot | contender gets empty |
| 8 | Atomic rollback | counter claim before failed INSERT | counter/rows stay `9` |

## Chống flaky

- Không dùng wall-clock timing để đặt interleaving.
- Mọi latch/future có timeout và restore interrupt flag.
- Actors có connections/transactions riêng; không outer test transaction.
- Test class chạy `SAME_THREAD` vì reset shared database state.
- SERIALIZABLE test assert winner count/SQLSTATE, không actor identity.
- Timeout thu thập `pg_stat_activity`, `pg_locks` và thread dump.
- H2 không được dùng làm bằng chứng predicate/MVCC/SSI.

## Production verification

Reconciliation:

```sql
select p.pool_id,
       p.capacity,
       p.used_slots,
       count(a.*) filter (where a.status = 'ACTIVE') as active_count
from processing_pool p
left join slot_allocation a on a.pool_id = p.pool_id
group by p.pool_id, p.capacity, p.used_slots
having count(a.*) filter (where a.status = 'ACTIVE') > p.capacity
    or p.used_slots <>
       count(a.*) filter (where a.status = 'ACTIVE');
```

Lock/SSI diagnostics:

```sql
select pid, state, wait_event_type, wait_event, xact_start, query
from pg_stat_activity
where datname = current_database();

select pid, locktype, mode, granted, relation::regclass
from pg_locks
where relation in (
    'processing_pool'::regclass,
    'slot_allocation'::regclass,
    'processing_slot'::regclass
);
```

Theo dõi affected-row `0`, `40001`, `55P03`, retry attempts, transaction duration,
active/capacity drift và duplicate replays.
