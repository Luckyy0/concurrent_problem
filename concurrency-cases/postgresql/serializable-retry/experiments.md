# Thực Nghiệm: SSI Và Cơ Chế Thử Lại Có Điều Phối (SSI and Retry Experiments)

## 1. Mục tiêu (Objectives)

Bộ kiểm thử (Test suite) này được thiết kế để chứng minh:

- Mức cô lập `READ COMMITTED` không ngăn chặn được lỗi dữ liệu khi 2 giao dịch đồng thời (concurrent transactions) xử lý cùng một nghiệp vụ, khiến giá trị bị thay đổi ngoài ý muốn.
- Mức cô lập `SERIALIZABLE` (SSI) ngăn chặn sai sót bằng cách hủy (abort) một giao dịch, trả về ngoại lệ `40001`.
- Giao dịch bị hủy (Victim) không được tái sử dụng để đọc hay ghi tiếp (tránh lỗi `25P02`).
- Giao dịch thử lại (Fresh retry) phải đọc lại toàn bộ trạng thái dữ liệu (state) và đưa ra quyết định dựa trên số liệu mới nhất.
- Yêu cầu lặp lại cùng một Command (Idempotent replay) sẽ không gây hiệu ứng phụ.
- Đảm bảo kiểm soát thời gian chờ bằng các cơ chế `timeout` cho tất cả các điều phối đa luồng.

> **Ghi chú quan trọng:** Thử nghiệm phải xác nhận giá trị tổng cuối cùng (final total) và kết quả được lưu (durable decisions) trong cơ sở dữ liệu thay vì chỉ bắt ngoại lệ, để chứng minh tính toàn vẹn của logic nghiệp vụ.

## 2. Thiết lập Môi trường Testcontainers (Testcontainers Setup)

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

Kiểm thử được định nghĩa một cách cách ly. Việc chạy nhiều kiểm thử song song (parallel testing) trên cùng một cấu trúc (Schema) có thể dẫn đến nhiễu (flaky test).

## 3. Tiện ích Điều phối (Coordination Helper)

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

## 4. Thực nghiệm 1 — Lỗi dữ liệu với `READ COMMITTED` (Baseline phá invariant)

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

Kiểm thử này xác nhận rằng ở mức `READ COMMITTED`, thao tác đọc điều kiện (predicate read) không khóa các giao dịch khác, cho phép cả hai tiến trình cập nhật dữ liệu, dẫn đến tổng số vượt mức giới hạn (120).

## 5. Thực nghiệm 2 — `SERIALIZABLE` tự động chặn một luồng (Creating a victim)

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

Lệnh kiểm thử không quy định cụ thể giao dịch nào phải bị hủy (abort). SSI bảo đảm rằng một trong hai sẽ bị loại bỏ với lỗi `40001`, và giao dịch còn lại thành công, duy trì tổng số là `90`.

## 6. Lõi thực thi (Core JDBC Attempt)

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

// ... các hàm insertReservation, insertDecision, executeAndReturnState tương tự mã ban đầu.
```

Đoạn mã này đóng gói các tác vụ giao dịch, xác định chính xác thời điểm gọi commit và trả về trạng thái thất bại thực tế (như `40001`).

## 7. Thực nghiệm 3 — Kết quả của việc Thử Lại (Fresh retry outcomes)

Sử dụng công cụ ngắt quãng (AttemptGate) trong Integration Test để mô phỏng một yêu cầu thử lại chính xác:
Sau khi giao dịch đầu tiên thất bại, tiến trình Coordinator sẽ gọi tạo giao dịch thử lại. Luồng xử lý mới phát hiện giá trị tổng đã được cập nhật thành `90`. Kết quả:
- Quyết định được đưa ra là `REJECTED`.
- Trạng thái dữ liệu được bảo vệ.
- Có tổng cộng 3 lần sinh Giao dịch vật lý mới (bao gồm lần lỗi và thử lại).

Cấu hình thử lại (Backoff) không nên can thiệp làm chậm hoặc treo các tác vụ Test. Quản lý thời gian trong Unit Test phải được cách ly khỏi hàm ngủ (sleep delay).

## 8. Thực nghiệm 4 — Tính lũy đẳng (Idempotent replay)

Khi lặp lại thao tác với một Command ID cũ (đã được quyết định `ACCEPTED`):

```java
long reservationsBefore = reservationCount();
long decisionsBefore = decisionCount();

LimitDecision replayed = coordinator.reserve(sameAcceptedCommand);

assertThat(replayed.outcome()).isEqualTo(ACCEPTED);
assertThat(reservationCount()).isEqualTo(reservationsBefore);
assertThat(decisionCount()).isEqualTo(decisionCount());
assertThat(activeTotal()).isEqualByComparingTo("90.00");
```

Giao dịch sẽ đọc kết quả từ bảng lưu trữ, trả về `ACCEPTED` mà không sinh thêm bản ghi reservation, đảm bảo an toàn tuyệt đối ngay cả khi luồng điều phối bị gọi lặp.

## 9. Thực nghiệm 5 — Khẳng định ranh giới mức cô lập (Effective isolation test)

Kiểm tra trực tiếp tại lớp thực thi:

```java
String isolation = jdbc.queryForObject(
        "select current_setting('transaction_isolation')",
        String.class
);
assertThat(isolation).isEqualTo("serializable");
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isTrue();
```

Xác minh này loại trừ khả năng Spring Proxy không thiết lập mức cô lập đúng. Môi trường kiểm thử không được bọc hàm kiểm tra trong `@Transactional` tại tầng Unit Test, để không làm lu mờ thiết kế ranh giới giao dịch bên dưới.

## 10. Thực nghiệm 6 — Quan sát khóa SIReadLock (Observability)

Sử dụng lệnh để giám sát khóa điều kiện do cơ chế SSI kích hoạt:

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

Hệ thống sẽ cấp một khóa `SIReadLock`. Không nên kỳ vọng quá lớn vào việc định cỡ (granularity) chính xác cho khóa này (có thể là row, page, relation) vì điều đó phụ thuộc vào Kế hoạch thực thi (Execution Plan) của PostgreSQL.

## 11. Thực nghiệm 7 — Giao dịch Đọc trì hoãn (Read-only deferrable)

Khi thiết lập giao dịch đọc thông thường cho báo cáo:

```sql
begin isolation level serializable read only deferrable;
select current_setting('transaction_isolation'),
       current_setting('transaction_read_only'),
       current_setting('transaction_deferrable');
```

Xác nhận hệ thống trả về thông số `serializable`, `on`, `on`. Việc này cho phép giao dịch đọc hoạt động tự do và nhận Snapshot an toàn mà không ảnh hưởng tới các luồng Ghi.

## 12. Phương thức tiện ích (Helper Methods)

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

## 13. Ma trận kiểm tra bao phủ (Coverage Matrix)

| Thí nghiệm (Experiment) | Điều kiện Kiểm thử Kỹ thuật | Điều kiện Đánh giá Nghiệp vụ |
| --- | --- | --- |
| 1 | Hai luồng `READ COMMITTED` đồng thời | Tổng tiền bị đội lên `120`, vi phạm toàn vẹn |
| 2 | Áp dụng `SERIALIZABLE` | Một luồng trả về `40001`, tổng duy trì mức `90` |
| 3 | Xử lý Thử lại (Fresh Retry) | Kết quả của một lệnh là `ACCEPTED`, lệnh kia `REJECTED`, tổng vẫn là `90` |
| 4 | Gọi lặp lệnh cũ (Idempotent replay) | Không thay đổi số lượng bản ghi tổng số |
| 5 | Mức độ cô lập của Proxy | Lệnh luôn được chạy dưới mức cô lập `serializable` thực sự |
| 6 | Kiểm tra `SIReadLock` | Xác nhận hiện diện của cơ chế giám sát khóa predicate của PostgreSQL |
| 7 | Chế độ Đọc Deferrable | Xác minh cấu hình dành riêng cho các tiến trình đọc dữ liệu an toàn |

## 14. Đảm bảo độ tin cậy kiểm thử (Anti-flaky Guidelines)

- Đồng bộ hóa các luồng xử lý (Latches) sau khi truy vấn điều kiện đọc (predicate read) hoàn tất.
- Tránh phụ thuộc vào mã định danh giao dịch (Identity) của luồng bị hủy, hệ thống không chỉ định luồng nào sẽ chết.
- Thiết lập cơ chế ngắt thời gian chờ (watchdog) bằng `statement_timeout` nhỏ hơn `Future.get`.
- Bất kỳ phát sinh lỗi nào cũng phải đóng giao dịch bằng `rollback` kèm cấu trúc mã an toàn (try-with-resources) cho connection.
- Làm sạch (Reset) hoàn toàn schema và dữ liệu (state) trước mỗi đợt kiểm tra độc lập.
- Thiết lập log chi tiết khi xảy ra lỗi chờ quá hạn (timeout) bằng cách quan sát `pg_stat_activity`, `pg_locks` để đánh giá hệ thống.

## 15. Hướng dẫn giám sát môi trường sản xuất (Production Observability)

Phân loại Log/Metrics cho ứng dụng:
- Bắt và theo dõi sự cố do SQLSTATE `40001`.
- Thống kê tỷ lệ Thử lại, thời gian chạy thử lại thành công (Success-after-retry) và số lần từ bỏ (Exhaustion).
- Xem xét tỷ lệ vi phạm Command ID (Duplicate command).
- Áp dụng thời hạn xử lý tối đa (Transaction Duration) và duy trì sự nhạy bén của Connection Pool.
- Tuyệt đối không ghi các dữ liệu nhạy cảm của khách hàng hay thông số PII vào Log khi quan sát sự cố.
