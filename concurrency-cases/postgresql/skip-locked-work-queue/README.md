# DB-010 — Concurrent workers với `FOR UPDATE SKIP LOCKED`

## Tóm tắt

Nhiều worker poll cùng bảng `work_job`. Plain `SELECT` làm hai worker lấy cùng
job; `FOR UPDATE` không có `SKIP LOCKED` lại khiến tất cả đứng sau row đầu tiên.

Pattern khuyến nghị:

```text
transaction ngắn: claim READY jobs bằng FOR UPDATE SKIP LOCKED
→ COMMIT claim
→ xử lý external work ngoài transaction
→ transaction ngắn: complete nếu claim_token vẫn khớp
```

`SKIP LOCKED` bỏ qua row đang bị transaction khác khóa và cho worker lấy row tiếp
theo. PostgreSQL cố ý trả một view không nhất quán, phù hợp work queue nhưng không
phù hợp query nghiệp vụ tổng quát.

> **Nói ngắn gọn:** row lock chỉ bảo vệ bước claim; lease, owner token và
> idempotency mới bảo vệ vòng đời job sau commit/crash.

## Actor và trạng thái dùng chung

| Thành phần | Vai trò |
| --- | --- |
| `work_job` | Authoritative queue table |
| Worker A/B/C | Các process/pod poll song song |
| External sink | API, email, payment gateway hoặc processor |
| Recovery sweeper | Requeue claim đã hết lease |

Jobs có `status`, `priority`, `available_at`, `claim_token`, `claimed_by`,
`lease_until` và `attempt_count`.

## Invariant

```text
Một READY job chỉ được một active claim sở hữu tại một thời điểm.

Chỉ worker có claim_token hiện tại được complete/retry job.

Crash không làm job mất vĩnh viễn; claim hết lease có thể được reclaim.

Một job có thể được xử lý lại, nên external effect phải idempotent theo job ID.
```

Case cung cấp at-least-once processing, không tuyên bố exactly-once external side
effect.

## Ranh giới transaction

### Claim

Một transaction `READ COMMITTED` chạy CTE:

```sql
with candidates as (
    select job_id
    from work_job
    where status = 'READY'
      and available_at <= clock_timestamp()
    order by priority desc, available_at, job_id
    for update skip locked
    limit :batchSize
)
update work_job j
set status = 'PROCESSING',
    claim_token = gen_random_uuid(),
    claimed_by = :workerId,
    lease_until = clock_timestamp() + :lease,
    attempt_count = attempt_count + 1
from candidates c
where j.job_id = c.job_id
returning j.*;
```

Locks giữ tới commit; worker chỉ bắt đầu external work sau commit.

### Complete

Transaction khác dùng conditional update:

```sql
update work_job
set status = 'DONE',
    completed_at = clock_timestamp(),
    claim_token = null,
    lease_until = null
where job_id = :jobId
  and status = 'PROCESSING'
  and claim_token = :claimToken;
```

Affected-row `1` là complete; `0` nghĩa worker đã mất ownership và không được ghi
đè state mới.

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong case |
| --- | --- |
| work claiming | Atomically chuyển job từ available sang owned |
| `SKIP LOCKED` | Bỏ qua selected rows chưa thể row-lock ngay |
| lock convoy | Nhiều worker xếp hàng sau cùng một locked row |
| inconsistent view | Mỗi worker có thể thấy subset khác vì locked rows bị bỏ qua |
| lease | Ownership có hạn tới `lease_until` |
| claim token | Owner/fencing token của một lần claim |
| stale worker | Worker cũ tiếp tục sau khi job đã được reclaim |
| starvation | Job liên tục bị skip và chờ quá lâu |
| at-least-once | Job có thể chạy lại sau crash/ambiguous completion |

## Điều hướng

- [Code lỗi: duplicate và lock convoy](broken-code.md)
- [Lock, visibility, fairness và crash analysis](analysis.md)
- [Atomic claim, token và recovery](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [PostgreSQL locks và lock lifetime](../../concepts/postgresql-locks.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production

- duplicate external effects khi hai workers chọn cùng READY row;
- throughput gần như một worker khi mọi poller block ở oldest job;
- connection pool bị giữ trong lúc worker gọi remote service;
- job stuck `PROCESSING` sau process crash nếu không có lease recovery;
- stale worker complete đè claim mới nếu không có token predicate;
- priority thấp hoặc row hay bị lock có thể starve;
- scale-out làm polling/database load tăng nếu batch/backoff không giới hạn.

## Hướng sửa khuyến nghị

1. Claim bằng một statement/transaction ngắn với deterministic `ORDER BY`,
   bounded batch và `FOR UPDATE SKIP LOCKED`.
2. Commit claim trước external processing.
3. Complete/fail bằng `job_id + claim_token`.
4. Requeue expired claims có attempt cap/dead-letter policy.
5. External sink deduplicate theo stable job/effect key.
6. Theo dõi oldest-ready age, lease expiry, reclaim và stale completion.

## Khi phù hợp

Dùng cho queue-like table, workload vừa phải và database đã là authoritative
store. Kafka/RabbitMQ phù hợp hơn khi cần retention, partition scale, consumer
groups hoặc replay chuyên dụng.

## Phạm vi

Case xử lý PostgreSQL work claiming. Kafka partition assignment không thuộc
scope. General pessimistic-lock selection thuộc `LOCK-003`; messaging delivery
semantics và outbox thuộc các case messaging.
