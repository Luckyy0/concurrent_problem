# Thực Nghiệm: Vòng Đời Và Hành Vi Khóa Trong PostgreSQL (Deterministic PostgreSQL lock experiments)

## 1. Mục tiêu (Objectives)

Bộ kiểm thử (Test suite) nhằm chứng minh rõ ràng cơ chế quản lý khóa của PostgreSQL thông qua các kịch bản thực tế:

1. Giao dịch đang giữ truy vấn `SELECT` thông thường (plain SELECT open) không đủ khả năng chặn các tiến trình thực hiện cập nhật (`UPDATE`).
2. Yêu cầu khóa `FOR UPDATE` sẽ ngăn chặn các luồng ghi (writer) và luồng lấy khóa khác (locking reader) tương tác trên cùng một bản ghi, buộc chúng phải vào hàng chờ.
3. Các tiến trình đóng vai trò "người quan sát" sử dụng `SELECT` thông thường không bị chặn bởi khóa cấp dòng (row lock), chúng sẽ đọc phiên bản dữ liệu cũ (old committed version).
4. Khóa chỉ được giải phóng (release) khi giao dịch hoàn tất thông qua lệnh `COMMIT` hoặc bị hủy bởi lệnh `ROLLBACK`.
5. Luồng chờ (waiter) sẽ tự động đánh giá lại điều kiện truy vấn (re-evaluate conditional predicate) sau khi giao dịch giữ khóa (holder commit) thành công.
6. Mức khóa toàn bảng `ACCESS EXCLUSIVE` có khả năng ngăn chặn cả các truy vấn `SELECT` thông thường.
7. Đánh giá cách Spring framework (thông qua `PESSIMISTIC_WRITE`) sinh mã SQL và quản lý thời gian sống của khóa (lock lifetime).

Cần sử dụng công cụ điều phối đa luồng định kỳ (latch/future kèm timeout) thay vì dùng lệnh ngâm thời gian tĩnh (`Thread.sleep`) để bảo đảm độ chính xác của kiểm thử.

> **Ghi chú quan trọng:** Mỗi kiểm thử mô phỏng một cấp độ khóa (lock mode) hoặc tình huống hiển thị (visibility). Khi kiểm thử thất bại, nguyên nhân có thể do việc nhận định sai lệnh tương thích hoặc cấu trúc ranh giới bị phá vỡ.

## 2. Thiết lập Môi trường Testcontainers (PostgreSQL Testcontainers)

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

Kiểm thử được thiết kế độc lập với transaction của Spring (không gắn `@Transactional` ở mức Test) để đảm bảo các tiến trình giao tiếp và xử lý thông qua các phiên (connection) vật lý cách ly nhau hoàn toàn.

## 3. Tiện ích Điều phối Đồng bộ (Coordination helper)

```java
private static void await(CountDownLatch latch, String step) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Vượt quá thời gian chờ: " + step);
        }
    } catch (InterruptedException ex) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Luồng bị ngắt khi chờ: " + step, ex);
    }
}
```

## 4. Thực nghiệm 1 — Truy vấn đọc thông thường không chặn luồng ghi (Experiment 1 — Plain SELECT does not reserve row)

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
            int first = selectQuota(connection, false); // Đọc 10
            firstReadDone.countDown();
            await(writerCommitted, "writer commit");
            int second = selectQuota(connection, false); // Đọc 8
            connection.commit();
            return List.of(first, second);
        }
    });

    Future<Void> writer = executor.submit(() -> {
        await(firstReadDone, "first plain read");
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            assertThat(updateQuota(connection, 8)).isEqualTo(1); // Ghi đè thành 8
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

Giao dịch đọc (Reader) mở phiên nhưng lệnh `SELECT` thông thường không tạo khóa cấp dòng (row lock). Do vậy, luồng ghi (Writer) hoàn toàn có thể thực thi `UPDATE` thành công.

## 5. Thực nghiệm 2 — Truy vấn đọc hiển thị phiên bản MVCC (Experiment 2 — Plain reader reads committed version)

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
            assertThat(updateQuota(connection, 12)).isEqualTo(1); // Cập nhật nháp
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

    assertThat(dashboard.get(5, TimeUnit.SECONDS)).isEqualTo(10); // Không đọc giá trị 12 nháp
    holder.get(5, TimeUnit.SECONDS);
    assertThat(committedQuota()).isEqualTo(12);
}
```

Mặc dù giá trị đang bị một giao dịch cập nhật thành `12` (kèm theo khóa độc quyền), luồng đọc quan sát (Dashboard) bằng `SELECT` thông thường không hề bị chặn và vẫn tiếp tục lấy được giá trị cũ (`10`) theo quy tắc MVCC.

## 6. Thực nghiệm 3 — Giới hạn quá hạn chờ khóa ghi (Experiment 3 — Incompatible writer timeout)

```java
@Test
void updateWaitsForForUpdateHolderAndTimesOutBoundedly() throws Exception {
    CountDownLatch rowLocked = new CountDownLatch(1);
    CountDownLatch contenderDone = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            selectQuota(connection, true); // Thiết lập khóa FOR UPDATE
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
                updateQuota(connection, 8); // Chờ khóa quá hạn
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

Kiểm thử xác nhận lỗi do khóa trả về mã `55P03` khi thời gian chờ vượt quá giới hạn thiết lập (`lock_timeout`).

## 7. Thực nghiệm 4 — Tranh chấp khi cùng yêu cầu khóa đọc (Experiment 4 — Locking reader contention)

Thay vì thực thi truy vấn ghi, luồng tranh chấp yêu cầu khóa thông qua:

```java
setLockTimeout(connection, "300ms");
selectQuota(connection, true); // Yêu cầu khóa SELECT ... FOR UPDATE
```

Luồng lấy khóa cấp dòng (`FOR UPDATE`) sẽ khóa chặn bất kỳ lệnh yêu cầu cấp khóa nào khác đối với cùng một dòng dữ liệu. Luồng sau sẽ phải chờ và phát sinh lỗi `55P03` (lock timeout). Tuy nhiên, nếu luồng thứ hai thay thế bằng `SELECT` thông thường, quá trình truy xuất diễn ra trôi chảy (như Thực nghiệm 2).

## 8. Thực nghiệm 5 — Giải phóng khóa khi Rollback (Experiment 5 — Rollback release lock)

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
            connection.rollback(); // Hủy bỏ
        }
        return null;
    });

    Future<Integer> waiter = executor.submit(() -> {
        await(holderUpdated, "holder update");
        waiterReady.countDown();
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            int affected = updateQuota(connection, 8); // Áp dụng thành công giá trị mới
            connection.commit();
            return affected;
        }
    });

    holder.get(5, TimeUnit.SECONDS);
    assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo(1);
    assertThat(committedQuota()).isEqualTo(8);
}
```

Lệnh `ROLLBACK` không những khôi phục giá trị về trạng thái ban đầu mà còn tháo dỡ hoàn toàn các khóa (lock release). Phiên bản cập nhật nháp (12) bị loại bỏ, và luồng chờ lập tức nhận được tài nguyên để cập nhật.

## 9. Thực nghiệm 6 — Lệnh Update kiểm tra phiên bản sau khi chờ khóa (Experiment 6 — Predicate recheck after holder commit)

Giả định một giao dịch (Luồng A) đã cập nhật giá trị và ghi chú phiên bản (`revision`) lên `6`. Luồng tranh chấp (Luồng B) nỗ lực cập nhật dữ liệu với câu truy vấn có kèm logic phiên bản kiểm chứng:

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

Khi luồng B phải chờ (do A đang khóa), ngay khi A commit, B sẽ đánh giá lại điều kiện truy vấn. Giá trị `revision` hiện tại đã là `6`, nhưng truy vấn của B mong chờ bản ghi có `revision` là `5`. Do đó:

```java
assertThat(waiterAffectedRows).isZero();
assertThat(committedQuota()).isEqualTo(12);
assertThat(committedRevision()).isEqualTo(6);
```

B không tạo lỗi hệ thống mà thay vào đó số lượng bản ghi bị tác động (affected-row) trả về giá trị `0`. Đây là tín hiệu cho phép ứng dụng tiến hành quá trình phục hồi (retry).

## 10. Thực nghiệm 7 — Khóa cấp bảng loại trừ chặn mọi truy vấn (Experiment 7 — `ACCESS EXCLUSIVE` table lock blocks plain SELECT)

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
                selectQuota(connection, false); // Bị chặn chờ timeout
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

Kiểm thử minh họa rằng khóa cấp bảng với mức độ `ACCESS EXCLUSIVE` gây ngừng toàn bộ quá trình đọc trên bảng dữ liệu tương ứng. Không nhầm lẫn hành vi này với khóa cấp dòng.

## 11. Thực nghiệm 8 — Giao diện mã của `PESSIMISTIC_WRITE` (Experiment 8 — Spring `PESSIMISTIC_WRITE`)

Kiểm tra quá trình Spring chuyển đổi yêu cầu từ mã nguồn JPA:

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

Kiểm thử này xác minh Hibernate thực hiện lệnh bảo vệ bằng cú pháp khóa cấp dòng `FOR UPDATE` đúng chuẩn khi tiếp nhận yêu cầu từ `PESSIMISTIC_WRITE`.

## 12. Phương thức tiện ích cốt lõi (Core helpers)

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

## 13. Ma trận kiểm thử (Coverage matrix)

| Kiểm thử (Experiment) | Hành vi Luồng 1 (Holder) | Hành vi Luồng 2 (Contender) | Kết quả mong đợi (Assertion) |
| --- | --- | --- | --- |
| 1 | Truy vấn không khóa (`plain SELECT`) | Lệnh `UPDATE` | Cập nhật được phép vượt lên. Luồng 1 đọc `10 -> 8`. |
| 2 | Truy vấn khóa (`FOR UPDATE`) + `UPDATE` (uncommitted) | Truy vấn hiển thị MVCC (`plain SELECT`) | Luồng đọc thu nhận giá trị đã commit cũ (`10`). |
| 3 | Truy vấn khóa (`FOR UPDATE`) | Lệnh `UPDATE` với giới hạn chờ | Luồng chờ vượt quá thời hạn, trả về `55P03`. |
| 4 | Truy vấn khóa (`FOR UPDATE`) | Truy vấn đòi hỏi khóa (`FOR UPDATE`) | Luồng yêu cầu khóa bị trả về `55P03`. |
| 5 | Truy vấn khóa (`FOR UPDATE`) + Lệnh hủy bỏ (`ROLLBACK`) | Lệnh `UPDATE` | Dữ liệu khôi phục về trạng thái cũ, luồng chờ xử lý thành công. |
| 6 | Truy vấn khóa (`FOR UPDATE`) + Cập nhật và `COMMIT` | Lệnh `UPDATE` có kiểm tra phiên bản | Số lượng dòng cập nhật báo về là `0` (affected-row `0`). |
| 7 | Khóa nguyên bảng (`ACCESS EXCLUSIVE`) | Truy vấn hiển thị (`plain SELECT`) | Tất cả truy vấn, kể cả đọc, bị chặn chờ hoặc trả về `55P03`. |
| 8 | Bọc đối tượng bằng Spring JPA (`PESSIMISTIC_WRITE`) | Truy vấn trực tiếp | Lệnh SQL sinh ra phù hợp để thiết lập khóa đối kháng trên hệ thống thật. |

## 14. Đảm bảo độ tin cậy kiểm thử và Môi trường thực tế (Anti-flaky and production verification)

- Giữ sự cách ly độc lập giữa các kết nối cơ sở dữ liệu và phiên giao dịch (Independent connections/transactions).
- Thiết lập cơ chế gửi tín hiệu điều phối đồng bộ (signals) chỉ sau khi quá trình xác nhận trạng thái khóa hoàn tất (đã cấp hoặc thay đổi).
- Bắt buộc phải gắn cờ `timeout` cho các khối điều phối (latches/futures) để đảm bảo không xảy ra hiện tượng treo không giới hạn.
- Duy trì trạng thái cơ sở dữ liệu làm sạch, trở về cấu hình ban đầu trước khi mỗi hàm kiểm tra được gọi (reset committed state).
- Trích xuất thông tin hệ thống của PostgreSQL (`pg_stat_activity`, `pg_locks`, `pg_blocking_pids`) đối với môi trường thực tế (Production) để quản lý hoặc cấu hình lệnh quá hạn phù hợp.
- Tuyệt đối hạn chế giả lập cơ sở dữ liệu bằng các bộ Database In-Memory (như H2) để kiểm định lỗi khóa, vì chúng không thực thi chính xác mô hình semantic của PostgreSQL.

Các chỉ số giám sát bắt buộc (Metrics):
- Thời gian chờ xin khóa (lock wait duration).
- Các lệnh lỗi giới hạn chờ (`55P03` lock timeout).
- Lỗi khóa chéo ngược chiều (deadlock `40P01`).
- Thời gian giao dịch duy trì khóa (transaction age).
- Các phiên bị đình trệ ở trạng thái chờ không giải quyết (idle in transaction).
