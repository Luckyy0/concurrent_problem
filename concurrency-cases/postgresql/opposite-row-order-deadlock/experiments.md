# Mở Phòng Thí Nghiệm: Ép Trẻ Trâu Kẹt Xe Và Bắt Lỗi (Các thí nghiệm PostgreSQL deadlock có điều phối)

## 1. Mục Tiêu Tối Thượng (Mục tiêu)

Bộ Lắp Ráp Test (Experiment suite) phải chứng minh được Cả Sự An Toàn Lẫn Tiến Độ Trơn Tru:

1. Bày Trận ÉP ĐƯỢC CHẮC CHẮN cái Vòng Luẩn Quẩn Kẹt Xe (wait-for cycle) Bằng Đúng 2 cái Giao Dịch Đồ Thật Của PostgreSQL;
2. Phải Đích Xác Có Đúng 1 Kẻ Chết Thay Lãnh Đạn `40P01`, Và Kẻ Đó Phải Nằm Im Đợi Dọn Dẹp Ở Trạng Thái Chết Chìm `25P02`;
3. Cái Tiền Vừa Trừ Lố Của Kẻ Chết Thay Lập Tức Bị Vứt Đi (rollback), Và Tổng Số Tiền Ở 2 Ví Vẫn Không Đổi 1 Xu;
4. Chứng minh được Luật Xếp Hàng Theo Chuẩn (canonical order) Cho Phép 2 Thằng Đi Ngược Chiều Băng Qua Đường Trót Lọt;
5. Đã Làm Lại (retry) Thì Phải Mở Giao Dịch Mới Tinh, Cấm Có Lấy Cái Cũ Ra Khè;
6. Vạch Chân Tướng Rõ Ràng Cho Tụi Nó Thấy Lỗi Kẹt Xe Đánh Nhau Khác Hoàn Toàn Lỗi Chờ Dài Cổ (`55P03` lock timeout);
7. Mọi Cuộc Chờ Đợi Từ Code Tới CSDL Đều Phải Bọc Đồng Hồ Đếm Giờ Chống Treo Máy (timeout).

Tuyệt Đối Không Lấy Thằng Đồ Chơi H2 ra Múa Rìu Qua Mắt Thợ Để Phán Xét Thằng Giám Thị (detector) Của PostgreSQL. Cũng Không Xài Cái Trò Đoán Mò Mò Bằng Thời Gian Căn Ke Code Chạy Tới Đâu; Mà Phải Dựng Hàng Rào Chặn (barrier) Ngay Trực Tiếp Sau Cục Khóa Dòng Số 1.

> **Nói ngắn gọn:** Viết Test Trình Độ Cao đéo phải chỉ là ngồi cầu nguyện chờ nó văng Lỗi (exception); Em Phải TỰ TAY DỰNG LÊN CÁI BẪY KẸT XE, Tự Bấm Đếm Quá Trình Dọn Rác (rollback), Rồi Dòm Lại Mớ Tiền Có Bị Lủng Hay Không Sau Khi Mọi Thằng Xung Đột Đã Cút Hết Ra Khỏi Khung Hình.

## 2. Bàn Chơi Testcontainers Đồ Thật (Môi trường PostgreSQL Testcontainers)

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

Trong Buổi Chơi Này, Sếp Cho Phép Em Thụt Cái Giờ Giám Thị Khám (`deadlock_timeout`) Xuống Thật Thấp Để Thấy Kịch Bản Xảy Ra Tức Thì Chứ Không Phải Chờ Dài Cổ. Lên Đấu Trường Thật (Production) MÀ CẮM ĐẦU COPY PASTE CÁI CẤU HÌNH RÁC NÀY MÀ ĐÉO ĐO ĐẠC GÌ THÌ COI CHỪNG BỊ ĐUỔI VIỆC NHÉ.

## 3. Thằng Vệ Sĩ Canh Cửa (Helper điều phối)

```java
private static void await(
        CountDownLatch latch,
        String description
) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Hết giờ rồi con trai: " + description);
        }
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Bị đứt gánh giữa đường: " + description, interrupted);
    }
}
```

Tất cả đống Lệnh `Future` bên dưới bắt buộc phải nhét Hẹn Giờ `get(timeout)`. Lỡ Code Em Viết Bị Thủng Bể, Nó Trở Thành Vòng Đợi Bất Tử (wait vô hạn), Thì Test Sẽ Nổ Tung Báo Lỗi Ngay Lập Tức Bằng Chẩn Đoán Dễ Đọc (diagnostic fail), Chứ Đéo Có Chuyện Treo Nguyên Cái Máy Chủ CI Của Đội Lên Đâu.

## 4. Cuộc Thí Nghiệm Đầu Tiên — Dựng Hiện Trường Kẹt Xe Và Bắt Tại Trận Việc Dọn Rác (Thí nghiệm 1)

Mỗi Thằng Lính Sẽ Lao Vào Trừ Tiền Của Đứa Chuyển (debit source) Ngay Sau Khi Ăn Được Ổ Khóa Số 1, Xong ĐỨNG HÁ MỎ DỪNG LẠI TẠI HÀNG RÀO CHẶN (barrier). Đến Khi Thấy Cả 2 Thằng Đều Đã Nắm Cục Khóa Đầu Tiên Trong Tay, Ta Bấm Nút Cho Cả 2 Cùng Phi Ra Tranh Ổ Khóa Thứ 2 (destination row).
Lúc Này, 1 Thằng Chắc Chắn Sẽ Nằm Xuống Làm Kẻ Chết Thay; Số Tiền Nó Vừa Trừ (partial debit) PHẢI LẬP TỨC BAY MÀU (rollback).

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

    // Chờ 2 anh tài túm được 2 Ổ khóa đầu tiên đã
    await(firstRowsLocked, "both source rows locked");
    
    // Nã súng bóp còi cho 2 thằng tranh nhau chui vô hẻm hẹp
    requestSecondRow.countDown();

    List<AttemptOutcome> outcomes = List.of(
            t1.get(8, TimeUnit.SECONDS),
            t2.get(8, TimeUnit.SECONDS)
    );

    // KIỂM TRA ĐẠO LÝ: Đúng 1 Kẻ Thắng (00000) - Và Đúng 1 Kẻ Chết (40P01)
    assertThat(outcomes)
            .extracting(AttemptOutcome::sqlState)
            .containsExactlyInAnyOrder("00000", "40P01");

    AttemptOutcome victim = outcomes.stream()
            .filter(outcome -> outcome.sqlState().equals("40P01"))
            .findFirst()
            .orElseThrow();
            
    // Bắt quả tang cái Xác Ướp Giao Dịch
    assertThat(victim.stateBeforeRollback()).isEqualTo("25P02");

    // Tiền chung phải Vẹn Nguyên Không Thiếu 1 Đồng
    assertThat(totalBalance()).isEqualByComparingTo("2000.00");
    
    // Và Đúng Đứa Thắng Nào Nhét Tiền Vô Trước
    assertThat(readBalances()).satisfiesAnyOf(
            balances -> assertThat(balances)
                    .containsExactly("101=900.00", "202=1100.00"),
            balances -> assertThat(balances)
                    .containsExactly("101=1070.00", "202=930.00")
    );
}
```

Chi tiết bên trong Công Cụ Gây Án (Task implementation):

```java
private static AttemptOutcome brokenTransferAttempt(
        long fromId,
        long toId,
        BigDecimal amount,
        CountDownLatch firstRowsLocked,
        CountDownLatch requestSecondRow
) throws SQLException {
    try (Connection connection = open()) {
        connection.setAutoCommit(false); // Cầm lái tay
        configureDeadlockTestSession(connection);

        try {
            lockRow(connection, fromId);
            debit(connection, fromId, amount);

            firstRowsLocked.countDown();
            await(requestSecondRow, "permission to lock destination"); // Ngựa Vằn Xếp Hàng Chờ Tiếng Còi

            lockRow(connection, toId);
            credit(connection, toId, amount);
            connection.commit();
            return AttemptOutcome.committed(fromId);
        } catch (SQLException failure) {
            // Xem lúc Chết Dở cái Tình Trạng Là Mã Mẹ Gì 
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
        // Đồng hồ Báo Thức cho Kẹt Xe reo thật sớm (100ms)
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

Chiếc đồng hồ Đợi Bạc Đầu (`lock_timeout`) đang bự hơn hẳn Giám Thị Chờ Kẹt Xe (`deadlock_timeout`). Nhờ Vậy Thằng Giám Thị Kẹt Xe Mới Có Cửa Dành Vé Phóng Cái Lỗi Kẹo Đồng `40P01` Vào Mặt Thằng Code Trước Lúc Bị Hủy Vì Hết Giờ. Còn Cục Chặn Ngoài Cùng `Future.get(8, TimeUnit.SECONDS)` Chỉ Làm Trò Thằng Chó Giữ Nhà Ngoại Cảnh Đứng Phá Hư Nếu Dây Trói Vượt Ngưỡng.

## 5. Cuộc Thí Nghiệm 2 — Cho Code Đàng Hoàng Xếp Hàng Theo Chuẩn Lên Chạy (Thí nghiệm 2)

Nhét Bộ Chữa Cháy Thật Sự (Chạy Qua Bùa Spring Chứ Đéo Làm Bằng Tay Nữa):

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

        // Hô Lên: Cả Hai Đứa Đồng Loạt Phi Ra!
        start.countDown();
        first.get(8, TimeUnit.SECONDS);
        second.get(8, TimeUnit.SECONDS);

        // KẾT QUẢ ĐẸP NHƯ TRANH VẼ!
        assertThat(balance(101)).isEqualByComparingTo("970.00");
        assertThat(balance(202)).isEqualByComparingTo("1030.00");
        assertThat(totalBalance()).isEqualByComparingTo("2000.00");
    }
}
```

Set Gắn Vô Đầu Giao Dịch Vài Cục Khóa Mốc Giờ Chờ Đợi trong File Test Profile Nhé. Test này Nã Khẩu Lệnh Gần Như Hệt Cái Cách Khách Nhấn Nút Trên Sân Thật.

## 6. Cuộc Thí Nghiệm 3 — Thử Thằng Tính Toán Xếp Hàng Chuẩn Nhỏ Gọn Nhất (Thí nghiệm 3)

Tách riêng Hạt Nhân Não Bộ Sắp Xếp Trật Tự Vào 1 Cục Object Nhỏ Xíu:

```java
record AccountLockOrder(long firstId, long secondId) {

    static AccountLockOrder of(long leftId, long rightId) {
        if (leftId == rightId) {
            throw new IllegalArgumentException("duplicate account");
        }
        // BAO ĐẸP: Bé đứng trước, Lớn Đứng Sau, Éo Quan Tâm Thằng Nào Chuyển
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

Thằng Lính Service Phải Nhắm Mắt Cầm Đúng Cái Vật Phẩm Này Lên Xài (không được tự ý hì hục ngồi code lại lệnh lặt vặt như trò `min/max`).

## 7. Cuộc Thí Nghiệm 4 — Người Chết Đội Mồ Sống Dậy Với Giao Dịch Tươi Mới (Thí nghiệm 4)

Test Thử Một Quả Làm Lại (Retry). Sếp xài Kỹ Thuật Đâm Code Ngoại Lai (test-only hook) Hệt Như Thằng Số 1 Ở Chỗ Chắn Vạch Khóa Đứa Thứ 1 Xong Mới Cho Lên Để Gây Án Ngay Lần Dạo Đầu (first attempts). Xong Rút Chốt Đó Đi Liền Tức Thì Để Thằng Đội Mồ Sống Lại (victim retry) Chạy Vượt Về Đích Êm Thấm.

Thò Vào 1 Đoạn Ăn Cắp Xem Giao Dịch Số Mấy:

```java
long transactionId = jdbc.queryForObject(
        "select txid_current()",
        Long.class
);
attemptProbe.record(command.commandId(), attemptNumber, transactionId);
```

Và Ta Ngó Lại Sự Kiện:

```java
start.countDown();
TransferReceipt firstReceipt = first.get(10, TimeUnit.SECONDS);
TransferReceipt secondReceipt = second.get(10, TimeUnit.SECONDS);

assertThat(firstReceipt).isNotNull();
assertThat(secondReceipt).isNotNull();
assertThat(attemptProbe.totalAttempts()).isEqualTo(3); // Hai Đứa Đi Cùng -> Chết 1 Đứa -> Nó Đội Mồ 1 Phát Thành 3
assertThat(attemptProbe.deadlockVictims()).isEqualTo(1);
assertThat(attemptProbe.transactionIds())
        .hasSize(3) // Tuyệt Đối Không Thằng Nào Trùng Mã Vạch Thằng Nào
        .doesNotHaveDuplicates(); 

assertThat(totalBalance()).isEqualByComparingTo("2000.00");
assertThat(balance(101)).isEqualByComparingTo("970.00");
assertThat(balance(202)).isEqualByComparingTo("1030.00");
```

3 Cục Mã Giao Dịch Kia Là Tên Của: Thằng Sinh Ra Ở Vạch Đích Ở Phát Gọi Số 1, Kẻ Chết Thay Ở Phát Số 1, Và Tấm Thân Mới Tinh Tươm Của Đứa Chết Thay Đội Mồ Bật Lên Lần Sau. Chấm Hết!

## 8. Cuộc Thí Nghiệm 5 — Rụng Răng Chờ Lâu Đéo Phải Bị Kẹt Xe! (Thí nghiệm 5)

Thằng Đại Ca Đứng Ngậm Cục Khóa Ví A; Một Thằng Culi Đứng Há Mỏ Đợi Cục Ví A Mà Chẳng Ôm Một Cục Rác Nào Trong Người MÀ Đại Ca Thèm Cả. Vòng Tròn Kẹt Xe ĐÉO HỀ CÓ (Graph không có cycle).
Bắt Buộc Thằng Culi Phải Nhận Về Bãi Rác Báo Mã `55P03` Tức Lên Hết Hạn Giờ Chờ Khóa (`lock_timeout` hết hạn):

```java
@Test
void oneWayWaitReturnsLockTimeoutNotDeadlock() throws Exception {
    var holderLocked = new CountDownLatch(1);
    var releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = open()) {
            connection.setAutoCommit(false);
            lockRow(connection, 101);
            holderLocked.countDown(); // La Lên: LÃO ĐẠI ĐÃ NẮM VÍ A!
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
                // Nhỏ giọt đồng hồ lại để test cho lẹ nè
                statement.execute("set local lock_timeout = '150ms'");
                statement.execute("set local statement_timeout = '2s'");
            }
            try {
                lockRow(connection, 101); // NHÀO VÔ XIN MÀ ĐÉO ĐƯỢC ĐÂU CON!
                connection.commit();
                return "00000";
            } catch (SQLException failure) {
                connection.rollback();
                return failure.getSQLState(); // TAO TRẢ LẠI TỜ GIẤY TỬ THẦN ĐÂY!
            }
        }
    });

    // BAO ĐẸP: Kết quả thằng há mỏ ăn nguyên mã ĐỢI BẠC ĐẦU HẾT GIỜ 55P03
    assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    releaseHolder.countDown();
    holder.get(5, TimeUnit.SECONDS);
}
```

Nhân viên Kiểm Thẩm Định Bắt Lỗi Retry (Retry classifier) CẤM TUYỆT ĐỐI Gọi Nhầm Cái Đống 55P03 Thành `40P01` Nha! Luật Của Em Vẫn Có Thể Khoan Hồng Cho Chạy Lại Khi Lỗi Chờ Lâu (Retry lock timeout), Nhớ Là Chờ Lâu Vẫn Chỉ Là Dấu Hiệu Lười Động Não; Nên Lấy Chìa Khóa Kiểm Mã (idempotency/deadline) Ra Dùng.

## 9. Cuộc Thí Nghiệm 6 — Đứt Cáp Đột Ngột Tự Nhả Khóa Cho Đứa Sau Hút Máu (Thí nghiệm 6)

Lão Đại Ôm Khóa A. Cho Đệ Vào Há Mỏ Hút Máu Ngay Lập Tức Xin Lấy Khóa. Lấy Khẩu Súng Lục Lên, Lãnh Tụ Cắt Dây Mạng Bóp Mất Xác Lão Đại TRƯỚC Khi Chốt Sổ:

```text
Cắt Dây Ống Thở Lão Đại (holder connection close)
→ Thằng PostgreSQL Thấy Lão Đại Tử Ếo Xong Tuyên Án Quăng Rác Giao Dịch
→ Vứt Bỏ Đi Cái Ổ Khóa Ví A (row A lock được release)
→ Đứa Đệ Vớ Lấy Vàng Cắn Răng Hô Khóa A Và Kịp Chốt Sổ Ở Phút 89
```

Quá Tuyệt Vời Cục Diện:

```java
assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo("ACQUIRED");
assertThat(totalBalance()).isEqualByComparingTo("2000.00");
// Trống vắng bóng người giao dịch, sạch sành sanh
assertThat(countOpenTransactionsForTestUsers()).isZero();
```

Trò Thử Này CHỈ KHẲNG ĐỊNH Việc DB Rất Mẫn Cán Rửa Tay Sạch Đẹp Vết Máu, CHỨ ĐÉO CHỨNG MINH ĐƯỢC cái Việc Cầm Code Dò Hỏi Kết Quả Nếu Đứt Mạng Lúc Cuối Trận Mập Mờ Nó Như Nào Nhé! (Vụ ambiguous outcome này đem test bên case nghiệp vụ riêng nha con trai).

## 10. Cuộc Thí Nghiệm Cuối: Sờ Tận Tay Sơ Đồ Bị Chặn (Thí nghiệm 7)

Kéo Đứng Thằng Chờ Trong Cái Mô Hình 1 Đứa Há Mỏ (one-way wait). Quét Lấy Số Phù Hiệu Người Háo Đợi `select pg_backend_pid()` RỒI SÚT CÂU HỎI TRỰC TIẾP TỪ THẰNG THEO DÕI:

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

Em sẽ Dùng Công Cụ Quét Lặp Giới Hạn Giờ (bounded polling helper) Chứ ĐÉO ĐƯỢC Nhét Cục Đợi Mù Quáng Xuyên Thủng Vô Test:

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
            throw new AssertionError("Đang soi mà bị dứt cáp");
        }
    }
    throw new AssertionError("Không thấy bóng hình Kẻ Thù Trước Khi Đồng Hồ Đánh " + timeout);
}
```

Nắm Tóc Cho Chắc: `wait_event_type = 'Lock'` VÀ TRONG BẢNG `blocking_pids` Hiện Rõ Rành Rành Số PID Của Lão Đại! Chờ Tụi Nó Đốt Xác Xong, Mọi Dấu Hiệu Phải Tiêu Tan Nhé (assert PID không còn chờ).
Cái Bảng Dò Ráp Này Tranh Tối Tranh Sáng Đảo Liên Tục; Nên Nhớ NÓ CHỈ LÀ BẰNG CHỨNG HÌNH SỰ Cho Dev Ngồi Sửa Lỗi, KHÔNG PHẢI LÀ CÔNG TẮC ĐỂ THẰNG CODE NHẢY NHÓT SỬA LỖI ỨNG DỤNG BÊN TRONG!

## 11. Các Mẹo Vặt Móc Thông Tin (Helper đọc trạng thái)

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

## 12. Bảng Tổng Hợp Chiến Trận Bao Phủ (Ma trận bao phủ)

| Màn Chơi Test | Máy Móc Sử Dụng | Câu Khẳng Định Lớn Nhất (Assertion) |
| --- | --- | --- |
| Màn 1 | Ngược Chiều Băng Qua 1 Cửa Chặn (Opposite row order + barrier) | Bắn Lỗi Chết Chùm `40P01`, Xác Nằm Đống `25P02`, Tiền Còn Nguyên `2000` |
| Màn 2 | Lính Tráng Ngoan Ngoãn Đi Có Hàng Lối Qua Spring | Ghi Sổ Hoàn Đẹp Mắt A=`970`, B=`1030`, Tổng Không Thất Thoát `2000` |
| Màn 3 | Não Nhỏ Cục Khóa Thuần Xếp Chuẩn | Bố Đảo Ngược Hướng Nó Vẫn Ói Ra Thứ Tự Cứng Như Đá `[101, 202]` |
| Màn 4 | Đội Cứu Thương Retry Vác Ống Dò Tx Mới Vào Mồm | Hiện Rõ 3 Cái Đầu Giao Dịch, 1 Cái Xác Chết Thay, Chạy Đủ Cả 2 Thằng Vào Cửa Đích Cuối |
| Màn 5 | Há Mỏ Chờ Tiếng Khóc Ở Bụi Cây (One-way row wait) | Văng Đúng Lỗi `55P03`, Cấm Gán Tội Cho Tụi Nó Đánh Nhau (deadlock) |
| Màn 6 | Bóp Cổ Lão Đại Cắt Cáp Giữa Đường | Đệ Vượt Lên Chốt Hàng Mượt Mà, Không Có Xác Giao Dịch Rơi Vãi |
| Màn 7 | Quét Sóng Rada Rình Sờ Cục Nghẽn Live | Ráp Nối Nắm Đuôi Áo Thằng Thằng Đợi Và Thằng Chặn Lên Nhãn `pg_blocking_pids` Thành Công Rực Rỡ |

## 13. Thuốc Lú Giúp Bộ Test Luôn Chắc Như Đá (Chống flaky)

- Ổ Khóa Rào Chắn bắt buộc Đặt Cắm Ngay Sau Phát Cắn Ổ Khóa Dòng Lần Đầu! Ngu Nghĩ Mà Đặt Bậy Ở Đầu Hàm Vào Giao Dịch.
- Phải Đảm Bảo Cục Này Đúng Chuẩn Theo Trật Tự Lớn Lên: `deadlock_timeout` Nhỏ Hơn `lock_timeout` Và Dễ Dãi Hơn `statement_timeout` Nằm Cũi Nhỏ Hơn Cục Quản Trò `Future watchdog`!
- ĐÉO BAO GIỜ Đi Cá Cược Chỉ Ra Số Má Cụ Thể "Thằng Này Chắc Chắn Mới Là Nạn Nhân Nhé".
- Reset Tiền Rác Giữa Các Lượt Đánh Khác Nhau; Cấm Cái Tội Bật Chạy Tứ Tung Song Song Nhiều Thằng Chung 1 Quần Đảo DB Schema Dễ Nát Xác!
- Hễ Nghe Chết Đứng Báo Lỗi Error Thì Phải Nhặt Xác Cắt Ống Trả Ổ Bằng Đồ Chơi Gói Gọn Dọn Dẹp try-with-resources.
- Bắn Bỏ Toàn Bộ Thằng Kéo Luồng `executor` Chắc Ăn Ở Vòng Shutdown Rửa Sạch (`finally`/lifecycle) Bằng Chốt Hẹn Giờ Bắt Buộc Hủy.
- Khi Cái Bộ Hẹn Giờ Quản Trò Ảo Tưởng Náo Loạn Kêu Oai Oái Vì Lỗi Không Mong (watchdog fail): NHỚ Dội Nguyên Đống Rác Thống Kê Dữ Liệu `pg_stat_activity`, `pg_locks` và Bảng Tội Phạm `pg_blocking_pids` TRƯỚC KHI BẤM NÚT QUÉT DỌN LÀM SẠCH VẾT TÍCH!
- Kéo 1 Nghìn Thằng Kêu Khẩu Hiệu Ngược Chiều Băng Đường Test Liên Tục Nhìn Chơi Cũng Lòi Hạn Số (Stress test), Nhưng TUYỆT ĐỐI NÓ KHÔNG ĐỦ TUỔI MÀ THAY THẾ CHO CÁI TEST CHỈ RA TOÀN CHÂN TƯỚNG KẾT ÁN CHU TRÌNH RÕ RÀNG 100% Này Đâu Nhen Cưng!

## 14. Vác Đi Đoán Lỗi Ở Chiến Trường Băng Giá (Xác minh trong production)

Ngồi Trực Soi Bảng:

```sql
select datname, deadlocks
from pg_stat_database
where datname = current_database();
```

Và Mix Nó Cùng Bữa Tiệc Này Nhé:

- Các Máy Đo Nhịp Tim Code Xoay Quanh Cái Đầu Lệnh Này Thôi: Đánh Nhau Rớt Đầu (deadlock), La Kêu Cấp Cứu Mới (retry), Mòn Cổ Mất Sức (exhaustion) và Độ Lết Bộ Bánh Xe Chậm Nhịp Trễ Xíu Tí Tẹo Thôi Đâu (latency);
- Chú Ý Mấy Cục Cuộn Log Hỏa Tốc Của PostgreSQL Quăng Graph Báo Lỗi Và Khung Hỏi Vặn Gây Án Ngay Cùng Chỗ Đó (query context);
- Cột Chờ Khát Máu Rụng Răng Khi Giành Kẹo Chỗ Xin Connection Ở Hồ Nước Code App Lỗi (pool active/pending/acquisition timeout);
- Đo Cục Tuổi Đời Thời Gian Treo Chờ Lác Mắt Và Dòng Cố Chạy Làm Bộ Ngủ Gật Trong Lệnh Code `idle in transaction`;
- Kiểm Nhãn Gắn Phiên Bản App Kéo Lên Thùng Để Còn Soi Được Mặt Thằng Mới Nhét Cái Code Chưa Xếp Hàng Theo Chuẩn Đẩy Lên Trên Này.

Cái Nút Bấm Đèn Còi Báo Động KHÔNG BAO GIỜ Bật Khùng Đính Chung Với Từng Số Thẻ Của Khách Bất Bất Ổn Lung Tung Đâu (high cardinality). Bật Lên Vì Phát Hiện Tần Xuất Đột Biến Giữa Trần Gian Tới Báo Cáo Thiệt Hại Thôi (rate/baseline và user impact). Khi Đã Tuyên Bố Bưng Code Đổ Bộ Ra Sản Phẩm Thực Dụng Rộng Rồi Đâu Mọi Vòng Đợi Quỷ Ám Luẩn Quẩn Đã Bị Vạch Mặt SẼ BIẾN MẤT VĨNH VIỄN KHÔNG ĐỂ LẠI DẤU VẾT GÌ VÀ NHỮNG VỤ ÉP PHẢI HY SINH CŨNG CHỈ CÒN ĐƯỢC TAY KHÔNG CHỚP MẮT TRONG MỘT NHỊP KIỂM SOÁT BẬT GỌI TỰ ĐỘNG THU DỌN SẠCH THU XẾP ỔN THỎA TRÒN RÕ TỊNH TIẾN QUY TẮC NHỮNG BẢO ĐẢM NỀN TẢNG TIỀN BẠC (invariant)!
