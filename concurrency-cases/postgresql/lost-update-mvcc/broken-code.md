# Broken JPA read-modify-write

## Entity không có conflict detector

```java
@Entity
@Table(name = "job_progress")
public class JobProgress {
    @Id
    private UUID jobId;

    private int completedUnits;
    private int totalUnits;

    protected JobProgress() {
    }

    public void addCompletedUnits(int delta) {
        if (delta <= 0) {
            throw new IllegalArgumentException(
                "Delta must be positive"
            );
        }
        if (completedUnits + delta > totalUnits) {
            throw new IllegalStateException(
                "Completed units exceed total units"
            );
        }
        completedUnits += delta;
    }

    public int getCompletedUnits() {
        return completedUnits;
    }
}
```

Entity không có `@Version`. Validation chạy trên value entity đã load, không phải
current value tại write time.

## Repository

```java
public interface JobProgressRepository
        extends JpaRepository<JobProgress, UUID> {
}
```

## Broken service

```java
@Service
public class JobProgressService {
    private final JobProgressRepository progress;

    public JobProgressService(JobProgressRepository progress) {
        this.progress = progress;
    }

    @Transactional
    public ProgressResult addCompletedUnits(
        UUID jobId,
        int delta
    ) {
        JobProgress job = progress.findById(jobId)
            .orElseThrow();

        int before = job.getCompletedUnits();
        job.addCompletedUnits(delta);

        return new ProgressResult(
            jobId,
            before,
            job.getCompletedUnits()
        );
    }
}
```

Method gọi qua Spring proxy đúng và mỗi request có transaction riêng. Bug không
phải self-invocation; bug là compound operation qua database boundary:

```text
read -> calculate in JVM -> absolute write
```

## SQL Hibernate thực hiện

SELECT:

```sql
select job_id, completed_units, total_units
from job_progress
where job_id = ?;
```

Dirty checking tại flush/commit tạo UPDATE tương đương:

```sql
update job_progress
set completed_units = ?,
    total_units = ?
where job_id = ?;
```

Tùy mapping/dynamic-update configuration, Hibernate có thể update ít hoặc nhiều
columns. Điểm quyết định là predicate chỉ có primary key:

```text
WHERE job_id = ?
```

Không có:

```text
AND version = ?
AND completed_units = old_value
```

Cả A và B UPDATE đều affected rows `1`, nên Hibernate không có conflict signal.

> **Nói ngắn gọn:** transaction đảm bảo mỗi UPDATE atomic, nhưng không biến ba bước
> read/calculate/write thành một atomic business operation.

## Concrete broken interleaving

```text
row initial = 10

Tx-A SELECT -> 10
Tx-B SELECT -> 10

Tx-A Java calculation -> 13
Tx-B Java calculation -> 14

Tx-A UPDATE SET completed_units = 13 WHERE job_id = ...
Tx-A COMMIT

Tx-B UPDATE SET completed_units = 14 WHERE job_id = ...
Tx-B COMMIT

final = 14, expected = 17
```

Nếu B UPDATE bắt đầu trước A commit, nó có thể wait row lock. Sau A commit, B vẫn
ghi parameter `14` đã tính từ stale value `10`.

## `save()` không sửa lost update

Explicit save:

```java
job.addCompletedUnits(delta);
progress.save(job);
```

vẫn merge/dirty-check cùng absolute state và cùng predicate không version. Thời
điểm repository method được gọi không thêm atomicity.

## `saveAndFlush()` chỉ làm conflict sớm hơn nếu có detector

```java
progress.saveAndFlush(job);
```

Không có `@Version`, flush chỉ gửi stale UPDATE sớm hơn. Row count vẫn `1`; không
có exception để retry.

## `synchronized` chỉ bảo vệ một object/JVM

```java
public synchronized ProgressResult addCompletedUnits(...) {
    // ...
}
```

Spring singleton lock có thể serialize calls trên một instance nhưng:

- App node 2 có singleton/monitor khác;
- multiple service objects/tests có lock khác;
- direct SQL/job worker không tham gia JVM lock;
- process restart không tạo durable coordination.

Authoritative row nằm ở PostgreSQL, nên invariant phải được bảo vệ tại database
write/transaction boundary.

## Validation cũng bị stale

Initial:

```text
completed = 95
total = 100
A delta = 3
B delta = 4
```

Cả actors validate `95 + delta <= 100` và pass. Absolute writes cuối có thể là
`98` hoặc `99`, che việc accepted total deltas lẽ ra thành `102`. Lost update làm
counter trông hợp lệ nhưng accepted-work invariant đã bị phá.

Database constraint `completed_units <= total_units` cần thiết nhưng không đủ để
phát hiện delta biến mất.

## Preconditions tái hiện

- Hai physical transactions ở effective `READ COMMITTED`.
- Plain SELECTs hoàn tất trước first commit.
- Không row lock trên read.
- Entity không có `@Version`.
- Update predicate không chứa old value/version.
- Cả methods return success.
- Second absolute UPDATE commit sau first.

## Những cách sửa chưa đủ

- Chỉ thêm `@Transactional`.
- Chỉ gọi `save()`/`saveAndFlush()`.
- Chỉ thêm in-memory lock.
- Chỉ kiểm tra `completed <= total` trong Java.
- Chỉ tăng isolation annotation nhưng không xử lý serialization failure.
- Retry mà vẫn reuse stale entity/transaction.
- Assume MVCC tự merge concurrent deltas.
