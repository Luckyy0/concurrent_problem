# Deterministic write-skew experiments

## Mục tiêu

Test suite chứng minh:

1. `READ COMMITTED` và `REPEATABLE READ` đều có thể commit write skew;
2. two version predicates cùng affected-row `1`;
3. `SERIALIZABLE` abort một actor với `40001`, final count giữ `1`;
4. roster guard lock serialize decisions;
5. conditional counter có đúng một successful decrement;
6. rollback không để assignment/counter dở dang.

Latch buộc cả actors đọc count trước khi update. Mọi wait/future có timeout; test
không dựa vào wall-clock delay.

> **Nói ngắn gọn:** test đúng phải assert both technical outcomes và final
> roster invariant, vì write skew thường không tạo exception ở isolation thấp hơn.

## PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
class WriteSkewIntegrationTest {

    private static final UUID ROSTER_ID =
        UUID.fromString("10000000-0000-0000-0000-000000000042");
    private static final UUID ALICE =
        UUID.fromString("20000000-0000-0000-0000-000000000001");
    private static final UUID BOB =
        UUID.fromString("20000000-0000-0000-0000-000000000002");

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
            create table on_call_roster (
                roster_id uuid primary key,
                name varchar(100) not null,
                active boolean not null,
                on_call_count integer not null,
                check (on_call_count >= 1)
            )
            """);
        jdbc.execute("""
            create table on_call_assignment (
                assignment_id uuid primary key,
                roster_id uuid not null references on_call_roster(roster_id),
                operator_id uuid not null,
                on_call boolean not null,
                version bigint not null,
                unique (roster_id, operator_id)
            )
            """);
    }

    @BeforeEach
    void resetState() {
        executor = Executors.newFixedThreadPool(2);
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        jdbc.update("delete from on_call_assignment");
        jdbc.update("delete from on_call_roster");
        jdbc.update(
            """
            insert into on_call_roster(
                roster_id, name, active, on_call_count
            ) values (?, 'NOC-NIGHT-42', true, 2)
            """,
            ROSTER_ID
        );
        insertAssignment(jdbc, ALICE);
        insertAssignment(jdbc, BOB);
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }

    @AfterAll
    static void closeDataSource() {
        dataSource.close();
    }
}
```

Test methods không có outer transaction; mỗi actor sở hữu JDBC connection riêng.

## Gate và outcome

```java
final class BothObservedGate {
    private final CountDownLatch observed = new CountDownLatch(2);
    private final CountDownLatch mayUpdate = new CountDownLatch(1);

    void observedInvariant() {
        observed.countDown();
        await(observed, "both invariant reads");
    }

    void releaseUpdates() {
        mayUpdate.countDown();
    }

    void awaitUpdatePermission() {
        await(mayUpdate, "update permission");
    }
}

record LeaveOutcome(
    long observedCount,
    int affectedRows,
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

## Broken actor helper

```java
private static LeaveOutcome runBrokenLeave(
    int isolation,
    UUID operatorId,
    BothObservedGate gate
) throws SQLException {
    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        connection.setTransactionIsolation(isolation);
        long observed = countOnCall(connection);
        gate.observedInvariant();
        gate.awaitUpdatePermission();

        try {
            int affected = markOffCall(connection, operatorId);
            connection.commit();
            return new LeaveOutcome(observed, affected, true, null);
        } catch (SQLException ex) {
            connection.rollback();
            return new LeaveOutcome(observed, 0, false, ex.getSQLState());
        }
    }
}

private static long countOnCall(Connection connection) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select count(*)
        from on_call_assignment
        where roster_id = ?
          and on_call
        """)) {
        statement.setObject(1, ROSTER_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
            return rs.getLong(1);
        }
    }
}

private static int markOffCall(
    Connection connection,
    UUID operatorId
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        update on_call_assignment
           set on_call = false,
               version = version + 1
         where roster_id = ?
           and operator_id = ?
           and on_call
           and version = 0
        """)) {
        statement.setObject(1, ROSTER_ID);
        statement.setObject(2, operatorId);
        return statement.executeUpdate();
    }
}
```

## Experiment 1 — `REPEATABLE READ` commits write skew

```java
@Test
void repeatableReadAllowsDifferentRowWriteSkew() throws Exception {
    List<LeaveOutcome> outcomes = runBrokenRace(
        Connection.TRANSACTION_REPEATABLE_READ
    );

    assertThat(outcomes).allSatisfy(outcome -> {
        assertThat(outcome.observedCount()).isEqualTo(2);
        assertThat(outcome.affectedRows()).isEqualTo(1);
        assertThat(outcome.committed()).isTrue();
    });
    assertThat(committedOnCallCount()).isZero();
}
```

Hai affected-row `1` chứng minh `@Version`-style predicates không conflict.

## Experiment 2 — `READ COMMITTED` cũng vỡ invariant

```java
@Test
void readCommittedRaceAlsoLeavesNoOperator() throws Exception {
    List<LeaveOutcome> outcomes = runBrokenRace(
        Connection.TRANSACTION_READ_COMMITTED
    );

    assertThat(outcomes.stream().filter(LeaveOutcome::committed).count())
        .isEqualTo(2);
    assertThat(committedOnCallCount()).isZero();
}
```

## Experiment 3 — `SERIALIZABLE` abort một actor

```java
@Test
void serializableKeepsAtLeastOneOperator() throws Exception {
    List<LeaveOutcome> outcomes = runBrokenRace(
        Connection.TRANSACTION_SERIALIZABLE
    );

    assertThat(outcomes.stream().filter(LeaveOutcome::committed).count())
        .isEqualTo(1);
    assertThat(outcomes.stream()
        .filter(outcome -> "40001".equals(outcome.sqlState()))
        .count()).isEqualTo(1);
    assertThat(committedOnCallCount()).isEqualTo(1);
}
```

Production retry của `40001` phải mở transaction mới; khi recount thấy `1`, loser
trả `LAST_OPERATOR_REQUIRED`.

## Shared broken-race runner

```java
private List<LeaveOutcome> runBrokenRace(int isolation) throws Exception {
    BothObservedGate gate = new BothObservedGate();
    Future<LeaveOutcome> alice = executor.submit(
        () -> runBrokenLeave(isolation, ALICE, gate)
    );
    Future<LeaveOutcome> bob = executor.submit(
        () -> runBrokenLeave(isolation, BOB, gate)
    );

    gate.releaseUpdates();
    return List.of(
        alice.get(5, TimeUnit.SECONDS),
        bob.get(5, TimeUnit.SECONDS)
    );
}
```

`observedInvariant()` vẫn buộc both COUNT hoàn tất trước either UPDATE dù
`releaseUpdates()` được gọi sớm.

## Experiment 4 — Guard row serialize quyết định

```java
record GuardedOutcome(boolean accepted, long observedCount) {
}

private static GuardedOutcome guardedLeave(
    UUID operatorId,
    CountDownLatch ready,
    CountDownLatch start
) throws SQLException {
    ready.countDown();
    await(start, "guarded start");

    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        try {
            lockRoster(connection);
            long count = countOnCall(connection);
            if (count <= 1) {
                connection.commit();
                return new GuardedOutcome(false, count);
            }
            assertThat(markOffCall(connection, operatorId)).isEqualTo(1);
            connection.commit();
            return new GuardedOutcome(true, count);
        } catch (SQLException ex) {
            connection.rollback();
            throw ex;
        }
    }
}

@Test
void guardRowAllowsOneLeaveAndRejectsTheOther() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Future<GuardedOutcome> alice = executor.submit(
        () -> guardedLeave(ALICE, ready, start)
    );
    Future<GuardedOutcome> bob = executor.submit(
        () -> guardedLeave(BOB, ready, start)
    );

    await(ready, "both guarded actors");
    start.countDown();
    List<GuardedOutcome> outcomes = List.of(
        alice.get(5, TimeUnit.SECONDS),
        bob.get(5, TimeUnit.SECONDS)
    );

    assertThat(outcomes.stream().filter(GuardedOutcome::accepted).count())
        .isEqualTo(1);
    assertThat(outcomes.stream().map(GuardedOutcome::observedCount))
        .containsExactlyInAnyOrder(2L, 1L);
    assertThat(committedOnCallCount()).isEqualTo(1);
}
```

## Experiment 5 — Guard lock timeout bounded

```java
@Test
void contenderGetsBoundedLockTimeout() throws Exception {
    CountDownLatch guardHeld = new CountDownLatch(1);
    CountDownLatch contenderDone = new CountDownLatch(1);

    Future<Void> owner = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            lockRoster(connection);
            guardHeld.countDown();
            await(contenderDone, "guard contender");
            connection.commit();
        }
        return null;
    });

    Future<String> contender = executor.submit(() -> {
        await(guardHeld, "guard held");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (Statement statement = connection.createStatement()) {
                statement.execute("set local lock_timeout = '300ms'");
            }
            try {
                lockRoster(connection);
                connection.commit();
                return "unexpected-lock";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                contenderDone.countDown();
            }
        }
    });

    assertThat(contender.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    owner.get(5, TimeUnit.SECONDS);
}
```

## Experiment 6 — Conditional counter có một winner

Hai actors update own rows rồi cùng decrement roster counter. Loser rollback own
row khi affected-row `0`:

```java
@Test
void conditionalCounterRollsBackSecondLeave() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Future<Boolean> alice = executor.submit(
        () -> leaveWithCounter(ALICE, ready, start)
    );
    Future<Boolean> bob = executor.submit(
        () -> leaveWithCounter(BOB, ready, start)
    );

    await(ready, "counter actors");
    start.countDown();
    List<Boolean> accepted = List.of(
        alice.get(5, TimeUnit.SECONDS),
        bob.get(5, TimeUnit.SECONDS)
    );

    assertThat(accepted).containsExactlyInAnyOrder(true, false);
    assertThat(committedRosterCount()).isEqualTo(1);
    assertThat(committedOnCallCount()).isEqualTo(1);
}
```

`leaveWithCounter` throws/rolls back when:

```sql
update on_call_roster
set on_call_count = on_call_count - 1
where roster_id = :rosterId
  and on_call_count > 1;
-- affected row 0 for loser
```

## Experiment 7 — Rollback release locks và state

```java
@Test
void failedGuardedAttemptLeavesBothOperatorsOnCall() {
    JdbcTemplate jdbc = new JdbcTemplate(dataSource);
    TransactionTemplate tx = new TransactionTemplate(
        new DataSourceTransactionManager(dataSource)
    );

    assertThatThrownBy(() -> tx.executeWithoutResult(status -> {
        jdbc.queryForObject(
            """
            select roster_id
            from on_call_roster
            where roster_id = ?
            for update
            """,
            UUID.class,
            ROSTER_ID
        );
        jdbc.update(
            """
            update on_call_assignment
            set on_call=false, version=version+1
            where roster_id=? and operator_id=?
            """,
            ROSTER_ID,
            ALICE
        );
        throw new IllegalStateException("force rollback");
    })).isInstanceOf(IllegalStateException.class);

    assertThat(committedOnCallCount()).isEqualTo(2);
}
```

## SQL helpers và inspector

```java
private static void lockRoster(Connection connection) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select roster_id
        from on_call_roster
        where roster_id = ?
        for update
        """)) {
        statement.setObject(1, ROSTER_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
        }
    }
}

private static long committedOnCallCount() {
    return new JdbcTemplate(dataSource).queryForObject(
        """
        select count(*)
        from on_call_assignment
        where roster_id=? and on_call
        """,
        Long.class,
        ROSTER_ID
    );
}

private static void insertAssignment(JdbcTemplate jdbc, UUID operatorId) {
    jdbc.update(
        """
        insert into on_call_assignment(
            assignment_id, roster_id, operator_id, on_call, version
        ) values (?, ?, ?, true, 0)
        """,
        UUID.randomUUID(),
        ROSTER_ID,
        operatorId
    );
}
```

`committedRosterCount()` đọc `on_call_roster.on_call_count`. Helper
`leaveWithCounter()` dùng JDBC transaction, update assignment conditionally, chạy
counter UPDATE, rollback và return `false` khi counter affected-row `0`.

## Coverage matrix

| Experiment | Mechanism | Technical outcome | Business outcome |
| --- | --- | --- | --- |
| 1 | RR + disjoint `@Version` writes | both affected `1` | unsafe count `0` |
| 2 | RC + disjoint writes | both commit | unsafe count `0` |
| 3 | SERIALIZABLE SSI | one `40001` | safe count `1` |
| 4 | Guard `FOR UPDATE` | second waits/recounts | one accepted |
| 5 | Guard + `lock_timeout` | contender `55P03` | owner unchanged |
| 6 | Conditional counter | loser affected `0`/rollback | rows/counter `1` |
| 7 | Forced rollback | lock/state rollback | count remains `2` |

## Chống flaky và production verification

- Connections/transactions độc lập; không outer test transaction.
- Latch đặt cả predicate reads trước writes; futures đều bounded.
- SERIALIZABLE test assert count/SQLSTATE, không actor identity.
- Test class chạy `SAME_THREAD`, reset committed state trước mỗi method.
- Timeout thu thập `pg_stat_activity`, `pg_locks` và thread dump.
- PostgreSQL Testcontainers là evidence; H2 không thay thế MVCC/SSI.

Production query:

```sql
select r.roster_id, r.on_call_count,
       count(a.*) filter (where a.on_call) as actual
from on_call_roster r
left join on_call_assignment a on a.roster_id = r.roster_id
where r.active
group by r.roster_id, r.on_call_count
having count(a.*) filter (where a.on_call) = 0
    or r.on_call_count <> count(a.*) filter (where a.on_call);
```

Theo dõi `40001`, `40P01`, `55P03`, retry attempts/exhaustion, lock waits và
unsafe-roster count.
