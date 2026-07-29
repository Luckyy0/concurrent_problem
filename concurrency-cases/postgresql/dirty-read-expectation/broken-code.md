# Broken dirty-read design

## Job state

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
                "Progress must be monotonic and <= 100"
            );
        }
        this.progressPercent = progressPercent;
    }

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

## Processor transaction

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
        job.reportProgress(80);

        jobs.flush();
        probe.afterProgressFlushed(jobId);

        if (failAfterFlush) {
            throw new ProcessingFailedException(jobId);
        }

        // Remaining local database work in the same transaction.
    }
}
```

`ProcessingProbe` là no-op hook trong production và controlled gate trong
integration test. Explicit flush đảm bảo UPDATE đã gửi tới PostgreSQL nhưng
transaction chưa commit.

## Watchdog kỳ vọng dirty read

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

    @Transactional(
        isolation = Isolation.READ_UNCOMMITTED,
        readOnly = true
    )
    public WatchdogDecision inspect(UUID jobId) {
        JobSnapshot observed = jobs.findById(jobId)
            .orElseThrow()
            .snapshot();

        String reportedIsolation = jdbc.queryForObject(
            "select current_setting('transaction_isolation')",
            String.class
        );

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

Design giả định annotation cho phép B thấy A value `80`. Trên PostgreSQL, B đọc old
committed tuple `20` và trả `startRecovery`.

> **Nói ngắn gọn:** annotation được xử lý, nhưng database không cung cấp phenomenon
> mà design đang dựa vào.

## SQL timeline

Session A:

```sql
begin;
update job_run
set progress_percent = 80
where job_id = :jobId;
-- no commit
```

Session B:

```sql
begin isolation level read uncommitted;
select current_setting('transaction_isolation');
select progress_percent
from job_run
where job_id = :jobId;
-- returns 20, not 80
commit;
```

Server có thể report label `read uncommitted`; behavior vẫn giống `READ
COMMITTED`. Test phải assert visibility, không suy luận semantics từ string label.

## Flush không phải commit

```java
jobs.saveAndFlush(job);
```

cũng không publish state. Flush:

- gửi UPDATE;
- tạo uncommitted tuple version;
- giữ transaction/locks;
- cho chính A thấy own write;
- không cho B plain SELECT thấy version đó.

Chỉ commit tạo cross-transaction visibility.

## Aborted writer

Nếu `failAfterFlush = true`:

```text
A flushes 80
B sees committed 20
A throws -> rollback
final committed progress = 20
```

Aborted `80` không trở thành visible cho B. PostgreSQL/VACUUM cleanup physical dead
version sau này theo MVCC rules.

## Migration assumption

Code/design port từ database khác có thể giả định:

```text
READ_UNCOMMITTED / NOLOCK
=> observe in-flight writes
```

Isolation names là standard vocabulary, nhưng databases được phép cung cấp stronger
guarantees. PostgreSQL maps requested RU semantics lên committed-only behavior.

## Plain SELECT không block

A UPDATE giữ row lock, nhưng B plain SELECT thường không wait incompatible row
lock; MVCC cho B đọc old committed tuple. Thêm query timeout không làm uncommitted
version visible.

`SELECT ... FOR UPDATE` là operation khác: nó có thể block tới A commit/rollback,
sau đó đọc/lock outcome phù hợp. Nó không phải dirty read.

## Preconditions tái hiện

- Initial committed progress là `20`.
- A UPDATE/flush `80` trong open transaction.
- B dùng independent connection/transaction.
- B SELECT xảy ra trước A completion.
- B request `READ_UNCOMMITTED`.
- Query là plain SELECT, không locking clause.
- Watchdog diễn giải old committed progress là stalled.

## Những cách sửa chưa đủ

- Chỉ thêm `saveAndFlush()`.
- Poll nhanh hơn trong khi A transaction vẫn open.
- Đổi label/default thành `read uncommitted`.
- Tăng query/transaction timeout.
- Dùng native SQL nhưng cùng plain SELECT.
- Assume reported isolation string chứng minh dirty-read capability.
- Đổi database chỉ để giữ một coordination design dựa trên dirty state.
- Dùng in-memory flag cho watchdog trong multi-instance deployment.
