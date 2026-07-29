# Deterministic PostgreSQL lock experiments

## Mục tiêu

Test suite chứng minh:

1. transaction giữ plain SELECT open không chặn UPDATE;
2. `FOR UPDATE` chặn incompatible writer/locking reader;
3. plain SELECT không chờ row lock và thấy old committed version;
4. commit/rollback release locks;
5. waiter re-evaluate conditional predicate sau holder commit;
6. `ACCESS EXCLUSIVE` table lock chặn plain SELECT;
7. Spring `PESSIMISTIC_WRITE` thực sự sinh locking SQL.

Dùng latch/future có timeout; không đoán interleaving bằng wall-clock delay.

> **Nói ngắn gọn:** mỗi test chỉ thay một operation/mode, nhờ vậy failure cho biết
> lock compatibility hay MVCC visibility nào đang sai.

## PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
class RowTableLockLifecycleIntegrationTest {

    private static final UUID TENANT_ID =
        UUID.fromString("10000000-0000-0000-0000-000000000042");

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

        new JdbcTemplate(dataSource).execute("""
            create table tenant_quota (
                tenant_id uuid primary key,
                quota integer not null check (quota >= 0),
                revision bigint not null
            )
            """);
    }

    @BeforeEach
    void resetState() {
        executor = Executors.newFixedThreadPool(3);
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        jdbc.update("delete from tenant_quota");
        jdbc.update(
            """
            insert into tenant_quota(tenant_id, quota, revision)
            values (?, 10, 5)
            """,
            TENANT_ID
        );
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

Test methods không có outer transaction. Actors dùng connections riêng.

## Coordination helper

```java
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

## Experiment 1 — Plain SELECT không chặn writer

```java
@Test
void openPlainReaderDoesNotReserveRow() throws Exception {
    CountDownLatch firstReadDone = new CountDownLatch(1);
    CountDownLatch writerCommitted = new CountDownLatch(1);

    Future<List<Integer>> reader = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            connection.setTransactionIsolation(
                Connection.TRANSACTION_READ_COMMITTED
            );
            int first = selectQuota(connection, false);
            firstReadDone.countDown();
            await(writerCommitted, "writer commit");
            int second = selectQuota(connection, false);
            connection.commit();
            return List.of(first, second);
        }
    });

    Future<Void> writer = executor.submit(() -> {
        await(firstReadDone, "first plain read");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            assertThat(updateQuota(connection, 8)).isEqualTo(1);
            connection.commit();
        }
        writerCommitted.countDown();
        return null;
    });

    assertThat(reader.get(5, TimeUnit.SECONDS)).containsExactly(10, 8);
    writer.get(5, TimeUnit.SECONDS);
    assertThat(committedQuota()).isEqualTo(8);
}
```

Reader transaction vẫn open ở lúc writer commit. Nếu plain SELECT giữ row lock,
writer không thể hoàn tất để signal latch.

## Experiment 2 — Plain reader đi qua `FOR UPDATE` holder

```java
@Test
void plainReaderSeesOldCommittedVersionWithoutWaitingForRowLock()
    throws Exception {

    CountDownLatch uncommittedUpdateReady = new CountDownLatch(1);
    CountDownLatch dashboardReadDone = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            assertThat(selectQuota(connection, true)).isEqualTo(10);
            assertThat(updateQuota(connection, 12)).isEqualTo(1);
            uncommittedUpdateReady.countDown();
            await(dashboardReadDone, "dashboard plain read");
            connection.commit();
        }
        return null;
    });

    Future<Integer> dashboard = executor.submit(() -> {
        await(uncommittedUpdateReady, "uncommitted locked update");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            int observed = selectQuota(connection, false);
            connection.commit();
            dashboardReadDone.countDown();
            return observed;
        }
    });

    assertThat(dashboard.get(5, TimeUnit.SECONDS)).isEqualTo(10);
    holder.get(5, TimeUnit.SECONDS);
    assertThat(committedQuota()).isEqualTo(12);
}
```

Dashboard không thấy dirty value `12`; nó đọc old committed `10`.

## Experiment 3 — Incompatible writer timeout

```java
@Test
void updateWaitsForForUpdateHolderAndTimesOutBoundedly() throws Exception {
    CountDownLatch rowLocked = new CountDownLatch(1);
    CountDownLatch contenderDone = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            selectQuota(connection, true);
            rowLocked.countDown();
            await(contenderDone, "writer timeout");
            connection.commit();
        }
        return null;
    });

    Future<String> contender = executor.submit(() -> {
        await(rowLocked, "row lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            setLockTimeout(connection, "300ms");
            try {
                updateQuota(connection, 8);
                connection.commit();
                return "unexpected-commit";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                contenderDone.countDown();
            }
        }
    });

    assertThat(contender.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    holder.get(5, TimeUnit.SECONDS);
    assertThat(committedQuota()).isEqualTo(10);
}
```

## Experiment 4 — Locking reader cũng chờ

Thay contender UPDATE bằng:

```java
setLockTimeout(connection, "300ms");
selectQuota(connection, true);
```

Khi A giữ `FOR UPDATE` same row, B locking SELECT nhận `55P03` sau bounded timeout.
Một plain SELECT ở cùng thời điểm vẫn return committed row như Experiment 2.

## Experiment 5 — Rollback release lock

```java
@Test
void rollbackReleasesRowLockAndDiscardsHolderVersion() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch waiterReady = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            selectQuota(connection, true);
            updateQuota(connection, 12);
            holderUpdated.countDown();
            await(waiterReady, "waiter ready");
            connection.rollback();
        }
        return null;
    });

    Future<Integer> waiter = executor.submit(() -> {
        await(holderUpdated, "holder update");
        waiterReady.countDown();
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            int affected = updateQuota(connection, 8);
            connection.commit();
            return affected;
        }
    });

    holder.get(5, TimeUnit.SECONDS);
    assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo(1);
    assertThat(committedQuota()).isEqualTo(8);
}
```

Waiter may reach lock before/after rollback depending scheduler; durable assertion
proves uncommitted `12` disappeared and lock did not survive transaction.

## Experiment 6 — Predicate recheck sau holder commit

A locks row và commits revision `6`. B đã prepared conditional UPDATE expecting
revision `5`:

```java
private static int updateIfRevisionMatches(
    Connection connection,
    int quota,
    long expectedRevision
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        update tenant_quota
           set quota=?,
               revision=revision+1
         where tenant_id=?
           and revision=?
        """)) {
        statement.setInt(1, quota);
        statement.setObject(2, TENANT_ID);
        statement.setLong(3, expectedRevision);
        return statement.executeUpdate();
    }
}
```

Controlled test asserts:

```java
assertThat(waiterAffectedRows).isZero();
assertThat(committedQuota()).isEqualTo(12);
assertThat(committedRevision()).isEqualTo(6);
```

B maps affected-row `0` thành conflict; it does not overwrite current row.

## Experiment 7 — `ACCESS EXCLUSIVE` chặn plain SELECT

```java
@Test
void accessExclusiveTableLockBlocksEvenPlainSelect() throws Exception {
    CountDownLatch tableLocked = new CountDownLatch(1);
    CountDownLatch readerDone = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (Statement statement = connection.createStatement()) {
                statement.execute("""
                    lock table tenant_quota in access exclusive mode
                    """);
            }
            tableLocked.countDown();
            await(readerDone, "table-lock reader timeout");
            connection.commit();
        }
        return null;
    });

    Future<String> reader = executor.submit(() -> {
        await(tableLocked, "access exclusive table lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            setLockTimeout(connection, "300ms");
            try {
                selectQuota(connection, false);
                connection.commit();
                return "unexpected-read";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                readerDone.countDown();
            }
        }
    });

    assertThat(reader.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    holder.get(5, TimeUnit.SECONDS);
}
```

Experiment 2 versus 7 là evidence rằng “row lock” và exact “table lock mode” không
thể gọi chung là “database đang lock nên reads block”.

## Experiment 8 — Spring `PESSIMISTIC_WRITE`

Integration test gọi real proxy/service với hook sau repository lock:

```java
@Test
void pessimisticRepositoryHoldsLockUntilServiceTransactionCommits()
    throws Exception {

    Future<Void> owner = executor.submit(
        () -> service.changeQuotaWithGate(TENANT_ID, 12, gate)
    );
    gate.awaitLocked();

    assertThat(lockInspector.hasWaitingWriter(TENANT_ID)).isFalse();
    Future<QuotaChangeResult> waiter = executor.submit(
        () -> service.changeQuota(TENANT_ID, 8)
    );

    gate.awaitWaiterObservedByDatabase();
    assertThat(lockInspector.blockingPids()).isNotEmpty();
    gate.allowCommit();

    owner.get(5, TimeUnit.SECONDS);
    waiter.get(5, TimeUnit.SECONDS);
}
```

Gate methods đều dùng bounded latches. SQL capture phải chứa `for update`; test
không được dựa chỉ vào annotation text.

## Core helpers

```java
private static int selectQuota(Connection connection, boolean forUpdate)
    throws SQLException {
    String suffix = forUpdate ? " for update" : "";
    try (PreparedStatement statement = connection.prepareStatement(
        "select quota from tenant_quota where tenant_id=?" + suffix
    )) {
        statement.setObject(1, TENANT_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
            return rs.getInt(1);
        }
    }
}

private static int updateQuota(Connection connection, int quota)
    throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        update tenant_quota
           set quota=?,
               revision=revision+1
         where tenant_id=?
        """)) {
        statement.setInt(1, quota);
        statement.setObject(2, TENANT_ID);
        return statement.executeUpdate();
    }
}

private static void setLockTimeout(Connection connection, String value)
    throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement(
        "select set_config('lock_timeout', ?, true)"
    )) {
        statement.setString(1, value);
        statement.execute();
    }
}

private static int committedQuota() {
    return new JdbcTemplate(dataSource).queryForObject(
        "select quota from tenant_quota where tenant_id=?",
        Integer.class,
        TENANT_ID
    );
}
```

## Coverage matrix

| Experiment | Holder | Contender | Assertion |
| --- | --- | --- | --- |
| 1 | plain SELECT tx open | UPDATE | writer commits; reader `10 -> 8` |
| 2 | FOR UPDATE + uncommitted UPDATE | plain SELECT | reads old committed `10` |
| 3 | FOR UPDATE row | UPDATE | contender `55P03` |
| 4 | FOR UPDATE row | FOR UPDATE | locking reader `55P03` |
| 5 | FOR UPDATE then rollback | UPDATE waiter | waiter commits `8` |
| 6 | holder commits revision change | conditional UPDATE | affected-row `0` |
| 7 | ACCESS EXCLUSIVE table | plain SELECT | reader `55P03` |
| 8 | JPA PESSIMISTIC_WRITE | service writer | SQL/wait visible until commit |

## Chống flaky và production verification

- Independent connections/transactions; no outer test transaction.
- Holder signals only after locking statement/update succeeds.
- Mọi latches/futures bounded và restore interrupt flag.
- Test class `SAME_THREAD`, reset committed state mỗi method.
- Timeout thu `pg_stat_activity`, `pg_locks`, `pg_blocking_pids` và thread dump.
- PostgreSQL Testcontainers là evidence; H2 không thay lock semantics.

Metrics: lock wait duration, `55P03`, `40P01`, transaction age, `idle in
transaction`, connection usage duration và waiting-connection count.
