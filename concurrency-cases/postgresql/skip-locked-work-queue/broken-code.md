# Code lỗi — Duplicate claim và lock convoy

## Schema

```sql
create table work_job (
    job_id uuid primary key,
    job_type varchar(64) not null,
    payload jsonb not null,
    status varchar(16) not null,
    priority integer not null default 0,
    available_at timestamptz not null,
    claim_token uuid,
    claimed_by varchar(128),
    lease_until timestamptz,
    attempt_count integer not null default 0,
    completed_at timestamptz,
    last_error varchar(512),
    check (status in ('READY', 'PROCESSING', 'DONE', 'DEAD'))
);

create index ix_work_job_claim
    on work_job(priority desc, available_at, job_id)
    where status = 'READY';
```

## Lỗi 1 — `SELECT` rồi mới xử lý

```java
public interface WorkJobRepository extends JpaRepository<WorkJob, UUID> {

    @Query(value = """
            select *
            from work_job
            where status = 'READY'
              and available_at <= clock_timestamp()
            order by priority desc, available_at, job_id
            limit :batchSize
            """, nativeQuery = true)
    List<WorkJob> findReady(int batchSize);
}

@Service
public class BrokenPollingWorker {

    private final WorkJobRepository jobs;
    private final ExternalProcessor processor;

    public BrokenPollingWorker(
            WorkJobRepository jobs,
            ExternalProcessor processor
    ) {
        this.jobs = jobs;
        this.processor = processor;
    }

    public void poll() {
        for (WorkJob job : jobs.findReady(10)) {
            processor.process(job.id(), job.payload());
            job.markDone();
            jobs.save(job);
        }
    }
}
```

Ở đây, Worker A và Worker B có thể cùng đọc được những dòng dữ liệu đang ở trạng thái `READY` trước khi bất kỳ ai kịp cập nhật (update). Các object hoặc luồng (thread) lưu trữ cục bộ không thể giúp ta thiết lập quyền sở hữu (ownership). Hậu quả là cả hai worker sẽ cùng gọi dịch vụ bên ngoài, và cuối cùng đều ghi đè trạng thái `DONE`.

Ngay cả khi bạn thêm `@Transactional` bao quanh hàm `poll()` thì cũng không giải quyết được vấn đề vì bản chất nó vẫn là một truy vấn `SELECT` thông thường. Hơn nữa, việc này còn khiến kết nối cơ sở dữ liệu bị giữ chặt suốt quá trình gọi I/O ra dịch vụ bên ngoài.

## Lỗi 2 — `FOR UPDATE` nhưng mọi worker lại phải xếp hàng

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query(value = """
        select *
        from work_job
        where status = 'READY'
          and available_at <= clock_timestamp()
        order by priority desc, available_at, job_id
        limit :batchSize
        for update
        """, nativeQuery = true)
List<WorkJob> lockReady(int batchSize);
```

Giả sử Worker A đang giữ dòng công việc cũ nhất, thì Worker B và C khi truy vấn cũng sẽ chọn đúng dòng đó (do cùng điều kiện và thứ tự sắp xếp) rồi bị chặn lại (block). Chúng không thể tự động "nhảy" sang công việc tiếp theo. Hàng đợi có 1.000 công việc vẫn có thể bị kẹt cứng (convoy) chỉ vì công việc đầu tiên đang bị khóa.

> **Nói ngắn gọn:** `FOR UPDATE` giúp loại bỏ việc lấy trùng công việc (duplicate) trong giao dịch, nhưng nó không cho phép các worker lấy việc song song (parallel claiming); `SKIP LOCKED` mới chính là cơ chế cho phép worker lướt qua những dòng đang bận để lấy việc tiếp theo.

## Lỗi 3 — Giữ lock xuyên suốt quá trình gọi external work

```java
@Transactional
public void pollAndProcess() {
    List<WorkJob> claimed = jobs.lockReady(10);

    for (WorkJob job : claimed) {
        processor.process(job.id(), job.payload()); // network I/O
        job.markDone();
    }
}
```

Khóa dòng (Row locks) chỉ được giải phóng (release) khi giao dịch hoàn tất (commit/rollback). Bất kỳ sự chậm trễ nào từ mạng (remote latency) cũng sẽ giữ chặt kết nối cơ sở dữ liệu, các khóa và cả trạng thái của giao dịch (transaction snapshot). Nếu tiến trình bị sập (crash), trạng thái công việc sẽ được rollback, nhưng tác vụ ở hệ thống bên ngoài thì đã thành công rồi; kết quả là khi hệ thống chạy lại, công việc đó sẽ bị xử lý lại từ đầu.

## Lỗi 4 — Lấy việc bằng hai bước không atomic

```java
@Transactional
public List<WorkJob> claim(String workerId, int batchSize) {
    List<WorkJob> selected = jobs.findReady(batchSize);
    selected.forEach(job -> job.markProcessing(workerId));
    return selected;
}
```

Lệnh `SELECT` bình thường không hề tạo khóa. Hai giao dịch khác nhau có thể cùng chọn ra những ID giống hệt nhau trước khi tiến trình kiểm tra (dirty checking) đẩy lệnh `UPDATE` xuống cơ sở dữ liệu. Nếu không dùng cơ chế kiểm soát phiên bản (như `@Version`), lệnh update sau sẽ âm thầm ghi đè mà không báo lỗi xung đột; còn nếu dùng `@Version`, lỗi xung đột chỉ lộ ra ở thời điểm lưu (flush), trong khi tác vụ bên ngoài có thể đã bị thực thi mất rồi nếu ứng dụng không cẩn thận chờ commit.

## Lỗi 5 — Trạng thái không đi kèm khả năng khôi phục ownership

```sql
update work_job
set status = 'PROCESSING',
    claimed_by = :worker
where job_id = :jobId
  and status = 'READY';
```

Mặc dù việc kiểm tra số dòng bị ảnh hưởng (affected-row) có thể ngăn hai worker cùng lấy một dòng, nhưng nếu hệ thống sập (crash), công việc đó sẽ kẹt vĩnh viễn ở trạng thái `PROCESSING`. Nếu chúng ta chỉ dùng một tiến trình dọn dẹp để đặt lại trạng thái về `READY`, worker cũ khi tỉnh lại vẫn có thể hoàn thành công việc sau worker mới, dẫn đến ghi đè kết quả.

Để giải quyết, chúng ta cần `lease_until` (thời hạn thuê) và một thẻ định danh (claim token) mới cho mỗi chu kỳ sở hữu. Bất kỳ thao tác hoàn thành nào cũng phải có điều kiện bắt buộc đi kèm với token này.

## Lỗi 6 — Dùng `SKIP LOCKED` nhưng không dùng `ORDER BY`

```sql
select job_id
from work_job
where status = 'READY'
for update skip locked
limit 10;
```

Nếu không có `ORDER BY`, PostgreSQL có quyền trả về một tập con bất kỳ sao cho tiện nhất với kế hoạch thực thi (plan) của nó. Điều này có nghĩa là bạn sẽ không có sự công bằng (fairness contract), không thể kiểm chứng được độ ưu tiên hay SLA, và một sự thay đổi nhỏ trong query plan có thể làm thay đổi hoàn toàn hành vi của hệ thống.

Tuy nhiên, ngay cả khi bạn dùng `ORDER BY`, `SKIP LOCKED` vẫn không đảm bảo thứ tự vào trước ra trước tuyệt đối (strict FIFO): một dòng ưu tiên nhất nếu đang bị khóa sẽ bị bỏ qua để nhường chỗ cho dòng tiếp theo nhằm duy trì tiến độ chung.

## Lỗi 7 — Ảo tưởng rằng `SKIP LOCKED` không bao giờ chờ

`SKIP LOCKED` chỉ áp dụng đối với các khóa cấp độ dòng (row-level locks). Truy vấn của bạn vẫn cần yêu cầu khóa cấp độ bảng (`ROW SHARE`) giống như thông thường, và nó hoàn toàn có thể bị kẹt lại bởi các lệnh thay đổi cấu trúc bảng (DDL), cạn kiệt connection pool, chậm trễ I/O hoặc xung đột tài nguyên.

Tuyệt đối không nên gỡ bỏ các giới hạn an toàn như `statement_timeout`, giới hạn thời gian chạy của ứng dụng, hay bỏ qua việc phối hợp khi thực hiện thay đổi cấu trúc bảng (migration) chỉ vì bạn nghĩ `SKIP LOCKED` đã lo liệu hết.

## Lỗi 8 — Hoàn thành (complete) mà không kiểm tra người sở hữu

```java
@Modifying
@Query("""
        update WorkJob j
        set j.status = 'DONE'
        where j.id = :jobId
        """)
int complete(UUID jobId);
```

Hãy hình dung diễn biến sau:

```text
Worker A lấy việc với token-A → bị khựng lại quá thời hạn thuê
Tiến trình quét đưa công việc về lại hàng đợi
Worker B lấy việc đó với token-B → bắt đầu xử lý
Worker A tỉnh lại → gọi hàm complete CHỈ KIỂM TRA jobId
```

Lúc này, Worker A đã trở thành một worker "ôi thiu" (stale worker) nhưng vẫn ngang nhiên ghi nhận `DONE`. Nếu Worker B đang xử lý lỗi và cần thử lại, hành động của A đã phá hỏng hoàn toàn luồng trạng thái (state machine) của hệ thống.

## Lỗi 9 — Gọi hàm nội bộ (Self-invocation) làm hỏng transaction

```java
public void poll() {
    List<ClaimedJob> batch = claimBatch(); // this.claimBatch()
}

@Transactional
public List<ClaimedJob> claimBatch() {
    // SELECT FOR UPDATE SKIP LOCKED
}
```

Trong Spring, việc gọi một hàm nội bộ từ trong cùng một class sẽ không đi qua proxy. Do đó, ranh giới lấy việc và commit (Claim/commit boundary) mà bạn tưởng tượng sẽ không hề tồn tại. Để giải quyết, hàm lấy việc phải được đặt ở một Bean riêng biệt, hoặc bạn phải sử dụng `TransactionTemplate`.

## Điều kiện tái hiện để kiểm thử

Để tái hiện các vấn đề này trong môi trường kiểm thử, bạn cần:
- Ít nhất hai kết nối vật lý (physical connections) độc lập đến PostgreSQL.
- Dữ liệu `READY` có thứ tự cố định (deterministic order) đã được commit.
- Một rào chắn (barrier) đặt ngay sau lệnh SELECT thông thường để tái hiện tình trạng lấy trùng (duplicate).
- Một connection giữ dòng dữ liệu đầu tiên để chứng minh hiện tượng kẹt (convoy) hoặc bỏ qua (skip).
- Sử dụng `Future.get(timeout)` và `statement_timeout` để đảm bảo test không bị treo mãi mãi.
- Phải dùng Testcontainers PostgreSQL, tuyệt đối không dùng H2 vì H2 không có cơ chế xử lý khóa (lock semantics) tương đồng.

## Các cách sửa chưa đủ hoặc sai lầm

- Dùng `synchronized`: Chỉ có tác dụng bảo vệ bên trong một máy ảo Java (JVM) duy nhất.
- Tăng số lượng worker/threads: Chỉ làm trầm trọng thêm tình trạng lấy trùng, chờ khóa và tăng tải cho database.
- Dùng `FOR UPDATE` không có `SKIP LOCKED`: Có thể giải quyết tính đúng đắn nhưng lại rước về hiện tượng kẹt hàng đợi (convoy).
- Dùng `SKIP LOCKED` rồi xử lý tác vụ trước khi commit: Tuổi thọ của khóa vẫn bị kéo dài không cần thiết.
- Cập nhật trạng thái `PROCESSING` mà không có thẻ token/lease: Hệ thống sập và worker quá hạn vẫn là bài toán chưa có lời giải.
- Thử lại tác vụ bên ngoài mà không có idempotency key: Sẽ gây ra việc xử lý trùng lặp.
- Lấy một lô (batch) mà không giới hạn: Phạm vi của giao dịch và khóa sẽ bị phình to quá mức.
- Chỉ theo dõi độ sâu hàng đợi (queue depth): Sẽ không thể nhận biết được các công việc đã bị ngâm quá lâu, các trường hợp quá hạn thuê hay công việc bị kẹt vĩnh viễn (starvation).
