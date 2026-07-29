# Deterministic non-repeatable-read experiments

## Mục tiêu

Test suite cần chứng minh riêng:

1. `READ COMMITTED` có thể thấy revision khác giữa hai SELECT.
2. Interleaving đó tạo decision vi phạm policy/evidence invariant.
3. `REPEATABLE READ` giữ stable snapshot nhưng không hứa latest-at-commit.
4. B rollback không làm revision mới visible.
5. `FOR SHARE` giữ updater chờ/fail theo bounded timeout.
6. Conditional validation trả affected-row `0` khi revision đổi.

Không dùng delay theo wall clock để điều phối. Latch đặt B commit chính xác giữa
hai statements; mọi `await` và `Future.get` đều có timeout.

> **Nói ngắn gọn:** test phải kiểm soát commit order và assert decision invariant,
> không chỉ in ra hai giá trị khác nhau rồi phụ thuộc scheduler.

## PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
class NonRepeatableReadIntegrationTest {

    private static final UUID MERCHANT_ID = UUID.fromString("10000000-0000-0000-0000-000000000042");
    private static final UUID COMMAND_ID = UUID.fromString("20000000-0000-0000-0000-000000000042");

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withDatabaseName("concurrency")
            .withUsername("test")
            .withPassword("test");

    private static HikariDataSource dataSource;
    private ExecutorService executor;

    @BeforeAll
    static void createDataSourceAndSchema() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(POSTGRES.getJdbcUrl());
        config.setUsername(POSTGRES.getUsername());
        config.setPassword(POSTGRES.getPassword());
        config.setMaximumPoolSize(6);
        dataSource = new HikariDataSource(config);

        new JdbcTemplate(dataSource).execute("""
            create table merchant_refund_policy (
                merchant_id uuid primary key,
                auto_refund_limit numeric(19, 2) not null,
                active boolean not null,
                revision bigint not null,
                check (auto_refund_limit >= 0)
            );

            create table refund_decision (
                id uuid primary key,
                command_id uuid not null unique,
                merchant_id uuid not null,
                amount numeric(19, 2) not null,
                outcome varchar(32) not null,
                evaluated_limit numeric(19, 2) not null,
                policy_revision bigint not null
            );
            """);
    }

    @BeforeEach
    void resetState() {
        executor = Executors.newFixedThreadPool(2);
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        jdbc.update("delete from refund_decision");
        jdbc.update("delete from merchant_refund_policy");
        jdbc.update(
            """
            insert into merchant_refund_policy(
                merchant_id, auto_refund_limit, active, revision
            ) values (?, 100.00, true, 7)
            """,
            MERCHANT_ID
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

Test methods không có outer `@Transactional`; mỗi actor tự mở JDBC connection và
physical transaction riêng.

## Coordination gate

```java
final class ReadCommitGate {
    private final CountDownLatch firstReadDone = new CountDownLatch(1);
    private final CountDownLatch writerCommitted = new CountDownLatch(1);

    void signalFirstRead() {
        firstReadDone.countDown();
    }

    void awaitFirstRead() {
        await(firstReadDone, "first read");
    }

    void signalWriterCommitted() {
        writerCommitted.countDown();
    }

    void awaitWriterCommit() {
        await(writerCommitted, "writer commit");
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
}

record PolicyRow(BigDecimal limit, boolean active, long revision) {
}

record TwoReads(
    PolicyRow first,
    PolicyRow second,
    String effectiveIsolation
) {
}
```

## JDBC helpers

```java
private static PolicyRow readPolicy(Connection connection) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select auto_refund_limit, active, revision
        from merchant_refund_policy
        where merchant_id = ?
        """)) {
        statement.setObject(1, MERCHANT_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
            return new PolicyRow(
                rs.getBigDecimal("auto_refund_limit"),
                rs.getBoolean("active"),
                rs.getLong("revision")
            );
        }
    }
}

private static String effectiveIsolation(Connection connection)
    throws SQLException {
    try (Statement statement = connection.createStatement();
         ResultSet rs = statement.executeQuery(
             "select current_setting('transaction_isolation')"
         )) {
        assertThat(rs.next()).isTrue();
        return rs.getString(1);
    }
}

private static void updatePolicyAndCommit(
    ReadCommitGate gate,
    BigDecimal newLimit
) throws SQLException {
    gate.awaitFirstRead();
    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        try (PreparedStatement statement = connection.prepareStatement("""
            update merchant_refund_policy
               set auto_refund_limit = ?,
                   revision = revision + 1
             where merchant_id = ?
            """)) {
            statement.setBigDecimal(1, newLimit);
            statement.setObject(2, MERCHANT_ID);
            assertThat(statement.executeUpdate()).isEqualTo(1);
        }
        connection.commit();
    }
    gate.signalWriterCommitted();
}

private static TwoReads readAroundCommit(
    int isolation,
    ReadCommitGate gate
) throws SQLException {
    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        connection.setTransactionIsolation(isolation);

        String actualIsolation = effectiveIsolation(connection);
        PolicyRow first = readPolicy(connection);
        gate.signalFirstRead();
        gate.awaitWriterCommit();
        PolicyRow second = readPolicy(connection);
        connection.commit();

        return new TwoReads(first, second, actualIsolation);
    }
}
```

Nếu writer fail trước commit, reader timeout thay vì treo. Future của writer vẫn
trả exception gốc cho test diagnostics.

## Experiment 1 — `READ COMMITTED` thấy revision khác

```java
@Test
void readCommittedUsesANewSnapshotForTheSecondSelect() throws Exception {
    ReadCommitGate gate = new ReadCommitGate();

    Future<TwoReads> reader = executor.submit(
        () -> readAroundCommit(Connection.TRANSACTION_READ_COMMITTED, gate)
    );
    Future<Void> writer = executor.submit(() -> {
        updatePolicyAndCommit(gate, new BigDecimal("50.00"));
        return null;
    });

    TwoReads observed = reader.get(5, TimeUnit.SECONDS);
    writer.get(5, TimeUnit.SECONDS);

    assertThat(observed.effectiveIsolation()).isEqualTo("read committed");
    assertThat(observed.first().revision()).isEqualTo(7);
    assertThat(observed.first().limit()).isEqualByComparingTo("100.00");
    assertThat(observed.second().revision()).isEqualTo(8);
    assertThat(observed.second().limit()).isEqualByComparingTo("50.00");
}
```

Assertion chính là committed business row thay đổi giữa hai controlled statement
snapshots.

## Experiment 2 — Broken decision vi phạm invariant

Reader giữ eligibility từ lần đầu, nhưng insert revision của lần hai:

```java
@Test
void mixingTwoSnapshotsCreatesAnImpossibleDecision() throws Exception {
    ReadCommitGate gate = new ReadCommitGate();
    UUID commandId = UUID.randomUUID();

    Future<Void> evaluator = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            connection.setTransactionIsolation(
                Connection.TRANSACTION_READ_COMMITTED
            );

            PolicyRow eligibility = readPolicy(connection);
            assertThat(new BigDecimal("80.00"))
                .isLessThanOrEqualTo(eligibility.limit());

            gate.signalFirstRead();
            gate.awaitWriterCommit();

            PolicyRow audit = readPolicy(connection);
            insertApprovedDecision(
                connection,
                commandId,
                new BigDecimal("80.00"),
                eligibility.limit(),
                audit.revision()
            );
            connection.commit();
        }
        return null;
    });

    Future<Void> writer = executor.submit(() -> {
        updatePolicyAndCommit(gate, new BigDecimal("50.00"));
        return null;
    });

    evaluator.get(5, TimeUnit.SECONDS);
    writer.get(5, TimeUnit.SECONDS);

    DecisionRow decision = inspector().decision(commandId);
    PolicyRow current = inspector().policy(MERCHANT_ID);

    assertThat(decision.outcome()).isEqualTo("APPROVED");
    assertThat(decision.policyRevision()).isEqualTo(current.revision());
    assertThat(decision.amount()).isGreaterThan(current.limit());
    assertThat(decision.evaluatedLimit()).isNotEqualByComparingTo(current.limit());
}
```

Test cố ý assert broken final state để khóa reproduction. Regression test cho
solution phải đảo lại thành:

```java
assertThat(decision.amount())
    .isLessThanOrEqualTo(policyHistory.limitAt(decision.policyRevision()));
```

## Experiment 3 — `REPEATABLE READ` giữ stable snapshot

```java
@Test
void repeatableReadKeepsTheTransactionSnapshot() throws Exception {
    ReadCommitGate gate = new ReadCommitGate();

    Future<TwoReads> reader = executor.submit(
        () -> readAroundCommit(Connection.TRANSACTION_REPEATABLE_READ, gate)
    );
    Future<Void> writer = executor.submit(() -> {
        updatePolicyAndCommit(gate, new BigDecimal("50.00"));
        return null;
    });

    TwoReads observed = reader.get(5, TimeUnit.SECONDS);
    writer.get(5, TimeUnit.SECONDS);

    assertThat(observed.effectiveIsolation()).isEqualTo("repeatable read");
    assertThat(observed.first().revision()).isEqualTo(7);
    assertThat(observed.second().revision()).isEqualTo(7);
    assertThat(inspector().policy(MERCHANT_ID).revision()).isEqualTo(8);
}
```

Cả hai assertions quan trọng: A stable ở revision `7`, nhưng current committed
policy sau cùng vẫn là revision `8`. Stable không đồng nghĩa latest.

## Experiment 4 — Updater rollback không visible

```java
@Test
void rolledBackRevisionIsNotVisibleToLaterReadCommittedStatement()
    throws Exception {

    ReadCommitGate gate = new ReadCommitGate();

    Future<TwoReads> reader = executor.submit(
        () -> readAroundCommit(Connection.TRANSACTION_READ_COMMITTED, gate)
    );
    Future<Void> writer = executor.submit(() -> {
        gate.awaitFirstRead();
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            updatePolicy(connection, new BigDecimal("50.00"));
            connection.rollback();
        }
        gate.signalWriterCommitted();
        return null;
    });

    TwoReads observed = reader.get(5, TimeUnit.SECONDS);
    writer.get(5, TimeUnit.SECONDS);

    assertThat(observed.first().revision()).isEqualTo(7);
    assertThat(observed.second().revision()).isEqualTo(7);
    assertThat(inspector().policy(MERCHANT_ID).revision()).isEqualTo(7);
}
```

Gate name `writerCommitted` ở helper biểu diễn “writer outcome published” trong
test này; production semantics vẫn phân biệt commit và rollback.

## Experiment 5 — `FOR SHARE` chặn policy updater

Dùng `lock_timeout` để có bounded, deterministic loser:

```java
@Test
void shareLockPreventsPolicyChangeUntilDecisionTransactionEnds()
    throws Exception {

    CountDownLatch shareLockHeld = new CountDownLatch(1);
    CountDownLatch writerFinished = new CountDownLatch(1);

    Future<Void> evaluator = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (PreparedStatement statement = connection.prepareStatement("""
                select revision
                from merchant_refund_policy
                where merchant_id = ?
                for share
                """)) {
                statement.setObject(1, MERCHANT_ID);
                try (ResultSet rs = statement.executeQuery()) {
                    assertThat(rs.next()).isTrue();
                    assertThat(rs.getLong(1)).isEqualTo(7);
                }
            }

            shareLockHeld.countDown();
            await(writerFinished, "writer lock timeout");
            connection.commit();
        }
        return null;
    });

    Future<String> writer = executor.submit(() -> {
        await(shareLockHeld, "share lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (Statement setting = connection.createStatement()) {
                setting.execute("set local lock_timeout = '300ms'");
            }
            try {
                updatePolicy(connection, new BigDecimal("50.00"));
                connection.commit();
                return "unexpected-commit";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                writerFinished.countDown();
            }
        }
    });

    assertThat(writer.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    evaluator.get(5, TimeUnit.SECONDS);
    assertThat(inspector().policy(MERCHANT_ID).revision()).isEqualTo(7);
}
```

SQLSTATE `55P03` là lock-not-available do configured timeout. Sau A commit, một
transaction mới của B có thể reload row và update thành revision `8`.

## Experiment 6 — Conditional validation phát hiện revision đổi

```java
@Test
void conditionalInsertReturnsZeroAfterPolicyRevisionChanges() {
    RefundPolicySnapshot first = inspector().snapshot(MERCHANT_ID);
    assertThat(first.revision()).isEqualTo(7);

    JdbcTemplate jdbc = new JdbcTemplate(dataSource);
    assertThat(jdbc.update(
        """
        update merchant_refund_policy
           set auto_refund_limit = 50.00,
               revision = 8
         where merchant_id = ?
        """,
        MERCHANT_ID
    )).isEqualTo(1);

    int inserted = jdbc.update(
        """
        insert into refund_decision(
            id, command_id, merchant_id, amount, outcome,
            evaluated_limit, policy_revision
        )
        select ?, ?, p.merchant_id, 80.00, 'APPROVED',
               p.auto_refund_limit, p.revision
          from merchant_refund_policy p
         where p.merchant_id = ?
           and p.revision = ?
           and p.active
           and 80.00 <= p.auto_refund_limit
        """,
        UUID.randomUUID(),
        UUID.randomUUID(),
        MERCHANT_ID,
        first.revision()
    );

    assertThat(inserted).isZero();
    assertThat(inspector().decisionCount()).isZero();
}
```

Test assert cả conflict signal và absence của side effect dở dang.

## Experiment 7 — Version history giữ audit evidence

Sau khi migrate schema versioned:

```java
@Test
void decisionStillJoinsItsImmutablePolicyAfterCurrentPointerMoves() {
    PolicyVersion revision7 = policyHistory.current(MERCHANT_ID);
    RefundDecisionId decisionId = decisionService.approve(
        COMMAND_ID,
        MERCHANT_ID,
        new BigDecimal("80.00")
    );

    policyAdmin.publishNextVersion(
        MERCHANT_ID,
        revision7.revision(),
        new BigDecimal("50.00")
    );

    AuditedDecision audited = inspector().auditedDecision(decisionId);

    assertThat(audited.policyRevision()).isEqualTo(7);
    assertThat(audited.evaluatedLimit()).isEqualByComparingTo("100.00");
    assertThat(audited.amount())
        .isLessThanOrEqualTo(audited.evaluatedLimit());
    assertThat(policyHistory.current(MERCHANT_ID).revision()).isEqualTo(8);
}
```

Đây là business-invariant regression test của recommended audit model.

## Shared helpers

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

private static void updatePolicy(
    Connection connection,
    BigDecimal newLimit
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        update merchant_refund_policy
           set auto_refund_limit = ?,
               revision = revision + 1
         where merchant_id = ?
        """)) {
        statement.setBigDecimal(1, newLimit);
        statement.setObject(2, MERCHANT_ID);
        assertThat(statement.executeUpdate()).isEqualTo(1);
    }
}

private static void insertApprovedDecision(
    Connection connection,
    UUID commandId,
    BigDecimal amount,
    BigDecimal evaluatedLimit,
    long policyRevision
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        insert into refund_decision(
            id, command_id, merchant_id, amount, outcome,
            evaluated_limit, policy_revision
        ) values (?, ?, ?, ?, 'APPROVED', ?, ?)
        """)) {
        statement.setObject(1, UUID.randomUUID());
        statement.setObject(2, commandId);
        statement.setObject(3, MERCHANT_ID);
        statement.setBigDecimal(4, amount);
        statement.setBigDecimal(5, evaluatedLimit);
        statement.setLong(6, policyRevision);
        assertThat(statement.executeUpdate()).isEqualTo(1);
    }
}
```

Imports chính:

```java
import java.math.BigDecimal;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.UUID;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

import javax.sql.DataSource;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.parallel.Execution;
import org.junit.jupiter.api.parallel.ExecutionMode;
import org.springframework.jdbc.core.JdbcTemplate;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
```

## Coverage matrix

| Experiment | Isolation/lock | Interleaving | Business assertion |
| --- | --- | --- | --- |
| 1 | `READ COMMITTED` | B commit giữa SELECTs | revision `7 -> 8` |
| 2 | `READ COMMITTED` | Mix S1 decision/S2 audit | approved evidence inconsistent |
| 3 | `REPEATABLE READ` | B commit giữa SELECTs | reader `7 -> 7`, current `8` |
| 4 | `READ COMMITTED` | B update rồi rollback | reader `7 -> 7` |
| 5 | `FOR SHARE` | B UPDATE khi A giữ lock | B `55P03`, policy unchanged |
| 6 | Revision predicate | B commit trước final insert | affected-row `0`, no decision |
| 7 | Immutable history | Pointer chuyển revision | old decision vẫn audit được |

## Chống flaky

- Không dùng wall-clock delay để đặt commit giữa SELECTs.
- Mọi latch/future có timeout và giữ interrupt flag.
- Reader/writer dùng connection riêng; test method không có outer transaction.
- Chạy class `SAME_THREAD` vì dùng shared container state.
- Reset committed data trước mỗi test.
- Khi timeout, dump `pg_stat_activity` và `pg_locks` thay vì tăng timeout tùy ý.
- Assert `current_setting('transaction_isolation')`, SQLSTATE và final rows.
- H2 không được dùng làm bằng chứng cho PostgreSQL MVCC.

## Production verification

Queries chẩn đoán:

```sql
select pid,
       application_name,
       state,
       wait_event_type,
       wait_event,
       xact_start,
       query_start,
       query
from pg_stat_activity
where datname = current_database();
```

```sql
select l.pid,
       l.locktype,
       l.mode,
       l.granted,
       l.relation::regclass
from pg_locks l
where l.relation in (
    'merchant_refund_policy'::regclass,
    'refund_decision'::regclass
);
```

Metrics/logs:

- `refund.policy_revision_mismatch`;
- `refund.conditional_insert_noop`;
- `refund.decision_retry`;
- `db.lock_wait` và `db.lock_timeout`;
- `db.serialization_failure`;
- transaction duration;
- approved decisions không join được immutable policy version.

Production canary nên kiểm tra join/invariant trên committed rows, không suy luận
từ exception count bằng `0`.
