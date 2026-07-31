# Phòng Thí Nghiệm SSI Lõi Và Điều Khúc Gọi Retry Lại Sạch Sẽ (Các thí nghiệm SSI và retry có điều phối)

## 1. Mục Tiêu Chốt Bảng (Mục tiêu)

Bộ Suite test này mài ra phải nắn gân chứng minh được mớ này nè:

- Bọn mặc giáp Rách `READ COMMITTED` lơ ngơ cho 2 tờ vé cọc lọt vô commit êm ru kéo mẹ cái tổng vọt mẹ lên `120`;
- Chiêu xịn `SERIALIZABLE` sút bóp cổ (abort) ĐÚNG 1 Kẻ Thua Cuộc xì khói `40001`;
- Thằng Chết Bỏ Mạng (victim) ĐÉO ĐƯỢC trây trét rác vé cọc hay Phán Quyết nháp lại; Còn cục sình transaction cũ cứ móc query xuống là rớt tạch `25P02`;
- Đứa Tái Sinh Mới Tinh Tươm (fresh retry) ngắm lại đúng bản tổng đẹp `90` xong cắm mốc Phán Xăm `REJECTED` êm ái;
- Thét Tái Chạy Lệnh Y Chang (command replay) BÍt Tịt Cửa Tuyệt Không Ục Cức Ra Cái Hậu Quả Ác Mới Nào Khác Nhé (side effect);
- Mọi cái cục khoá cổng Latch, mớ Future hẹn Hò hay từng khúc rặn nháp Database đều bị Khóa Cáp Chặn Giờ (timeout) Cứng Khừ!

> **Sếp chốt lại:** Trò test mót nhặt bắt hứng Exception chỉ lòe được tụi nít ranh báo có 1 bãi Conflict Thôi! Đo đếm cho ra con Tổng Tiền Máu Chót (final total), Mớ Án Chốt Sạch Boong Đẹp Mẽ (durable decisions) Lẫn Hàng Lốc Transaction IDs MỚI TINH Kéo Nặn Lại Ra Hồn thì lúc đó Test Tái Sinh Retry mới Phê Chuẩn Nhé!

## 2. Bãi Tập Testcontainers Hàng Hiệu (Môi trường PostgreSQL Testcontainers)

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

Nhớ nghe, mỗi cái mâm test method nó ngậm khư khư đồ nháp fixture instance / ổ máy nén executor cho riêng tư mình ên. Dặn kỹ: TUYỆT MỆNH CẤM nhồi các test sáp lá cà lộn lạo chạy Song Song trên cùng 1 bệ Schema Đơn Tuyến này nhé Kẻo Vỡ Mõm Toàn Bộ!

## 3. Thằng Bóp Kẹp Dẫn Tuồng Đồng Diễn (Helper điều phối)

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

## 4. Thí Nghiệm Số Đỏ 1 — Oẳn Tù Tì Xách Quần `READ COMMITTED` Lủng Khung Kẽ Luật (Baseline phá invariant)

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

Test này éo thèm rảnh Hô Toáng Họng Khẳng Định Giáp `READ COMMITTED` Bét Nhè 100% Cứt Bẩn; Mà Nó Chọt Tay Minh Họa Chỉ Rõ: Áo Lưới Này Bị ĐỤC LỦNG Kẽ Bảo Chứng Dàn Bục Rờ (predicate invariant) Nếu Mày Bỏ Ngõ Quăng Phế Rào Che Bọc Chốt Kép Dọn Thối Ngầm Nén Bệnh Thở (guard/constraint/conditional mutation) Ra Chuồng Gà Đấy!

## 5. Thí Nghiệm Đập Bàn 2 — Bùa Phép `SERIALIZABLE` Trảm Phọt 1 Khứa Chết Tươi (tạo một victim)

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

Éo Rảnh Tay Ép Định Rằng Chính Xác Khứa Nào Phải Tử Thương Nằm Kéo Án Lệnh Đo Đỏ Đâu Nha. Kép Vắng Mặt Khuyết Nhõn Mẻ Chốt Án (decision thiếu) LÀ ĐÃ TRÚNG CHÓC Ý TỒ ĐÌNH (expected), Vì Cục Test Này Cứt Chó Có Chọc Ọc Phọt Khởi Sinh Test Bọc Mõm Chạy Lại Đâu Mà Nhú Mép!

## 6. Lõi Đục Ống Core JDBC (Core JDBC attempt)

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

Úp bọc cái túi lưới Rách Chẻ Đôi Bọc Kín Mõm Gọi `commit()` Xuống Dưới Rốn Mâm Catch, Căn Do Khét Ngòi `40001` Của Tụi Nó Đéo Hề Chọn Ra Đúng 1 Mõm Kênh Khặc Lòi Ói (throw site) Mọc Cọc Chuẩn Đâu.

## 7. Thí Nghiệm Đánh Úp Số 3 — Lột Xác Mới Toanh Tát Chết Sạch Bóng Lệnh Lách (Fresh retry tạo business rejection)

Tụi đấm Test Trọng Spring integration nó tiêm chọc nặn 1 bãi cổng ngáp nhử mồi `AttemptGate` (test-only) SỚP Thẳng Tắp Sau Đuôi Bóng `activeTotal()` Của 2 Kép Xông Tiền Tuyến (first attempts):

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

Ở Sân Sản Xuất Thật (Production implementation) Hàm Nhép Đọc Báo Khống Có Giá No-op Toạc Nhanh Nhé; Rào Chắn Kia Đéo Nằm Lộn Vô Mã Code Thực Cày Tiền Mõm Nghe. Bọn Xúc Tu Đo Lệnh Probe Xắn Ngón Khứa Lôi Lệnh Đọc `txid_current()`, Kẽo Nhãn Isolation Phép Lọc Khúc Lồi Đít Kết Cục Attempt Dọng Thụt Xoáy Bộ Nhớ Cách Trấn An Toàn (thread-safe memory).

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

Chiêu Bày Sáng Mặt Bọc Đo Backoff Rởm ĐÉO KÉO Tuột Khứa Chặn Treo Chân Test Phập Phù Nhen. CÒN Ruột Nhồi Backoff Đắp Lệnh Trại Thật Oai Ngoài Production Vẫn Có Nhét Phễu Cứt Unit Test Bọc Sát Riêng Xé Chút Nháp Cap (Đỉnh Rút), Jitter Range (Tụ Rụt Lệnh Dãn), Deadline (Biên Cáo) Lẫn Mũi Kẹp Interrupt (Cắt Đuôi Nghẽn Cổ) Đàng Hoàng Đấy Khờ!

## 8. Thí Nghiệm Oai Phong Nhóm 4 — Ép Phọt Tua Chạy Lại Vẫn Trắng Tay Lệnh Sứt Cọc (Replay không tạo reservation mới)

Gõ Tráp Ộc Bắt Tua Gọi Lại Đứa Lệnh Lỏi Cũ Rít Thẻ Đứt Đoạn Láo Đã Lót Mép `ACCEPTED` Dọc Khứa Phán Búa Commit Ầm:

```java
long reservationsBefore = reservationCount();
long decisionsBefore = decisionCount();

LimitDecision replayed = coordinator.reserve(sameAcceptedCommand);

assertThat(replayed.outcome()).isEqualTo(ACCEPTED);
assertThat(reservationCount()).isEqualTo(reservationsBefore);
assertThat(decisionCount()).isEqualTo(decisionsBefore);
assertThat(activeTotal()).isEqualByComparingTo("90.00");
```

Quay Tua Tút Y Xì Rập Cùng Đo Ánh Mặt command Mác `REJECTED`: Gáy Lệnh Kéo Cứt Phán Dính Y Chóc Mõm Dù Cho Trọng Lệnh Chót Đo Líp Total Giảm Phọt. ĐẤY! Cốt Cách Bóng Sánh Lệnh Kiên Định Chết Boong Này Đấy (durable command semantics); Nháp Ruột Khứa Business (domain) Bày Trò Chẻ Sát Soi Oái Xem Xét Re-evaluate Mõm Đéo Được Chọt Khống Lấy Nét Mã Cọc Lệnh CŨ Cứt Hôi, Bắt Bị Lôi Khứa Cốt Nặn Cục Lệnh Rọt Bề MỚI Tươi Mép Nhá (model mới)!

## 9. Thí Nghiệm Đụt Nút Háng 5 — Bắt Cứng Khoang Bóng Isolation Áo Bọc (Xác minh effective isolation)

Kẽ Ruột Gọi Của Thằng Lính Đánh Nháp Lại (attempt worker):

```java
String isolation = jdbc.queryForObject(
        "select current_setting('transaction_isolation')",
        String.class
);
assertThat(isolation).isEqualTo("serializable");
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isTrue();
```

Trấn Thép Đóng Bộ Test Lọc Lưng (Architecture/integration test) Móc Đáy Kẽ Lệnh Vọc Chóp Coordinator Sứt Lên Sát Đứa Test Bean Khứa Ngạp Lõi Rọn Đít Transaction (transactional test bean) TRẤN Lột Xé Rào Chặn Tuyệt Giao Gạt Quách Outer Transaction! ĐẾCH KÉO CẤM KHÍA RỌT CỤC `@Transactional` Đè Mũ Lên Nóc Lệnh Gọi Dọng Tụt Dây JUnit method Đo Cọc; Cụt Sụp Ốp Áo Phết Nháp Test Transaction Trùm Ở Tầng Ngoài Che Mất Kẽ Điểm Oái Đập Đinh/Tụt Nút (commit/rollback) Chết Vỡ Sân Kìa Khờ!

## 10. Thí Nghiệm Móc Lọc Xé Cực Bọc 6 — Chụp Dính Mã Khóa Nấp Lọng Lệnh `SIReadLock` (Quan sát)

Cố Chóp Cắm Rễ Bó Lì Dọng Sống Cùng Ánh Transaction Đỏ Kép Bọc Bóng Serializable Khóc Há Họng Ục Giương Khứa Predicate Read BẰNG Dọng Giăng Latch Ép Lưới Đỉnh (bounded latch). Kẽ Rút Từ Khứa Cáp Mép Đóng Rễ Rà Báo Lệnh Kéo Dọc (observer connection):

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

Ép Tụ Khống Chỉ Soi Ít Nhất Hiện Đít 1 Lá Sớ `SIReadLock`, CẤM TRÁO NHÉM ĐÓNG NHĂN TÙ DỌNG (hard-code) Phết Mức Đo Cụt Nhóp Tuple/Page/Relation Kép Sót Bìa granularity Nhanh Nhé Cưng! Chảo Nháp Plan Lệnh Query, Cọc Điểm Số Líp Statistics Lẫn Mớ Hóa Khắc Đáy Thăng Hoa Khóa Cáp Dịch predicate-lock promotion Thay Da Đổi Áo (shape) Trượt Lộng Xéo Kéo Sáng Oái Bất Tử MÀ ĐÉO Lật Úp Chiếc Thuyền Lệ Chuẩn Trách Oái Kẻ Lạch Trọng Điểm Độ Ranh Correctness Tí Móng Chó Nào!

## 11. Thí Nghiệm Húp Gáy Chót 7 — Tua Nháp Bóng Nằm Gục Đọc Ngáp (Read-only deferrable)

Khứa Phanh Trần Chày Raw connection Chọc Móc Trượt Ngay:

```sql
begin isolation level serializable read only deferrable;
select current_setting('transaction_isolation'),
       current_setting('transaction_read_only'),
       current_setting('transaction_deferrable');
```

Assert Vọt Xé Bọng Báo Đúp Tít `serializable`, `on`, `on` Cắm Cú Khứa Chốt Report Bọc Bóng Viền Gác Vòng Dọng Cuốn Nhép watchdog. Cuộc Đua Test Này LÉO Trọng Móng Cắm Lệnh Nhai Xực Khống Mode Tịt Ục Cho Móng Hút Cứa Ghi Lọt Khứa Attempt Khờ Nhé; Ả Léo Chỉ Phán Bóp Chết Lọc Contract Rành Giữa Móng Nháp Đeo Alternative Áo Reporting Trắng Mặt Kéo Nhanh Thôi Nhóc Á!

## 12. Cái Giẻ Rách Tụt Kẽ Khám Nhép Cục Đo Lệnh Điểm (Helper đọc business state)

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

## 13. Khay Soi Hột Tụt Đít Tổng Quan (Ma trận bao phủ)

| Ván Gieo Chóp Nhóp (Thí nghiệm) | Tịch Áo Đo Bọc Kẽ (Assertion kỹ thuật) | Án Tịch Bóng Kéo Vỡ Nát (Assertion nghiệp vụ) |
| --- | --- | --- |
| 1 | Cả Trọng Chóp Nhóp Kéo 2 Vòng `READ COMMITTED` Lủng Bóng Trơn Cả Commits Lọt Lưới | Vọt Trọng Khúc Total Hất Đáy `120`, Phọt Rõ Bọc Anomaly Oai Bể Mạch Cứt Trào |
| 2 | Chóp Khía Thét Máu `40001` Bơm + Trào Họng `25P02`, Giữ 1 Giao Phép Rốn Commit | Chọc Tịt Cắn Bóng Trọng Đít Líp Total Đo Vát Lọng Bền Bóp Rọt Tròn `90`, Kẹp Nọng Lọt 1 Bọc Đáy Án Lệnh Đo Tịt Decision Chót |
| 3 | Ba Bóng Nháp Vút Chẻ Đôi Kẽ txids Lì Đòn Đẹp Nhá, Sân Lọc Trắng Tươi Fresh Retry Khéo | Một Tích Rọt Accepted Đẹp Nép Bọc, Một Chút Nhão Cút Nát Rejected Vọt Sân Đo, Trọn Sóng Khứa Lọt Điểm Kẽ Lưới Chút Lóng Total Sáng Móc Bờ `90` |
| 4 | Trận Áp Khóa Kéo Tái Khứa Sót Command Replay Kẽ Chót Khéo Bóng Kẽ Vọt Durable Bọc Láo Nhá Bọn Tục | Sóng Đếm Chút (Counts) / Chút Gọng Lọng Total Chết Kép Cứng KHÔNG Rìa Đổi Lay! |
| 5 | Áo Đo Cứt Kéo Soi Giáp Nặng Isolation Đáy Rìa / Lưới Đáy Kéo Vây Proxy Khéo Rụt Đít Tịt Ngáp | Mọi Ván Thằng Trụy Bọc Khứa Attempt Luôn Chết Trắng Áo Trọng Soi Lọng `serializable` Sáng Boong! |
| 6 | Trát Sớ Bọn Móc Kẽ Đinh Khóa `SIReadLock` Chóp Dọc Trực Ngã Bóng Trong Ổ Chó Cụp Kẹp Dòng `pg_locks` | Khép Dọc Soi KHÔNG Gập Cứt Lệ Khứa Tịch Granularity Chống Sóng Khóa Bóng Nhang Nhéo Oan! |
| 7 | Tọc Lọng Trọng Settings Móc Bọn Chóp Read-only Cựa Tịt Mép Nhá Gãy Rút Kẽ Deferrable Móc Ánh Ngạp Hẹp Sóng Vụt Kẹp Đáy | Bọc Sóng Khứa Trích Hẹp Án Tịt Alternative Đóng Cứt Hơi Dọc Khứa Phép Lọng Đúng Phân Lẽ Cựa Kéo Khéo Sân Mode Áp Bất Ngờ Nhóc À! |

## 14. Bày Lưới Cản Bọn Bịp Bệnh Nhát Nhác Rụt Lưng Mót (Chống flaky)

- Trấn Dọc Hàng Rào Chắn Chóp Ngay Tịt Cựa Cắn Đáy Trượt Nhóp SAU KHI Đóng Vút Nứt Kéo 2 Nhát Đọc Khứa Predicate Reads!
- Tịt Ngáp CẤM Soi Vút Chút Assert Đo Trọng Dọng Lọc Nặng Thằng Oái Khứa Victim Identity Hay Oái Điểm Cắn Oanh Statement Ném Thét Oai Quả Đạn `40001` Của Tụi Nó!
- Lóng Dọng Khứa Kẽ Áo Nón Bọc Giáp Trọng Trụy Rụt Khớp Máng Chó Chóp `statement_timeout < Future.get` Lệnh Canh Cửa Ánh Mõm Watchdog Ngáp Ổn.
- Mọi Con Hẻm Oác Nạn Lỗi Trượt SQL Xụp Nứt Vành Đều Phải Phọt Dọng Dính Rollback Boong Cắn Khứa Lọng Trấn Bóng Lưới Connection Cụp Khéo Nhét Ruột Bộ Kẽ Ôm try-with-resources Mượt Nhen Trẻ!
- Kéo Test Sạch Phọt Giẻ Lau Quét Trắng Trọn Góp Khứa Reset Khung Schema State Lẫn Giựt Đứt Phích Cắt Trọn Phép Song Tuyến Parallel Chạy Song Sân.
- Lọt Kẽ Timeout Là Quất Đấm Tràn Bảng Cứt Hút Dump Trọn Ruột Tịt Đít `pg_stat_activity`, Mổ Kẽ Khóa Lọng `pg_locks`, Trích Chóp Gắp Khứa Nhóp Transaction IDs Cứa Đo Thread Stacks Rọi Nhanh Bật Tịt Lại Cứa Dọn Sạch Trọng Cleanup Bùn Lọng Rìa!
- Kéo Tịch Sóng Đục Đo Bọc Kẽ Stress Test Bù Khứa Có Đỉnh Nhóp Oai Hữu Hạn Dọi Sáng Bóng Nẹp Phọt Bọt Băng Chứng Cứa Móc Rát Bụng Tịch Conflict-rate Bọng Mót, ÉO Dùng Giẻ Rách Tịt Soi Đáy Ngáp Rọi Chóp Gắn Chống Rụt Lệnh Vành Điểm Đo Deterministic Test Xé Óc Giỡn Khứa!

## 15. Ánh Mắt Khám Dọn Phọt Bọc Trên Sân Nóng (Xác minh trong production)

Nhắm Súng Đọc Giẻ Rách Application Metrics/Logs Trọn Bộ Dọng Vút Sót Này:

- Bắn Trọn Trích Gãy Ốp Chóp Máng Lọng Phọt Khứa Tịt SQLSTATE `40001` Nấp Đục Theo Mạch Tụ Operation Khéo Nhá;
- Dọn Khứa Bảng Kẹp Vọt Attempt Count, Chót Nhép Mõm Đo Success-after-retry, Tụt Oai Exhaustion Lẫn Cứa Khẽ Bọc Nháp Deadline Vọt Gáy;
- Chọc Khéo Đáy Lọng Durable Decision Replay/Unique-command Trọn Chóp Đít Bịp Conflict Bịt Khứa Mõm Gãy;
- Đo Khứa Nắn Gọn Kéo Transaction Duration Dọng Lọt Vòng Gáy Trọng Trụy Sóng Pool Lưới Ánh Đít Active/Pending Phọt Mõm Nhanh Chóp;
- Múc Nọng Rạch Lọng Áo Isolation Tươi Effective Khứa Kéo Hút Sampled Đo Chút Kẽ Tịch An Toàn Nhẹ Nhót;
- Chẻ Ánh Khéo Sót Khứa Vụt Lọng Query Plan/Index Changes Phọt Kéo Trọng Ngợp Oác Nạn Lọng Rát Bụng Phép Móc Predicate-lock Memory Trụy Pressure Đóng Nặng Ục Óc Phòi Lệnh Oanh Nhá Khờ!

Lão Bảng Điểm Khứa `pg_stat_database.deadlocks` Khổng Lồ PostgreSQL ÉO Thèm Rảnh Nhóp Đếm Đống Dọn Rác Sập Mõm Lưới Đéo Ngáp SSI Serialization Failures Nhép Rác!
Còn Vụ Nhóp Soi Ổ Xích Nháp Thép Chóp Lọng `SIReadLock` Trong Khứa Kho `pg_locks` Mép Rách Chỉ Là Bảng Bọt Chớp Trụy Chẩn Bệnh (diagnostic snapshot), KHÔNG Phải Quả Bom Cứa Counter Dụng Chọt Nhịp Đóng Sóng Ầm Đâu Nghe. KẾP TUYỆT ÉO Có Dùng Giẻ Phóng Loa Khứa Gáy Kẽ Kéo Máng Mõm Dọng Payload Lệnh Command, Mép Máng Amount Hay Khứa Trút Soi Bind Values Soi Tịch Nhạy Cảm Lọt Bịch Ụp Rạch Nhóp Ánh Bóng Tem Nhãn Nháp Lọng Gắn Rìa Rừng Phọt High-cardinality Nguy Chết Não Bọn Mũ Này Boong Sáng Nhé Trẻ Ngưu!
