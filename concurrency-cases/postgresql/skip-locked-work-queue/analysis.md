# Phân tích claiming, fairness và crash recovery

## Trạng thái ban đầu

Jobs J1–J4 đều `READY`, cùng priority, `available_at` tăng dần. Worker A và B poll
trên hai connections/transactions khác nhau.

## Timeline plain `SELECT` tạo duplicate

| Bước | Worker A | Worker B |
| ---: | --- | --- |
| 1 | đọc J1 | |
| 2 | | đọc J1 |
| 3 | gọi external effect cho J1 | gọi cùng effect cho J1 |
| 4 | ghi DONE | ghi DONE |

Read → process → write không atomic. Final row DONE không chứng minh effect chỉ
chạy một lần.

## Timeline `FOR UPDATE` tạo convoy

| Bước | Worker A | Worker B |
| ---: | --- | --- |
| 1 | lock J1 | |
| 2 | giữ J1 để xử lý | xin J1 và block |
| 3 | J2/J3 vẫn READY | không tiến tới J2 |
| 4 | commit | mới acquire/re-evaluate |

Không có wait-for cycle nên đây không phải deadlock. Một hot/slow first row có
thể serialize pollers và giữ nhiều pool connections.

## Timeline với `SKIP LOCKED`

| Bước | Worker A | Worker B |
| ---: | --- | --- |
| 1 | lock J1 | |
| 2 | | skip J1, lock J2 |
| 3 | update J1 → PROCESSING | update J2 → PROCESSING |
| 4 | commit claim | commit claim |
| 5 | process J1 ngoài transaction | process J2 ngoài transaction |

Hai workers nhận disjoint rows. Row lock lifetime chỉ bao claim statement/
transaction, không bao external work.

> **Nói ngắn gọn:** `SKIP LOCKED` đổi “chờ đúng row đầu” thành “lấy một row khả
> dụng khác”; đổi lại, strict FIFO không còn được bảo đảm.

## MVCC và lock behavior

Ở `READ COMMITTED`, claim statement lấy statement snapshot. Candidate scan:

1. lọc committed rows thỏa `READY`/`available_at`;
2. theo `ORDER BY` tìm candidate;
3. thử acquire `FOR UPDATE` row lock;
4. row chưa thể lock ngay bị skip;
5. dừng khi đủ `LIMIT`;
6. `UPDATE ... RETURNING` đổi ownership trong cùng statement.

Worker khác không thấy uncommitted `PROCESSING` version nhưng gặp row lock và skip.
Sau commit, row không còn thỏa READY predicate.

`SKIP LOCKED` chỉ bỏ qua row-level lock. `SELECT FOR UPDATE` vẫn lấy table-level
`ROW SHARE`; incompatible `ACCESS EXCLUSIVE` DDL có thể chặn query. Pool wait,
I/O, query execution và transaction commit cũng còn latency.

## Vì sao CTE claim atomic?

Candidate SELECT nằm trong CTE của cùng top-level UPDATE:

```text
lock candidate rows
→ update chính rows đó
→ return token/state
→ commit
```

Không có cửa sổ để application trả IDs rồi transaction khác claim trước UPDATE.
Nếu transaction rollback/crash trước commit, cả status/token/attempt increment
rollback và locks release.

Mỗi returned row phải mang token do database vừa ghi. Caller không bắt đầu work
trước khi proxy/template commit thành công.

## View không nhất quán là feature có chủ ý

Một worker không thấy locked READY rows trong result của poll đó. Vì thế:

- không dùng kết quả để tính queue depth, billing hoặc business report;
- không kết luận “không còn job” khi batch rỗng; jobs có thể đang locked;
- không dùng strict FIFO claim làm correctness invariant;
- phù hợp khi goal là phân phối work khả dụng giữa consumers.

Dashboard dùng plain aggregate/read replica strategy riêng, không dùng
`SKIP LOCKED` result làm toàn bộ queue view.

## Fairness và starvation

`ORDER BY priority DESC, available_at, job_id` tạo preference ổn định nhưng không
tạo strict fairness. J1 có thể liên tục bị skip nếu:

- transaction khác giữ lock lâu;
- J1 fail/requeue ngay với priority cao;
- workers luôn fill batch bằng jobs priority cao mới;
- claim query/index plan không phù hợp.

Containment:

- transaction claim cực ngắn;
- bounded batch;
- retry `available_at` với backoff;
- max attempts và `DEAD`/dead-letter state;
- priority aging hoặc reserved capacity nếu business cần;
- metric oldest READY age, not only queue depth;
- sweeper phát hiện expired lease/long lock.

Fairness là policy cần test/observe, không phải guarantee tự động của
`SKIP LOCKED`.

## Lease và ownership epoch

Claim tạo `claim_token` mới và `lease_until`. Token đại diện ownership epoch:

```text
J1/token-A expires
→ sweeper requeue
→ Worker B claim J1/token-B
→ Worker A resume với token-A
→ conditional complete affected-row 0
```

Token ngăn stale worker mutate queue row. Nó không tự ngăn external sink nhận
effect cũ; sink cần idempotency key/fencing support riêng.

Lease duration phải lớn hơn expected processing nhưng không quá lớn đến mức crash
recovery chậm. Long jobs cần heartbeat/lease extension conditional theo current
token và có maximum execution policy.

## Failure matrix

| Failure point | Database state | Recovery |
| --- | --- | --- |
| Trước claim commit | Rollback, job vẫn READY | Poller khác claim |
| Sau claim commit, trước work | PROCESSING tới lease | Sweeper requeue |
| Trong external work | Effect có thể fail/không rõ | Retry idempotently |
| Effect xong, trước complete | Job sẽ chạy lại | Sink dedupe theo job/effect key |
| Stale worker complete | Token mismatch, affected-row `0` | Worker bỏ result, log metric |
| Complete commit xong, response mất | DONE durable | Re-read state/token |
| Sweeper crash | Expired rows còn PROCESSING | Sweeper khác/idempotent next run |

## At-least-once, không phải exactly-once

Database transaction không bao external service. Crash sau effect trước DONE tạo
redelivery hợp lệ. Các lựa chọn:

- external API nhận idempotency key = stable effect ID;
- ghi local outbox/inbox rồi delivery riêng;
- sink hỗ trợ conditional/fenced operation;
- reconciliation phát hiện effect/state lệch.

Không kéo remote call vào database transaction để “đạt exactly once”; cách đó chỉ
kéo dài lock và vẫn không có distributed atomic commit.

## Complete, fail và retry

Complete/fail đều conditional theo token:

```sql
where job_id = :jobId
  and status = 'PROCESSING'
  and claim_token = :claimToken
```

Failure có thể:

- retryable: set READY, `available_at = now() + backoff`, clear ownership;
- terminal: set DEAD, retain redacted error metadata;
- lost ownership: affected-row `0`, không thay đổi row.

Backoff tránh poison job chiếm đầu queue liên tục. Error payload/log không lưu
secret hoặc full sensitive response.

## Timeout, deadlock và isolation

Claim thường dùng `READ COMMITTED`. `SKIP LOCKED` giảm row wait nhưng không loại
mọi deadlock/table wait. Đặt bounded `statement_timeout`; `lock_timeout` vẫn hữu
ích cho locks khác.

Nếu claim/update chạm thêm tables theo trigger/foreign key, định nghĩa lock order
và xử lý `40P01`. `SERIALIZABLE` không cần thiết chỉ để claim rows và có thể tăng
abort/retry.

## Crash và transaction release

Connection mất trước commit làm PostgreSQL rollback và release row locks. Sau
claim commit, process crash không rollback ownership vì claim đã durable; lease
recovery là bắt buộc.

Dùng database time (`clock_timestamp()`/`CURRENT_TIMESTAMP` theo semantics mong
muốn) để tránh lệch clock giữa pods. Trong một statement, chọn rõ liệu cần
transaction timestamp hay wall-clock hiện tại.

## Multi-instance

PostgreSQL row locks coordinate mọi application instance. JVM mutex không cần cho
correctness claim và có thể làm một node tự serialize.

Scale-out tăng concurrent claim queries. Cần giới hạn worker concurrency, batch
size, polling interval/jitter và tổng connection budget. Empty poll nên backoff,
không busy-loop.

## Nguyên nhân gốc theo layer

| Layer | Vấn đề/cơ chế |
| --- | --- |
| Application | Plain read/process/write hoặc giữ transaction qua remote work |
| Spring | Proxy boundary quyết định claim đã commit trước processing hay chưa |
| Hibernate/JDBC | Native CTE/affected-row/token mapping |
| PostgreSQL | MVCC, row locks, skip semantics và atomic UPDATE |
| External sink | Idempotency quyết định duplicate effect |

## Khả năng quan sát (`observability`)

Theo dõi:

- READY depth và oldest READY age theo priority;
- claim batch size/latency/empty polls;
- PROCESSING age, expired leases, reclaim và DEAD count;
- stale completion affected-row `0`;
- attempt distribution và external idempotency replay;
- pool active/pending, statement timeout và `40P01`;
- `pg_stat_activity`, `pg_locks`, `pg_blocking_pids` khi claim bất ngờ chờ.

Không dùng worker/job ID làm metric label high-cardinality; giữ chúng trong
structured trace/log có sampling.
