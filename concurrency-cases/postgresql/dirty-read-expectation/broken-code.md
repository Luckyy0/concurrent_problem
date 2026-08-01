# Đoạn Mã Lỗi: Ảo Tưởng Về Đọc Bẩn (Broken dirty-read design)

## 1. Dữ liệu trạng thái công việc (Job state)

```java
@Entity
@Table(name = "job_run")
public class JobRun {
    @Id
    private UUID jobId;

    @Enumerated(EnumType.STRING)
    private JobStatus status;

    private int progressPercent;
    private long generation;

    protected JobRun() {
    }

    public void reportProgress(int progressPercent) {
        if (status != JobStatus.RUNNING) {
            throw new IllegalStateException("Tiến trình không ở trạng thái RUNNING");
        }
        if (
            progressPercent < this.progressPercent
                || progressPercent > 100
        ) {
            throw new IllegalArgumentException(
                "Tiến độ phải tăng dần (monotonic) và không vượt quá 100"
            );
        }
        this.progressPercent = progressPercent;
    }

    // Bóc tách trạng thái hiện tại thành Data Transfer Object (DTO)
    public JobSnapshot snapshot() {
        return new JobSnapshot(
            jobId,
            status,
            progressPercent,
            generation
        );
    }
}
```

## 2. Dịch vụ xử lý giữ giao dịch lâu (Processor transaction)

```java
@Service
public class JobProcessor {
    private final JobRunRepository jobs;
    private final ProcessingProbe probe;

    public JobProcessor(
        JobRunRepository jobs,
        ProcessingProbe probe
    ) {
        this.jobs = jobs;
        this.probe = probe;
    }

    @Transactional
    public void processCurrentUnit(
        UUID jobId,
        boolean failAfterFlush
    ) {
        JobRun job = jobs.findById(jobId).orElseThrow();
        
        // 1. Cập nhật tiến độ lên 80% (mới chỉ cập nhật trên Entity)
        job.reportProgress(80);

        // 2. Ép Hibernate tạo lệnh SQL UPDATE và gửi xuống CSDL (Nhưng chưa commit)
        jobs.flush();
        probe.afterProgressFlushed(jobId); // Hook dùng để hỗ trợ kiểm thử

        // 3. Giả lập một lỗi ngoại lệ hoặc quá trình xử lý I/O kéo dài
        if (failAfterFlush) {
            throw new ProcessingFailedException(jobId);
        }

        // Thực hiện các thao tác xử lý nghiệp vụ dài hạn khác tại đây...
    }
}
```

Đối tượng `ProcessingProbe` đóng vai trò như một cơ chế đồng bộ trong quá trình kiểm thử (test barrier) để kiểm soát trình tự thực thi. Lệnh `flush()` yêu cầu đẩy câu lệnh UPDATE xuống PostgreSQL, tuy nhiên giao dịch (Transaction) bao ngoài vẫn chưa kết thúc. 

## 3. Dịch vụ giám sát kỳ vọng đọc bẩn (Watchdog kỳ vọng dirty read)

```java
@Service
public class JobWatchdog {
    private final JobRunRepository jobs;
    private final JdbcTemplate jdbc;

    public JobWatchdog(
        JobRunRepository jobs,
        JdbcTemplate jdbc
    ) {
        this.jobs = jobs;
        this.jdbc = jdbc;
    }

    // Yêu cầu mức độ cô lập READ_UNCOMMITTED (Đọc bẩn)
    @Transactional(
        isolation = Isolation.READ_UNCOMMITTED,
        readOnly = true
    )
    public WatchdogDecision inspect(UUID jobId) {
        // Lấy dữ liệu trạng thái
        JobSnapshot observed = jobs.findById(jobId)
            .orElseThrow()
            .snapshot();

        // Kiểm tra cấu hình Isolation hiện tại của hệ thống CSDL
        String reportedIsolation = jdbc.queryForObject(
            "select current_setting('transaction_isolation')",
            String.class
        );

        // Đánh giá: Tiến độ đạt 80% thì bình thường, ngược lại kích hoạt khôi phục
        return observed.progressPercent() >= 80
            ? WatchdogDecision.healthy(
                observed,
                reportedIsolation
            )
            : WatchdogDecision.startRecovery(
                observed,
                reportedIsolation
            );
    }
}
```

Sai lầm trong thiết kế này nằm ở việc lập trình viên tin rằng thông số `@Transactional(isolation = Isolation.READ_UNCOMMITTED)` sẽ ép buộc PostgreSQL cung cấp dữ liệu chưa commit của các giao dịch khác (như tiến độ `80%` đang chờ xử lý). 
Trong thực tế, PostgreSQL tuân thủ nghiêm ngặt nguyên tắc MVCC và luôn trả về giá trị đã commit cuối cùng (`20%`). Do đó, Watchdog sẽ đánh giá sai tình trạng hệ thống và gọi hàm `startRecovery` một cách không cần thiết.

> **Nói ngắn gọn:** Annotation cấu hình của Spring truyền yêu cầu chính xác xuống CSDL, và PostgreSQL tiếp nhận. Tuy nhiên, hành vi truy xuất dữ liệu dở dang (dirty read phenomenon) bị PostgreSQL từ chối thực hiện từ cấp độ kiến trúc.

## 4. Trình tự SQL thực thi tại CSDL (SQL timeline)

Phiên làm việc của Processor (Session A):

```sql
begin;
update job_run
set progress_percent = 80
where job_id = :jobId;
-- Giao dịch đang giữ trạng thái mở, chưa gọi lệnh commit...
```

Phiên làm việc của Watchdog (Session B, thực thi sau A):

```sql
begin isolation level read uncommitted; -- Yêu cầu mức Đọc Bẩn
select current_setting('transaction_isolation'); -- CSDL phản hồi: "read uncommitted"
select progress_percent
from job_run
where job_id = :jobId;
-- PostgreSQL trả về giá trị đã commit gần nhất: 20, KHÔNG PHẢI 80.
commit;
```

Máy chủ báo cáo cấu hình là `read uncommitted` nhưng thực tế vận hành giống hệt như `READ COMMITTED`. Khi xây dựng kịch bản kiểm thử, chúng ta phải tập trung kiểm tra dữ liệu kết quả (`20` hay `80`), thay vì kiểm tra chuỗi trạng thái cấu hình.

## 5. Phân biệt Xả dữ liệu (Flush) và Xác nhận (Commit)

Sử dụng hàm:
```java
jobs.saveAndFlush(job);
```
Chỉ thực hiện việc đồng bộ câu lệnh SQL xuống cơ sở dữ liệu, không có tác dụng chia sẻ (publish) dữ liệu cho các giao dịch khác. Thao tác này thực hiện các nhiệm vụ:
- Gửi lệnh `UPDATE` xuống PostgreSQL.
- PostgreSQL tạo ra một phiên bản dữ liệu nháp (uncommitted tuple version).
- Giữ vững giao dịch và các khóa (locks) liên quan.
- Đảm bảo tính nhất quán (visibility) chỉ cho giao dịch hiện hành (Processor).
- **Ngăn chặn hoàn toàn** các giao dịch đọc thông thường khác nhìn thấy dữ liệu nháp này.

Sự kiện `COMMIT` mới là ranh giới duy nhất công bố dữ liệu ra toàn hệ thống (cross-transaction).

## 6. Xử lý khi giao dịch ghi thất bại (Aborted writer)

Nếu Processor gặp ngoại lệ (`failAfterFlush = true`):

```text
Processor ghi giá trị 80 và gọi flush.
Watchdog vào đọc và nhận giá trị 20 (phiên bản đã commit cũ).
Processor gặp lỗi và Rollback giao dịch.
Kết quả cuối cùng trên CSDL: Tiến độ vẫn là 20.
```

Phiên bản giá trị `80` chưa từng được công bố và sẽ bị PostgreSQL đánh dấu là dữ liệu rác để công cụ `VACUUM` dọn dẹp sau này theo quy trình MVCC.

## 7. Giả định tương thích nền tảng (Migration assumption)

Khi chuyển đổi mã nguồn từ các CSDL hỗ trợ Đọc Bẩn (như SQL Server, MySQL), các lập trình viên thường có giả định:

```text
Sử dụng cờ READ_UNCOMMITTED / NOLOCK sẽ giúp đọc được dữ liệu đang cập nhật giữa chừng (in-flight writes).
```

Tuy nhiên, tiêu chuẩn SQL chỉ định nghĩa các mức độ cô lập (Isolation levels) nhằm thiết lập yêu cầu tối thiểu. Các nhà sản xuất RDBMS hoàn toàn có quyền cung cấp các cơ chế an toàn hơn để loại bỏ những rủi ro bất thường. PostgreSQL ánh xạ (maps) yêu cầu `READ_UNCOMMITTED` sang luồng xử lý của `READ_COMMITTED` (chỉ đọc dữ liệu đã commit).

## 8. Truy vấn thông thường không bị khóa chặn (Plain SELECT không block)

Lệnh `UPDATE` của Processor nắm giữ khóa cấp dòng (row lock). Ngược lại, lệnh `SELECT` thông thường của Watchdog không yêu cầu cấp khóa và không bị khóa chặn (block). Cơ chế MVCC hỗ trợ Watchdog lấy dữ liệu từ phiên bản đã commit trước đó một cách mượt mà. Kể cả khi bạn thiết lập `Query Timeout` rất cao, câu lệnh đọc cũng sẽ ngay lập tức trả về `20` thay vì đợi giá trị `80`.

Sử dụng câu lệnh `SELECT ... FOR UPDATE` (Đọc có khóa) lại là một cơ chế đồng bộ hoàn toàn khác: Giao dịch của Watchdog sẽ buộc phải chờ Processor xử lý xong (commit/rollback) rồi mới tiếp tục, tuy nhiên nó không giúp đọc được dữ liệu nháp.

## 9. Điều kiện tái hiện hiện tượng (Preconditions)

- Giá trị đã commit ban đầu là `20`.
- Processor thực hiện `UPDATE` và `flush` giá trị `80` bên trong một giao dịch chưa đóng.
- Watchdog sử dụng một kết nối (connection) độc lập.
- Watchdog truy vấn trước khi Processor kết thúc quá trình xử lý (completion).
- Mức độ cô lập yêu cầu là `READ_UNCOMMITTED`.
- Câu lệnh truy vấn là đọc thông thường (plain SELECT), không đính kèm khóa truy cập.
- Watchdog dựa vào kết quả truy vấn lỗi thời để đưa ra các quyết định xử lý hệ thống nguy hiểm (Recovery).

## 10. Các giải pháp tạm bợ và sai lầm (Những cách sửa chưa đủ)

Việc sửa chữa các sự cố này thường bị sa đà vào các biện pháp đối phó không mang lại hiệu quả:
- Chèn thêm nhiều phương thức `saveAndFlush()` một cách vô định.
- Chạy vòng lặp kiểm tra liên tục (Polling) trong khi Processor vẫn đang giữ giao dịch.
- Can thiệp thay đổi cấu hình mặc định của phiên làm việc (Session default).
- Nâng giới hạn thời gian chờ của Query hoặc Transaction (Timeout).
- Từ bỏ Spring Data JPA và chuyển sang dùng Native SQL với các lệnh `SELECT` truyền thống.
- Sử dụng việc CSDL trả về nhãn "read uncommitted" để khẳng định rằng logic Đọc Bẩn đã hoạt động.
- Từ bỏ hệ quản trị CSDL MVCC an toàn để chọn một CSDL chấp nhận thiết kế chia sẻ trạng thái bẩn rủi ro cao.
- Khai báo các cờ (flags) trong bộ nhớ ứng dụng (in-memory) để chia sẻ tiến độ, nhưng thất bại khi hệ thống được mở rộng (scaling/load-balancing) sang nhiều máy chủ khác nhau.
