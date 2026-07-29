# Các thí nghiệm SSI và retry có điều phối

## Mục tiêu

Suite phải chứng minh:

- `READ COMMITTED` cho phép hai decisions cùng commit và total thành `120`;
- `SERIALIZABLE` abort đúng một actor với `40001`;
- victim không để lại reservation/decision và transaction cũ trả `25P02`;
- fresh retry thấy total `90` rồi commit `REJECTED`;
- command replay không tạo side effect mới;
- mọi latch, future và database statement đều có timeout.

> **Nói ngắn gọn:** exception assertion chỉ chứng minh conflict; final total,
> durable decisions và transaction IDs mới chứng minh retry đúng.

## Môi trường PostgreSQL Testcontainers

```java
package example.limit;

import static org.assertj.core.api.Assertions.assertThat;

import java.math.BigDecimal;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class SerializableLimitIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("ssi_cases")
                    .withUsername("cases")
                    .withPassword("cases");

    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    @BeforeAll
    static void createSchema() throws SQLException {
        try (Connection connection = open();
                Statement sql = connection.createStatement()) {
            sql.execute("""
                    create table merchant_limit (
                        merchant_id bigint primary key,
                        limit_amount numeric(19, 2) not null
                    )
                    """);
            sql.execute("""
                    create table credit_reservation (
                        reservation_id uuid primary key,
                        command_id uuid not null unique,
                        merchant_id bigint not null,
                        amount numeric(19, 2) not null,
                        status varchar(16) not null
                    )
                    """);
            sql.execute("""
                    create index ix_credit_reservation_scope
                    on credit_reservation(merchant_id, status)
                    """);
            sql.execute("""
                    create table limit_command_decision (
                        command_id uuid primary key,
                        merchant_id bigint not null,
                        requested_amount numeric(19, 2) not null,
                        outcome varchar(16) not null
                    )
                    """);
        }
    }

    @BeforeEach
    void reset() throws SQLException {
        try (Connection connection = open();
                Statement sql = connection.createStatement()) {
            sql.execute(
                    "truncate limit_command_decision, credit_reservation"
            );
            sql.execute("delete from merchant_limit");
            sql.execute("""
                    insert into merchant_limit values (7, 100.00)
                    """);
            sql.execute("""
                    insert into credit_reservation
                    values (
                      '00000000-0000-0000-0000-000000000060',
                      '10000000-0000-0000-0000-000000000060',
                      7, 60.00, 'ACTIVE'
                    )
                    """);
        }
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }

    static Connection open() throws SQLException {
        return DriverManager.getConnection(
                POSTGRES.getJdbcUrl(),
                POSTGRES.getUsername(),
                POSTGRES.getPassword()
        );
    }
}
```

Mỗi test method có fixture instance/executor riêng. Không chạy các tests này song
song trên cùng schema.

## Helper điều phối

```java
static void await(CountDownLatch latch, String description) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Timed out: " + description);
        }
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Interrupted: " + description, interrupted);
    }
}

record AttemptOutcome(
        UUID commandId,
        String sqlState,
        String stateBeforeRollback,
        long transactionId
) {
    boolean committed() {
        return "00000".equals(sqlState);
    }
}
```

## Thí nghiệm 1 — Baseline `READ COMMITTED` phá invariant

```java
@Test
void readCommittedAllowsBothPredicateDecisions() throws Exception {
    var bothRead = new CountDownLatch(2);
    var write = new CountDownLatch(1);

    Future<AttemptOutcome> first = executor.submit(() ->
            runAttempt(
                    Connection.TRANSACTION_READ_COMMITTED,
                    UUID.fromString("20000000-0000-0000-0000-000000000001"),
                    bothRead,
                    write
            )
    );
    Future<AttemptOutcome> second = executor.submit(() ->
            runAttempt(
                    Connection.TRANSACTION_READ_COMMITTED,
                    UUID.fromString("20000000-0000-0000-0000-000000000002"),
                    bothRead,
                    write
            )
    );

    await(bothRead, "both actors read total 60");
    write.countDown();

    assertThat(List.of(
            first.get(8, TimeUnit.SECONDS),
            second.get(8, TimeUnit.SECONDS)
    )).allMatch(AttemptOutcome::committed);
    assertThat(activeTotal()).isEqualByComparingTo("120.00");
    assertThat(decisionOutcomes()).containsExactly("ACCEPTED", "ACCEPTED");
}
```

Test không nói `READ COMMITTED` luôn sai; nó chứng minh isolation này không bảo vệ
predicate invariant nếu không có guard/constraint/conditional mutation khác.

## Thí nghiệm 2 — `SERIALIZABLE` tạo một victim

```java
@Test
void serializableAbortsOneAttemptAndPreservesLimit() throws Exception {
    var bothRead = new CountDownLatch(2);
    var write = new CountDownLatch(1);

    Future<AttemptOutcome> first = executor.submit(() ->
            runAttempt(
                    Connection.TRANSACTION_SERIALIZABLE,
                    UUID.fromString("30000000-0000-0000-0000-000000000001"),
                    bothRead,
                    write
            )
    );
    Future<AttemptOutcome> second = executor.submit(() ->
            runAttempt(
                    Connection.TRANSACTION_SERIALIZABLE,
                    UUID.fromString("30000000-0000-0000-0000-000000000002"),
                    bothRead,
                    write
            )
    );

    await(bothRead, "both serializable snapshots read total 60");
    write.countDown();

    List<AttemptOutcome> outcomes = List.of(
            first.get(8, TimeUnit.SECONDS),
            second.get(8, TimeUnit.SECONDS)
    );
    assertThat(outcomes)
            .extracting(AttemptOutcome::sqlState)
            .containsExactlyInAnyOrder("00000", "40001");
    assertThat(outcomes.stream()
            .filter(outcome -> "40001".equals(outcome.sqlState()))
            .findFirst()
            .orElseThrow()
            .stateBeforeRollback()).isEqualTo("25P02");
    assertThat(activeTotal()).isEqualByComparingTo("90.00");
    assertThat(decisionOutcomes()).containsExactly("ACCEPTED");
}
```

Không assert command cụ thể là victim. Một decision còn thiếu là expected vì test
này chưa chạy application retry.

## Core JDBC attempt

```java
static AttemptOutcome runAttempt(
        int isolation,
        UUID commandId,
        CountDownLatch bothRead,
        CountDownLatch write
) throws SQLException {
    try (Connection connection = open()) {
        connection.setTransactionIsolation(isolation);
        connection.setAutoCommit(false);
        try (Statement sql = connection.createStatement()) {
            sql.execute("set local statement_timeout = '4s'");
        }

        long txid = queryLong(connection, "select txid_current()");
        BigDecimal active = queryAmount(connection, """
                select coalesce(sum(amount), 0)
                from credit_reservation
                where merchant_id = 7 and status = 'ACTIVE'
                """);
        assertThat(active).isEqualByComparingTo("60.00");
        bothRead.countDown();
        await(write, "permission to write");

        try {
            insertReservation(connection, commandId);
            insertDecision(connection, commandId, "ACCEPTED");
            connection.commit();
            return new AttemptOutcome(commandId, "00000", "00000", txid);
        } catch (SQLException failure) {
            String failedState = executeAndReturnState(connection, "select 1");
            connection.rollback();
            return new AttemptOutcome(
                    commandId,
                    failure.getSQLState(),
                    failedState,
                    txid
            );
        }
    }
}

static void insertReservation(
        Connection connection,
        UUID commandId
) throws SQLException {
    try (PreparedStatement sql = connection.prepareStatement("""
            insert into credit_reservation
            values (?, ?, 7, 30.00, 'ACTIVE')
            """)) {
        sql.setObject(1, UUID.randomUUID());
        sql.setObject(2, commandId);
        sql.executeUpdate();
    }
}

static void insertDecision(
        Connection connection,
        UUID commandId,
        String outcome
) throws SQLException {
    try (PreparedStatement sql = connection.prepareStatement("""
            insert into limit_command_decision
            values (?, 7, 30.00, ?)
            """)) {
        sql.setObject(1, commandId);
        sql.setString(2, outcome);
        sql.executeUpdate();
    }
}

static String executeAndReturnState(Connection connection, String statement) {
    try (Statement sql = connection.createStatement()) {
        sql.execute(statement);
        return "00000";
    } catch (SQLException failure) {
        return failure.getSQLState();
    }
}
```

Catch bao cả `commit()`, vì `40001` không có một throw site duy nhất.

## Thí nghiệm 3 — Fresh retry tạo business rejection

Spring integration test inject một test-only `AttemptGate` ngay sau
`activeTotal()` của hai first attempts:

```java
interface AttemptGate {
    void afterRead(UUID commandId, int attemptNumber);
}

final class TwoFirstAttemptsGate implements AttemptGate {
    private final CountDownLatch bothRead = new CountDownLatch(2);
    private final CountDownLatch continueWrite = new CountDownLatch(1);

    @Override
    public void afterRead(UUID commandId, int attemptNumber) {
        if (attemptNumber != 1) {
            return;
        }
        bothRead.countDown();
        await(bothRead, "two first attempts reached predicate read");
        continueWrite.countDown();
    }
}
```

Production implementation là no-op; barrier không nằm trong production code.
Probe ghi `txid_current()`, isolation và attempt outcome vào thread-safe memory.

```java
start.countDown();
LimitDecision firstResult = first.get(10, TimeUnit.SECONDS);
LimitDecision secondResult = second.get(10, TimeUnit.SECONDS);

assertThat(List.of(firstResult.outcome(), secondResult.outcome()))
        .containsExactlyInAnyOrder(ACCEPTED, REJECTED);
assertThat(activeTotal()).isEqualByComparingTo("90.00");
assertThat(decisionOutcomes())
        .containsExactlyInAnyOrder("ACCEPTED", "REJECTED");
assertThat(attemptProbe.totalAttempts()).isEqualTo(3);
assertThat(attemptProbe.sqlStates()).contains("40001");
assertThat(attemptProbe.transactionIds())
        .hasSize(3)
        .doesNotHaveDuplicates();
assertThat(attemptProbe.isolations())
        .containsOnly("serializable");
assertThat(recordingBackoff.invocations()).isEqualTo(1);
```

Recording backoff không delay test nhưng production backoff implementation có
unit test riêng cho cap, jitter range, deadline và interrupt.

## Thí nghiệm 4 — Replay không tạo reservation mới

Gọi lại command đã `ACCEPTED` sau khi commit:

```java
long reservationsBefore = reservationCount();
long decisionsBefore = decisionCount();

LimitDecision replayed = coordinator.reserve(sameAcceptedCommand);

assertThat(replayed.outcome()).isEqualTo(ACCEPTED);
assertThat(reservationCount()).isEqualTo(reservationsBefore);
assertThat(decisionCount()).isEqualTo(decisionsBefore);
assertThat(activeTotal()).isEqualByComparingTo("90.00");
```

Lặp tương tự với command `REJECTED`: outcome giữ nguyên dù total sau đó giảm.
Đây là durable command semantics; nếu domain muốn rejection có thể re-evaluate,
phải model một command mới thay vì reuse ID.

## Thí nghiệm 5 — Xác minh effective isolation

Trong attempt worker:

```java
String isolation = jdbc.queryForObject(
        "select current_setting('transaction_isolation')",
        String.class
);
assertThat(isolation).isEqualTo("serializable");
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isTrue();
```

Architecture/integration test còn gọi coordinator từ một transactional test bean
và assert guard từ chối outer transaction. Không annotate JUnit method
`@Transactional`; outer test transaction sẽ che commit/rollback thật.

## Thí nghiệm 6 — Quan sát `SIReadLock`

Giữ một serializable transaction mở sau predicate read bằng bounded latch. Từ
observer connection:

```sql
select locktype,
       mode,
       relation::regclass,
       page,
       tuple,
       granted
from pg_locks
where pid = :reader_pid
  and mode = 'SIReadLock';
```

Assert có ít nhất một row `SIReadLock`, nhưng không hard-code tuple/page/relation
granularity. Query plan, statistics và predicate-lock promotion có thể thay đổi
shape mà không đổi correctness.

## Thí nghiệm 7 — Read-only deferrable

Raw connection chạy:

```sql
begin isolation level serializable read only deferrable;
select current_setting('transaction_isolation'),
       current_setting('transaction_read_only'),
       current_setting('transaction_deferrable');
```

Assert `serializable`, `on`, `on` và report hoàn tất trong outer watchdog. Test
không dùng mode này cho write attempt; nó chỉ khóa contract của reporting
alternative.

## Helper đọc business state

```java
static BigDecimal activeTotal() throws SQLException {
    try (Connection connection = open()) {
        return queryAmount(connection, """
                select coalesce(sum(amount), 0)
                from credit_reservation
                where merchant_id = 7 and status = 'ACTIVE'
                """);
    }
}

static List<String> decisionOutcomes() throws SQLException {
    try (Connection connection = open();
            Statement sql = connection.createStatement();
            ResultSet rows = sql.executeQuery("""
                    select outcome
                    from limit_command_decision
                    order by outcome
                    """)) {
        var result = new java.util.ArrayList<String>();
        while (rows.next()) {
            result.add(rows.getString(1));
        }
        return List.copyOf(result);
    }
}
```

## Ma trận bao phủ

| Thí nghiệm | Assertion kỹ thuật | Assertion nghiệp vụ |
| --- | --- | --- |
| 1 | Hai `READ COMMITTED` commits | Total `120`, minh họa anomaly |
| 2 | `40001` + `25P02`, một commit | Total `90`, một decision |
| 3 | Ba unique txids, fresh retry | Một accepted, một rejected, total `90` |
| 4 | Durable decision replay | Counts/total không đổi |
| 5 | Effective isolation/proxy | Mọi attempt là `serializable` |
| 6 | `SIReadLock` hiện trong `pg_locks` | Không phụ thuộc granularity |
| 7 | Read-only deferrable settings | Report alternative đúng mode |

## Chống flaky

- Barrier đặt sau cả hai predicate reads.
- Không assert victim identity hoặc exact statement ném `40001`.
- `statement_timeout < Future.get` watchdog.
- Mọi SQL error path rollback và connection dùng try-with-resources.
- Test reset schema state và tắt parallel execution cùng schema.
- Khi timeout, dump `pg_stat_activity`, `pg_locks`, transaction IDs và thread
  stacks trước cleanup.
- Stress test hữu hạn bổ sung conflict-rate evidence, không thay deterministic
  test.

## Xác minh trong production

Theo dõi application metrics/logs:

- SQLSTATE `40001` theo operation;
- attempt count, success-after-retry, exhaustion và deadline;
- durable decision replay/unique-command conflict;
- transaction duration và pool active/pending;
- effective isolation sampled an toàn;
- query plan/index changes và predicate-lock memory pressure.

PostgreSQL `pg_stat_database.deadlocks` không đếm serialization failures.
`SIReadLock` trong `pg_locks` là diagnostic snapshot, không phải counter. Không
log command payload, amount hoặc bind values nhạy cảm ở high-cardinality labels.
