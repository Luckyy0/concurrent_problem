# Deterministic unique-claim experiments

## Mục tiêu

Test suite chứng minh:

1. Thao tác kiểm tra rồi chèn (check-then-insert) mà không có ràng buộc sẽ tạo ra hai rows;
2. ràng buộc duy nhất có tên (named unique constraint) chỉ cho phép một bên thắng bền bỉ;
3. bên thua có thể chờ kết quả transaction của bên thắng;
4. bên thắng bị rollback cho phép bên chờ (waiter) trở thành bên thắng;
5. lệnh `ON CONFLICT DO NOTHING RETURNING` trả về một ID và một kết quả rỗng (empty);
6. lỗi `23505` làm transaction bị hủy và siêu dữ liệu (metadata) chính xác của ràng buộc được giữ lại;
7. Lệnh `saveAndFlush()` của Hibernate làm lộ xung đột ở ranh giới của lần thử insert.

Không dùng độ trễ thời gian thực (wall-clock delay) để sắp xếp sự xen kẽ (interleaving). Các cơ chế Latch/future đều có thời gian chờ (timeout).

> **Nói ngắn gọn:** bài kiểm thử (test) phải xác nhận (assert) có một business key bền bỉ duy nhất và kết quả nghiệp vụ của bên thua, thay vì chỉ xác nhận rằng một chủ thể đã ném ra ngoại lệ.

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

Mỗi chủ thể mở connection/transaction riêng biệt; phương thức kiểm thử không có `@Transactional` bao ngoài.

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

## Thí nghiệm 1 — Thao tác kiểm tra rồi chèn bị lỗi tạo ra dữ liệu trùng lặp

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

## Thí nghiệm 2 — Unique constraint chỉ có một bên thắng

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

`trySafeInsert()` bắt ngoại lệ `PSQLException` bên ngoài kết quả của statement, thực hiện rollback connection và lấy tên ràng buộc từ `getServerErrorMessage()`.

## Thí nghiệm 3 — Bên thua chờ khóa duy nhất chưa commit

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

Kết quả quá thời gian chờ (timeout) không được ánh xạ thành dữ liệu trùng lặp; bên thắng chưa commit tại thời điểm bên thua bị quá thời gian.

## Thí nghiệm 4 — Bên thắng rollback cho phép bên chờ thực hiện lệnh insert

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

Bài kiểm thử không phụ thuộc vào việc bên chờ đã bị chặn (block) trong bao lâu; kết quả bất biến sau lần rollback đầu tiên là bên chờ commit đúng một row.

## Thí nghiệm 5 — `ON CONFLICT DO NOTHING RETURNING`

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

Trợ thủ yêu cầu quyền (Claim helper) chạy transaction riêng biệt và dùng:

```sql
insert into work_item_safe(...)
values (...)
on conflict (tenant_id, external_reference) do nothing
returning work_item_id;
```

## Thí nghiệm 6 — `23505` abort transaction

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

Đây là bằng chứng hồi quy (regression evidence) cho yêu cầu “phải bắt ngoại lệ bên ngoài transaction đã thất bại”.

## Thí nghiệm 7 — Hibernate flush boundary

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

Mỗi lời gọi `insertAttempt.insert()` sử dụng mức lan truyền `REQUIRES_NEW` và gọi `saveAndFlush()`. Thao tác đọc (Reader) chạy sau khi lần thử thất bại đã bị rollback.

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

Tên bảng chỉ nhận các hằng số kiểm thử (test constants), không nhận đầu vào không đáng tin cậy.

## Ma trận độ phủ (Coverage matrix)

| Thí nghiệm | Điều kiện được kiểm soát | Xác nhận kỹ thuật | Xác nhận nghiệp vụ |
| --- | --- | --- | --- |
| 1 | cả hai thao tác kiểm tra trước các lệnh insert | no conflict | số lượng không an toàn là 2 |
| 2 | cùng một unique key | một lỗi `23505` đúng ràng buộc | số lượng an toàn là 1 |
| 3 | bên thắng chưa commit | bên thua bị lỗi `55P03` trong giới hạn | không có row thứ hai |
| 4 | rollback đầu tiên | bên chờ commit | count `1` |
| 5 | các lệnh `DO NOTHING` đồng thời | một ID/một rỗng | count `1` |
| 6 | truy vấn sau lỗi `23505` | lỗi `25P02` | yêu cầu rollback |
| 7 | lệnh JPA `saveAndFlush` | bộ bọc đã phân loại | ID hiện tại không đổi |

## Chống lỗi ngẫu nhiên (flaky) và xác minh trên production

- Sử dụng các connections/transactions độc lập; không dùng transaction kiểm thử bao ngoài.
- Các Latches kiểm soát các thao tác kiểm tra chưa tồn tại và kết quả transaction.
- Mọi futures/waits đều được giới hạn thời gian (bounded), cờ ngắt (interrupt flag) được khôi phục.
- Lớp kiểm thử dùng `SAME_THREAD`, đặt lại (reset) các bảng đã commit sau mỗi bài kiểm thử.
- Khi quá thời gian chờ, hệ thống thu thập `pg_stat_activity`, `pg_locks` và thread dump.
- Sử dụng PostgreSQL Testcontainers, không dùng H2, để làm bằng chứng cho trạng thái chờ khóa duy nhất hoặc mã SQLSTATE.

Production checks:

```sql
select tenant_id, external_reference, count(*)
from work_item
group by tenant_id, external_reference
having count(*) > 1;
```

Theo dõi các vi phạm duy nhất (unique violations) theo constraint, quyền sở hữu không làm gì (claim no-op), thời gian chờ/timeout, các transactions bị hủy, sự sai lệch tải trọng (payload mismatch) và tỷ lệ trùng lặp so với tạo mới (duplicate-to-created ratio).
