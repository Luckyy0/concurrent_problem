# Code lỗi — duplicate claim và lock convoy

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

## Lỗi 1 — `SELECT` rồi xử lý

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

A và B có thể đọc cùng committed READY rows trước khi actor nào update. Entity
object/local thread khác nhau không tạo ownership. Cả hai gọi external processor,
sau đó cùng ghi `DONE`.

Thêm `@Transactional` quanh `poll()` không sửa plain SELECT; nó còn giữ connection
qua remote I/O.

## Lỗi 2 — `FOR UPDATE` nhưng mọi worker xếp hàng

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

Nếu Worker A giữ oldest row, B/C đều chọn cùng row theo snapshot/order rồi block.
Chúng không tự “nhảy” sang job kế tiếp. Queue có 1.000 jobs vẫn có thể bị convoy
sau job đầu.

> **Nói ngắn gọn:** `FOR UPDATE` loại duplicate trong transaction nhưng không tạo
> parallel claiming; `SKIP LOCKED` mới cho poller đi qua row đang bận.

## Lỗi 3 — giữ lock qua external work

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

Row locks chỉ release tại commit/rollback. Remote latency giữ database connection,
locks và transaction snapshot. Process crash rollback status nhưng không hoàn tác
external effect đã thành công; job sẽ chạy lại.

## Lỗi 4 — claim hai bước không atomic

```java
@Transactional
public List<WorkJob> claim(String workerId, int batchSize) {
    List<WorkJob> selected = jobs.findReady(batchSize);
    selected.forEach(job -> job.markProcessing(workerId));
    return selected;
}
```

`SELECT` không khóa. Hai transactions có thể chọn cùng IDs trước khi dirty
checking flushes `UPDATE`. Nếu không có `@Version`, later update không báo
conflict; có `@Version` thì conflict chỉ lộ ở flush nhưng external work có thể đã
bắt đầu nếu caller không chờ commit.

## Lỗi 5 — status không có recovery ownership

```sql
update work_job
set status = 'PROCESSING',
    claimed_by = :worker
where job_id = :jobId
  and status = 'READY';
```

Affected-row check ngăn duplicate claim cho một row, nhưng process crash để job
`PROCESSING` vĩnh viễn. Nếu sweeper chỉ đổi status về READY, worker cũ hồi phục
có thể complete sau worker mới và ghi đè kết quả.

Cần `lease_until` và claim token mới cho mỗi ownership epoch. Completion phải
conditional theo token.

## Lỗi 6 — `SKIP LOCKED` nhưng không có `ORDER BY`

```sql
select job_id
from work_job
where status = 'READY'
for update skip locked
limit 10;
```

Không có `ORDER BY`, PostgreSQL được phép trả subset theo plan thuận tiện. Không
có fairness contract, priority/SLA không thể kiểm chứng và plan change có thể đổi
hành vi.

Ngay cả có `ORDER BY`, `SKIP LOCKED` không bảo đảm strict FIFO: row đầu đang lock
bị bỏ qua để giữ progress.

## Lỗi 7 — tưởng `SKIP LOCKED` không bao giờ chờ

`SKIP LOCKED` chỉ áp dụng cho row-level locks. Query vẫn acquire table-level
`ROW SHARE` theo cách thông thường và có thể chờ incompatible DDL/table lock,
connection pool, I/O hoặc statement resource khác.

Không bỏ `statement_timeout`, application deadline và migration coordination chỉ
vì có `SKIP LOCKED`.

## Lỗi 8 — complete không kiểm tra owner

```java
@Modifying
@Query("""
        update WorkJob j
        set j.status = 'DONE'
        where j.id = :jobId
        """)
int complete(UUID jobId);
```

Timeline:

```text
A claim token-A → pause quá lease
sweeper requeue
B claim token-B → process
A resume → complete chỉ theo jobId
```

A là stale worker nhưng vẫn ghi `DONE`. Nếu B đang retry/fail, A phá state machine.

## Self-invocation làm transaction không đúng

```java
public void poll() {
    List<ClaimedJob> batch = claimBatch(); // this.claimBatch()
}

@Transactional
public List<ClaimedJob> claimBatch() {
    // SELECT FOR UPDATE SKIP LOCKED
}
```

Call nội bộ không qua Spring proxy. Claim/commit boundary có thể không tồn tại như
code review tưởng. Claimer phải là bean riêng hoặc dùng `TransactionTemplate`.

## Điều kiện tái hiện

- ít nhất hai physical PostgreSQL connections;
- nhiều committed READY jobs có deterministic order;
- barrier sau plain SELECT để tái hiện duplicate;
- holder giữ first row để chứng minh convoy/skip;
- `Future.get(timeout)` và `statement_timeout` để test không treo;
- Testcontainers PostgreSQL, không dùng H2 cho lock semantics.

## Các cách sửa chưa đủ

- `synchronized`: chỉ bảo vệ một JVM.
- Tăng worker threads/pool: tăng duplicate/lock wait/database load.
- `FOR UPDATE` không `SKIP LOCKED`: correctness có thể tốt hơn nhưng convoy.
- `SKIP LOCKED` rồi xử lý trước commit: lock lifetime vẫn dài.
- Status `PROCESSING` không lease/token: crash và stale worker chưa được giải quyết.
- Retry external call không idempotency key: duplicate effect.
- Batch không giới hạn: transaction/lock footprint lớn.
- Chỉ monitor queue depth: không thấy oldest age, expired leases và starvation.
