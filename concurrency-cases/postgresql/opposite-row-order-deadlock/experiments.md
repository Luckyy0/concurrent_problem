# Thực Nghiệm: Tái Hiện Deadlock Trong PostgreSQL (Controlled deadlock experiments)

## 1. Mục tiêu thử nghiệm (Objectives)

Bộ kiểm thử tương tranh (Concurrency test suite) cần chứng minh được tính an toàn và hiệu suất hệ thống thông qua các tiêu chí:

1. Thiết lập chính xác một tình huống chờ vòng tròn (wait-for cycle) dẫn đến deadlock thông qua sự tương tác giữa hai giao dịch PostgreSQL thực tế.
2. Đảm bảo hệ thống xác định chính xác một giao dịch nạn nhân (victim), nhận ngoại lệ `40P01` và rơi vào trạng thái `25P02`.
3. Khẳng định số tiền thay đổi của giao dịch nạn nhân sẽ bị khôi phục (rollback), bảo toàn nguyên vẹn tổng số dư tài khoản.
4. Chứng minh giải pháp áp dụng trật tự khóa chuẩn (canonical order) cho phép cả hai luồng tương tác an toàn.
5. Kiểm chứng cơ chế thử lại (retry) bắt buộc sử dụng một giao dịch cơ sở dữ liệu (database transaction) hoàn toàn mới.
6. Phân biệt rõ ràng lỗi deadlock (`40P01`) và lỗi vượt quá thời gian chờ khóa (`55P03` - lock timeout).
7. Áp dụng giới hạn thời gian thực thi (timeout) trên các lệnh chờ của cơ sở dữ liệu và mã nguồn Java để tránh tình trạng treo hệ thống.

Việc sử dụng các cơ sở dữ liệu in-memory như H2 không đáp ứng được yêu cầu đánh giá chính xác hành vi của trình phát hiện deadlock (deadlock detector) trong PostgreSQL. Các bài kiểm thử này phải sử dụng cơ chế điều phối luồng đồng bộ (synchronization barriers) ngay sau lệnh cấp khóa đầu tiên thay vì dùng `Thread.sleep` mang tính xác suất.

## 2. Thiết lập Môi trường Testcontainers (PostgreSQL Testcontainers setup)

```java
package example.transfer;

import static org.assertj.core.api.Assertions.assertThat;

import java.math.BigDecimal;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.time.Duration;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.function.BooleanSupplier;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class PostgreSqlDeadlockIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("deadlock_cases")
                    .withUsername("cases")
                    .withPassword("cases");

    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    @BeforeAll
    static void createSchema() throws SQLException {
        try (Connection connection = open();
                Statement statement = connection.createStatement()) {
            statement.execute("""
                    create table account (
                        id bigint primary key,
                        balance numeric(19, 2) not null
                            check (balance >= 0)
                    )
                    """);
        }
    }

    @BeforeEach
    void resetRows() throws SQLException {
        try (Connection connection = open();
                Statement statement = connection.createStatement()) {
            statement.execute("truncate table account");
            statement.execute("""
                    insert into account(id, balance)
                    values (101, 1000.00), (202, 1000.00)
                    """);
        }
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }

    private static Connection open() throws SQLException {
        return DriverManager.getConnection(
                POSTGRES.getJdbcUrl(),
                POSTGRES.getUsername(),
                POSTGRES.getPassword()
        );
    }
}
```

Để tối ưu thời gian thực thi trong môi trường test, giá trị `deadlock_timeout` được thiết lập ở mức thấp. Việc áp dụng cấu hình này vào môi trường sản xuất (Production) cần phải xem xét cẩn thận dựa trên đặc điểm phân bổ tài nguyên.

## 3. Tiện ích Điều phối luồng (Synchronization helper)

```java
private static void await(
        CountDownLatch latch,
        String description
) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Vượt quá thời gian chờ: " + description);
        }
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Luồng bị ngắt khi chờ: " + description, interrupted);
    }
}
```

Tất cả các đối tượng `Future` cần đi kèm với phương thức `get(timeout)` nhằm ngăn chặn tình trạng chờ vô hạn (infinite wait) trong trường hợp có lỗi luồng. Điều này đảm bảo CI/CD pipeline không bị treo khi kiểm thử thất bại.

## 4. Thực nghiệm 1 — Tái hiện Deadlock và Xác minh Rollback

Thử nghiệm giả lập trường hợp hai luồng thực hiện thao tác trên tài khoản nguồn sau khi được cấp khóa dòng đầu tiên, sau đó tiếp tục yêu cầu khóa tài khoản đích.
Lệnh `CountDownLatch` điều phối hai tiến trình cùng tiến vào giai đoạn yêu cầu khóa thứ hai, ép buộc xảy ra deadlock. Giao dịch bị chọn làm nạn nhân (victim) phải thực hiện rollback để đảo ngược trạng thái số dư tạm thời (partial debit).

```java
record AttemptOutcome(
        long fromId,
        String sqlState,
        String stateBeforeRollback
) {
    static AttemptOutcome committed(long fromId) {
        return new AttemptOutcome(fromId, "00000", "00000");
    }

    static AttemptOutcome failed(
            long fromId,
            String sqlState,
            String stateBeforeRollback
    ) {
        return new AttemptOutcome(fromId, sqlState, stateBeforeRollback);
    }
}

@Test
void oppositeOrderCreatesOneVictimAndRollsBackItsDebit() throws Exception {
    var firstRowsLocked = new CountDownLatch(2);
    var requestSecondRow = new CountDownLatch(1);

    Future<AttemptOutcome> t1 = executor.submit(() ->
            brokenTransferAttempt(
                    101, 202, new BigDecimal("100.00"),
                    firstRowsLocked, requestSecondRow
            )
    );
    Future<AttemptOutcome> t2 = executor.submit(() ->
            brokenTransferAttempt(
                    202, 101, new BigDecimal("70.00"),
                    firstRowsLocked, requestSecondRow
            )
    );

    // Chờ hai luồng được cấp khóa của tài khoản nguồn
    await(firstRowsLocked, "both source rows locked");
    
    // Tiếp tục yêu cầu khóa tài khoản đích
    requestSecondRow.countDown();

    List<AttemptOutcome> outcomes = List.of(
            t1.get(8, TimeUnit.SECONDS),
            t2.get(8, TimeUnit.SECONDS)
    );

    // Đánh giá: Có duy nhất một giao dịch thành công (00000) và một giao dịch lỗi deadlock (40P01)
    assertThat(outcomes)
            .extracting(AttemptOutcome::sqlState)
            .containsExactlyInAnyOrder("00000", "40P01");

    AttemptOutcome victim = outcomes.stream()
            .filter(outcome -> outcome.sqlState().equals("40P01"))
            .findFirst()
            .orElseThrow();
            
    // Kiểm tra trạng thái aborted của giao dịch nạn nhân
    assertThat(victim.stateBeforeRollback()).isEqualTo("25P02");

    // Đảm bảo tổng số dư hai tài khoản không đổi
    assertThat(totalBalance()).isEqualByComparingTo("2000.00");
    
    // Xác minh kết quả dựa vào luồng thành công
    assertThat(readBalances()).satisfiesAnyOf(
            balances -> assertThat(balances)
                    .containsExactly("101=900.00", "202=1100.00"),
            balances -> assertThat(balances)
                    .containsExactly("101=1070.00", "202=930.00")
    );
}
```

Chi tiết triển khai (Task implementation):

```java
private static AttemptOutcome brokenTransferAttempt(
        long fromId,
        long toId,
        BigDecimal amount,
        CountDownLatch firstRowsLocked,
        CountDownLatch requestSecondRow
) throws SQLException {
    try (Connection connection = open()) {
        connection.setAutoCommit(false); 
        configureDeadlockTestSession(connection);

        try {
            lockRow(connection, fromId);
            debit(connection, fromId, amount);

            firstRowsLocked.countDown();
            await(requestSecondRow, "permission to lock destination");

            lockRow(connection, toId);
            credit(connection, toId, amount);
            connection.commit();
            return AttemptOutcome.committed(fromId);
        } catch (SQLException failure) {
            String failedTransactionState =
                    executeAndReturnSqlState(connection, "select 1");
            connection.rollback();
            return AttemptOutcome.failed(
                    fromId,
                    failure.getSQLState(),
                    failedTransactionState
            );
        }
    }
}

private static void configureDeadlockTestSession(
        Connection connection
) throws SQLException {
    try (Statement statement = connection.createStatement()) {
        // Cấu hình phát hiện deadlock nhanh cho môi trường test (100ms)
        statement.execute("set local deadlock_timeout = '100ms'");
        statement.execute("set local lock_timeout = '2s'");
        statement.execute("set local statement_timeout = '4s'");
    }
}

private static void lockRow(
        Connection connection,
        long accountId
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
            select id
            from account
            where id = ?
            for update
            """)) {
        statement.setLong(1, accountId);
        try (ResultSet rows = statement.executeQuery()) {
            if (!rows.next()) {
                throw new SQLException("Missing account " + accountId);
            }
        }
    }
}

private static void debit(
        Connection connection,
        long accountId,
        BigDecimal amount
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
            update account
            set balance = balance - ?
            where id = ?
              and balance >= ?
            """)) {
        statement.setBigDecimal(1, amount);
        statement.setLong(2, accountId);
        statement.setBigDecimal(3, amount);
        if (statement.executeUpdate() != 1) {
            throw new SQLException("Insufficient funds: " + accountId);
        }
    }
}

private static void credit(
        Connection connection,
        long accountId,
        BigDecimal amount
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
            update account
            set balance = balance + ?
            where id = ?
            """)) {
        statement.setBigDecimal(1, amount);
        statement.setLong(2, accountId);
        if (statement.executeUpdate() != 1) {
            throw new SQLException("Missing account " + accountId);
        }
    }
}

private static String executeAndReturnSqlState(
        Connection connection,
        String sql
) {
    try (Statement statement = connection.createStatement()) {
        statement.execute(sql);
        return "00000";
    } catch (SQLException failure) {
        return failure.getSQLState();
    }
}
```

Việc thiết lập giá trị `lock_timeout` lớn hơn `deadlock_timeout` nhằm đảm bảo hệ thống có đủ thời gian ưu tiên ghi nhận lỗi deadlock (`40P01`) trước khi hết hạn thời gian chờ khóa. Các giới hạn timeout ở cấp luồng Java `Future.get(8, TimeUnit.SECONDS)` đóng vai trò giám sát tổng thể quá trình thực thi.

## 5. Thực nghiệm 2 — Đánh giá Trật tự khóa chuẩn (Canonical Order)

Kiểm chứng hành vi hệ thống sau khi áp dụng mô hình chuẩn:

```java
@SpringBootTest
@Testcontainers
class OrderedTransferIT {

    @Autowired
    private OrderedTransferAttempt attempts;

    @Autowired
    private JdbcTemplate jdbc;

    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    @Test
    void opposingTransfersUseSameOrderAndPreserveTotal() throws Exception {
        var start = new CountDownLatch(1);

        TransferCommand aToB = command(
                101, 202, new BigDecimal("100.00")
        );
        TransferCommand bToA = command(
                202, 101, new BigDecimal("70.00")
        );

        Future<TransferReceipt> first = executor.submit(() -> {
            await(start, "common start");
            return attempts.execute(aToB);
        });
        Future<TransferReceipt> second = executor.submit(() -> {
            await(start, "common start");
            return attempts.execute(bToA);
        });

        // Kích hoạt thực thi đồng thời
        start.countDown();
        first.get(8, TimeUnit.SECONDS);
        second.get(8, TimeUnit.SECONDS);

        // Đảm bảo số dư được cập nhật và tổng tiền được bảo toàn
        assertThat(balance(101)).isEqualByComparingTo("970.00");
        assertThat(balance(202)).isEqualByComparingTo("1030.00");
        assertThat(totalBalance()).isEqualByComparingTo("2000.00");
    }
}
```

Kiểm thử này cung cấp một mô phỏng sát với thực tế, áp dụng chiến lược khóa đã đề xuất để đảm bảo độ tin cậy của ứng dụng khi gặp tranh chấp.

## 6. Thực nghiệm 3 — Đánh giá Logic Sắp xếp ID (ID Sorting)

Tách biệt logic xác định trật tự tài nguyên:

```java
record AccountLockOrder(long firstId, long secondId) {

    static AccountLockOrder of(long leftId, long rightId) {
        if (leftId == rightId) {
            throw new IllegalArgumentException("duplicate account");
        }
        return leftId < rightId
                ? new AccountLockOrder(leftId, rightId)
                : new AccountLockOrder(rightId, leftId);
    }
}

@Test
void directionDoesNotChangeLockOrder() {
    assertThat(AccountLockOrder.of(101, 202))
            .isEqualTo(new AccountLockOrder(101, 202));
    assertThat(AccountLockOrder.of(202, 101))
            .isEqualTo(new AccountLockOrder(101, 202));
}
```

Cấu trúc trên thiết lập quy tắc bắt buộc để lấy ID tài nguyên cần khóa trước, bất kể đó là tài khoản chuyển (source) hay tài khoản nhận (destination).

## 7. Thực nghiệm 4 — Khởi tạo Giao dịch Mới khi Thử Lại (Retry Transaction)

Mô phỏng trường hợp lỗi trong lần chạy đầu tiên và đánh giá giao dịch thử lại:

```java
long transactionId = jdbc.queryForObject(
        "select txid_current()",
        Long.class
);
attemptProbe.record(command.commandId(), attemptNumber, transactionId);
```

Kiểm thử hành vi xử lý:

```java
start.countDown();
TransferReceipt firstReceipt = first.get(10, TimeUnit.SECONDS);
TransferReceipt secondReceipt = second.get(10, TimeUnit.SECONDS);

assertThat(firstReceipt).isNotNull();
assertThat(secondReceipt).isNotNull();
assertThat(attemptProbe.totalAttempts()).isEqualTo(3); 
assertThat(attemptProbe.deadlockVictims()).isEqualTo(1);
assertThat(attemptProbe.transactionIds())
        .hasSize(3) 
        .doesNotHaveDuplicates(); 

assertThat(totalBalance()).isEqualByComparingTo("2000.00");
assertThat(balance(101)).isEqualByComparingTo("970.00");
assertThat(balance(202)).isEqualByComparingTo("1030.00");
```

Quá trình này xác nhận hệ thống tạo ba giao dịch (Transaction ID) độc lập: giao dịch thành công đầu tiên, giao dịch nạn nhân và giao dịch mới khi thực hiện thử lại.

## 8. Thực nghiệm 5 — Phân biệt Lỗi Quá hạn Chờ Khóa (Lock Timeout)

Kiểm thử tình huống một chiều: một luồng giữ khóa A và một luồng khác chờ khóa A mà không gây tranh chấp chéo (circular wait). Do không có vòng lặp phụ thuộc, lỗi nhận được sẽ là `55P03` (lock timeout):

```java
@Test
void oneWayWaitReturnsLockTimeoutNotDeadlock() throws Exception {
    var holderLocked = new CountDownLatch(1);
    var releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = open()) {
            connection.setAutoCommit(false);
            lockRow(connection, 101);
            holderLocked.countDown(); 
            await(releaseHolder, "release lock holder");
            connection.commit();
        }
        return null;
    });

    Future<String> waiter = executor.submit(() -> {
        await(holderLocked, "holder locked row 101");
        try (Connection connection = open()) {
            connection.setAutoCommit(false);
            try (Statement statement = connection.createStatement()) {
                statement.execute("set local lock_timeout = '150ms'");
                statement.execute("set local statement_timeout = '2s'");
            }
            try {
                lockRow(connection, 101); 
                connection.commit();
                return "00000";
            } catch (SQLException failure) {
                connection.rollback();
                return failure.getSQLState(); 
            }
        }
    });

    // Xác nhận lỗi là lock timeout (55P03), không phải deadlock (40P01)
    assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    releaseHolder.countDown();
    holder.get(5, TimeUnit.SECONDS);
}
```

Ứng dụng không được nhầm lẫn mã `55P03` với `40P01`. Quá hạn khóa phản ánh sự chậm trễ hoặc khối lượng công việc tồn đọng (latency/contention), thay vì kiến trúc khóa đối kháng chéo. 

## 9. Thực nghiệm 6 — Mất Kết Nối Đột Ngột Giải Phóng Khóa

Mô phỏng ngắt kết nối (simulate crash) của giao dịch đang giữ khóa để kiểm tra cơ chế giải phóng tài nguyên của PostgreSQL:

```text
Tiến trình 1 bị đóng đột ngột (connection close).
PostgreSQL tự động rollback giao dịch chưa được commit.
Khóa bản ghi (row lock) được giải phóng.
Tiến trình 2 nhận được khóa và hoàn thành giao dịch.
```

Kiểm tra kết quả:

```java
assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo("ACQUIRED");
assertThat(totalBalance()).isEqualByComparingTo("2000.00");
assertThat(countOpenTransactionsForTestUsers()).isZero();
```

Trường hợp này chứng minh khả năng tự làm sạch (self-recovery) của CSDL. Tuy nhiên, nó cũng nhắc nhở về tầm quan trọng của việc kiểm tra trạng thái cập nhật (idempotency key/status) trong môi trường phân tán nếu có sự gián đoạn mạng bất ngờ.

## 10. Thực nghiệm 7 — Truy xuất Thông tin Chặn Khóa từ PostgreSQL

Thiết lập mô phỏng chặn một chiều (one-way wait), thu thập PID của tiến trình chờ thông qua `select pg_backend_pid()` và truy vấn trạng thái khóa hiện hành của PostgreSQL:

```sql
select a.pid,
       a.wait_event_type,
       a.wait_event,
       pg_blocking_pids(a.pid) as blocking_pids,
       a.xact_start,
       a.query_start
from pg_stat_activity a
where a.pid = :waiter_pid;
```

Cơ chế quét thông tin (polling) được sử dụng để truy vấn trạng thái:

```java
static void awaitCondition(
        Duration timeout,
        BooleanSupplier condition
) {
    long deadline = System.nanoTime() + timeout.toNanos();
    while (System.nanoTime() < deadline) {
        if (condition.getAsBoolean()) {
            return;
        }
        java.util.concurrent.locks.LockSupport.parkNanos(
                Duration.ofMillis(10).toNanos()
        );
        if (Thread.currentThread().isInterrupted()) {
            throw new AssertionError("Tiến trình kiểm tra bị gián đoạn");
        }
    }
    throw new AssertionError("Yêu cầu không được thỏa mãn trong thời hạn " + timeout);
}
```

Kết quả mong đợi: `wait_event_type = 'Lock'` và mảng `blocking_pids` chứa PID của tiến trình giữ khóa. 

## 11. Các hàm hỗ trợ đọc dữ liệu (Helper methods)

```java
private static BigDecimal totalBalance() throws SQLException {
    try (Connection connection = open();
            Statement statement = connection.createStatement();
            ResultSet row = statement.executeQuery(
                    "select sum(balance) from account"
            )) {
        row.next();
        return row.getBigDecimal(1);
    }
}

private static List<String> readBalances() throws SQLException {
    try (Connection connection = open();
            Statement statement = connection.createStatement();
            ResultSet rows = statement.executeQuery("""
                    select id, balance
                    from account
                    order by id
                    """)) {
        var result = new java.util.ArrayList<String>();
        while (rows.next()) {
            result.add(
                    rows.getLong("id")
                            + "="
                            + rows.getBigDecimal("balance").toPlainString()
            );
        }
        return List.copyOf(result);
    }
}
```

## 12. Ma trận các kịch bản kiểm thử (Test coverage matrix)

| Thực nghiệm | Tình huống kỹ thuật | Xác nhận kết quả (Assertion) |
| --- | --- | --- |
| 1 | Khóa chéo ngược chiều với điều phối | Ngoại lệ `40P01`, trạng thái aborted `25P02`, dữ liệu không thất thoát. |
| 2 | Sắp xếp trình tự chuẩn thông qua Spring | Giao dịch hoàn thành không xung đột, số dư được cập nhật. |
| 3 | Thuật toán sắp xếp khóa | Tính ổn định của Object lưu trữ trật tự. |
| 4 | Chạy thử lại (Retry) với giao dịch mới | Giao dịch mới được tạo, không trùng Transaction ID. |
| 5 | Chờ khóa lâu một chiều (Lock timeout) | Ngoại lệ `55P03`, không bị nhầm lẫn với deadlock. |
| 6 | Đứt kết nối đột ngột của tiến trình giữ khóa | Giao dịch chờ tiếp tục thực thi và thành công, không để lại rác dữ liệu. |
| 7 | Truy vấn giám sát tắc nghẽn PostgreSQL | Truy vết thành công qua hệ thống `pg_blocking_pids`. |

## 13. Khuyến nghị chống Flaky Test (Anti-flaky practices)

- Sử dụng cơ chế điều phối (Barrier) như Latch thay vì `Thread.sleep` để kiểm soát thứ tự thực thi.
- Cài đặt giá trị `deadlock_timeout` nhỏ hơn `lock_timeout`, và cả hai đều phải nhỏ hơn `statement_timeout` và timeout của `Future`.
- Làm sạch (reset) dữ liệu ở giữa các lần kiểm thử (test run) để ngăn chặn các tác động từ dữ liệu dư thừa.
- Không gắn cứng định danh về giao dịch nào sẽ trở thành nạn nhân (victim).
- Dọn dẹp an toàn các kết nối và lệnh thông qua cấu trúc `try-with-resources`.
- Quản lý quá trình kết thúc luồng (shutdown executor) có kiểm soát để tránh tài nguyên tồn đọng.
- Kiểm thử hành vi hệ quản trị cơ sở dữ liệu (RDBMS) không nên dùng in-memory databases (như H2).

## 14. Giám sát hệ thống thực tế (Production verification metrics)

Truy vấn nhanh cơ sở dữ liệu:

```sql
select datname, deadlocks
from pg_stat_database
where datname = current_database();
```

Các chỉ số phân tích bổ sung:

- Lỗi timeout chờ kết nối (connection pool pending/active).
- Lỗi khóa quá thời hạn (`lock_timeout`).
- Các giao dịch ở trạng thái treo (`idle in transaction`).
- Chỉ báo về việc sử dụng retry cho mã lỗi `40P01` với số lượng giới hạn và backoff phù hợp.
- Ghi nhận `deadlock_detected_total` theo các tác vụ định danh để cảnh báo tần suất vượt kiểm soát. 

Triển khai quy tắc sắp xếp tài nguyên (Canonical ordering) một cách nhất quán sẽ ngăn chặn tận gốc hiện tượng chờ chéo vòng tròn này.
