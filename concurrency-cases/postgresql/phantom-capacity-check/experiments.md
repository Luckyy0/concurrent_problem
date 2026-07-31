# Buộc Tội Hiện Trường Tranh Gian Sức Chứa (Deterministic predicate-capacity experiments)

## 1. Mục Tiêu Lên Thớt (Mục tiêu)

Bộ Trảm Kiểm Tra (Test suite) Này Dựng Lên Cốt Găm Đinh Trọng Án Báo Rõ Rằng:

1. Cuốc Cuốc Kháo Nhìn COUNT ở `READ COMMITTED` Mà Xem Thì Banh Mắt Mới Chọc Thấy Được Cái Bóng Kép Commit Kín.
2. Quả Đấm Ngược Đếm-Xong-Nhét Hai Nhát Đủ Lực Thổi Dạt Giới Hạn Cứu (active count) Tốc Trần Lên `11`.
3. Bọc Kéo `REPEATABLE READ` Ngậm Băng Tĩnh Ổn (snapshot) Đó Nhưng Ép Túng Sức Chứa (capacity) Dưới Chân Vẫn Lỏng Nát Khống Nổi.
4. Trùm Cháy `SERIALIZABLE` Ốp Cửa Trảm Cắt Ngược Một Khách Nhả Chết Kịp Rớt Rành Nền Cho Ổn Số Dư `10`.
5. Đóng Van Đếm Thần Thánh (conditional counter) Chỉ Tặng Duy Nhất Giải Vô Địch Winner Cho 1 Kẻ Tới Bờ!
6. Bịt Ống Áp Sếp Khóa Ngực Bố Mẹ Parent Row Có Góc Góc Chờ Đầu Tử Loser Cắn Xích Chết Timeout Ế Rõ Ngon (bounded loser).
7. Nắm Vé Giành Ghế Khóa Gõ `SKIP LOCKED` Thọc Ngực Lẻ Kép KHÔNG Sớt Cho Hai Kẻ Chung 1 Ghe Ghế Slot.
8. Kẹp Phanh Xé Ruột Rollback KHÔNG Trôi Xoắn Cục Bộ Bơm Đếm Trôi Dốc Nào (counter drift).

Ngâm Chốt Chờ Tĩnh Kép Ổn Định Gây Chết Sẽ Xài Barrier Nhé Đừng Ngu Chơi Đi Xài Đo Cáp Phút Chờ Hên Xui Đoán Đồng Hồ Gỗ Tích. Tất Bật `await` Lẫn Gọi Chóp Kép `Future.get` PHẢI Găm Timeout Bịt Đầu Đội Giới Trừ Sụt Đuôi Rớt Chờ Hoài.

> **Sếp chốt lại:** Quật Test Kiện Bịt Vá Áo Lỗi Thì Dò Phải Tra Tận Tủy Cắm `active <= capacity`, Nhìn Kỹ Mấy Đứa Thắng Winner Cùng Hậu Tích Khắc Đá Án Vệt Cát Khói Commit Rõ Nhé; Chứ Soi Tìm Kéo Chặn Quát Tách Exception Độc Cụt Không Chút Thấm Án Gì Khỏi Ngộp Nước!!!

## 2. Thùng Máy Kép Chứa Trạm Chọt Testcontainers (PostgreSQL Testcontainers)

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
class PhantomCapacityIntegrationTest {

    private static final UUID POOL_ID =
        UUID.fromString("10000000-0000-0000-0000-000000000042");
    private static final UUID EXISTING_REQUEST_ID =
        UUID.fromString("20000000-0000-0000-0000-000000000042");

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
            create table processing_pool (
                pool_id uuid primary key,
                capacity integer not null,
                used_slots integer not null,
                check (
                    capacity >= 0
                    and used_slots >= 0
                    and used_slots <= capacity
                )
            )
            """);
        jdbc.execute("""
            create table slot_allocation (
                allocation_id uuid primary key,
                pool_id uuid not null references processing_pool(pool_id),
                request_id uuid not null,
                status varchar(16) not null,
                unique (pool_id, request_id)
            )
            """);
        jdbc.execute("""
            create table processing_slot (
                pool_id uuid not null references processing_pool(pool_id),
                slot_no integer not null,
                allocation_id uuid,
                primary key (pool_id, slot_no),
                unique (allocation_id)
            )
            """);
    }

    @BeforeEach
    void resetState() {
        executor = Executors.newFixedThreadPool(2);
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        jdbc.update("delete from processing_slot");
        jdbc.update("delete from slot_allocation");
        jdbc.update("delete from processing_pool");
        jdbc.update(
            """
            insert into processing_pool(pool_id, capacity, used_slots)
            values (?, 10, 9)
            """,
            POOL_ID
        );
        for (int number = 1; number <= 9; number++) {
            jdbc.update(
                """
                insert into slot_allocation(
                    allocation_id, pool_id, request_id, status
                ) values (?, ?, ?, 'ACTIVE')
                """,
                UUID.randomUUID(),
                POOL_ID,
                number == 1
                    ? EXISTING_REQUEST_ID
                    : UUID.randomUUID()
            );
        }
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }

    @AfterAll
    static void closePool() {
        dataSource.close();
    }
}
```

Bãi Chạy Test Methods KHÔNG Thèm Mang Bọc Vỏ Tròn Ngoài (outer transaction). Bất Kỳ Ai Gọi Lính Cựa Kép Đều Cầm Súng Tựa Ống Nước Cắm Xoay Cuộc Tự Đích Ngôi Lên Chơi (connection riêng).

## 3. Trấn Rào Giữ Cửa Cho Bọn Chơi Đều Góc (Coordination helpers)

```java
final class BothCountedGate {
    private final CountDownLatch counted = new CountDownLatch(2);
    private final CountDownLatch mayInsert = new CountDownLatch(1);

    void counted() {
        counted.countDown();
        await(counted, "both predicate reads");
    }

    void releaseInserts() {
        mayInsert.countDown();
    }

    void awaitInsertPermission() {
        await(mayInsert, "insert permission");
    }
}

record AttemptOutcome(
    long observedCount,
    boolean committed,
    String sqlState
) {
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

## 4. Bàn Tay Vàng Gọi Nghịch DB (JDBC helpers)

```java
private static long activeCount(Connection connection) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select count(*)
        from slot_allocation
        where pool_id = ?
          and status = 'ACTIVE'
        """)) {
        statement.setObject(1, POOL_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
            return rs.getLong(1);
        }
    }
}

private static void insertActive(
    Connection connection,
    UUID requestId
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        insert into slot_allocation(
            allocation_id, pool_id, request_id, status
        ) values (?, ?, ?, 'ACTIVE')
        """)) {
        statement.setObject(1, UUID.randomUUID());
        statement.setObject(2, POOL_ID);
        statement.setObject(3, requestId);
        assertThat(statement.executeUpdate()).isEqualTo(1);
    }
}

private static AttemptOutcome brokenAttempt(
    int isolation,
    UUID requestId,
    BothCountedGate gate
) throws SQLException {
    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        connection.setTransactionIsolation(isolation);
        long observed = activeCount(connection);
        gate.counted();
        gate.awaitInsertPermission();

        try {
            insertActive(connection, requestId);
            connection.commit();
            return new AttemptOutcome(observed, true, null);
        } catch (SQLException ex) {
            connection.rollback();
            return new AttemptOutcome(observed, false, ex.getSQLState());
        }
    }
}
```

## 5. Màn Mổ Xẻ 1 — Kéo Vành Áo Chọt Bóng Ma Committed Ở `READ COMMITTED` (Experiment 1)

```java
@Test
void laterReadCommittedStatementSeesNewMatchingRow() throws Exception {
    CountDownLatch firstCountDone = new CountDownLatch(1);
    CountDownLatch writerCommitted = new CountDownLatch(1);

    Future<List<Long>> reader = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            connection.setTransactionIsolation(
                Connection.TRANSACTION_READ_COMMITTED
            );
            long first = activeCount(connection);
            firstCountDone.countDown();
            await(writerCommitted, "writer commit");
            long second = activeCount(connection);
            connection.commit();
            return List.of(first, second);
        }
    });

    Future<Void> writer = executor.submit(() -> {
        await(firstCountDone, "first count");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            insertActive(connection, UUID.randomUUID());
            connection.commit();
        }
        writerCommitted.countDown();
        return null;
    });

    assertThat(reader.get(5, TimeUnit.SECONDS)).containsExactly(9L, 10L);
    writer.get(5, TimeUnit.SECONDS);
}
```

## 6. Màn Mổ Xẻ 2 — Bể Đáy Rác Chốt Tổng Lên Cột Nhảy Đỉnh Bóp Trán `11` (Experiment 2)

```java
@Test
void readCommittedCountThenInsertExceedsCapacity() throws Exception {
    BothCountedGate gate = new BothCountedGate();

    Future<AttemptOutcome> a = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_READ_COMMITTED,
            UUID.randomUUID(),
            gate
        )
    );
    Future<AttemptOutcome> b = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_READ_COMMITTED,
            UUID.randomUUID(),
            gate
        )
    );

    gate.releaseInserts();
    AttemptOutcome resultA = a.get(5, TimeUnit.SECONDS);
    AttemptOutcome resultB = b.get(5, TimeUnit.SECONDS);

    assertThat(resultA.observedCount()).isEqualTo(9);
    assertThat(resultB.observedCount()).isEqualTo(9);
    assertThat(resultA.committed()).isTrue();
    assertThat(resultB.committed()).isTrue();
    assertThat(inspector().activeCount()).isEqualTo(11);
    assertThat(inspector().activeCount())
        .isGreaterThan(inspector().capacity());
}
```

Mấy Chú Bắt Bẻ Test Lỗi Bể Cái Rào Mà Đập Khẳng Định Ngang Xương Xảy Lỗi (assert violation) Mục Đích Để Éo Có Kêu Trở Lại Dựa Vào Trò Ngẫu Nhiên Xác Suất Gây Họa Hóc Mép Đi Lặng Đi Kép Kêu Gì Á (reproduction không phụ thuộc xác suất).

## 7. Màn Mổ Xẻ 3 — Sóng `REPEATABLE READ` Còn Cứng Đầu Xuyên Toạc Ngang Gãy Cánh Sức Chứa (Experiment 3)

```java
@Test
void repeatableReadStableSnapshotsDoNotEnforceCapacity() throws Exception {
    BothCountedGate gate = new BothCountedGate();

    Future<AttemptOutcome> a = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_REPEATABLE_READ,
            UUID.randomUUID(),
            gate
        )
    );
    Future<AttemptOutcome> b = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_REPEATABLE_READ,
            UUID.randomUUID(),
            gate
        )
    );

    gate.releaseInserts();
    List<AttemptOutcome> outcomes = List.of(
        a.get(5, TimeUnit.SECONDS),
        b.get(5, TimeUnit.SECONDS)
    );

    assertThat(outcomes).allSatisfy(outcome -> {
        assertThat(outcome.observedCount()).isEqualTo(9);
        assertThat(outcome.committed()).isTrue();
    });
    assertThat(inspector().activeCount()).isEqualTo(11);
}
```

Bản Test Này KHÔNG Gọi Điếm Result Đó Là Chọc Ma Trồi Lên Trông Lồi Mặt Ra Nào Khéo Nhé! (visible phantom). Lý Do Bóng Tối Thằng Kia Nó Cắt Ngang Có Được Cảm Thấy Khúc Dòng Mới Nhú Sờ Trong Cái Hộp Khung Cảnh Tĩnh Chết Tịch Đó Đâu Nào Á (stable snapshot). Trọng Án Bóp Còi Dọng Ép Assertion Nhấn Chú Mục Trúng Cái Luật Rốn Thủng Vỡ Toạc Kẹp Đầu Đuôi Sau Cuối Kìa (final invariant).

## 8. Màn Mổ Xẻ 4 — Búa Tạ `SERIALIZABLE` Thẳng Tay Vả Phá Cuộc Chơi Nhép Cổ Bỏ Cục (Experiment 4)

```java
@Test
void serializableTurnsPredicateRaceIntoRetryableAbort() throws Exception {
    BothCountedGate gate = new BothCountedGate();

    Future<AttemptOutcome> a = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_SERIALIZABLE,
            UUID.randomUUID(),
            gate
        )
    );
    Future<AttemptOutcome> b = executor.submit(
        () -> brokenAttempt(
            Connection.TRANSACTION_SERIALIZABLE,
            UUID.randomUUID(),
            gate
        )
    );

    gate.releaseInserts();
    List<AttemptOutcome> outcomes = List.of(
        a.get(5, TimeUnit.SECONDS),
        b.get(5, TimeUnit.SECONDS)
    );

    assertThat(outcomes.stream().filter(AttemptOutcome::committed).count())
        .isEqualTo(1);
    assertThat(outcomes.stream()
        .filter(outcome -> "40001".equals(outcome.sqlState()))
        .count()).isEqualTo(1);
    assertThat(inspector().activeCount()).isEqualTo(10);
    assertThat(inspector().activeCount())
        .isLessThanOrEqualTo(inspector().capacity());
}
```

Éo Màng Bức Gắt Là Cu A Hoặc Em Bứa Thằng B Nào Sẽ Thành Loser Kẻ Quỳ Mõm Khóc (loser identity). Lên Trận Bật Dọn Vết Production TỰ Bật Đuôi Mở Hút Máy Lại Áo Transaction Kép Trắng Khác Nhé; Khúc Test Chỉ Gõ Nhịp Đập Assert Báo Ràng Kín Cái Hợp Đồng Gãy Đụng Kẹp Mức Cơ Sở Dữ Liệu Đất Này Dọc Sâu Đáy Nhấp (database conflict contract).

## 9. Màn Mổ Xẻ 5 — Mài Nhẵn Lồng Đếm Chỉ Mở Lọt Lòng Được Duy Nhất Một Khứa Người Thắng Bức Tóc (Experiment 5)

```java
record ClaimOutcome(boolean accepted, int affectedRows) {
}

private static ClaimOutcome atomicClaim(
    UUID requestId,
    CountDownLatch ready,
    CountDownLatch start
) throws SQLException {
    ready.countDown();
    await(start, "atomic claim start");

    try (Connection connection = dataSource.getConnection()) {
        connection.setAutoCommit(false);
        try (PreparedStatement claim = connection.prepareStatement("""
            update processing_pool
               set used_slots = used_slots + 1
             where pool_id = ?
               and used_slots < capacity
            """)) {
            claim.setObject(1, POOL_ID);
            int affected = claim.executeUpdate();
            if (affected == 1) {
                insertActive(connection, requestId);
            }
            connection.commit();
            return new ClaimOutcome(affected == 1, affected);
        } catch (SQLException ex) {
            connection.rollback();
            throw ex;
        }
    }
}

@Test
void conditionalCounterAcceptsOnlyOneConcurrentRequest() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Future<ClaimOutcome> a = executor.submit(
        () -> atomicClaim(UUID.randomUUID(), ready, start)
    );
    Future<ClaimOutcome> b = executor.submit(
        () -> atomicClaim(UUID.randomUUID(), ready, start)
    );

    await(ready, "both claimers ready");
    start.countDown();
    List<ClaimOutcome> outcomes = List.of(
        a.get(5, TimeUnit.SECONDS),
        b.get(5, TimeUnit.SECONDS)
    );

    assertThat(outcomes.stream().filter(ClaimOutcome::accepted).count())
        .isEqualTo(1);
    assertThat(outcomes.stream().mapToInt(ClaimOutcome::affectedRows).sum())
        .isEqualTo(1);
    assertThat(inspector().usedSlots()).isEqualTo(10);
    assertThat(inspector().activeCount()).isEqualTo(10);
}
```

## 10. Màn Mổ Xẻ 6 — Nhót Khóa Lão Tướng Chặn Sọc Buộc Kẻ Đụng Đầu Phải Ói Hơi Đứt Xích Tắt Kịp Nữa Lời (Experiment 6)

```java
@Test
void parentRowLockMakesSecondAllocatorWaitOrTimeout() throws Exception {
    CountDownLatch parentLocked = new CountDownLatch(1);
    CountDownLatch contenderFinished = new CountDownLatch(1);

    Future<Void> owner = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            lockParent(connection);
            parentLocked.countDown();
            await(contenderFinished, "contender timeout");
            connection.commit();
        }
        return null;
    });

    Future<String> contender = executor.submit(() -> {
        await(parentLocked, "parent lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            try (Statement statement = connection.createStatement()) {
                statement.execute("set local lock_timeout = '300ms'");
            }
            try {
                lockParent(connection);
                connection.commit();
                return "unexpected-lock";
            } catch (SQLException ex) {
                connection.rollback();
                return ex.getSQLState();
            } finally {
                contenderFinished.countDown();
            }
        }
    });

    assertThat(contender.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    owner.get(5, TimeUnit.SECONDS);
}
```

Lôi Theo 1 Cuộc Test Kéo Trận Mặt Đỉnh Lớp Nghiệp Vụ Cấp Cao (service-level) Soi Chóp Cho Chủ Nhân Thắng Vé Owner Giật Cửa Cống Xong Thằng Cùi Contender Quệt Áo Trắng Lùi Lui Retry/Recount Xong PHẢI Tạt Lại Kiểm Sạch Rằng Đầu Phát `ACCEPTED`, Bị Chặn Cái Hai Bật Trống `FULL`, Giữ Kín Cực Đỉnh Đầu Đáy Đầm Kép `10`.

## 11. Màn Mổ Xẻ 7 — Câu Gọi Rẽ Bóng Mây Khép `SKIP LOCKED` Éo Thưởng Trùng Cho Cu Kép Hai Khúc Trên Cùng Căn Dịch Ghế (Experiment 7)

```java
@Test
void skipLockedReturnsNoCandidateWhileOnlySlotIsClaimed() throws Exception {
    JdbcTemplate jdbc = new JdbcTemplate(dataSource);
    jdbc.update(
        """
        insert into processing_slot(pool_id, slot_no, allocation_id)
        values (?, 1, null)
        """,
        POOL_ID
    );

    CountDownLatch slotLocked = new CountDownLatch(1);
    CountDownLatch contenderDone = new CountDownLatch(1);
    UUID allocationId = UUID.randomUUID();

    Future<OptionalInt> owner = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            OptionalInt slot = selectFreeSlotSkipLocked(connection);
            slotLocked.countDown();
            await(contenderDone, "slot contender");
            assignSlot(connection, slot.orElseThrow(), allocationId);
            connection.commit();
            return slot;
        }
    });

    Future<OptionalInt> contender = executor.submit(() -> {
        await(slotLocked, "slot lock");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            OptionalInt slot = selectFreeSlotSkipLocked(connection);
            connection.commit();
            contenderDone.countDown();
            return slot;
        }
    });

    assertThat(contender.get(5, TimeUnit.SECONDS)).isEmpty();
    assertThat(owner.get(5, TimeUnit.SECONDS)).hasValue(1);
    assertThat(inspector().assignedSlotCount()).isEqualTo(1);
}
```

Thằng Vét Câu `selectFreeSlotSkipLocked()` Rập Ngay Y Khuôn Bỏ Lệnh Chặt `FOR UPDATE SKIP LOCKED LIMIT 1` Ra Khỏi Ổ Giải Pháp Chữa Bệnh Nằm Bên Kia Đó; Trấn Gài Khóa Dòng Kép Cháy Row Lock Gồng Nắm Bền Dẻo Thở Nín Chờ Đến Mốc Khép Đỉnh Cùng Chốt Vực Chết Gãy Trắng Đất Cuộc Cuốn Xả Lưng Giao Dịch (connection commit/rollback).

## 12. Màn Mổ Xẻ 8 — Cầm Lấy Miếng Vé Xong Dán Tụt Tay Rollback Chết Vét Lăn Dòng Giữ Một Đôi Dính Không Rụng Bể Sổ Tiêu Hao (Experiment 8)

```java
@Test
void failedAllocationRollsBackCounterClaim() {
    JdbcTemplate jdbc = new JdbcTemplate(dataSource);
    TransactionTemplate transactionTemplate = new TransactionTemplate(
        new DataSourceTransactionManager(dataSource)
    );

    assertThatThrownBy(() -> transactionTemplate.executeWithoutResult(status -> {
        int claimed = jdbc.update(
            """
            update processing_pool
               set used_slots = used_slots + 1
             where pool_id = ?
               and used_slots < capacity
            """,
            POOL_ID
        );
        assertThat(claimed).isEqualTo(1);

        jdbc.update(
            """
            insert into slot_allocation(
                allocation_id, pool_id, request_id, status
            ) values (?, ?, ?, 'ACTIVE')
            """,
            UUID.randomUUID(),
            POOL_ID,
            EXISTING_REQUEST_ID
        );
    })).isInstanceOf(DataIntegrityViolationException.class);

    assertThat(inspector().usedSlots()).isEqualTo(9);
    assertThat(inspector().activeCount()).isEqualTo(9);
}
```

Cái Bẫy Đã Sẵn Rào ID Dính Bẩy `EXISTING_REQUEST_ID` Cắm Nằm Phục Găm Chốt Từ Lúc Giữa Màn Nhử Mồi Đi Trước Test Để Dính Chắc Con Rác Độc Vướng Sốc Nổ Phá Áo Ngực Trùng Rớt Quả Hàng (unique violation) Đâm Nhót Đập Kép Cùng Nguyên Rạp Bọn Áo Transaction Cháy Chung Lửa Cùng Khung Tích Cho Quẩy Tán Loạn! Xé Đáy Bỏ Tưởng Cho Rảnh Không Ngâm Vuốt Khóc Lùi Trốn Kẹp Hút Nắm Miệng Giấu Kép Mù Catch Exception Ở Khúc Ruột Nửa Tròn Trong Trôi Lềnh Bệnh Transaction Rồi Ém Trắng Cụp Commit Kéo Gượng Kịch Bể Ụp Ép Sập Chết Đều Nghe Mấy Anh!

## 13. Kho Đồ Chơi Nhặt Phụ Phụ Đo Đạc Cắt Kéo Thêm Dụng (Helpers còn lại)

```java
private static void lockParent(Connection connection) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select pool_id
        from processing_pool
        where pool_id = ?
        for update
        """)) {
        statement.setObject(1, POOL_ID);
        try (ResultSet rs = statement.executeQuery()) {
            assertThat(rs.next()).isTrue();
        }
    }
}

private static OptionalInt selectFreeSlotSkipLocked(Connection connection)
    throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        select slot_no
        from processing_slot
        where pool_id = ?
          and allocation_id is null
        order by slot_no
        for update skip locked
        limit 1
        """)) {
        statement.setObject(1, POOL_ID);
        try (ResultSet rs = statement.executeQuery()) {
            return rs.next()
                ? OptionalInt.of(rs.getInt(1))
                : OptionalInt.empty();
        }
    }
}

private static void assignSlot(
    Connection connection,
    int slotNo,
    UUID allocationId
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
        update processing_slot
           set allocation_id = ?
         where pool_id = ?
           and slot_no = ?
           and allocation_id is null
        """)) {
        statement.setObject(1, allocationId);
        statement.setObject(2, POOL_ID);
        statement.setInt(3, slotNo);
        assertThat(statement.executeUpdate()).isEqualTo(1);
    }
}
```

Kính Lúp Dò Đường Chụp Vạch Bắn Lén Đếm Số Sau Áo `inspector()` Ánh Giương Lướt Nhóm Thầy Trò Đọc Chờ Kết Quả Đọng Đá Gắn Ống Bút Mới `JdbcTemplate` Xé Phẳng Tuột Nằm Ở Khung Ngoại Dịch Rời Trôi Lềnh Máy Sàn Tĩnh (ngoài actor transactions) Để Lắng Đọc Bức Vách Tình Trạng Chốt Nhót Yên Commit Ghi Rành Mạch: `capacity`, Số Chỗ Sài Mòn `used_slots`, Nhịp Đếm Người Còn Đú (active count) Và Số Vết Xé Cắt Chia Assigned Slot Chỗ.

## 14. Bảng Sớ Đo So Kéo Bao Gấp Chéo Đâm Toạt Rách Cuộc Hủy Diệt (Coverage matrix)

| Trò Nghịch Lửa (Experiment) | Chiêu Kẹp Thần Thủ Cửa Trục (Mechanism) | Cuộc Cờ Bày Mưu Ép Trục Quay (Controlled order) | Bản Trảm Cuộc Phán Cuối Vạch (Invariant assertion) |
| --- | --- | --- | --- |
| 1 | Cắn Miếng Khói Snap Đứt Kẹp Chắn RC | B Gõ Chốt Búa Tạt Bóng Sang Giữa Nhịp COUNTs | Bóp Số Lộ Trắng Quả Tình `9 -> 10` |
| 2 | Tai Nạn Rạn Bụng Tụt Chỗ Áo RC | Hai Thằng Nhe Nanh Chớp Đếm Sóng Bể Đít Tranh Chưa Kịp Bơm Nhét Nhồi Kéo Mâm Dồn Gắn Phụt Ngay Lưng Nhập Nhằng | Vỡ Trận Văng Dội Hất Ngang Lên `11` |
| 3 | Màng Đóng Đá Chắc Gỗ Áo Bền Chặn Trôi RR Snap | Rượt Sút Chung Nhe Sóng Răng Cả 2 Đếm Ánh Trước Ngõ Chui Bụng Cửa INSERT Dồn Cháy Nhót | Cả 2 Đâm Cùng Chốt Ván Liều Rượt Xuyên Mốc Vọt Ngõ `11` |
| 4 | Trùm Tử Hình Rập Tắt Cửa SSI Trọn Cục | Gắn Song Pháo Mắt Thần Khựng Gắng Sợ Đụng Ngõ Nhét Nặn Gắn Lưới Điều Kiện Tát Mặt Phẳng Tịch Sót Bóng Sóng Ngõ Rách Đọc Kíp Ván Lướt Vội Vào | Một Khứa Oẳng Đọt Án Tạt Tiếng Bể Lỗi Trảm Chìm Hủy Chạy Khựng Oan Mệnh Móc Đầu Ép Bỏ Trùm `40001`, Kết Cục Mượt Sạch Y Số Vững Đỉnh Đạt `10` |
| 5 | Lò So Nhịp Cò Mút Chỉ Định Chóp Thắng Lên Một Bước Một Thôi | Úp Song Chạm Thẳng Gắn Búa Xẻ Dòng Cửa Tụ Điểm Búng Gạt Cực Kép Cuộc UPDATE Đều Bóp Quật | Kẻ Búng Độc Cáo Báo Một Vết Trúng Lệ, Số Giữ Đỉnh Y Còn `10` |
| 6 | Ông Cậu Phụ Khống Kép Lệ Chắn Khóa Phép Gông Bền `FOR UPDATE` | Chú Tướng Chủ Giữ Rít Quấn Dày Row Lock | Kẻ Cùi Oẳng Đuổi Gãy Giới Lệ Tắc Tiết Tiếng Nghẹn Ố `55P03` |
| 7 | Cầm Tráp Cú Chui Sợ Gãy Lộn Lướt Ọc Bơm Bám Mớ Hôi Của Bịp Dày Vé Hộp Rút Chọn Đi Lặn Giành Khe Xí Ngõ Nhét Cửa Đứt `SKIP LOCKED` | Trùm Lính Nằm Ôm Chờ Kéo Ém Vút Slot Đít Thôi | Kẻ Ngước Xin Trút Hụt Ộp Khống Nát Tay Nắm Khí Trời Chơi Rách Sót Thùng Tiêu Đòi Cuốn Nhịp (empty) |
| 8 | Lỗi Nát Bưng Ộp Cuộc Kẹp Chảy Chết Bỏ Ụp (Atomic rollback) | Hô Cò Gọi Chụp Kép Nhét Slot Trướng Mũi Sớm Lệnh Chết Mắc Chui INSERT Rụng Hụt Đuôi | Sổ Đếm Gắn Bóp Rớt Kép/Row Văng Sạch Còn Nguyên Hình Bóng Rõ Trút Giếng Y Sóng `9` |

## 15. Dán Bùa Lên Quần Tránh Đóng Phim Đứt Dây Ngắn Dạng Flaky Lỗi Chạy Ma Mù Nhỏ (Chống flaky)

- TUYỆT ĐỐI ÉO Dùng Ba Cái Trò Mồi Dưỡng Quăng Treo Đồng Hồ Tick Thời Gian Đón Nhanh Đoạt Đầu Đuôi Làm Cơ Quan Mở Khe Rẽ Bóng Chọc Trễ Nghịch Dây Rối Interleaving Ụp Ngốc.
- Tấc Bịch Mũi Mồi Móc Phá Tách Treo Mọi Kéo Future/Latch Buộc Găm Đinh Hẹn Timeout Dẹp Lì Giới Tới Đi Lùi Khống Lọt Rủi Khứ Máng Lệnh Restore Vớt Interrupt Flag Ráng Lên Nghe Rõ!
- Tụi Dân Cày Actors Ôm Xài Bơm Ngắn Vòng Vòi Giao Khống Ổn Connection Tươi Kịp/Transaction Độc Tuyệt Hoàn Cự Trơn Kín Đi Chống Khác Hoàn Toàn Riêng; Xé Toạc Vứt Bỏ Outer Test Khống Chéo Sân Transaction Kẹp Chống.
- Đám Lũ Test Class Sét Trận Hò Ép Giáp Khít `SAME_THREAD` Trút Máng Reset Lệnh Giũ Sạch Sân Chơi Lật Đất Chung Đáy Database State Chuyển Kịp Gấp.
- Bức Tường Chống Trảm SERIALIZABLE Kép Kiểm Giao Án Đếm So Cựa Thằng Sống Số Má Winner Cột Mệnh Án/SQLSTATE Nín Mỏ Chờ Lỗi Trúng Kẻ Kia Xéo Cuộc, Chứ Không Thích Hỏi Tội Gõ Tên Ai Đui Đứt Chạm Liệt (actor identity) Trận Lãnh Thua Kia Là Đứa Đểu Xui Rủi Nào Nghen Rách Tay!
- Sập Hụt Giãn Khống Quát Đất Timeout Gọi Xí Kéo Cào Dọn Hốt Số Mã Ráp Máy Đếm Giật Gương Lọc Mạch Truy Số Máy Áp Thu Bịp Ngay Sổ Phạt Mực Tụ `pg_stat_activity`, Khóa Mép Khựng `pg_locks` Gộp Sòng Cả Bụm Cứt Văng Khói Hót Rác Đội Hỏa Thread Dump Soi Án Đi Rạch Sâu Nghe.
- Bịt Nút Khóa Xó Bịch H2 Trạm Giả Ở Máy Đồ Chơi Khống Tí Đuôi Tàu Ra Rìa Ngay Nhé Đéo Hề CÓ CỬA Mang Vào Chường Làm Giấy Dịch Bịp Bằng Chứng Gọi Thư Xác Hóa Thấy Gì Đâu Nhé Luyện Cáo MVCC/Predicate Lẫn Chướng Sát Tích Trận Hỏa Khí Điếm Áo SSI!

## 16. Sớ Giấy Kêu Oan Kiểm Chứng Án Khống Tại Lò Trảm Khốc Liệt Giữa Lũ Máy Cày Sản Xuất Thật (Production verification)

Bộ Truy Vấn Kiểm Soát Sai Số Chết Tiệt Đấu Lọc Đối Chứng Giới Bứt Tiếng Kép Khống Dối (Reconciliation):

```sql
select p.pool_id,
       p.capacity,
       p.used_slots,
       count(a.*) filter (where a.status = 'ACTIVE') as active_count
from processing_pool p
left join slot_allocation a on a.pool_id = p.pool_id
group by p.pool_id, p.capacity, p.used_slots
having count(a.*) filter (where a.status = 'ACTIVE') > p.capacity
    or p.used_slots <>
       count(a.*) filter (where a.status = 'ACTIVE');
```

Soi Bóp Ống Kính Đâm Rọi Soi Bức Chéo Đáy Lock Khóa Giằng Níu Sôi Rắn Bứt Đo Tín Lỗi Thòng Trói Gắn Mép Bịp SSI Diagnostics:

```sql
select pid, state, wait_event_type, wait_event, xact_start, query
from pg_stat_activity
where datname = current_database();

select pid, locktype, mode, granted, relation::regclass
from pg_locks
where relation in (
    'processing_pool'::regclass,
    'slot_allocation'::regclass,
    'processing_slot'::regclass
);
```

Giương Mắt Rình Quét Theo Đuôi Chắn Rập Con Số Khống Ác Bức Affected-row Khựng Sượng Nổ `0`, Tiếng Còi SSI Vỡ Bọc Chét Não Đứt `40001`, Bất Lực Cuốn Lồi Ngược Áp Chó Kẹp Kìm Chết Góc Giữ Timeout Rõ Toạc Xích Tiếng Khống `55P03`, Tần Suất Xé Hơi Giật Nước Uống Máng Nạp Gân Cứu Ráng Mạng Sinh Retry Attempts Nhè Đuôi Kéo Kịp Tát, Độ Chảy Nghẽn Ép Họng Kẹt Đo Áo Giờ Trôi Lệ Transaction Duration Nút Ngạt Rặn Chảy Kịp Rụng Mực Mồ Hôi, Khung Viễn Rạc Kẹp Active Số Nhót Capacity Cứt Chệch Lỗi Drift Lồi Giằng Bọng Kép Kịp Ngán Khóa Trượt Vết Cát Áo Vàng Duplicate Cọ Khớp Vất Đâm Sổ Đuôi Phá Kêu Kháo Chép Nạp Gọi Giả Cổ Tái Kéo Chết Đụng Chui Rặn Xung Nhót Y Bịch Kép Khống Mơ Đổ Duplicate Replays Đó Kẻo Quên.
