# Kiểm Thử Tầm Nhìn Dữ Liệu Tương Tranh (Deterministic visibility experiments)

## 1. Mục tiêu kiểm thử

Bộ bài kiểm thử (Test suite) này được thiết kế để xác thực cơ chế xử lý Đọc Bẩn (Dirty Read) của PostgreSQL, qua đó chứng minh các nguyên lý sau:

1. Việc thực thi lệnh UPDATE và xả (flush) dữ liệu nhưng trì hoãn việc commit sẽ tạo ra dữ liệu chưa hoàn chỉnh (uncommitted data).
2. Lệnh đọc thông thường (plain SELECT) dù được cấu hình ở mức `READ_UNCOMMITTED` vẫn không bị chặn lại (non-blocking), và sẽ bỏ qua các khóa (locks) đang chờ xử lý.
3. Luồng đọc (Reader) chỉ nhận được kết quả đã commit trước đó (ví dụ: `20`), tuyệt đối không đọc được giá trị chưa commit (`80`).
4. Nếu luồng ghi (Writer) hủy bỏ (rollback), giá trị vĩnh viễn là `20`. Nếu luồng ghi chốt sổ (commit), luồng đọc khởi tạo sau đó sẽ thấy giá trị mới (`80`).
5. Giá trị cấu hình cách ly (isolation label) do CSDL phản hồi không đại diện cho hành vi Đọc Bẩn thực tế.
6. Các giải pháp thay thế như cơ chế Nhịp Tim (Heartbeat) và Bảng Trạng Thái phải chứng minh được hiệu năng và khả năng trích xuất Dữ Liệu Đã Chốt.

> **Nói ngắn gọn:** Cách chuẩn xác nhất để kiểm tra độ tin cậy của dữ liệu là sử dụng rào chắn (Barrier) tạm giữ luồng ghi, tạo một luồng đọc độc lập lấy kết quả, sau đó mới cho phép luồng ghi kết thúc (commit hoặc rollback) để xác thực tính bất biến của luồng đọc.

## 2. Cấu hình môi trường PostgreSQL Testcontainers

Các bài kiểm thử tương tranh phải chạy trên PostgreSQL thực tế, tuyệt đối không dùng CSDL nhúng giả lập như H2:

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class DirtyReadExpectationIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Lưu ý quan trọng: Các phương thức test không được phép đính kèm annotation `@Transactional`. Các tác nhân mô phỏng (Writer, Reader, Inspector) phải là những Bean độc lập để kiểm soát giao dịch riêng biệt qua kết nối CSDL (connections).

Cấu trúc Bảng:

```sql
create table job_run (
    job_id uuid primary key,
    status varchar(32) not null,
    progress_percent integer not null
        check (progress_percent between 0 and 100),
    generation bigint not null,
    owner_token uuid,
    lease_until timestamptz
);

create table job_attempt_heartbeat (
    -- Bảng theo dõi tiến độ riêng biệt
    job_id uuid not null,
    generation bigint not null,
    owner_token uuid not null,
    progress_percent integer not null
        check (progress_percent between 0 and 100),
    last_seen_at timestamptz not null,
    primary key (job_id, generation)
);
```

Dữ liệu mô phỏng ban đầu:

```sql
insert into job_run(
    job_id, status, progress_percent, generation
)
values (:jobId, 'RUNNING', 20, 7);
```

## 3. Rào chắn đồng bộ luồng xử lý (Writer gate)

Sử dụng `CountDownLatch` trong Java để tạm dừng tiến trình ghi ngay sau khi dữ liệu đã được đẩy xuống CSDL (flush):

```java
final class WriterGate {
    private final CountDownLatch flushed = new CountDownLatch(1);
    private final CountDownLatch allowCompletion =
        new CountDownLatch(1);

    // Luồng ghi tạm dừng tại đây sau khi flush
    void afterFlush() {
        flushed.countDown(); // Phát tín hiệu đã hoàn thành flush
        awaitOrFail(allowCompletion, Duration.ofSeconds(5)); // Chờ lệnh đi tiếp
    }

    // Các luồng kiểm thử chờ tín hiệu flush
    void awaitFlushed() {
        awaitOrFail(flushed, Duration.ofSeconds(5));
    }

    // Luồng kiểm thử cấp phép hoàn tất giao dịch
    void release() {
        allowCompletion.countDown();
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for writer gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

Trong khối mã thực thi nghiệp vụ, hãy gọi `entityManager.flush()` trước hàm `gate.afterFlush()`. Cần đảm bảo luôn giải phóng luồng (`release()`) trong khối `finally` để tránh việc luồng (Thread) bị khóa vĩnh viễn.

## 4. Công cụ theo dõi giao dịch đọc (Reader observation)

```java
public record ReadObservation(
    int progress,
    String reportedIsolation,
    int jdbcIsolation
) {}
```

```java
@Transactional(
    isolation = Isolation.READ_UNCOMMITTED,
    readOnly = true
)
public ReadObservation observe(UUID jobId) {
    // 1. Kiểm tra cấu hình Isolation của JDBC
    int jdbcIsolation = jdbc.execute(
        (ConnectionCallback<Integer>)
            Connection::getTransactionIsolation
    );
    // 2. Kiểm tra cấu hình Isolation phản hồi từ PostgreSQL
    String reported = jdbc.queryForObject(
        "select current_setting('transaction_isolation')",
        String.class
    );
    // 3. Thực thi kiểm tra dữ liệu thật sự: Đọc giá trị tiến độ!
    int progress = jdbc.queryForObject(
        """
        select progress_percent
        from job_run
        where job_id = ?
        """,
        Integer.class,
        jobId
    );
    return new ReadObservation(
        progress,
        reported,
        jdbcIsolation
    );
}
```

Các phản hồi về `reportedIsolation` có thể khác nhau tùy phiên bản (read committed hay uncommitted), nhưng để kiểm chứng tính đúng đắn (correctness), hệ thống chỉ được phép tin tưởng vào kết quả lấy được tại thuộc tính `progress`.

## 5. Thí nghiệm 1 — Yêu cầu Đọc Bẩn nhưng không trả về dữ liệu bẩn

```java
@Test
void readUncommittedRequestStillCannotSeeDirtyRow()
        throws Exception {
    WriterGate gate = writerProbe.install();
    ExecutorService actor = Executors.newSingleThreadExecutor();

    try {
        // 1. Luồng xử lý chính được chỉ định gặp lỗi và rollback
        Future<Throwable> writerOutcome = actor.submit(() ->
            catchThrowable(() ->
                processor.processCurrentUnit(
                    JOB_ID,
                    true // Kích hoạt biến lỗi failAfterFlush
                )
            )
        );

        // 2. Chờ luồng xử lý thực hiện flush
        gate.awaitFlushed();

        // 3. Khởi tạo truy vấn giám sát dữ liệu
        ReadObservation observed = watchdog.observe(JOB_ID);

        // KẾT QUẢ: Dù cấu hình READ_UNCOMMITTED, vẫn nhận giá trị cũ 20
        assertThat(observed.progress()).isEqualTo(20);
        // Cấu hình Isolation có thể hiển thị dưới nhiều nhãn, nhưng giá trị là không đổi
        assertThat(observed.reportedIsolation())
            .isIn("read uncommitted", "read committed");
        assertThat(observed.jdbcIsolation()).isIn(
            Connection.TRANSACTION_READ_UNCOMMITTED,
            Connection.TRANSACTION_READ_COMMITTED
        );

        // 4. Giải phóng cổng luồng để luồng A nhận Rollback exception
        gate.release();
        Throwable failure =
            writerOutcome.get(5, TimeUnit.SECONDS);
        assertThat(failure)
            .isInstanceOf(ProcessingFailedException.class);

        // Xác minh trạng thái cuối cùng
        assertThat(inspector.progress(JOB_ID)).isEqualTo(20);
    } finally {
        gate.release();
        writerProbe.clear();
        actor.shutdownNow();
        assertThat(actor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Luồng Watchdog hoàn thành quá trình truy vấn độc lập trong khi rào chắn của luồng Writer đang được đóng, chứng tỏ truy vấn không bị chặn (non-blocking) và luôn trả về dữ liệu đã commit (20).

## 6. Thí nghiệm 2 — Chỉ hiển thị sau khi hoàn tất Commit (Before commit old, after commit new)

Nếu quá trình ghi hoàn tất thành công (commit) mà không gặp lỗi:

```java
@Test
void laterStatementSeesValueOnlyAfterWriterCommits()
        throws Exception {
    WriterGate gate = writerProbe.install();
    ExecutorService actor = Executors.newSingleThreadExecutor();

    try {
        Future<Void> writer = actor.submit(() -> {
            processor.processCurrentUnit(JOB_ID, false);
            return null;
        });

        gate.awaitFlushed();
        assertThat(watchdog.observe(JOB_ID).progress())
            .isEqualTo(20);

        gate.release();
        writer.get(5, TimeUnit.SECONDS);

        assertThat(watchdog.observe(JOB_ID).progress())
            .isEqualTo(80);
    } finally {
        gate.release();
        writerProbe.clear();
        actor.shutdownNow();
        assertThat(actor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Truy vấn thứ hai của Watchdog sẽ thấy Bức Ảnh Chụp (Snapshot) mới sau khi Processor hoàn tất quá trình commit, đáp ứng nguyên lý (normal committed visibility) thay vì Đọc Bẩn.

## 7. Thí nghiệm 3 — Khả năng vô hình của Dữ liệu chèn mới (INSERT)

Nguyên tắc Đọc Bẩn áp dụng đồng thời trên tất cả thao tác CSDL. Đối với lệnh chèn dữ liệu:

```sql
insert into job_run(
    job_id, status, progress_percent, generation
)
values (:newJob, 'RUNNING', 1, 1);
-- Gọi flush, không commit
```

Kết quả truy vấn giám sát:

```java
assertThat(watchdog.find(NEW_JOB_ID)).isEmpty();
```

Trường hợp này thể hiện nguyên tắc loại bỏ ảo tưởng thiết kế khi ứng dụng chỉ kiểm tra các thay đổi trên "Dòng Cũ" nhưng bỏ sót dữ liệu chèn "Dòng Mới". Toàn bộ dữ liệu chỉ tồn tại trong Tầm Nhìn Dữ Liệu sau khi commit thành công.

## 8. Thí nghiệm 4 — Truy vấn kèm Khóa (FOR UPDATE) tạo trạng thái chờ

Nếu Watchdog sử dụng cấu trúc truy vấn kèm khóa bảo vệ:

```java
@Transactional
public JobSnapshot lockAndRead(UUID jobId) {
    return jobs.findByIdForUpdate(jobId)
        .orElseThrow()
        .snapshot();
}
```

Kiểm tra số lượng truy vấn đang trong trạng thái chờ khóa:

```sql
select count(*)
from pg_stat_activity
where datname = current_database()
  and wait_event_type = 'Lock';
```

KẾT QUẢ: Hệ thống báo cáo có ít nhất một giao dịch đang chờ cấp phép khóa (`waiter count >= 1`). 
Luồng Watchdog sẽ bị chặn lại cho đến khi quá trình ghi hoàn tất (commit hoặc rollback), sau đó lấy trạng thái cuối cùng (outcome) làm kết quả. Chờ đợi khóa không mang ý nghĩa của thao tác Đọc Bẩn (dirty visibility), vì vậy không nên áp dụng giải pháp này cho các logic cập nhật tiến độ liên tục (Polling) trên giao diện.

## 9. Thí nghiệm 5 — Khả năng hiển thị của cơ chế Nhịp Tim tách biệt (Independent heartbeat visible)

Cơ chế cập nhật trạng thái nhỏ (heartbeat) có thể tách biệt khỏi giao dịch lớn nếu áp dụng phương thức tạo giao dịch nguyên bản độc lập (`REQUIRES_NEW`):

```java
heartbeatPublisher.publish(new Heartbeat(
    JOB_ID,
    7,
    OWNER_TOKEN,
    80,
    TEST_NOW
));
```

Trạng thái hệ thống trong thời gian giao dịch chính đang trì hoãn:

```java
// Bảng nghiệp vụ trung tâm duy trì trạng thái cũ
assertThat(watchdog.observe(JOB_ID).progress()).isEqualTo(20);
// Nhịp tim được cập nhật và hiển thị thành công
assertThat(heartbeatReader.read(JOB_ID, 7).progressPercent())
    .isEqualTo(80);
```

Hệ quả khi giao dịch chính bị gián đoạn (Rollback):

```java
assertThat(inspector.progress(JOB_ID)).isEqualTo(20);
assertThat(heartbeatReader.read(JOB_ID, 7).progressPercent())
    .isEqualTo(80);
```

Bài học: Cập nhật nhịp tim tiến độ "Đang xử lý" là một thiết kế hệ thống tốt, tuy nhiên cần được cấp phát môi trường dữ liệu và quản lý giao dịch tách biệt khỏi kết quả nghiệp vụ cuối cùng.

## 10. Thí nghiệm 6 — Lỗi cấu trúc khóa đệ quy (`REQUIRES_NEW` Self-deadlock)

Nếu một phương thức đang giữ khóa cấp dòng thông qua Giao dịch Bên Ngoài (Outer Transaction), việc gọi thêm Giao dịch Độc Lập (`REQUIRES_NEW`) trỏ vào chính Dòng dữ liệu đó sẽ sinh ra lỗi vòng lặp khóa khép kín (Deadlock):

```text
Giao Dịch Bên Ngoài chiếm giữ Khóa Dòng.
Giao Dịch Độc Lập chờ Giao Dịch Bên Ngoài trả khóa.
Giao Dịch Bên Ngoài chờ Giao Dịch Độc Lập hoàn tất để chạy tiếp.
Kết quả: Quá trình Thread bị treo và gây Deadlock.
```

Hệ thống sẽ bị ngắt kết nối mà không cần chịu ảnh hưởng bởi máy chủ khác. Đây là lý do kiến trúc lưu trữ Nhịp Tim cần có định dạng bảng dữ liệu riêng.

## 11. Thí nghiệm 7 — Giành quyền giải cứu đồng thời (Atomic recovery claim)

Các luồng Watchdog tham gia giải cứu sẽ thực thi lệnh cấp phát điều kiện nguyên tử:

```sql
update job_run
set generation = generation + 1,
    owner_token = :owner,
    lease_until = :leaseUntil
where job_id = :jobId
  and generation = 7
  and status = 'RUNNING'
  and lease_until < :databaseNow -- Kiểm tra hết hạn sử dụng
returning generation, owner_token;
```

Xác thực tính độc quyền cấp phát:

```java
assertThat(results.successCount()).isEqualTo(1); // 1 Yêu cầu cấp phép
assertThat(results.noOpCount()).isEqualTo(1); // 1 Yêu cầu bị loại
assertThat(inspector.generation(JOB_ID)).isEqualTo(8);
assertThat(inspector.ownerToken(JOB_ID))
    .isEqualTo(results.winnerToken());
```

Cấu trúc yêu cầu nguyên tử (atomic write) giúp quản lý an toàn trong hệ thống phân tán, xử lý triệt để bài toán khôi phục.

## 12. Thí nghiệm 8 — Độ tin cậy của cấu hình Isolation (Label test)

Kịch bản ghi chú nhãn cô lập (Isolation label):

```text
PostgreSQL server version
pgJDBC version
requested Spring isolation
Connection.getTransactionIsolation()
current_setting('transaction_isolation')
observed progress during uncommitted writer
```

Xác định tính chính xác của chương trình bắt buộc phải dựa vào thao tác truy vấn kiểm chứng Tầm Nhìn Dữ Liệu (visibility correctness). Cấu hình chuỗi Label không mang tính quyết định hành vi thiết kế nghiệp vụ, và không được dùng làm bằng chứng để loại bỏ các kiểm thử chức năng.

## 13. Công cụ Thanh Tra Tách Biệt (Inspector)

```java
@Service
class CommittedJobInspector {
    private final JdbcTemplate jdbc;

    // Đóng vai trò giám sát khách quan thông qua REQUIRES_NEW
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    int progress(UUID jobId) {
        return jdbc.queryForObject(
            """
            select progress_percent
            from job_run
            where job_id = ?
            """,
            Integer.class,
            jobId
        );
    }
}
```

Phương thức độc lập xác minh Trạng thái Chốt Cuối Cùng bằng việc thiết lập hoàn toàn một Giao Dịch Mới (Tách Biệt)

## 14. Tổng hợp mức độ bao phủ (Coverage matrix)

| Sự Kiện / Hành Vi | Trạng Thái Thiết Lập Truy Vấn Đọc | Giá Trị Tiến Độ |
| --- | --- | --- |
| Đọc Bẩn (RU), Giao dịch Ghi đang chờ | Thực thi khi Ghi chưa hoàn thành | 20 (Phiên bản Cũ) |
| Đọc Bẩn (RU), Giao dịch Ghi Rollback | Lấy số liệu sau Hủy giao dịch | 20 (Duy trì nguyên vẹn) |
| Đọc Bẩn (RU), Giao dịch Ghi Commit | Khởi động truy vấn ngay sau Commit | 80 (Hiển thị phiên bản mới) |
| Đọc dữ liệu mới (INSERT) | Đọc khi Ghi chưa hoàn thành | Không Dữ Liệu (Rỗng) |
| Đọc có Khóa `FOR UPDATE`, Ghi Commit | Chờ đến khi Mở Khóa hoàn tất | 80 |
| Đọc có Khóa `FOR UPDATE`, Ghi Rollback| Chờ đến khi Mở Khóa hoàn tất | 20 |
| Cập nhật Nhịp Tim Độc Lập | Kích hoạt khi Bảng Chính vẫn giữ giao dịch | Bảng Nhịp Tim (80), Bảng Chính (20) |

## 15. Kinh nghiệm xử lý kiểm thử chập chờn (Anti-Flaky)

- Sử dụng rào chắn đồng bộ luồng (Writer flush gate) để điều phối chu trình thứ tự.
- Chức năng kiểm thử yêu cầu Hẹn giờ Timeout (Latch/Future) để chặn đứng sự cố lặp không hồi kết.
- Yêu cầu giải phóng `release()` tại lệnh `finally`.
- Các thử nghiệm xác minh Dữ liệu Yêu cầu CSDL thật thay vì H2 giả lập.
- Thẩm định logic phụ thuộc trên kết quả trả về của Dữ Liệu thay cho Nhãn.
- Cấu hình Timeout và trích xuất dữ liệu Admin Diagnostic (`pg_stat_activity`) hỗ trợ giải quyết Đọc Khóa.

## 16. Chẩn đoán hệ thống Thực Tế (Production verification)

Các thông số cần thu thập, đối chiếu cho môi trường thực tế:

- Biểu đồ Phân phối tuổi thọ Giao dịch (Transaction Lifetime Distribution).
- Dữ liệu định danh Nhịp Tim mới nhất (Heartbeat Recency).
- Số lượng cấp quyền Khôi Phục (Recovery Claim Rate).
- Số lần từ chối Cập Nhật (Generation Rejection Count).
- Tỷ lệ Giao dịch Thành công / Rollback.

Trong trường hợp phát sinh thắc mắc tại sao hệ thống chỉ đọc được các thông tin trễ, vui lòng dựa vào lý luận MVCC để bảo vệ quan điểm phát triển. Việc phản hồi dữ liệu muộn hơn giao dịch là tính năng nguyên lý hiển nhiên, không phải là sự cố đứt kết nối mạng hay hỏng CSDL (Replication lag/Corruption).
