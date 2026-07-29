# Giải pháp atomic claim, owner token và lease recovery

## Mục tiêu thiết kế

Claim, process và complete là ba boundary riêng:

```text
Tx-claim → COMMIT → external process → Tx-complete
```

Không giữ connection/row lock qua external work. Queue state chấp nhận
at-least-once và từ chối stale owner.

## Giải pháp 1 — Atomic batch claim bằng CTE

### DTO

```java
public record ClaimedJob(
        UUID jobId,
        String jobType,
        String payload,
        UUID claimToken,
        Instant leaseUntil,
        int attemptCount
) {
}
```

### SQL

```sql
with candidates as (
    select job_id
    from work_job
    where status = 'READY'
      and available_at <= clock_timestamp()
    order by priority desc, available_at, job_id
    for update skip locked
    limit :batch_size
)
update work_job j
set status = 'PROCESSING',
    claim_token = gen_random_uuid(),
    claimed_by = :worker_id,
    lease_until = clock_timestamp() + (:lease_seconds * interval '1 second'),
    attempt_count = j.attempt_count + 1
from candidates c
where j.job_id = c.job_id
returning j.job_id,
          j.job_type,
          j.payload::text,
          j.claim_token,
          j.lease_until,
          j.attempt_count;
```

Deterministic `ORDER BY` và partial index:

```sql
create index ix_work_job_claim
    on work_job(priority desc, available_at, job_id)
    where status = 'READY';
```

Index usefulness phải được xác minh bằng production-like data/query plan; không
ép plan chỉ để giữ một predicate-lock/lock shape tưởng tượng.

### Claimer dùng JDBC cho `UPDATE RETURNING`

```java
@Repository
public class JdbcJobClaimer {

    private final NamedParameterJdbcTemplate jdbc;

    public JdbcJobClaimer(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public List<ClaimedJob> claim(
            String workerId,
            int batchSize,
            Duration lease
    ) {
        if (batchSize < 1 || batchSize > 100) {
            throw new IllegalArgumentException("invalid batch size");
        }
        var parameters = new MapSqlParameterSource()
                .addValue("worker_id", workerId)
                .addValue("batch_size", batchSize)
                .addValue("lease_seconds", lease.toSeconds());

        return jdbc.query(CLAIM_SQL, parameters, (row, index) ->
                new ClaimedJob(
                        row.getObject("job_id", UUID.class),
                        row.getString("job_type"),
                        row.getString("payload"),
                        row.getObject("claim_token", UUID.class),
                        row.getTimestamp("lease_until").toInstant(),
                        row.getInt("attempt_count")
                )
        );
    }
}
```

### Transactional claim service

```java
@Service
public class JobClaimService {

    private final JdbcJobClaimer claimer;

    public JobClaimService(JdbcJobClaimer claimer) {
        this.claimer = claimer;
    }

    @Transactional(
            propagation = Propagation.REQUIRES_NEW,
            isolation = Isolation.READ_COMMITTED
    )
    public List<ClaimedJob> claimBatch(
            String workerId,
            int batchSize,
            Duration lease
    ) {
        return List.copyOf(claimer.claim(workerId, batchSize, lease));
    }
}
```

Caller chỉ nhận list sau proxy commit. `REQUIRES_NEW` phù hợp vì polling root
không có outer unit; nếu caller đang giữ transaction, guard/architecture phải
ngăn independent boundary ngoài ý muốn.

> **Nói ngắn gọn:** một SQL statement chọn, khóa và đổi owner; transaction commit
> trước khi job rời database boundary.

## Giải pháp 2 — Processor ngoài transaction

```java
@Component
public class JobWorker {

    private final JobClaimService claims;
    private final JobCompletionService completions;
    private final ExternalProcessor processor;

    public JobWorker(
            JobClaimService claims,
            JobCompletionService completions,
            ExternalProcessor processor
    ) {
        this.claims = claims;
        this.completions = completions;
        this.processor = processor;
    }

    public int poll(String workerId) {
        List<ClaimedJob> batch = claims.claimBatch(
                workerId,
                20,
                Duration.ofMinutes(2)
        );

        for (ClaimedJob job : batch) {
            try {
                processor.process(job.jobId(), job.payload());
                completions.complete(job.jobId(), job.claimToken());
            } catch (RetryableProcessingException failure) {
                completions.retryLater(
                        job.jobId(),
                        job.claimToken(),
                        failure.safeCode()
                );
            } catch (RuntimeException failure) {
                completions.deadLetter(
                        job.jobId(),
                        job.claimToken(),
                        failure.getClass().getSimpleName()
                );
            }
        }
        return batch.size();
    }
}
```

`processor` nhận stable idempotency/effect key derived từ `jobId`. Sample xử lý
tuần tự trong một poll; production có thể đưa batch vào bounded executor nhưng
không vượt connection/downstream capacity.

Không claim nhiều hơn số job worker có thể bắt đầu trước khi lease gần hết.
Nếu processing time biến động lớn, giảm batch, claim theo available execution
slot hoặc heartbeat từng current token; nếu không, job cuối batch có thể bị
reclaim trước khi worker bắt đầu xử lý.

## Complete/fail conditional theo token

```java
public interface WorkJobRepository extends JpaRepository<WorkJob, UUID> {

    @Modifying
    @Query(value = """
            update work_job
            set status = 'DONE',
                completed_at = clock_timestamp(),
                claimed_by = null,
                claim_token = null,
                lease_until = null
            where job_id = :jobId
              and status = 'PROCESSING'
              and claim_token = :claimToken
            """, nativeQuery = true)
    int complete(UUID jobId, UUID claimToken);

    @Modifying
    @Query(value = """
            update work_job
            set status = 'READY',
                available_at = clock_timestamp() + interval '30 seconds',
                claimed_by = null,
                claim_token = null,
                lease_until = null,
                last_error = :safeCode
            where job_id = :jobId
              and status = 'PROCESSING'
              and claim_token = :claimToken
            """, nativeQuery = true)
    int retryLater(UUID jobId, UUID claimToken, String safeCode);
}
```

Service:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void complete(UUID jobId, UUID token) {
    if (jobs.complete(jobId, token) != 1) {
        throw new LostJobOwnershipException(jobId, token);
    }
}
```

`retryLater`/`deadLetter` dùng cùng affected-row contract. Backoff được tính theo
attempt/policy; `30 seconds` chỉ minh họa SQL.

## Lease recovery

Sweeper atomically requeue expired claims:

```sql
with expired as (
    select job_id
    from work_job
    where status = 'PROCESSING'
      and lease_until < clock_timestamp()
    order by lease_until, job_id
    for update skip locked
    limit :batch_size
)
update work_job j
set status = case
        when attempt_count >= :max_attempts then 'DEAD'
        else 'READY'
    end,
    available_at = clock_timestamp() + interval '30 seconds',
    claimed_by = null,
    claim_token = null,
    lease_until = null,
    last_error = 'LEASE_EXPIRED'
from expired e
where j.job_id = e.job_id
returning j.job_id, j.status;
```

Nhiều sweepers có thể chạy; `SKIP LOCKED` chia expired rows. Token cũ mất hiệu lực.
Heartbeat cho long job cũng phải:

```sql
update work_job
set lease_until = clock_timestamp() + :lease
where job_id = :job_id
  and status = 'PROCESSING'
  and claim_token = :claim_token
  and lease_until >= clock_timestamp();
```

Affected-row `0` nghĩa ownership đã mất; worker phải dừng nếu có thể.

## Empty poll và backpressure

Batch rỗng không chứng minh queue rỗng; rows có thể locked hoặc chưa đến
`available_at`. Worker dùng bounded polling delay có jitter/adaptive backoff.
Admission control giới hạn:

- poller concurrency;
- batch size;
- in-flight external calls;
- per-tenant/type quotas;
- total database connections.

Không busy-loop `SKIP LOCKED` khi toàn bộ queue đang bận.

## Fairness policy

Base order:

```sql
order by priority desc, available_at, job_id
```

Nếu low priority starvation không chấp nhận được, dùng aging:

```text
effective priority = base priority + bounded waiting-age bucket
```

Hoặc tách queues/reserved worker capacity. Mọi policy phải có index/query-plan
review và metric oldest age theo class; strict FIFO không tương thích với skip.

## Failure behavior

| Outcome | Worker behavior |
| --- | --- |
| Claim trả empty | Backoff poll; không báo queue empty tuyệt đối |
| Claim transaction rollback | Không process returned objects |
| External retryable failure | Conditional requeue với backoff |
| Terminal failure/max attempts | Conditional DEAD |
| Lease expired | Sweeper reclaim/dead-letter |
| Stale complete | Affected-row `0`, không overwrite |
| Crash after effect | Redelivery; sink idempotency/reconciliation |
| `40P01`/timeout | Rollback short transaction; bounded retry theo policy |

## Các phương án khác

### `FOR UPDATE` không skip

Dùng khi strict ordering/serialization quan trọng hơn throughput và wait được
bounded. Không phù hợp nhiều independent jobs.

### Atomic `UPDATE ... WHERE status='READY'`

Claim một known job ID bằng affected-row check rất tốt. Với “lấy next N jobs”,
CTE skip-locked giải quyết selection + ownership trong database.

### Advisory lock

Thêm protocol key/unlock/crash complexity và mọi writer phải tuân thủ; row status,
token và lease vẫn cần. Không nhỏ hơn row claim ở đây.

### Message broker

Kafka/RabbitMQ/SQS cung cấp delivery, retention/backpressure/consumer primitives
chuyên dụng nhưng database side effects vẫn cần idempotency/outbox. Chọn theo
failure model, không vì `SKIP LOCKED` “không scale vô hạn”.

## So sánh trade-off

| Cách | Duplicate claim | Throughput/latency | Fairness | Crash recovery | Vận hành | Multi-instance |
| --- | --- | --- | --- | --- | --- | --- |
| CTE `SKIP LOCKED` + token/lease | Ngăn active duplicate | Parallel, short claim Tx | Best-effort | Có sweeper | Trung bình | Có |
| Blocking `FOR UPDATE` | Ngăn trong Tx | Convoy/wait | Gần order hơn | Cần status/lease | Thấp | Có |
| Plain SELECT | Không ngăn | Nhanh nhưng sai | Không đáng tin | Không | Thấp | Không đúng |
| Known-ID conditional UPDATE | Ngăn bằng affected row | Tốt cho known job | Caller chọn ID | Cần lease | Thấp | Có |
| Broker | Theo broker semantics | Scale/backpressure tốt hơn | Partition-dependent | Redelivery/DLQ | Cao | Có |

## Checklist trước production

- [ ] Claim SQL có predicate, deterministic order, bounded limit và partial index.
- [ ] Claim commit trước external processing.
- [ ] Complete/fail/heartbeat đều kiểm tra current token.
- [ ] Lease expiry, max attempts và DEAD recovery được định nghĩa.
- [ ] Downstream idempotency key dùng stable job/effect ID.
- [ ] Empty poll có jitter/backoff; worker concurrency bounded.
- [ ] Oldest READY/PROCESSING age và stale completion được alert.
- [ ] DDL/table-lock wait vẫn có timeout/coordination.
- [ ] Testcontainers chứng minh disjoint claims, convoy, skip và stale owner.
