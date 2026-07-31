# Phòng Thí Nghiệm: Ép Lỗi Đọc Không Lặp Lại Bằng Mọi Giá (Deterministic non-repeatable-read experiments)

## 1. Mục tiêu (Objective)

Đã viết Test suite thì phải chứng minh cho bằng được mấy cái này:

1. Chế độ `READ COMMITTED` có thể quăng ra cái `revision` khác nhau giữa hai lệnh SELECT liên tiếp.
2. Việc chọt ngang (Interleaving) đó đủ sức tạo ra một Quyết định Vi phạm trắng trợn Luật Lệ kinh doanh.
3. Chế độ `REPEATABLE READ` ráng giữ Bức Ảnh Ổn Định nhưng đéo hứa hẹn nó là Bức Ảnh Mới Nhất (latest-at-commit).
4. Khi ông B Xù Kèo (rollback), phiên bản mới tuyệt đối KHÔNG ĐƯỢC lòi ra.
5. Xài Ổ Khóa `FOR SHARE` phải bắt thằng Cập Nhật chờ hoặc Văng Lỗi (fail) đàng hoàng theo mốc Hết Giờ (bounded timeout).
6. Viết Lệnh Thẩm Định (Conditional validation) phải chả về số dòng cập nhật (affected-row) = `0` khi Luật đổi.

Nhớ nha: Tuyệt đối KHÔNG dùng mấy cái lệnh dở hơi như Ngủ (Delay theo wall clock) để điều phối. Phải xài Cổng Chặn (Latch) ép sếp B chốt lệnh CHÍNH XÁC ngay giữa 2 lệnh Đọc; Mọi lệnh chờ (`await`) hay `Future.get` ĐỀU PHẢI có giới hạn thời gian (timeout).

> **Nói ngắn gọn:** Viết Test là phải Nắm Thóp trật tự Chốt Sổ và tự tay soi Cổng Kiểm Soát Lỗi, chứ KHÔNG CHỈ in mồm ra 2 cái giá trị khác nhau rồi chắp tay cầu nguyện cho Trình Lập Lịch (scheduler) hệ điều hành nó chạy trúng!

## 2. Dựng Trại Testcontainers (PostgreSQL Testcontainers)

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

Nhắc lại: Trong Test KHÔNG Gắn `@Transactional` lồng bên ngoài (outer). Mỗi diễn viên phải Tự Mở Kết Nối JDBC và Giao dịch vật lý RIÊNG BIỆT của mình!

## 3. Cổng Điều Phối (Coordination gate)

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
                throw new AssertionError("Hết giờ! Đứng chờ " + step + " quá lâu!");
            }
        } catch (InterruptedException ex) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Đang chờ " + step + " thì bị đá văng!", ex);
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

## 4. Các Đồ Nghề JDBC Chặt Chẽ (JDBC helpers)

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

Kể cả thằng chép (writer) lăn đùng ra chết ngỏm trước khi Chốt, thằng Đọc (reader) cũng sẽ văng lỗi Timeout chứ không treo máy. Bằng chứng Phạm Tội gốc rễ (exception) vẫn nằm trong `Future` để soi.

## 5. Thí Nghiệm 1 — `READ COMMITTED` Lòi Ra 2 Cái Revision

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

Bắt quả tang tận tay sự tráo trở của dữ liệu Kinh doanh ngay giữa 2 lần bấm máy chụp ảnh (statement snapshots).

## 6. Thí Nghiệm 2 — Râu Ông Nọ Cắm Cằm Bà Kia (Broken decision vi phạm invariant)

Người đọc giữ kết quả xét duyệt từ Tấm Ảnh đầu, nhưng lại thó cái Revision của Tấm Ảnh thứ hai nhét vô CSDL:

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

    // Assert CỐ TÌNH SAI LỆCH để Bắt Bài cái Bug!
    assertThat(decision.outcome()).isEqualTo("APPROVED");
    assertThat(decision.policyRevision()).isEqualTo(current.revision());
    assertThat(decision.amount()).isGreaterThan(current.limit());
    assertThat(decision.evaluatedLimit()).isNotEqualByComparingTo(current.limit());
}
```

Bài Test này TỰ GIÀY VÒ bản thân, cố tình Assert Cái Kết Tệ Hại để Khóa Cứng (reproduction) cái Bug. Lúc code xong giải pháp, phải lật ngược (Regression test) cái cờ lại thành:

```java
assertThat(decision.amount())
    .isLessThanOrEqualTo(policyHistory.limitAt(decision.policyRevision()));
```

## 7. Thí Nghiệm 3 — `REPEATABLE READ` Ôm Khư Khư Tấm Ảnh Đầu

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
    assertThat(observed.second().revision()).isEqualTo(7); // VẪN LÀ 7 NÀY!
    assertThat(inspector().policy(MERCHANT_ID).revision()).isEqualTo(8); // THỰC TẾ LÀ 8 RỒI NÀY!
}
```

Lưu ý đoạn này rất đắt giá: Ông A vẫn Đọc được Revision ổn định `7`, nhưng dữ liệu Thật Dưới Database đã chốt là `8`. Đứng Yên Không Đổi (Stable) Không Có Nghĩa là Mới Nhất (latest)!

## 8. Thí Nghiệm 4 — Đứa Ghi Xù Kèo (Updater rollback không visible)

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
            connection.rollback(); // B XÙ KÈO!
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

Chữ `writerCommitted` ở trong helper chỉ có nghĩa là "Cờ Báo Xong Chuyện" thôi nhé, đừng nhầm lẫn với Commit và Rollback Thật trên Production.

## 9. Thí Nghiệm 5 — Bùa `FOR SHARE` Khóa Chặn Mõm Thằng Đổi Luật

Gắn trò Timeout Mới Nhất (`lock_timeout`) để Ép Kẻ Xâm Nhập Bỏ Cuộc:

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

Mã Lỗi SQLSTATE `55P03` nghĩa là Éo-Có-Khóa-Mà-Đợi-Quá-Lâu (lock-not-available). Tránh vỏ dưa gặp vỏ dừa, khi A chốt xong, một Giao dịch MỚI TINH của B vẫn được phép Đọc lại rồi UPDATE lên `8`.

## 10. Thí Nghiệm 6 — Chốt Chặn Bằng Điều Kiện (Conditional validation)

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

Đoạn Test ngầu lòi, vừa kiểm chứng cờ Lỗi Tranh Chấp, vừa chứng tỏ không có "Rác Dữ Liệu" (side effect) nào lọt vô DB.

## 11. Thí Nghiệm 7 — Kho Lịch Sử Cứng Nhắc Giữ Mạng (Version history giữ audit evidence)

Sau khi Chuyển Đổi sang Bảng Lưu Lịch Sử (Migrate schema versioned):

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

Đây là đỉnh cao của Audit Model Regression Test: Dù dòng thời gian trôi đi, Bằng Chứng Quyết Định vẫn Gắn Liền Mãi Mãi với Dữ Liệu Quá Khứ.

## 12. Bãi Rác Bọc Sẵn (Shared helpers)

```java
private static void await(CountDownLatch latch, String step) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Hết giờ chờ " + step);
        }
    } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Bị đá văng lúc đang đợi " + step, ex);
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

Import các Bùa Cần Thiết:

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

## 13. Ma Trận Bao Phủ Bài Test (Coverage matrix)

| Cuộc Thí Nghiệm | Mức Cô lập/Khóa (Isolation/lock) | Việc Trộn Nhau (Interleaving) | Mục Tiêu Kinh Doanh Đã Bắt Được |
| --- | --- | --- | --- |
| 1 | `READ COMMITTED` | B Chốt giữa 2 phát SELECT | Revision Đảo Chiều `7 -> 8` |
| 2 | `READ COMMITTED` | Râu S1 Cắm Audit S2 | Bằng chứng Duyệt Tan Nát |
| 3 | `REPEATABLE READ` | B Chốt giữa 2 phát SELECT | A Vẫn Đọc `7 -> 7`, DB là `8` |
| 4 | `READ COMMITTED` | B Ghi Xong Xù Kèo | Đứa Đọc Vẫn Thấy `7 -> 7` |
| 5 | `FOR SHARE` | B GHI đè lúc A Giam Khóa | B Văng Lỗi `55P03`, Luật Đứng Yên |
| 6 | Gắn Điều Kiện Đuôi | B Chốt Trị Số trước Lệnh Ghi Sổ | Cập nhật dòng `0`, Đóng sổ sớm |
| 7 | Lịch Sử Bất Khả Xâm Phạm | Cổng chỉ tới Revision Mới Nhất | Quyết định xưa Cũ vẫn Dò Lại Được |

## 14. Cách Chống Đứt Gãy Test Nhảm Nhí (Chống flaky)

- Cấm Tuyệt Đối trò Ngủ 1 giây để chốt sổ!
- Mọi nút bấm Cổng (latch)/Tương lai (future) bắt buộc xài Timeout và Giữ cờ ngắt mạch (interrupt flag).
- Người Đọc/Người Ghi Bắt buộc Dùng 2 Kênh (connection) riêng. File Test không Đội Mũ Giao Dịch ngoài cùng.
- Phải nhốt các hàm chạy vào Chung 1 luồng (`SAME_THREAD`) vì xài chung CSDL Container.
- Diệt sạch cỏ (Reset committed data) trước khi chạy mỗi hàm test.
- Khi bị Dính Timeout, in mọe cái Dump `pg_stat_activity` và `pg_locks` ra màn hình, đừng có rảnh mà vặn tăng thời gian Timeout nhé.
- Xác Minh Kép: Xem Trị Số Mức Cô Lập đang xài (`current_setting`), Mã SQLSTATE văng ra và Tình Trạng Hàng Cuối cùng.
- Cấm Lấy Cái CSDL Đồ Chơi H2 ra để bao biện cho Tính Năng MVCC Khủng Bố Của PostgreSQL!

## 15. Bộ Câu Hỏi Bắt Lỗi Cho Dân Cấp Cứu Production (Production verification)

Bắn Câu Hình Tội Phạm:

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

Soi Băng Nhóm Cầm Khóa:

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

Bảng Phong Thần Metrics/Logs:

- `refund.policy_revision_mismatch`: Phát còi hụ Lệch Phiên Bản!
- `refund.conditional_insert_noop`: Vỗ Tay vì Ghi Đè Hụt!
- `refund.decision_retry`: Sổ tay số lần Thử Lại Quyết Định;
- `db.lock_wait` và `db.lock_timeout`: Khóc thé vì Chờ Khóa quá lâu;
- `db.serialization_failure`: Chết Ngắt vì Vi Phạm Phân Tách;
- Đo thời gian ngâm Giao dịch (transaction duration);
- Cảnh Cáo Số Lượng Quyết Định Đã Duyệt Mà Éo Dò Được Lịch Sử.

Chó săn (Canary) thả vô Production thì phải rà soát được Cái Invariant trên Đống Dữ Liệu Đã Chốt Sổ, đéo được phán an toàn bằng cách "Ồ, không thấy Bắn Exception Count lên đồ thị".
