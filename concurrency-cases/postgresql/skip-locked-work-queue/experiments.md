# Các thí nghiệm `SKIP LOCKED` với PostgreSQL thật

## Mục tiêu

- Dùng lệnh `SELECT` thông thường cho thấy hai tiến trình có thể lấy cùng một ID công việc.
- Kiểm chứng `FOR UPDATE` không có `SKIP LOCKED` sẽ tạo ra tắc nghẽn một chiều.
- Kiểm chứng `SKIP LOCKED` thực sự bỏ qua dòng J1 đang bị khóa và chuyển sang lấy J2.
- Việc lấy một lô công việc song song không bao giờ bị giao nhau.
- Khi có sự cố, các khóa lấy việc sẽ tự động được giải phóng.
- Các công việc hết hạn thuê có thể được lấy lại để xử lý tiếp; một tiến trình quá hạn nếu báo hoàn thành sẽ nhận về số dòng bị ảnh hưởng là `0`.
- Đảm bảo tính nhất quán qua các yếu tố: quyền sở hữu, trạng thái, số lần thử và tính lũy đẳng.

> **Nói ngắn gọn:** Các kịch bản kiểm thử phải chứng minh được việc phân tách quyền sở hữu độc lập và khả năng phục hồi sau sự cố, chứ không chỉ đơn thuần là kiểm tra xem câu truy vấn có chứa chuỗi `SKIP LOCKED` hay không.

## PostgreSQL Testcontainers fixture

```java
@Testcontainers
class SkipLockedQueueIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("queue_cases")
                    .withUsername("cases")
                    .withPassword("cases");

    private final ExecutorService executor = Executors.newFixedThreadPool(4);

    @BeforeAll
    static void schema() throws SQLException {
        try (Connection connection = open();
                Statement sql = connection.createStatement()) {
            sql.execute("""
                    create table work_job (
                        job_id uuid primary key,
                        payload jsonb not null,
                        status varchar(16) not null,
                        priority integer not null,
                        available_at timestamptz not null,
                        claim_token uuid,
                        claimed_by varchar(128),
                        lease_until timestamptz,
                        attempt_count integer not null default 0,
                        completed_at timestamptz
                    )
                    """);
            sql.execute("""
                    create index ix_work_job_claim
                    on work_job(priority desc, available_at, job_id)
                    where status = 'READY'
                    """);
        }
    }

    @BeforeEach
    void seed() throws SQLException {
        try (Connection connection = open();
                Statement sql = connection.createStatement()) {
            sql.execute("truncate work_job");
            for (int i = 1; i <= 8; i++) {
                try (PreparedStatement insert = connection.prepareStatement("""
                        insert into work_job(
                          job_id, payload, status, priority, available_at
                        ) values (?, '{}'::jsonb, 'READY', 0, ?)
                        """)) {
                    insert.setObject(1, jobId(i));
                    insert.setTimestamp(
                            2,
                            Timestamp.from(Instant.parse(
                                    "2026-01-01T00:00:00Z"
                            ).plusSeconds(i))
                    );
                    insert.executeUpdate();
                }
            }
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

## Helper điều phối

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
```

Tất cả các kết nối đều phải đặt `statement_timeout`. Mã kiểm thử dùng `Future.get(timeout)` làm hàng rào bảo vệ bên ngoài để ngăn test bị treo.

## Thí nghiệm 1 — Lệnh SELECT thông thường trả về các ID trùng lặp

```java
@Test
void plainSelectDoesNotClaim() throws Exception {
    var bothSelected = new CountDownLatch(2);
    var returnRows = new CountDownLatch(1);

    Callable<List<UUID>> select = () -> {
        try (Connection connection = open()) {
            List<UUID> ids = selectReady(connection, 2, "");
            bothSelected.countDown();
            await(returnRows, "return duplicate selections");
            return ids;
        }
    };

    Future<List<UUID>> a = executor.submit(select);
    Future<List<UUID>> b = executor.submit(select);
    await(bothSelected, "both plain selects");
    returnRows.countDown();

    assertThat(a.get(5, TimeUnit.SECONDS))
            .containsExactlyElementsOf(b.get(5, TimeUnit.SECONDS));
    assertThat(readyCount()).isEqualTo(8);
}
```

## Thí nghiệm 2 — `FOR UPDATE` gây ra tắc nghẽn

Chúng ta dùng một transaction giữ khóa công việc J1 (Holder). Sau đó một tiến trình khác (Waiter) chạy truy vấn `LIMIT 1 FOR UPDATE` có sắp xếp và kèm theo `lock_timeout='200ms'`:

```java
assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
assertThat(readyCount()).isEqualTo(8);
releaseHolder.countDown();
holder.get(5, TimeUnit.SECONDS);
```

Biểu đồ chờ khóa lúc này chỉ có Waiter đang chờ Holder, không hề có vòng lặp nào. Do đó, đây không phải là lỗi deadlock, mà đơn thuần là sự tắc nghẽn chờ tài nguyên.

## Thí nghiệm 3 — `SKIP LOCKED` lấy dòng kế tiếp

```java
@Test
void skipLockedPassesLockedFirstJob() throws Exception {
    var firstLocked = new CountDownLatch(1);
    var release = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = open()) {
            connection.setAutoCommit(false);
            lockJob(connection, jobId(1));
            firstLocked.countDown();
            await(release, "release J1");
            connection.rollback();
        }
        return null;
    });

    await(firstLocked, "J1 locked");
    try (Connection worker = open()) {
        worker.setAutoCommit(false);
        List<UUID> ids = selectReady(
                worker,
                1,
                "for update skip locked"
        );
        assertThat(ids).containsExactly(jobId(2));
        worker.rollback();
    } finally {
        release.countDown();
    }
    holder.get(5, TimeUnit.SECONDS);
}
```

## Thí nghiệm 4 — Hai lệnh lấy việc nguyên tử tạo ra các lô độc lập

```java
@Test
void concurrentClaimBatchesNeverOverlap() throws Exception {
    var start = new CountDownLatch(1);

    Future<List<ClaimedJob>> a = executor.submit(() -> {
        await(start, "common claim start");
        return claimService.claimBatch("A", 3, Duration.ofMinutes(1));
    });
    Future<List<ClaimedJob>> b = executor.submit(() -> {
        await(start, "common claim start");
        return claimService.claimBatch("B", 3, Duration.ofMinutes(1));
    });

    start.countDown();
    List<ClaimedJob> first = a.get(8, TimeUnit.SECONDS);
    List<ClaimedJob> second = b.get(8, TimeUnit.SECONDS);

    assertThat(first).hasSize(3);
    assertThat(second).hasSize(3);
    assertThat(first).extracting(ClaimedJob::jobId)
            .doesNotContainAnyElementsOf(
                    second.stream().map(ClaimedJob::jobId).toList()
            );
    assertThat(Stream.concat(first.stream(), second.stream())
            .map(ClaimedJob::jobId)
            .distinct()).hasSize(6);
    assertThat(processingCount()).isEqualTo(6);
    assertThat(readyCount()).isEqualTo(2);
    assertThat(allClaimTokens()).doesNotContainNull().doesNotHaveDuplicates();
}
```

Phía gọi thông qua Service bắt buộc phải đi qua proxy, đảm bảo rằng lô công việc trả về chỉ có thể được sử dụng SAU KHI transaction đã được commit.

## Lấy việc nguyên tử thông qua SQL test

Sử dụng JDBC thuần túy để ánh xạ lệnh `UPDATE RETURNING`:

```java
static List<ClaimedJob> claim(
        Connection connection,
        String worker,
        int limit
) throws SQLException {
    connection.setAutoCommit(false);
    try (PreparedStatement sql = connection.prepareStatement("""
            with candidates as (
              select job_id
              from work_job
              where status = 'READY'
                and available_at <= clock_timestamp()
              order by priority desc, available_at, job_id
              for update skip locked
              limit ?
            )
            update work_job j
            set status='PROCESSING',
                claim_token=gen_random_uuid(),
                claimed_by=?,
                lease_until=clock_timestamp() + interval '1 minute',
                attempt_count=attempt_count+1
            from candidates c
            where j.job_id=c.job_id
            returning j.job_id, j.claim_token, j.lease_until,
                      j.attempt_count
            """)) {
        sql.setInt(1, limit);
        sql.setString(2, worker);
        List<ClaimedJob> result = mapRows(sql.executeQuery());
        connection.commit();
        return result;
    } catch (SQLException failure) {
        connection.rollback();
        throw failure;
    }
}
```

## Thí nghiệm 5 — Rollback làm công việc có thể lấy lại

Một kết nối thực thi câu lệnh SQL lấy việc nhưng sau đó lại gọi `rollback` thay vì `commit`. Ngay sau đó, một kết nối thứ hai tiến hành lấy việc:

```java
assertThat(firstUncommittedIds).containsExactly(jobId(1));
assertThat(secondCommittedIds).containsExactly(jobId(1));
assertThat(job(jobId(1)).status()).isEqualTo("PROCESSING");
assertThat(job(jobId(1)).attemptCount()).isEqualTo(1);
```

Lưu ý rằng số lần thử của lần lấy việc bị rollback xem như không hề tồn tại. Nếu bạn đóng kết nối mà không commit, kết quả dọn dẹp hệ thống cũng sẽ hoàn toàn giống như vậy.

## Thí nghiệm 6 — Hoàn thành trễ bị từ chối

```text
Tiến trình A lấy J1, nhận token-A
Test can thiệp chỉnh sửa lease_until lùi về quá khứ
Tiến trình quét trả J1 lại hàng đợi
Tiến trình B lấy J1, nhận token-B
Tiến trình A gọi complete(token-A) → trả về 0 (thất bại)
Tiến trình B gọi complete(token-B) → trả về 1 (thành công)
```

Xác nhận qua mã kiểm thử:

```java
assertThat(complete(jobId(1), tokenA)).isZero();
assertThat(complete(jobId(1), tokenB)).isEqualTo(1);
assertThat(job(jobId(1)).status()).isEqualTo("DONE");
assertThat(job(jobId(1)).attemptCount()).isEqualTo(2);
```

Chúng ta không dùng lệnh `sleep` của hệ thống để đợi hết hạn thuê; thay vào đó, mã kiểm thử sẽ chủ động cập nhật `lease_until` một cách có kiểm soát, hoặc dùng cơ chế giả lập đồng hồ database.

## Thí nghiệm 7 — Tác động bên ngoài lũy đẳng sau sự cố

Một điểm đến giả lập lưu giữ một khóa duy nhất `effect_key = job_id`:

```text
Tiến trình B lấy J1 → gọi sink apply(J1) → giả lập sự cố trước khi báo hoàn thành
Hết hạn thuê/Bị lấy lại → Tiến trình C lấy J1 → gọi sink apply(J1) thêm lần nữa
Tiến trình C báo hoàn thành thành công với thẻ định danh hiện tại
```

Xác nhận:

```java
assertThat(sink.deliveryAttempts(jobId(1))).isEqualTo(2);
assertThat(sink.committedEffects(jobId(1))).isEqualTo(1);
assertThat(job(jobId(1)).status()).isEqualTo("DONE");
```

Điểm đến giả lập phải thiết kế mô phỏng được việc kiểm tra lấy việc độc nhất; việc chỉ dùng một danh sách trên bộ nhớ không được đồng bộ sẽ không đủ để chứng minh tính lũy đẳng.

## Thí nghiệm 8 — Khóa bảng vẫn có thể chặn

Mở một phiên giao dịch thực hiện DDL:

```sql
lock table work_job in access exclusive mode;
```

Dù tiến trình gọi `FOR UPDATE SKIP LOCKED`, lệnh này vẫn yêu cầu khóa `ROW SHARE` trên toàn bảng. Do đó, truy vấn vẫn sẽ bị chặn và báo lỗi `55P03` khi hết giới hạn `lock_timeout`. Bài kiểm thử này làm rõ một chi tiết quan trọng: cơ chế bỏ qua chỉ có tác dụng đối với các khóa ở cấp độ dòng mà thôi.

## Thí nghiệm 9 — Sự công bằng và tình trạng chết đói

Chúng ta thực hiện hai kịch bản kiểm thử riêng biệt:

1. Khi không có tắc nghẽn: Lấy việc từng lô một và đảm bảo thứ tự lấy ra luôn tuân theo `priority DESC, available_at, job_id`.
2. Khi một transaction cố tình giữ dòng cũ nhất: Nhiều lần lấy việc liên tiếp sẽ đành phải lấy các công việc mới hơn; công việc cũ nhất sẽ không thể xuất hiện trong kết quả. Tuy nhiên, ngay sau khi nhả khóa hoặc rollback, lần lấy việc tiếp theo BẮT BUỘC phải lấy được công việc cũ nhất đó.

Chúng ta tuyệt đối không mong chờ thứ tự vào trước ra trước một cách cứng nhắc khi hệ thống đang tắc nghẽn. Thay vào đó, trong các bài test hiệu năng, bạn nên thêm các độ ưu tiên khác nhau, ghi nhận lại độ tuổi của công việc cũ nhất và phải đảm bảo rằng trong thời gian giới hạn, mọi công việc cuối cùng đều sẽ được hoàn thành hoặc hủy bỏ.

## Core helpers

```java
static List<UUID> selectReady(
        Connection connection,
        int limit,
        String lockingClause
) throws SQLException {
    String sqlText = """
            select job_id
            from work_job
            where status='READY'
              and available_at <= clock_timestamp()
            order by priority desc, available_at, job_id
            limit ?
            """ + lockingClause;
    try (PreparedStatement sql = connection.prepareStatement(sqlText)) {
        sql.setInt(1, limit);
        try (ResultSet rows = sql.executeQuery()) {
            var ids = new ArrayList<UUID>();
            while (rows.next()) {
                ids.add(rows.getObject(1, UUID.class));
            }
            return List.copyOf(ids);
        }
    }
}
```

Lưu ý rằng mã trên môi trường thực tế tuyệt đối không được phép ghép chuỗi SQL từ đầu vào của người dùng. Hàm trợ giúp này chỉ dùng riêng cho các tham số hằng số của môi trường kiểm thử.

## Ma trận bao phủ

| Thí nghiệm | Cơ chế kiểm tra | Yêu cầu logic |
| --- | --- | --- |
| 1 | Lỗi SELECT thông thường | Trả về ID trùng lặp, trạng thái vẫn là READY |
| 2 | Khóa dòng chặn luồng | Phát sinh `55P03`, lock convoy không phải deadlock |
| 3 | Lách khóa `SKIP LOCKED` | Bỏ qua J1 đang khóa, tiến trình lấy ngay J2 |
| 4 | Lấy việc song song nguyên tử | Nhận về hai lô riêng biệt, thẻ định danh độc nhất |
| 5 | Transaction bị Rollback/Đóng kết nối | Cùng công việc vẫn có thể được lấy lại, số lần thử không tăng |
| 6 | Thẻ định danh và thời hạn thuê | Hoàn thành trễ báo `0`, chủ sở hữu mới báo `1` |
| 7 | Điểm đến lũy đẳng | Hai lần thử giao, nhưng chỉ ghi một kết quả |
| 8 | Khóa độc quyền `ACCESS EXCLUSIVE` | Lách khóa không thể vượt qua khóa bảng |
| 9 | Thứ tự / Chống chết đói | Duy trì tính ưu tiên ổn định, sau khi nhả khóa sẽ lấy được việc |

## Chống flaky và xác minh production

- Đặt rào chắn ngay sau điểm khóa/đọc dữ liệu chính xác.
- Đặt rào bảo vệ theo thứ tự `lock_timeout < statement_timeout < Future`.
- Tuyệt đối không dùng thời gian ngủ cố định; mọi thao tác chờ đều phải có giới hạn.
- Đảm bảo thực hiện rollback, đóng kết nối và tắt tiến trình thực thi trong phần dọn dẹp hệ thống.
- Nếu gặp lỗi hết giờ, phải tự động thu thập `pg_stat_activity`, `pg_locks`, `pg_blocking_pids` để tiện gỡ lỗi.
- Liên tục theo dõi độ trễ lấy việc, tần suất truy vấn rỗng, độ tuổi lớn nhất của trạng thái READY, số lần quá hạn thuê, số lần lấy lại công việc, số lần hoàn thành trễ, tỷ lệ bị DEAD và tính năng chống trùng lặp của hệ thống phía sau.
- Kế hoạch thực thi truy vấn và index bắt buộc phải được thử nghiệm với khối lượng dữ liệu gần với môi trường production; tuyệt đối không lấy chỉ số benchmark từ môi trường Testcontainers trên máy tính cá nhân!
