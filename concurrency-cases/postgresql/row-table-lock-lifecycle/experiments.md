# Bày Đồ Chơi Thực Nghiệm Bắt Lỗi Khóa Chặn (Deterministic PostgreSQL lock experiments)

## 1. Đích Ngắm (Mục tiêu)

Bài test chuẩn bài dọn đường mài kiếm chứng minh ngay:

1. Thằng Mở Sổ Cầm Vòng Giao Dịch (transaction) Đang Giương Đọc Trơn (plain SELECT open) Hổng Đủ Tuổi Chặn Kẻ Đâm Sửa (UPDATE);
2. Nẹt Lệnh Khóa Khét Lẹt `FOR UPDATE` Sẽ Xích Lũ Viết Cắn Lệnh Kéo Lọc Khác Nhóm Bất Tuân (incompatible writer/locking reader) Bắt Đứng Chờ Ngáp Tới Chết;
3. Khán Giả Đi Dòm Ngó Chay (plain SELECT) Rất Khôn Éo Chờ Rớt Dây Lệnh Ộc Ở Quanh Ô Khóa Dòng (row lock), Nó Rút Tuột Kép Tụ Cuối Cùng Nhìn Về Quá Khứ Cũ (old committed version);
4. Buông Hơi Đập Kép (commit) Cúp Nguồn Bể Cuộn (rollback) Mới Xõa Văng Kéo Đứt Thả Sạch (release locks);
5. Lão Nhót Đứng Đợi Sẽ Cắn Tính Lại Bộ Chống Điều Kiện (re-evaluate conditional predicate) Sau Khi Thằng Ở Trên Dập Vạch Đóng Chốt Sổ Xong (holder commit);
6. Trùm Vây Bảng Sạch Nẹt Phép Giăng Lưới `ACCESS EXCLUSIVE` Thách Kẻ Đi Dòm Chay Cả Trơn Tuột SELECT Cũng Mắc Nghẹn!
7. Lưới Áo Cáo Spring `PESSIMISTIC_WRITE` Nó Bơm Nhét Hàng Mốc Lệnh Khóa (locking SQL) Ra Cái Mã Sao Cùng Nhau Chứng Kiến Đi Nha!

Mượn Tuyệt Kỹ Mỏ Neo Cuộn Trống Báo Thức Đoạt Thời Lượng Hẹn Trước (latch/future có timeout) Nhé; Cấm Nặn Ngớ Ngẩn Ngồi Mò Bóng Canh Sai Đồng Hồ Nhão Đoạn Rủi Ro Giữa Chừng Nhen Cưng!

> **Sếp chốt lại:** Từng Bãi Test Khóa Đi Đúng 1 Nhịp Mode Xoắn Nháp Xoay Áo Mới Rút Đi; Cái Bể Nứt Đoán Sai Nhức (failure) Cho Anh Rõ Bộ Khép Ráp Kéo Sóng Tương Thích Nào Hoặc Lệnh Mở Ảo Cặp Hiện Hình Lầm Lạc Đang Lỗi Ở Đâu Để Ụp Cứu Phốt Á!

## 2. Kích Hộp Giữ Lưới Ném Gọn Ảo Tưởng Nằm Quanh (PostgreSQL Testcontainers)

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
class RowTableLockLifecycleIntegrationTest {

    private static final UUID TENANT_ID =
        UUID.fromString("10000000-0000-0000-0000-000000000042");

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

        new JdbcTemplate(dataSource).execute("""
            create table tenant_quota (
                tenant_id uuid primary key,
                quota integer not null check (quota >= 0),
                revision bigint not null
            )
            """);
    }

    @BeforeEach
    void resetState() {
        executor = Executors.newFixedThreadPool(3);
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        jdbc.update("delete from tenant_quota");
        jdbc.update(
            """
            insert into tenant_quota(tenant_id, quota, revision)
            values (?, 10, 5)
            """,
            TENANT_ID
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

Nhét Tay Thử Gỡ Nhột Ngoài Method Test ĐỂ KHÔNG Bọc Cái Kén Transaction Tự Chặn Ống Kép Láo. Các Sếp Trẻ Diễn Trò Đều Vớ Ống Khác Nhau Đấm Dây Bơi Đứng Trụ Connection Phân Ly Trắng Chợ Riêng Phệt!

## 3. Dây Đo Phạt Nghẹn Gọi Rớt Chụp (Coordination helper)

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
```

## 4. Cuộc Thí Nghiệm Số 1 — Kẻ Nhìn Đểu Chay Không Sức Ép Dòng Cắn (Experiment 1 — Plain SELECT không chặn writer)

```java
@Test
void openPlainReaderDoesNotReserveRow() throws Exception {
    CountDownLatch firstReadDone = new CountDownLatch(1);
    CountDownLatch writerCommitted = new CountDownLatch(1);

    Future<List<Integer>> reader = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            connection.setTransactionIsolation(
                Connection.TRANSACTION_READ_COMMITTED
            );
            int first = selectQuota(connection, false);
            firstReadDone.countDown();
            await(writerCommitted, "writer commit");
            int second = selectQuota(connection, false);
            connection.commit();
            return List.of(first, second);
        }
    });

    Future<Void> writer = executor.submit(() -> {
        await(firstReadDone, "first plain read");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            assertThat(updateQuota(connection, 8)).isEqualTo(1);
            connection.commit();
        }
        writerCommitted.countDown();
        return null;
    });

    assertThat(reader.get(5, TimeUnit.SECONDS)).containsExactly(10, 8);
    writer.get(5, TimeUnit.SECONDS);
    assertThat(committedQuota()).isEqualTo(8);
}
```

Thằng Xoi Đọc Vẫn Cười Tủm Tỉm Treo Lơ Lửng (Reader transaction open) Bất Bấp Lúc Thằng Sửa Nắm Gọn Đít Chốt Đinh Ngay Đáy Trống. Thế Nghĩa Là Ngó Trơn SELECT Nhột Cái Khóa Xích Row Lỗ Nào, Nên Anh Sửa Bút Vẫn Múa Bọc Đinh Nện Xuống Ra Dấu Lật Ụp Signal Nhé!

## 5. Cuộc Thí Nghiệm Số 2 — Ngó Chay Hồn Nhiên Bước Xuyên Quanh Áo Lấp `FOR UPDATE` Của Trùm Đỉnh Cụt (Experiment 2 — Plain reader đi qua `FOR UPDATE` holder)

```java
@Test
void plainReaderSeesOldCommittedVersionWithoutWaitingForRowLock()
    throws Exception {

    CountDownLatch uncommittedUpdateReady = new CountDownLatch(1);
    CountDownLatch dashboardReadDone = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            assertThat(selectQuota(connection, true)).isEqualTo(10);
            assertThat(updateQuota(connection, 12)).isEqualTo(1);
            uncommittedUpdateReady.countDown();
            await(dashboardReadDone, "dashboard plain read");
            connection.commit();
        }
        return null;
    });

    Future<Integer> dashboard = executor.submit(() -> {
        await(uncommittedUpdateReady, "uncommitted locked update");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            int observed = selectQuota(connection, false);
            connection.commit();
            dashboardReadDone.countDown();
            return observed;
        }
    });

    assertThat(dashboard.get(5, TimeUnit.SECONDS)).isEqualTo(10);
    holder.get(5, TimeUnit.SECONDS);
    assertThat(committedQuota()).isEqualTo(12);
}
```

Ông Dashboard ÉO Xoi Bẩn Thấy Hình Đít Bẩn Ngâm Rỉ `12`; Ảnh Chỉ Cóp Nhặt Tờ Quota Cũ Quen Chốt Ngon Lâu Nhớ `10`. Vững Sót Vượt!

## 6. Cuộc Thí Nghiệm Số 3 — Phá Đứt Họng Khía Cạnh Hẹn Ổ Chờ Đứt Lõi (Experiment 3 — Incompatible writer timeout)

```java
@Test
void updateWaitsForForUpdateHolderAndTimesOutBoundedly() throws Exception {
    CountDownLatch rowLocked = new CountDownLatch(1);
    CountDownLatch contenderDone = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            selectQuota(connection, true);
            rowLocked.countDown();
            await(contenderDone, "writer timeout");
            connection.commit();
        }
        return null;
    });

    Future<String> contender = executor.submit(() -> {
        await(rowLocked, "row lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            setLockTimeout(connection, "300ms");
            try {
                updateQuota(connection, 8);
                connection.commit();
                return "unexpected-commit";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                contenderDone.countDown();
            }
        }
    });

    assertThat(contender.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    holder.get(5, TimeUnit.SECONDS);
    assertThat(committedQuota()).isEqualTo(10);
}
```

## 7. Cuộc Thí Nghiệm Số 4 — Lãnh Ấn Lệnh Đọc Cũng Dính Ngáp Ruồi Chờ (Experiment 4 — Locking reader cũng chờ)

Lấy Cái Tấm Che Chắn Cờ Kẻ Phá Nát (contender UPDATE) Ép Sang Trò:

```java
setLockTimeout(connection, "300ms");
selectQuota(connection, true);
```

Giữa Màn Giao Đâm Bác Lão A Nhấn Giữ Quật Trấn `FOR UPDATE` Cùng Trúng Chót Khúc Củ Dòng Á, Quỷ Khép Nách Nhóp Đọc Chắn Kẹp SELECT Khác Quật Hút Nhận Ọc Lỗi `55P03` Bóp Bắn Đứt Phanh (bounded timeout). Song Ngang Tầm Đó Nảy Lệnh Tụt Chay Gọn Trơn Bình SELECT Vẫn Bay Vút Kéo Xoi Lượm Committed Row Đi Bốc Y Chang Số Đo 2 Đời Nớ Á (như Experiment 2)!

## 8. Cuộc Thí Nghiệm Số 5 — Đạp Hủy Cờ Quật Bay Kéo Tách Xích Khóa Lủng Lỗ (Experiment 5 — Rollback release lock)

```java
@Test
void rollbackReleasesRowLockAndDiscardsHolderVersion() throws Exception {
    CountDownLatch holderUpdated = new CountDownLatch(1);
    CountDownLatch waiterReady = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            selectQuota(connection, true);
            updateQuota(connection, 12);
            holderUpdated.countDown();
            await(waiterReady, "waiter ready");
            connection.rollback();
        }
        return null;
    });

    Future<Integer> waiter = executor.submit(() -> {
        await(holderUpdated, "holder update");
        waiterReady.countDown();
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            int affected = updateQuota(connection, 8);
            connection.commit();
            return affected;
        }
    });

    holder.get(5, TimeUnit.SECONDS);
    assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo(1);
    assertThat(committedQuota()).isEqualTo(8);
}
```

Bọn Lượn Hóng Chờ (waiter) Nhặt Được Cửa Xích Trước/Hay Chặn Sau Dấu Bể Hủy Rollback Nhẹ Lệ Phụ Trách Thừa Hưởng Đo Cuộn (scheduler); Có Phát Súng Cắm Bảng Xét Trị Điển Chứng Tịch Lũ Nón Ảo Lố `12` Nháp Kín Cứt Trôi Lặn Vỡ Tiêu (disappeared) Vẫn Rách Lép Xích Đứt Bóng Án Theo Giao Dịch Không Thể Tái Sống Liều Hết Nhen!

## 9. Cuộc Thí Nghiệm Số 6 — Test Cố Lệnh Kép Vá Sau Sóng Rớt Trút Commit Bủa Đầu (Experiment 6 — Predicate recheck sau holder commit)

Ông Trùm A Nắm Lưới Khoác Gắn Row Áo Dọng Đóng Sổ Revision Chốt Mệnh Lên `6`. Cu Nhóc B Phun Súng Gắn Đạn Test Vá Ốp Kẹp Tịch UPDATE Trực Chờ Xin Yêu Cầu Gốc Móc Mã Khóa Sống `5`:

```java
private static int updateIfRevisionMatches(
    Connection connection,
    int quota,
    long expectedRevision
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        update tenant_quota
           set quota=?,
               revision=revision+1
         where tenant_id=?
           and revision=?
        """)) {
        statement.setInt(1, quota);
        statement.setObject(2, TENANT_ID);
        statement.setLong(3, expectedRevision);
        return statement.executeUpdate();
    }
}
```

Nằm Ở Đáy Tích Khóa Đỉnh Vượt:

```java
assertThat(waiterAffectedRows).isZero();
assertThat(committedQuota()).isEqualTo(12);
assertThat(committedRevision()).isEqualTo(6);
```

Khứa B Trút Nhờ Sụt Nhận Quả Cáo Trắng Khất Affected-row Sập Rơi Trượt `0` Ép Đo Tròn Cứ Hóc Kẹp Giữa (conflict); Phá Quật Tranh Cũ Đội Đè Nháp Này Tự Đứt Mõm Giao Khách Phá Đọc Xóa Sập Gãy! (it does not overwrite current row).

## 10. Cuộc Thí Nghiệm Số 7 — Đao Chém Nguyên Làng Bảng Ngăn Luôn Mắt Thần Đọc (Experiment 7 — `ACCESS EXCLUSIVE` chặn plain SELECT)

```java
@Test
void accessExclusiveTableLockBlocksEvenPlainSelect() throws Exception {
    CountDownLatch tableLocked = new CountDownLatch(1);
    CountDownLatch readerDone = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (Statement statement = connection.createStatement()) {
                statement.execute("""
                    lock table tenant_quota in access exclusive mode
                    """);
            }
            tableLocked.countDown();
            await(readerDone, "table-lock reader timeout");
            connection.commit();
        }
        return null;
    });

    Future<String> reader = executor.submit(() -> {
        await(tableLocked, "access exclusive table lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            setLockTimeout(connection, "300ms");
            try {
                selectQuota(connection, false);
                connection.commit();
                return "unexpected-read";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                readerDone.countDown();
            }
        }
    });

    assertThat(reader.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    holder.get(5, TimeUnit.SECONDS);
}
```

Ghép Bàn 2 Khứa Experiment Dọc Án Quật Vào Số 7 Cho Mấy Đồng Chí Bẻ Dáng Mà Đo: “Khóa Dòng” Nó Oải Mồm Hoàn Toàn Ộc Lạc Với Trò Đo Bốc Áp Chắn Rõ Phép Vênh Đích Nhất Lệnh Quét Đi “Khóa Sập Cả Bàn Mode”. Bê Chữ Kêu Đất Rợn Người "Bóng Database Khóa Điên Lên Nên Phọt Đứa Đọc Chết Tắc Hết Cả" Lên Tòa Cãi Mõm Là Bị Khỏ Trách Ngược Liền!

## 11. Cuộc Thí Nghiệm Số 8 — Chiêu Cắn Móc Lệnh `PESSIMISTIC_WRITE` Của Giáo Phái Spring (Experiment 8 — Spring `PESSIMISTIC_WRITE`)

Đẩy Trận Dội Thử Integration Kẹp Proxy/Service Điên Điệt Thật Có Khúc Néo Đóng Móc Mỏ Mồi Dọc Gờ Vịn Đít Trượt Repository:

```java
@Test
void pessimisticRepositoryHoldsLockUntilServiceTransactionCommits()
    throws Exception {

    Future<Void> owner = executor.submit(
        () -> service.changeQuotaWithGate(TENANT_ID, 12, gate)
    );
    gate.awaitLocked();

    assertThat(lockInspector.hasWaitingWriter(TENANT_ID)).isFalse();
    Future<QuotaChangeResult> waiter = executor.submit(
        () -> service.changeQuota(TENANT_ID, 8)
    );

    gate.awaitWaiterObservedByDatabase();
    assertThat(lockInspector.blockingPids()).isNotEmpty();
    gate.allowCommit();

    owner.get(5, TimeUnit.SECONDS);
    waiter.get(5, TimeUnit.SECONDS);
}
```

Dàn Lệnh Giăng Trống Dấu Gate Có Cộc Giữ Ổ Đo (bounded latches). Quái Chiêu Hứng Bắt Đo Chộp Dính Móc Lọt Khe Kéo Lấy SQL Mới Chuẩn Sóng Dây Cho Rành Rọi Phệt Bộ Ánh Chữ Trọn Nét `for update`; Níu Bám Bóng Phép Vành Annotate Giả Áo Suông Chỉ Đi Vào Đất Chết Thôi Khờ Á!

## 12. Bọc Súng Rút Nhanh Móc Cú Nhỏ Xung (Core helpers)

```java
private static int selectQuota(Connection connection, boolean forUpdate)
    throws SQLException {
    String suffix = forUpdate ? " for update" : "";
    try (PreparedStatement statement = connection.prepareStatement(
        "select quota from tenant_quota where tenant_id=?" + suffix
    )) {
        statement.setObject(1, TENANT_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
            return rs.getInt(1);
        }
    }
}

private static int updateQuota(Connection connection, int quota)
    throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        update tenant_quota
           set quota=?,
               revision=revision+1
         where tenant_id=?
        """)) {
        statement.setInt(1, quota);
        statement.setObject(2, TENANT_ID);
        return statement.executeUpdate();
    }
}

private static void setLockTimeout(Connection connection, String value)
    throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement(
        "select set_config('lock_timeout', ?, true)"
    )) {
        statement.setString(1, value);
        statement.execute();
    }
}

private static int committedQuota() {
    return new JdbcTemplate(dataSource).queryForObject(
        "select quota from tenant_quota where tenant_id=?",
        Integer.class,
        TENANT_ID
    );
}
```

## 13. Mảng Đọ Dây Quất Khống Trình Đo Đạc (Coverage matrix)

| Chuyến Test (Experiment) | Tay Ôm Trùm (Holder) | Tay Giằng Cửa (Contender) | Khép Sổ Ghi Bàn Lỗi Gì (Assertion) |
| --- | --- | --- | --- |
| 1 | SELECT Trơn Móc Mở Giao Dịch | Sút Nạn Bể Lấp Sửa Đỉnh Cũ Ngón Cập UPDATE | Sửa Vượt Chốt Gấp; Đứa Xem Thơm Bốc Tới `10 -> 8` |
| 2 | Khóa Ác `FOR UPDATE` + Đập UPDATE Đang Giấu Nháp | Tướng Xem Khán Dòm `plain SELECT` | Ánh Thấy Trọn Cứ Áo Bóc Số `10` Kỷ Niệm Cũ Chưa Sửa Xóa |
| 3 | Tút Góc Dòng Cờ Gài `FOR UPDATE` | Kẻ Bạo Liều Nện Ốp UPDATE | Nhét Vá Quần Ợ Lỗi `55P03` |
| 4 | Trực Chạm Cầm Ngó Cục `FOR UPDATE` | Lính Tiếp Quát Đòi Cửa Khóa Đập Kéo Mạng Đọc Chắn Theo Nhau `FOR UPDATE` | Dính Bọn Hồi Mỏ Chờ Lật Ép `55P03` Nốt |
| 5 | Quẳng Ánh Kéo Chốt Bể Lồi Đo Rollback Chôn | Kẻ Phụt Nhờ Hóng Chờ Cựa Chọc Bút | Dọn Chờ Ụp Quật Bỏ Sửa Ra Bóng Lại Khởi Hồn Vụt Ra `8` Nhé Nhóc! |
| 6 | Thằng Tựa Đỉnh Buông Xả Mỏ Báo Hủy Bịch Cũ Lật Cắm Đinh Đổi Lắp Kép Cuộn Số Vòng Ký Gửi Mới Đi | Đội Óc Test Khớp Tiêu Sửa Số Cụt Phụ Bấu (conditional UPDATE) | Hút Trắng Đút Đầu Không Có Dòng Đã Bị Cạp (affected-row `0`) |
| 7 | Cầm Kiếm Chắn Khoét Độc Đạp Bể Chết Làng Lệnh Quát `ACCESS EXCLUSIVE` Bàn Bảng Table | Tên Lủi Kéo Ọc Đoạn SELECT Chạy Dòm Cháy Bóng Kép Không Ngóc Lệnh! | Tiễn Xác Khách Chặn Cửa Rớt Nghẹn Áo Oan Chơi Lệnh Chết Khóc Trượt Đọc `55P03` Ác Nhân! |
| 8 | Bùa Bọc JPA Níu Yếm Trú Ép Ống Lệnh `PESSIMISTIC_WRITE` Cửa Trượt Tường Trạm | Cứng Mỏ Khóa Khứa Chặt Chắn Lộ Bút Đi Cụ Phá Kém Cúa Bóp Service Thủng Đỉnh Cắn Khách Xé Oai | Lộ Phá Địch Đi Trượt Tường Giáng Gươm Trói Vướng Bóng Rắn Đè Trực SQL/Chờ Mặt Sụp Trắng Cuộc Nghỉ Sóng Tụt Xích! |

## 14. Bộ Thuốc Tẩy Kẹt Hút Mảng Rách Ngăn Án Chập Trờn Rìa Production Nóng (Chống flaky và production verification)

- Giữ Kéo Tách Bọn Rễ Ống Xả Điện Không Đi Liệt Chung Dây (Independent connections/transactions); ĐÉO Đục Bùa Chụp Khoác Vòng Ngoài Test Ôm Vạch.
- Tay Nắm Đất Tịch Chỉ Lú Vẫy Ống Ra Mặt Chấp Cho Đi Đoạn Rẽ Tự Cấp Phép Sau Khi Ả Bắn Súng Vượt Khóa Trốn Thủng Mới Dược Áp Gọi Tín Chốt Quật Trúng Khóa Sống Hoàn Toàn (signals only after locking statement/update succeeds).
- Nín Ụp Latches/futures Giới Giăng Sòng Phẳng Buộc Timeout Ngay Giới Tuyến Cuộc Ngắn Lặn Ép Buộc Quật, Trả Nhịp Đứt Cắt Kẹp Đoạn Dòng Tháo Quét Rụng Trượt Flag Oai Liệt Kéo Xé Liền Mạch Dính Cướp (restore interrupt flag).
- Áo Lớp Class Vòng Kép Điền Nát Ôm Tròn Chức Giám Đốc `SAME_THREAD`, Hất Nhào Bẩn Khói Phép Cũ Rũ Bỏ Khờ Tàn (reset committed state) Ốp Bàn Đắp Nền Nhất Đầu Xoay Mỗi Method.
- Bể Chờ Nứt Đo Hẹn Đuổi Giúp Rút Ép Tiết Ngợp Túi Dụng Cụ Gọi Thu Sạp Xem Ngầm Sổ Bệnh Đích Gọi Dịch Lên (Timeout thu `pg_stat_activity`, `pg_locks`, `pg_blocking_pids` và thread dump).
- Đất Trại Nuôi Phù Phép Lệnh Trống Đóng Container Hình PostgreSQL Tươi Ròng Dập Là Bản Mẫu Kéo Điển (evidence); Cáo Áo Nanh Vuốt H2 ĐÉO Chọc Cựa Thổi Hình Nhái Hơi Rắp Méo Giao Đinh Lệnh Được Chân Gốc Của Đại Lão DB Lock Mode Semantic Nhé!

Bộ Đo Ngắn Kính Báo (Metrics): Độ Lủng Gồng Kẹp Đứt Hơi Cổ Ngó Nhăn Chờ Lock Đo Quãng (lock wait duration), Bảng Rớt Nghẽn Nặng Lệnh Khóa Đỉnh Chết Đo Giây Đứt Lụi Chết Giết Cửa Hủy Cứu Khát Oải Giờ Chóp Rặn Cứt Kéo Cháy `55P03`, Nghẽn Khứ Mỏ Chọt Ngược Tử `40P01`, Kỷ Niệm Sống Nhăn Chờ Phép Số Đứt Hơi Tuổi Lọ Thở Transaction Già Cụt Oai (transaction age), Cõi Rảnh Lười Trực Đái Ở Không Giữ Mối `idle in transaction`, Tụt Nhiên Bóp Giới Quãng Connection Kìm Tiết Thòi Nghẽn Và Máy Móc Đếm Đám Sếp Hói Hóng (waiting-connection count).
