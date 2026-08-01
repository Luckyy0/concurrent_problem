# Thực Nghiệm: Tái Hiện Hiện Tượng Non-Repeatable Read (Deterministic non-repeatable-read experiments)

## 1. Mục tiêu (Objective)

Hệ thống Test suite cần chứng minh được các luận điểm kỹ thuật sau:

1. Chế độ `READ COMMITTED` có thể trả về các `revision` khác nhau giữa hai lệnh SELECT liên tiếp trong cùng một giao dịch.
2. Việc chèn ngang (Interleaving) giao dịch cập nhật có thể dẫn đến một quyết định vi phạm quy tắc kinh doanh.
3. Chế độ `REPEATABLE READ` duy trì snapshot ổn định, nhưng không đảm bảo đó là dữ liệu mới nhất tại thời điểm commit (latest-at-commit).
4. Khi giao dịch cập nhật bị hủy (rollback), phiên bản mới tuyệt đối không được hiển thị cho các giao dịch khác.
5. Việc sử dụng khóa `FOR SHARE` bắt buộc giao dịch cập nhật phải chờ, hoặc ném ngoại lệ (fail) khi vượt quá thời gian chờ (bounded timeout).
6. Lệnh cập nhật có điều kiện (Conditional validation) phải trả về số dòng cập nhật (affected-row) = `0` khi chính sách đã bị thay đổi.

Lưu ý: Không sử dụng các lệnh ngưng trệ (sleep/delay theo wall clock) để điều phối các luồng. Cần sử dụng cơ chế đồng bộ (Latch/Barrier) để ép giao dịch cập nhật thực thi chính xác giữa hai lệnh SELECT. Mọi thao tác chờ (`await`) hoặc `Future.get` đều phải cấu hình thời gian chờ tối đa (timeout).

> **Ghi chú quan trọng:** Kiểm thử tương tranh (concurrency testing) yêu cầu kiểm soát chính xác thứ tự thực thi và cơ chế xác minh lỗi, không dựa vào xác suất hệ điều hành.

## 2. Thiết lập Testcontainers (PostgreSQL Testcontainers)

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

Lưu ý: Không áp dụng `@Transactional` bao ngoài (outer transaction) ở cấp độ phương thức kiểm thử. Mỗi luồng phải tự quản lý kết nối JDBC và giao dịch vật lý riêng biệt.

## 3. Cơ chế điều phối (Coordination gate)

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
                throw new AssertionError("Timeout: Chờ " + step + " quá lâu.");
            }
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Luồng bị ngắt khi chờ " + step, ex);
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

## 4. Các tiện ích JDBC (JDBC helpers)

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

Với cấu trúc này, nếu luồng cập nhật (writer) gặp sự cố trước khi commit, luồng đọc (reader) sẽ phát hiện timeout thay vì bị treo (hang) vô thời hạn. Ngoại lệ gốc sẽ được lưu trữ trong `Future` để phục vụ phân tích.

## 5. Thực nghiệm 1 — `READ COMMITTED` dẫn đến khác biệt Revision

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

Kiểm thử xác minh dữ liệu kinh doanh bị thay đổi giữa hai lệnh SELECT (statement snapshots) trong cùng một giao dịch.

## 6. Thực nghiệm 2 — Quyết định vi phạm Invariant (Broken decision)

Giao dịch xét duyệt sử dụng kết quả đánh giá từ Snapshot đầu, nhưng lại trích xuất Revision từ Snapshot thứ hai để lưu trữ:

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

    // Assert này kiểm tra sự tồn tại của dữ liệu mâu thuẫn.
    assertThat(decision.outcome()).isEqualTo("APPROVED");
    assertThat(decision.policyRevision()).isEqualTo(current.revision());
    assertThat(decision.amount()).isGreaterThan(current.limit());
    assertThat(decision.evaluatedLimit()).isNotEqualByComparingTo(current.limit());
}
```

Test này chủ đích xác nhận hệ thống có thể tạo ra dữ liệu bất nhất. Khi triển khai giải pháp khắc phục, bài test cần được cập nhật (Regression test) để phản ánh hành vi đúng:

```java
assertThat(decision.amount())
    .isLessThanOrEqualTo(policyHistory.limitAt(decision.policyRevision()));
```

## 7. Thực nghiệm 3 — `REPEATABLE READ` bảo lưu Snapshot ban đầu

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
    assertThat(observed.second().revision()).isEqualTo(7); // Dữ liệu bảo lưu
    assertThat(inspector().policy(MERCHANT_ID).revision()).isEqualTo(8); // Dữ liệu thực tế tại bảng
}
```

Giao dịch đọc giữ nguyên Revision `7`, trong khi dữ liệu gốc đã được commit là `8`. Điều này minh họa trạng thái ổn định (Stable) không đồng nghĩa với dữ liệu mới nhất (Latest).

## 8. Thực nghiệm 4 — Rollback giao dịch cập nhật

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
            connection.rollback(); // Giao dịch bị hủy
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

Tín hiệu `writerCommitted` trong class điều phối chỉ có ý nghĩa đánh dấu giao dịch đã kết thúc, bao gồm cả hành động commit hoặc rollback.

## 9. Thực nghiệm 5 — Khóa `FOR SHARE` bảo vệ bản ghi

Cấu hình `lock_timeout` để kiểm tra việc giao dịch cập nhật không thể thực thi khi dữ liệu đang bị khóa:

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

SQLSTATE `55P03` biểu thị ngoại lệ "lock-not-available" do vượt quá `lock_timeout`.

## 10. Thực nghiệm 6 — Kiểm tra điều kiện (Conditional validation)

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

Kiểm thử xác minh dữ liệu không được ghi vào hệ thống khi phiên bản chính sách đã thay đổi.

## 11. Thực nghiệm 7 — Tham chiếu lịch sử bất biến (Version history audit)

Sau khi chuyển đổi sang mô hình dữ liệu lưu trữ lịch sử:

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

Kết quả: Bằng chứng quyết định được đảm bảo tham chiếu nhất quán với phiên bản chính sách gốc (revision 7), ngay cả khi con trỏ hiện tại trỏ đến phiên bản mới (revision 8).

## 12. Mã nguồn hỗ trợ (Shared helpers)

```java
private static void await(CountDownLatch latch, String step) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Timeout chờ " + step);
        }
    } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Luồng bị ngắt khi đợi " + step, ex);
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

Danh sách Import:

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

## 13. Mức độ bao phủ kiểm thử (Coverage matrix)

| Thực nghiệm | Cấu hình (Isolation/Lock) | Tương tác (Interleaving) | Đặc tả kinh doanh (Business requirement) |
| --- | --- | --- | --- |
| 1 | `READ COMMITTED` | Cập nhật chèn giữa hai lệnh SELECT | Dữ liệu bị thay đổi trong giao dịch (`7 -> 8`) |
| 2 | `READ COMMITTED` | Sử dụng chéo hai Snapshot | Bằng chứng audit bị sai lệch |
| 3 | `REPEATABLE READ` | Cập nhật chèn giữa hai lệnh SELECT | Snapshot đọc ổn định tại `7`, DB đã nhận `8` |
| 4 | `READ COMMITTED` | Cập nhật bị rollback | Giao dịch đọc không bị ảnh hưởng |
| 5 | `FOR SHARE` | Cập nhật khi bản ghi đang bị khóa | Ngoại lệ `55P03` do hết thời gian chờ |
| 6 | Cập nhật có điều kiện | Giá trị thay đổi trước lệnh ghi | Trả về `0` dòng, dừng xử lý |
| 7 | Lịch sử bất biến | Cập nhật phiên bản mới | Quyết định quá khứ tham chiếu nhất quán |

## 14. Hướng dẫn giảm thiểu Flaky Test

- Tránh việc sử dụng `Thread.sleep()` để quản lý luồng.
- Sử dụng Latch, Barrier hoặc Future với timeout rõ ràng để theo dõi tiến trình.
- Cấu hình từng actor (đọc và cập nhật) qua kết nối riêng biệt; không bọc Test bằng `@Transactional` bên ngoài.
- Thiết lập `ExecutionMode.SAME_THREAD` đối với cơ sở dữ liệu dùng chung (Testcontainers).
- Dọn dẹp dữ liệu trước khi thực thi từng case.
- In cấu trúc database diagnostic như `pg_stat_activity` hoặc `pg_locks` trong thông báo lỗi timeout để chẩn đoán.
- So sánh các thông số thực tế như Isolation Level, SQLSTATE và trạng thái hệ thống.
- Sử dụng cơ sở dữ liệu thực (ví dụ PostgreSQL) thay vì cơ sở dữ liệu in-memory (H2) khi kiểm tra hành vi MVCC.

## 15. Kiểm tra hệ thống (Production verification)

Truy vấn giao dịch và trạng thái lock:

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

Truy vấn đối tượng nắm giữ khóa:

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

Các chỉ số (Metrics) quan trọng:

- `refund.policy_revision_mismatch`: Phát hiện sai lệch phiên bản.
- `refund.conditional_insert_noop`: Số lần ghi có điều kiện không ảnh hưởng.
- `refund.decision_retry`: Tần suất phải thử lại quyết định hoàn tiền.
- `db.lock_wait` và `db.lock_timeout`: Tình trạng nghẽn khóa.
- `db.serialization_failure`: Lỗi do vi phạm tính tuần tự hóa.
- Đo thời gian xử lý giao dịch.
- Đếm số lượng quyết định không có audit log hợp lệ tương ứng.
