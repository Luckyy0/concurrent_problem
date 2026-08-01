# Bài Tập Thực Hành Vi Phạm Trạng Thái Tập Hợp (Deterministic predicate-capacity experiments)

## 1. Mục Tiêu Kiểm Thử (Mục tiêu)

Bộ bài kiểm thử (Test suite) này được thiết kế nhằm mục đích xác thực các khía cạnh sau:

1. Việc thực thi lệnh COUNT ở mức `READ COMMITTED` có thể dẫn đến hiện tượng bóng ma (phantom row) nếu giao dịch khác commit thành công.
2. Mô hình đếm-rồi-chèn (Count-then-insert) tạo ra lỗ hổng vượt mức sức chứa (active count) khiến tổng số lên `11` dù giới hạn là `10`.
3. Mức độ cô lập `REPEATABLE READ` tạo snapshot ổn định nhưng vẫn thất bại trong việc bảo vệ giới hạn sức chứa (capacity).
4. Mức độ cô lập `SERIALIZABLE` chủ động phát hiện lỗi và hủy (abort) giao dịch vi phạm để bảo toàn kết quả ở mức `10`.
5. Phương án sử dụng bộ đếm có điều kiện (conditional counter) đảm bảo chỉ duy nhất 1 giao dịch được cấp phát thành công.
6. Phương án sử dụng khóa cấp cha (parent row lock) bắt buộc giao dịch phía sau phải chờ (wait) và có thể rơi vào trạng thái hết hạn thời gian chờ (timeout).
7. Phương án `SKIP LOCKED` cho phép lấy dòng ngay lập tức mà không chặn đứng các giao dịch khác, giao dịch đến sau sẽ nhận về kết quả rỗng (empty).
8. Nếu quá trình chèn dữ liệu thất bại, trạng thái rollback sẽ đảm bảo bộ đếm (counter) không bị sai lệch.

Chúng ta sẽ sử dụng bộ rào chắn (Barrier) để điều phối thời gian (interleaving) một cách chính xác thay vì dùng `sleep` không đáng tin cậy. Các thao tác chờ đợi (`await`) luôn được cấu hình thời gian (Timeout) cụ thể để tránh treo test.

> **Nói ngắn gọn:** Các bài kiểm thử tích hợp (Integration test) cần đối chiếu với điều kiện thực tế của dữ liệu (active <= capacity) và kiểm tra kỹ các thông tin từ database sau khi mọi thứ được commit, thay vì chỉ dừng ở mức kiểm tra các ngoại lệ.

## 2. Thiết Lập Hệ Thống PostgreSQL (PostgreSQL Testcontainers)

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

Các phương thức test được cấu hình chạy thủ công bằng JDBC để kiểm soát nghiêm ngặt các tham số về kết nối và giao dịch.

## 3. Rào Chắn Điều Phối Trình Tự (Coordination helpers)

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

## 4. Công Cụ Hỗ Trợ Giao Tiếp Database (JDBC helpers)

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
        
        // Đọc giá trị
        long observed = activeCount(connection);
        
        // Chờ đồng bộ cả 2 luồng đọc xong
        gate.counted();
        gate.awaitInsertPermission();

        try {
            // Chèn giá trị mới
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

## 5. Kiểm Thử 1 — Phát Hiện Bóng Ma Trong Mức `READ COMMITTED` (Experiment 1)

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

## 6. Kiểm Thử 2 — Mô Hình Count-Then-Insert Vi Phạm Giới Hạn `11` (Experiment 2)

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

Đây là bài kiểm thử có tính quyết định. Hai giao dịch xen kẽ sẽ luôn chắc chắn sinh ra dữ liệu dư thừa 11 (không bị flaky) do rào chắn (Barrier) đồng bộ các tiến trình một cách ổn định.

## 7. Kiểm Thử 3 — Mức `REPEATABLE READ` Không Thể Bảo Vệ Quy Tắc Tập Hợp (Experiment 3)

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

Trong phương thức test này, các luồng không sử dụng kỹ thuật "đọc lại lần nữa" (visible phantom). Kịch bản chứng minh một snapshot tĩnh (stable snapshot) vẫn không thể xử lý những tương tác chèn dòng mới phá vỡ giới hạn cuối cùng (final invariant).

## 8. Kiểm Thử 4 — Mức `SERIALIZABLE` Buộc Giao Dịch Bị Xung Đột Hủy Bỏ (Experiment 4)

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

Kiểm thử xác minh rằng SSI của database phát hiện lỗi xung đột tập hợp (database conflict contract). Một giao dịch sẽ cam kết thành công và giao dịch còn lại sẽ gặp mã lỗi `40001`. Việc giao dịch nào thất bại phụ thuộc vào trình cơ sở dữ liệu.

## 9. Kiểm Thử 5 — Bộ Đếm Có Điều Kiện Chỉ Cho Phép Một Yêu Cầu Thành Công (Experiment 5)

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

## 10. Kiểm Thử 6 — Khóa Cấp Cha Buộc Yêu Cầu Tương Tranh Phải Chờ Hoặc Bị Hủy Bỏ Lỗi Timeout (Experiment 6)

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

Kiểm thử xác thực chiến lược cấp độ hệ thống (service-level): Giao dịch nắm giữ khóa (owner) sẽ chặn giao dịch cạnh tranh (contender). Contender có thể retry hoặc trả về kết quả lỗi tùy theo logic nghiệp vụ, đảm bảo tổng số giới hạn được tôn trọng nghiêm ngặt.

## 11. Kiểm Thử 7 — `SKIP LOCKED` Nhanh Chóng Trả Về Tập Rỗng Thay Vì Tạo Chờ Đợi (Experiment 7)

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

Nếu một slot hiện đã được chọn và giữ khóa, tùy chọn `SKIP LOCKED` cho phép câu lệnh bỏ qua nó mà không gây ách tắc hàng chờ của DB.

## 12. Kiểm Thử 8 — Tiến Trình Bị Lỗi Dẫn Đến Hủy Bỏ Và Bảo Toàn Dữ Liệu Bộ Đếm (Experiment 8)

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

Thiết lập một request ID đã tồn tại trước đó sẽ sinh ra lỗi `unique violation`. Kịch bản này kiểm tra quá trình rollback được thiết kế toàn vẹn của ứng dụng: cả quá trình tăng bộ đếm và chèn dòng dữ liệu đều bị hoàn tác (rollback) triệt để trong cùng giao dịch, ngăn sự cố mất cân bằng số liệu.

## 13. Các Phương Thức Hỗ Trợ Test Nội Bộ (Helpers còn lại)

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

Đối tượng `inspector()` giúp theo dõi các giá trị được cam kết ổn định cuối cùng (ngoài actor transactions), thông qua `JdbcTemplate` mới.

## 14. Tổng Hợp Bao Phủ Các Kịch Bản (Coverage matrix)

| Kiểm thử (Experiment) | Cơ chế xử lý (Mechanism) | Trình tự điều phối (Controlled order) | Phán quyết ranh giới tổng thể (Invariant assertion) |
| --- | --- | --- | --- |
| 1 | Bóng ma xuất hiện với Snapshot RC | B commit vào khoảng giữa 2 thao tác COUNT | Báo cáo chênh lệch dòng: `9 -> 10` |
| 2 | Lỗi Count-Then-Insert mức RC | Cả A và B chạy đếm trước khi chèn lệnh | Tình trạng bị lỗi dội quá mức lên `11` |
| 3 | Ổn định cấu trúc Snapshot RR | Cả A và B đều đọc trước, ghi sau | Cả 2 giao dịch vọt vượt định mức lên `11` |
| 4 | Kích hoạt SSI an toàn với cơ chế `SERIALIZABLE` | Các dòng tranh chấp lọc đè lên nhau | Một giao dịch sẽ bị ngắt bỏ (abort) chuẩn xác `40001`, giữ mốc 10 |
| 5 | Atomic update để kiểm tra bộ đếm | Hai truy vấn `UPDATE` thực thi đồng thời | Kịch bản 1 thành công (Update = 1), giữ chuẩn mốc 10 |
| 6 | Parent lock kết hợp timeout | Giao dịch tranh chấp (contender) chờ | `55P03` báo hiệu thời gian chờ đã đạt giới hạn |
| 7 | Bỏ qua dữ liệu chờ bằng `SKIP LOCKED` | Truy vấn lấy Slot trống không trùng | Yêu cầu sau nhận kết quả rỗng (empty) |
| 8 | Lỗi dữ liệu dẫn tới ngắt Rollback | Kích hoạt lỗi ngoại lệ rào chắn Unique C | Counter và dữ liệu trở về chuẩn `9` như trước|

## 15. Kinh Nghiệm Phòng Chống Bài Test Chập Chờn (Chống flaky)

- Tránh sử dụng các khoảng thời gian chờ cứng (câu lệnh sleep) để mô phỏng tương tác song song. Thay vào đó, hãy sử dụng các cơ chế đồng bộ rào chắn (Barrier hoặc Latch).
- Các luồng Future/Latch bắt buộc khai báo thời hạn Timeout để ngăn việc ứng dụng treo nếu có luồng thất bại.
- Các giao dịch độc lập cần kết nối (Connection) và quy trình hoạt động độc lập hoàn toàn khỏi quy chuẩn Test của Outer Transaction.
- Thiết lập `ExecutionMode.SAME_THREAD` trên Test class để dọn sạch Database State trước mỗi bài Test.

## 16. Mẹo Kiểm Toán Sự Cố Trên Môi Trường Thực Tế (Production verification)

Truy vấn xác minh tình trạng lệch kiểm toán dữ liệu (Reconciliation):

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

Theo dõi thông tin tiến trình hoặc Lock chẩn đoán (SSI Diagnostics):

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

Giám sát các con số và mã lỗi để cải thiện hiệu năng vận hành. Theo dõi tỉ lệ khóa chờ timeout (`55P03`) hay tần suất giao dịch báo lỗi (`40001`), đi kèm kịch bản sửa đổi mức Retry hoặc Timeout phù hợp cho nghiệp vụ. Đo lường thường xuyên kết quả xác thực trực tiếp trên các Database Metrics.
