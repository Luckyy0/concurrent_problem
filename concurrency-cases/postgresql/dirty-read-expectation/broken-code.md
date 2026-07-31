# Đoạn Code Ảo Tưởng Đọc Bẩn (Broken dirty-read design)

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
            throw new IllegalStateException("Job is not RUNNING");
        }
        if (
            progressPercent < this.progressPercent
                || progressPercent > 100
        ) {
            throw new IllegalArgumentException(
                "Progress must be monotonic and <= 100" // Tiến độ chỉ được tăng dần, cấm đi lùi!
            );
        }
        this.progressPercent = progressPercent;
    }

    // Copy trạng thái hiện tại ra một DTO gọn nhẹ
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

## 2. Kẻ làm mướn ngâm tôm (Processor transaction)

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
        
        // 1. Cập nhật tiến độ lên 80% (mới sửa trong RAM)
        job.reportProgress(80);

        // 2. Ép đẩy lệnh SQL UPDATE xuống Database ngay lập tức (Nhưng chưa thèm Commit!)
        jobs.flush();
        probe.afterProgressFlushed(jobId); // Cái cờ (hook) này dùng để Test bắt trúng thời điểm

        // 3. Chơi lầy: Ném lỗi giả bộ chết để Rollback, hoặc rề rà làm cái gì đó rất lâu
        if (failAfterFlush) {
            throw new ProcessingFailedException(jobId);
        }

        // Còn làm một nùi việc lưu Database khác ở dưới này...
    }
}
```

Cái `ProcessingProbe` chỉ là cái móc (hook) vô hại trên Production, nhưng khi chạy Test thì dùng nó làm rào chắn để canh me chính xác thời điểm. Chỗ gọi hàm `flush()` nhằm ép lòi ra lệnh UPDATE gửi thẳng xuống PostgreSQL, tuy nhiên cái Transaction thì vẫn đang mở toang (chưa hề commit).

## 3. Người gác cổng ngây thơ đòi Đọc Bẩn (Watchdog kỳ vọng dirty read)

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

    // Dán lá bùa xin ĐỌC_BẨN (READ_UNCOMMITTED)
    @Transactional(
        isolation = Isolation.READ_UNCOMMITTED,
        readOnly = true
    )
    public WatchdogDecision inspect(UUID jobId) {
        // Đọc dữ liệu ra
        JobSnapshot observed = jobs.findById(jobId)
            .orElseThrow()
            .snapshot();

        // Check thử xem DB có cấp cho mình cái Mác Đọc Bẩn thật không?
        String reportedIsolation = jdbc.queryForObject(
            "select current_setting('transaction_isolation')",
            String.class
        );

        // Quyết định: 80% thì khen Khỏe, thấp hơn thì hô hoán "Treo rồi, Cứu!"
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

Tác giả đoạn Code này ảo tưởng tin rằng cái lá bùa `@Transactional` kia đủ linh nghiệm để giúp Ông B nhìn xuyên tường, thấy được con số `80` mà Ông A đang giấu. 
Nhưng ở đất PostgreSQL, B chỉ hốt được cái cục `20` cũ rích đã chốt từ kiếp nào, rồi ngơ ngác trả về lệnh `startRecovery` (Chạy lại từ đầu).

> **Nói ngắn gọn:** Bạn nạp cái Annotation, Spring xử lý và truyền lệnh đàng hoàng, PostgreSQL gật đầu nhận lệnh. NHƯNG cái hành động (phenomenon) mà bạn mong muốn thì Database nó từ chối phục vụ!

## 4. Dòng thời gian SQL thực tế chạy dưới DB (SQL timeline)

Kênh của Ông A (Session A):

```sql
begin;
update job_run
set progress_percent = 80
where job_id = :jobId;
-- Im lặng, chưa thèm commit...
```

Kênh của Ông B (Session B, chạy sau A):

```sql
begin isolation level read uncommitted; -- Dạ xin phép con Đọc Bẩn ạ!
select current_setting('transaction_isolation'); -- DB trả lời: "Uh mày đang Đọc Bẩn đấy!"
select progress_percent
from job_run
where job_id = :jobId;
-- NHƯNG kết quả vẫn vả vào mặt con số 20, KHÔNG PHẢI 80!
commit;
```

Máy chủ (Server) có thể hiện cái Nhãn `read uncommitted` ra cho bạn yên lòng; nhưng cách hành xử của nó giống y đúc `READ COMMITTED`. 
Bởi vậy, Code Test là phải kiểm tra (assert) vào con số thật sự Đọc được (tầm nhìn - visibility), chứ đừng rảnh rỗi đọc cái chuỗi Label rồi tự thủ dâm tinh thần.

## 5. Xả (Flush) không bao giờ là Chốt (Commit)

Bạn dùng hàm:
```java
jobs.saveAndFlush(job);
```
Cái này cũng KHÔNG HỀ công bố dữ liệu ra ngoài (publish state). Việc Flush chỉ làm đúng 5 trò:
- Gửi lệnh UPDATE xuống DB;
- DB tạo ra 1 cái bóng dở dang (uncommitted tuple version);
- Giữ chặt cái Giao dịch và Khóa (locks);
- Cho chính Ông A tự sướng nhìn thấy thứ mình vừa Ghi;
- **Giấu nhẹm** không cho lệnh SELECT chay của B thấy cái bóng đó.

Chỉ khi bạn gõ lệnh Commit thì thiên hạ (cross-transaction) mới nhìn thấy.

## 6. Chuyện người thua cuộc (Aborted writer)

Nếu vô tình vấp lỗi `failAfterFlush = true`:

```text
Ông A xả (flush) số 80 xuống
Ông B nhảy vào đọc thấy số 20 (bản đã chốt cũ)
Ông A lăn đùng ra chết (ném exception) -> Rút lại lời nói (rollback)
Kết quả chung cuộc tiến độ chốt sổ vẫn là 20.
```

Con số `80` đẻ non kia KHÔNG BAO GIỜ có cơ hội ngóc đầu lên làm phiền ông B. PostgreSQL và công cụ hút bụi (VACUUM) sau này sẽ âm thầm hốt dọn cái xác vật lý đó theo luật MVCC.

## 7. Ảo mộng bê code (Migration assumption)

Bê code từ Database khác (như SQL Server, MySQL) sang có thể nảy sinh ảo tưởng:

```text
Cứ set READ_UNCOMMITTED / gắn cờ NOLOCK
=> Chắc chắn tao sẽ ngó trộm được Data đang ghi nửa chừng (in-flight writes).
```

Mấy cái tên mức độ cô lập (Isolation names) chỉ là từ vựng chuẩn mực để gọi tên, chứ các Hãng Database được toàn quyền Cung cấp luật lệ khắt khe hơn. PostgreSQL ánh xạ (maps) luật chơi `READ_UNCOMMITTED` thành trò "Chỉ Đọc Hàng Đã Chốt" (committed-only).

## 8. Lệnh Đọc Chay không thèm chờ Khóa (Plain SELECT không block)

Lệnh UPDATE của A giữ cái Khóa Dòng (row lock), nhưng lệnh Đọc Chay của B chả thèm quan tâm cái Khóa đối nghịch đó; luật MVCC cho phép B đi lối tắt móc cái dòng cũ đã chốt ra mà xài. Dù bạn có tăng cái Query Timeout lên 1 tiếng đồng hồ thì cũng vô vọng, chả bao giờ làm cái bóng chưa chốt kia lòi ra được.

Dùng lệnh `SELECT ... FOR UPDATE` (Đọc Lấy Khóa) lại là 1 game hoàn toàn khác: nó sẽ đứng xếp hàng chờ A chốt sổ/hủy kèo, sau đó lượm kết quả phù hợp (outcome). Trò đó KHÔNG PHẢI là Đọc Bẩn!

## 9. Điều kiện hội tụ để nổ Bug (Preconditions tái hiện)

- Tiến độ gốc Đã Chốt ban đầu là `20`.
- Ông A gửi UPDATE và Xả (flush) số `80` trong cái Giao dịch đang mở toang.
- Ông B dùng Kết nối/Giao dịch độc lập (independent connection).
- Ông B phát lệnh SELECT vào lúc A chưa làm xong nhiệm vụ (chưa completion).
- Ông B cầu xin `READ_UNCOMMITTED`.
- Lệnh query là Đọc Chay (plain SELECT), không có trò ép Khóa (locking clause).
- Ông Watchdog (B) vỗ đùi cái đét, kết luận tiến độ dậm chân tại chỗ cũ rích là do "Job Treo Rồi!".

## 10. Những ngõ cụt chắp vá (Những cách sửa chưa đủ)

Nhiều pháp sư chọn những trò đi vào ngõ cụt sau, nhưng vĩnh viễn không sửa được:
- Cố nhét thêm `saveAndFlush()` hòng nặn Data ra.
- Trả thù bằng cách lặp vòng lặp Hỏi liên tục (Poll) vào lúc Ông A còn đang ngâm Giao dịch.
- Đổi cài đặt Label/Default của Session thành `read uncommitted`.
- Tăng Timeout của Query/Transaction lên thật cao.
- Quăng cái thư viện Spring Data đi, ráng gõ SQL chay (Native SQL) nhưng vẫn dùng lệnh Đọc Chay.
- Tự sướng với cái chuỗi (String) Label Isolation đọc được từ Database và lấy đó làm bằng chứng "Hệ thống có Đọc Bẩn".
- Đổi cả hệ quản trị Cơ Sở Dữ Liệu chỉ để chứa chấp cái thiết kế (Design) lố bịch chuyên đi ngó trộm Dữ Liệu Rác (dirty state).
- Thêm cờ Biến trong RAM (in-memory flag) cắm vào Code để Watchdog đọc ké, nhưng vỡ mồm khi Load Balancer quăng Watchdog qua cái Server (multi-instance) khác.
