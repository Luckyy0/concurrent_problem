# Atomic, optimistic and pessimistic solutions

## Chọn theo business intent

Intent của case là cộng commutative deltas vào một row. Database có thể thực hiện
phép cộng trực tiếp, nên atomic conditional SQL là solution đơn giản nhất.

Nếu mutation cần load nhiều fields/domain rules trong Java, optimistic `@Version`
hoặc pessimistic lock là credible alternatives.

> **Nói ngắn gọn:** gửi intent `+delta` tới authoritative store thay vì gửi absolute
> value được tính từ snapshot cũ.

## Solution 1 — Atomic conditional delta

### SQL

```sql
update job_progress
set completed_units = completed_units + :delta
where job_id = :jobId
  and :delta > 0
  and completed_units + :delta <= total_units;
```

Behavior:

1. UPDATE acquire row lock.
2. Concurrent updater có thể wait.
3. Sau holder commit, PostgreSQL xử lý current row version/re-evaluate predicate.
4. Expression dùng current `completed_units`.
5. Affected rows `1` = applied; `0` = not applied.

Với A `+3`, B `+4`:

```text
10 -> 13 -> 17
```

Commit order có thể đảo nhưng final sum vẫn `17` nếu cả predicates pass.

### Spring repository

```java
public interface JobProgressRepository
        extends JpaRepository<JobProgress, UUID> {

    @Modifying(
        flushAutomatically = true,
        clearAutomatically = true
    )
    @Query(
        value = """
            update job_progress
               set completed_units = completed_units + :delta
             where job_id = :jobId
               and :delta > 0
               and completed_units + :delta <= total_units
            """,
        nativeQuery = true
    )
    int addCompletedUnits(UUID jobId, int delta);
}
```

```java
@Service
public class AtomicJobProgressService {
    private final JobProgressRepository progress;

    public AtomicJobProgressService(
        JobProgressRepository progress
    ) {
        this.progress = progress;
    }

    @Transactional
    public ProgressApplyResult addCompletedUnits(
        UUID jobId,
        int delta
    ) {
        if (delta <= 0) {
            throw new IllegalArgumentException(
                "Delta must be positive"
            );
        }

        int changed = progress.addCompletedUnits(jobId, delta);
        return changed == 1
            ? ProgressApplyResult.applied(jobId, delta)
            : ProgressApplyResult.notApplied(jobId, delta);
    }
}
```

Bulk/native update bypasses managed entity state, nên clear/transaction design phải
tránh trả một stale `JobProgress` đã load trước đó.

### `RETURNING` khi cần new value

Dùng `JdbcTemplate`/jOOQ/native DAO:

```sql
update job_progress
set completed_units = completed_units + :delta
where job_id = :jobId
  and :delta > 0
  and completed_units + :delta <= total_units
returning completed_units, total_units;
```

Zero returned rows là no-apply. New value đến từ cùng atomic statement, không cần
SELECT sau update để đoán.

### Database constraint

```sql
alter table job_progress
add constraint ck_job_progress_range
check (
    total_units >= 0
    and completed_units >= 0
    and completed_units <= total_units
);
```

Constraint là defense-in-depth cho range; atomic UPDATE bảo vệ delta composition.

## Solution 2 — Optimistic locking bằng `@Version`

### Entity

```java
@Entity
@Table(name = "job_progress")
public class JobProgress {
    @Id
    private UUID jobId;

    private int completedUnits;
    private int totalUnits;

    @Version
    private long version;

    public void addCompletedUnits(int delta) {
        // validate and mutate
    }
}
```

Generated update:

```sql
update job_progress
set completed_units = :newValue,
    total_units = :total,
    version = :nextVersion
where job_id = :jobId
  and version = :expectedVersion;
```

A affected rows `1`; B stale expected version affected rows `0`. Hibernate emits
optimistic conflict and B transaction rollback.

### Mỗi retry là transaction mới

```java
@Service
public class ProgressAttemptService {
    private final JobProgressRepository progress;
    private final ProgressCommandRepository commands;

    public ProgressAttemptService(
        JobProgressRepository progress,
        ProgressCommandRepository commands
    ) {
        this.progress = progress;
        this.commands = commands;
    }

    @Transactional
    public ProgressResult addOnce(
        UUID commandId,
        UUID jobId,
        int delta
    ) {
        ProgressCommand existing =
            commands.findByCommandId(commandId).orElse(null);
        if (existing != null) {
            return ProgressResult.replayed(existing);
        }

        JobProgress job = progress.findById(jobId)
            .orElseThrow();
        job.addCompletedUnits(delta);
        commands.save(ProgressCommand.applied(
            commandId,
            jobId,
            delta
        ));
        progress.flush();
        return ProgressResult.applied(commandId, jobId);
    }
}
```

```java
@Service
public class ProgressRetryCoordinator {
    private final ProgressAttemptService attempts;

    public ProgressRetryCoordinator(
        ProgressAttemptService attempts
    ) {
        this.attempts = attempts;
    }

    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 4,
        backoff = @Backoff(
            delay = 20,
            maxDelay = 200,
            multiplier = 2.0,
            random = true
        )
    )
    public ProgressResult add(
        UUID commandId,
        UUID jobId,
        int delta
    ) {
        return attempts.addOnce(commandId, jobId, delta);
    }
}
```

Coordinator không transactional; attempt bean có transaction mới cho mỗi call.
Example timing values không phải production benchmark. Retry reloads A's committed
`13`, rồi calculates `17`.

Optimistic solution phù hợp khi mutation logic phức tạp hơn simple delta và
conflicts tương đối hiếm. Hot key có thể tạo retry amplification.

## Solution 3 — Pessimistic row lock trước read

```java
public interface JobProgressRepository
        extends JpaRepository<JobProgress, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select j from JobProgress j where j.jobId = :jobId")
    Optional<JobProgress> findByIdForUpdate(UUID jobId);
}
```

```java
@Transactional
public ProgressResult addCompletedUnits(
    UUID jobId,
    int delta
) {
    JobProgress job = progress.findByIdForUpdate(jobId)
        .orElseThrow();
    int before = job.getCompletedUnits();
    job.addCompletedUnits(delta);
    return ProgressResult.applied(
        jobId,
        before,
        job.getCompletedUnits()
    );
}
```

Timeline:

```text
A locks/read 10 -> writes 13 -> commit
B waits before read -> locks/read 13 -> writes 17 -> commit
```

Loser blocks thay vì fail/retry. Giữ transaction ngắn, deterministic order nếu
lock nhiều rows, và có lock/statement deadline. Không gọi remote I/O khi giữ lock.

## Solution 4 — Compare-and-set old value

Khi muốn optimistic conflict mà không dùng entity version:

```sql
update job_progress
set completed_units = :newValue
where job_id = :jobId
  and completed_units = :expectedOldValue;
```

Affected rows `0` báo stale read. Application rollback/retry trong transaction mới
và reload. Cách này chỉ an toàn nếu compared columns đại diện đầy đủ state ảnh
hưởng decision; explicit version thường rõ hơn cho aggregate.

## Solution 5 — Stronger isolation với full retry

`REPEATABLE READ`/`SERIALIZABLE` có thể abort one updater bằng SQLSTATE `40001`.
Application phải:

- để failed transaction rollback;
- retry toàn unit trong transaction mới;
- reload/revalidate;
- giữ attempt/deadline bounded;
- bảo vệ external side effects/idempotency.

Đây là lựa chọn cho invariants rộng hơn một atomic row expression. Với counter
delta đơn giản, atomic SQL thường có throughput/operational contract rõ hơn.

## Solution 6 — Durable progress commands/events

Nếu reconciliation/audit quan trọng, lưu mỗi distinct completion command:

```sql
create table progress_command (
    command_id uuid primary key,
    job_id uuid not null,
    delta integer not null check (delta > 0),
    applied_at timestamptz not null
);
```

Insert command và update projection trong cùng transaction. Unique command ID ngăn
duplicate delivery; atomic delta bảo vệ distinct concurrent commands.

```text
duplicate-command prevention != concurrent-mutation safety
```

Append-only commands cho phép rebuild/reconcile projection nhưng tăng storage và
workflow complexity.

## Conflict và loser behavior

| Solution | Conflict detector | Loser |
| --- | --- | --- |
| Atomic delta | Row lock + current predicate | Wait, then apply/no-apply |
| `@Version` | Affected rows `0` | Fail, rollback, bounded retry |
| `FOR UPDATE` | Incompatible row lock | Block/timeout, then fresh read |
| Compare-and-set | Old-value predicate | Affected rows `0`, retry/fail |
| Serializable | Serialization graph | SQLSTATE `40001`, full retry |

## Trade-off comparison

| Lựa chọn | Correctness | Contention behavior | DB load | Complexity |
| --- | --- | --- | --- | --- |
| Atomic delta | Cao cho expressible single-row intent | Short row serialization | Thấp | Thấp |
| Optimistic version | Detects stale aggregate | Retry under conflict | Tăng theo retries | Vừa |
| Pessimistic lock | Fresh read under lock | Blocking queue | Connections held while wait | Vừa |
| Strong isolation | Bảo vệ rộng hơn | Abort/retry | Có thể cao | Vừa-cao |
| Event ledger + projection | Audit/rebuild | Cần atomic/idempotent apply | Storage thêm | Cao |

## Recommendation cho case này

1. Dùng atomic conditional delta và check affected-row count.
2. Thêm database range constraint.
3. Dùng durable command ID nếu delivery có thể duplicate.
4. Chỉ dùng optimistic/pessimistic/serializable khi mutation/invariant phức tạp hơn.
5. Integration test bằng PostgreSQL với concrete two-actor interleaving.
6. Reconcile accepted delta sum với projection khi business critical.

## Production checklist

### Write contract

- [ ] Intent được biểu diễn là delta/conditional mutation khi có thể.
- [ ] Affected-row count map thành explicit result.
- [ ] No stale absolute write by ID only.
- [ ] Constraint bảo vệ current-row range.
- [ ] Duplicate command và concurrent mutation được xử lý riêng.

### Alternative coordination

- [ ] Optimistic retry dùng transaction/context mới.
- [ ] Pessimistic lock acquire trước read/calculate.
- [ ] Lock transaction ngắn và có timeout budget.
- [ ] Serialization failures được allowlist/retry bounded.
- [ ] Multi-instance correctness nằm ở PostgreSQL.

### Operations

- [ ] Conflict/lock/retry metrics theo aggregate key an toàn.
- [ ] Projection có reconciliation source khi cần.
- [ ] Test assert final invariant, không chỉ response success.
- [ ] Effective isolation được xác nhận.
- [ ] Hot-key strategy được benchmark bằng workload đại diện.
