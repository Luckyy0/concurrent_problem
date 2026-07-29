# Deterministic unique-claim experiments

## Mục tiêu

Test suite chứng minh:

1. check-then-insert không constraint tạo hai rows;
2. named unique constraint chỉ cho một durable winner;
3. loser có thể wait winner transaction outcome;
4. winner rollback cho phép waiter trở thành winner;
5. `ON CONFLICT DO NOTHING RETURNING` trả one ID/one empty;
6. `23505` làm transaction abort và exact constraint metadata được giữ;
7. Hibernate `saveAndFlush()` surface conflict ở insert-attempt boundary.

Không dùng wall-clock delay để sắp interleaving. Latch/future đều có timeout.

> **Nói ngắn gọn:** test phải assert one durable business key và domain loser
> outcome, không chỉ assert một actor đã ném exception.

## PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
class UniqueConstraintConcurrencyIntegrationTest {

    private static final UUID TENANT_ID =
        UUID.fromString("10000000-0000-0000-0000-000000000042");
    private static final String REFERENCE = "CASE-9001";

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
            create table work_item_unsafe (
                work_item_id uuid primary key,
                tenant_id uuid not null,
                external_reference varchar(100) not null,
                status varchar(32) not null
            )
            """);
        jdbc.execute("""
            create table work_item_safe (
                work_item_id uuid primary key,
                tenant_id uuid not null,
                external_reference varchar(100) not null,
                status varchar(32) not null,
                constraint uk_work_item_tenant_reference
                    unique (tenant_id, external_reference)
            )
            """);
    }

    @BeforeEach
    void resetState() {
        executor = Executors.newFixedThreadPool(2);
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        jdbc.update("delete from work_item_unsafe");
        jdbc.update("delete from work_item_safe");
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

Mỗi actor mở connection/transaction riêng; test method không có outer
`@Transactional`.

## Gate helpers

```java
final class BothCheckedGate {
    private final CountDownLatch checked = new CountDownLatch(2);
    private final CountDownLatch mayInsert = new CountDownLatch(1);

    void checkedAbsent() {
        checked.countDown();
        await(checked, "both absence checks");
    }

    void releaseInserts() {
        mayInsert.countDown();
    }

    void awaitInsertPermission() {
        await(mayInsert, "insert permission");
    }
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

## Experiment 1 — Broken check-then-insert tạo duplicates

```java
@Test
void absenceChecksDoNotReserveBusinessKey() throws Exception {
    BothCheckedGate gate = new BothCheckedGate();

    Future<UUID> a = executor.submit(() -> unsafeCreate(gate));
    Future<UUID> b = executor.submit(() -> unsafeCreate(gate));

    gate.releaseInserts();
    UUID idA = a.get(5, TimeUnit.SECONDS);
    UUID idB = b.get(5, TimeUnit.SECONDS);

    assertThat(idA).isNotEqualTo(idB);
    assertThat(count("work_item_unsafe")).isEqualTo(2);
}

private static UUID unsafeCreate(BothCheckedGate gate) throws SQLException {
    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        assertThat(exists(connection, "work_item_unsafe")).isFalse();
        gate.checkedAbsent();
        gate.awaitInsertPermission();
        UUID id = insert(connection, "work_item_unsafe");
        connection.commit();
        return id;
    }
}
```

## Experiment 2 — Unique constraint có one winner

```java
record InsertOutcome(boolean created, String sqlState, String constraint) {
}

@Test
void uniqueConstraintAllowsExactlyOneDurableRow() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Future<InsertOutcome> a = executor.submit(
        () -> trySafeInsert(ready, start)
    );
    Future<InsertOutcome> b = executor.submit(
        () -> trySafeInsert(ready, start)
    );

    await(ready, "both unique inserters");
    start.countDown();
    List<InsertOutcome> outcomes = List.of(
        a.get(5, TimeUnit.SECONDS),
        b.get(5, TimeUnit.SECONDS)
    );

    assertThat(outcomes.stream().filter(InsertOutcome::created).count())
        .isEqualTo(1);
    InsertOutcome loser = outcomes.stream()
        .filter(outcome -> !outcome.created())
        .findFirst()
        .orElseThrow();
    assertThat(loser.sqlState()).isEqualTo("23505");
    assertThat(loser.constraint())
        .isEqualTo("uk_work_item_tenant_reference");
    assertThat(count("work_item_safe")).isEqualTo(1);
}
```

`trySafeInsert()` catches `PSQLException` outside statement result, rollback
connection, và lấy constraint từ `getServerErrorMessage()`.

## Experiment 3 — Loser waits uncommitted unique key

```java
@Test
void conflictingInsertCannotPassUncommittedWinner() throws Exception {
    CountDownLatch winnerInserted = new CountDownLatch(1);
    CountDownLatch loserFinished = new CountDownLatch(1);

    Future<Void> winner = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            insert(connection, "work_item_safe");
            winnerInserted.countDown();
            await(loserFinished, "loser lock timeout");
            connection.commit();
        }
        return null;
    });

    Future<String> loser = executor.submit(() -> {
        await(winnerInserted, "winner insert");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (Statement setting = connection.createStatement()) {
                setting.execute("set local lock_timeout = '300ms'");
            }
            try {
                insert(connection, "work_item_safe");
                connection.commit();
                return "unexpected-commit";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                loserFinished.countDown();
            }
        }
    });

    assertThat(loser.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    winner.get(5, TimeUnit.SECONDS);
    assertThat(count("work_item_safe")).isEqualTo(1);
}
```

Timeout outcome không được map thành duplicate; winner chưa commit tại thời điểm
loser timeout.

## Experiment 4 — Winner rollback cho phép waiter insert

```java
@Test
void waiterCanBecomeWinnerAfterFirstInsertRollsBack() throws Exception {
    CountDownLatch firstInserted = new CountDownLatch(1);
    CountDownLatch waiterReady = new CountDownLatch(1);

    Future<Void> first = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            insert(connection, "work_item_safe");
            firstInserted.countDown();
            await(waiterReady, "waiter ready");
            connection.rollback();
        }
        return null;
    });

    Future<UUID> waiter = executor.submit(() -> {
        await(firstInserted, "first uncommitted insert");
        waiterReady.countDown();
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            UUID id = insert(connection, "work_item_safe");
            connection.commit();
            return id;
        }
    });

    first.get(5, TimeUnit.SECONDS);
    UUID winnerId = waiter.get(5, TimeUnit.SECONDS);
    assertThat(winnerId).isNotNull();
    assertThat(count("work_item_safe")).isEqualTo(1);
}
```

Test không phụ thuộc waiter đã block bao lâu; invariant/outcome sau first rollback
là waiter commit đúng một row.

## Experiment 5 — `ON CONFLICT DO NOTHING RETURNING`

```java
@Test
void onConflictReturnsOneIdAndOneEmpty() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Callable<Optional<UUID>> claim = () -> {
        ready.countDown();
        await(start, "upsert start");
        return claimWithOnConflict();
    };

    Future<Optional<UUID>> a = executor.submit(claim);
    Future<Optional<UUID>> b = executor.submit(claim);
    await(ready, "both upsert actors");
    start.countDown();

    List<Optional<UUID>> outcomes = List.of(
        a.get(5, TimeUnit.SECONDS),
        b.get(5, TimeUnit.SECONDS)
    );
    assertThat(outcomes.stream().filter(Optional::isPresent).count())
        .isEqualTo(1);
    assertThat(outcomes.stream().filter(Optional::isEmpty).count())
        .isEqualTo(1);
    assertThat(count("work_item_safe")).isEqualTo(1);
}
```

Claim helper chạy transaction riêng và dùng:

```sql
insert into work_item_safe(...)
values (...)
on conflict (tenant_id, external_reference) do nothing
returning work_item_id;
```

## Experiment 6 — `23505` abort transaction

```java
@Test
void queryAfterUniqueViolationInSameTransactionGetsAbortedState()
    throws Exception {

    insertCommittedSafeRow();

    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        SQLException duplicate = catchThrowableOfType(
            () -> insert(connection, "work_item_safe"),
            SQLException.class
        );
        assertThat(duplicate.getSQLState()).isEqualTo("23505");

        SQLException aborted = catchThrowableOfType(
            () -> exists(connection, "work_item_safe"),
            SQLException.class
        );
        assertThat(aborted.getSQLState()).isEqualTo("25P02");
        connection.rollback();
    }
}
```

Đây là regression evidence cho requirement “catch outside failed transaction”.

## Experiment 7 — Hibernate flush boundary

Spring integration test không có outer transaction:

```java
@Test
void saveAndFlushSurfacesExactConstraintAtAttemptBoundary() {
    UUID firstId = insertAttempt.insert(TENANT_ID, REFERENCE);

    DataIntegrityViolationException duplicate = catchThrowableOfType(
        () -> insertAttempt.insert(TENANT_ID, REFERENCE),
        DataIntegrityViolationException.class
    );

    assertThat(classifier.isUniqueViolation(
        duplicate,
        "uk_work_item_tenant_reference"
    )).isTrue();
    assertThat(reader.findByBusinessKey(TENANT_ID, REFERENCE).id())
        .isEqualTo(firstId);
}
```

Mỗi `insertAttempt.insert()` là `REQUIRES_NEW` và gọi `saveAndFlush()`. Reader chạy
sau failed attempt đã rollback.

## Core JDBC helpers

```java
private static UUID insert(Connection connection, String table)
    throws SQLException {
    UUID id = UUID.randomUUID();
    String sql = """
        insert into %s(
            work_item_id, tenant_id, external_reference, status
        ) values (?, ?, ?, 'OPEN')
        """.formatted(table);
    try (PreparedStatement statement = connection.prepareStatement(sql)) {
        statement.setObject(1, id);
        statement.setObject(2, TENANT_ID);
        statement.setString(3, REFERENCE);
        assertThat(statement.executeUpdate()).isEqualTo(1);
    }
    return id;
}

private static boolean exists(Connection connection, String table)
    throws SQLException {
    String sql = """
        select exists(
            select 1 from %s
            where tenant_id=? and external_reference=?
        )
        """.formatted(table);
    try (PreparedStatement statement = connection.prepareStatement(sql)) {
        statement.setObject(1, TENANT_ID);
        statement.setString(2, REFERENCE);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
            return rs.getBoolean(1);
        }
    }
}

private static long count(String table) {
    return new JdbcTemplate(dataSource).queryForObject(
        """
        select count(*) from %s
        where tenant_id=? and external_reference=?
        """.formatted(table),
        Long.class,
        TENANT_ID,
        REFERENCE
    );
}
```

Table name chỉ nhận test constants, không nhận untrusted input.

## Coverage matrix

| Experiment | Controlled condition | Technical assertion | Business assertion |
| --- | --- | --- | --- |
| 1 | both checks before inserts | no conflict | unsafe count `2` |
| 2 | same unique key | one `23505` exact constraint | safe count `1` |
| 3 | winner uncommitted | loser `55P03` bounded | no second row |
| 4 | first rollback | waiter commits | count `1` |
| 5 | concurrent `DO NOTHING` | one ID/one empty | count `1` |
| 6 | query after `23505` | `25P02` | requires rollback |
| 7 | JPA `saveAndFlush` | classified wrapper | existing ID intact |

## Chống flaky và production verification

- Independent connections/transactions; no outer test transaction.
- Latches control absence checks and transaction outcomes.
- Mọi futures/waits bounded, interrupt flag được restore.
- Test class `SAME_THREAD`, reset committed tables mỗi test.
- Timeout thu thập `pg_stat_activity`, `pg_locks` và thread dump.
- PostgreSQL Testcontainers, không H2, là evidence cho unique wait/SQLSTATE.

Production checks:

```sql
select tenant_id, external_reference, count(*)
from work_item
group by tenant_id, external_reference
having count(*) > 1;
```

Theo dõi unique violations theo constraint, claim no-op, wait/timeout, aborted
transactions, payload mismatch và duplicate-to-created ratio.
