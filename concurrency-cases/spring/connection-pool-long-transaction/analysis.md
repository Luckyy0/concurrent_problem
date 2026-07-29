# Pool-starvation analysis

## Initial state

Giả sử một instance có pool capacity `3` trong scenario minh họa:

```text
idle connections   = 3
active connections = 0
pending borrowers  = 0

P-42 status = RISK_PENDING, version = 12
P-99 status = RISK_PENDING, version = 4
```

Remote risk dependency đang chậm nhưng chưa fail. Đây là finite slow wait, không
phải connection leak; connections sẽ được trả nếu transactions cuối cùng kết thúc.

## Timeline cascading failure

| Bước | Request A — P-42 | Duplicate B — P-42 | Request C — P-99 | Unrelated U |
| --- | --- | --- | --- | --- |
| T0 | Borrow conn-1 | | | |
| T1 | Lock P-42 | | | |
| T2 | Chờ remote | Borrow conn-2 | | |
| T3 | | Chờ lock P-42 | Borrow conn-3 | |
| T4 | | | Lock P-99, chờ remote | |
| T5 | Pool active=3 | | | Chờ connection |
| T6 | Vẫn chờ | Vẫn chờ | Vẫn chờ | Acquisition timeout |
| T7 | Remote timeout/return | | | Caller U đã fail |
| T8 | Rollback/commit, trả conn-1 | B mới acquire lock | | Retry traffic có thể đến |

Không có circular wait:

```text
B waits for A row lock
A waits for remote service
C waits for remote service
U waits for any pool connection
```

PostgreSQL deadlock detector không thể phá chuỗi này vì remote service/pool không
tạo database wait-for cycle.

> **Nói ngắn gọn:** mọi wait đều có thể “hợp lệ” riêng lẻ nhưng cùng giữ finite
> resources theo thứ tự xấu, tạo cascading starvation.

## Connection được acquire và giữ khi nào?

Spring có thể tạo logical transaction trước khi JDBC connection được lấy lazily.
Trong case này, `findByIdForUpdate()` chắc chắn cần connection. Sau SELECT:

- EntityManager/transaction bind connection vào request thread;
- Hibernate không trả connection cho pool giữa transaction;
- row lock thuộc PostgreSQL transaction;
- remote call không làm transaction end;
- connection chỉ trở lại pool sau commit/rollback/cleanup.

Không đồng nhất một open EntityManager với active JDBC connection trong mọi config.
Evidence cần lấy từ pool metrics và PostgreSQL sessions, không chỉ từ annotation.

## PostgreSQL activity nhìn thấy gì?

Trong timeline:

- A có thể hiện `idle in transaction` sau SELECT khi Java đang chờ HTTP;
- B hiện `active`, `wait_event_type = 'Lock'` khi `FOR UPDATE` block;
- C có thể `idle in transaction` trên order khác;
- U chưa có PostgreSQL backend/query vì còn chờ trong Hikari pool.

Diagnostic query bằng một admin connection độc lập:

```sql
select pid,
       application_name,
       state,
       wait_event_type,
       wait_event,
       xact_start,
       query_start,
       pg_blocking_pids(pid) as blockers,
       query
from pg_stat_activity
where datname = current_database()
order by xact_start nulls last;
```

`idle in transaction` cũng có thể do application bug/path khác; correlation ID,
stack trace/telemetry và pool usage phải nối evidence.

## Lock lifetime bằng transaction lifetime

A acquire row lock ở T1. Lock không release khi:

- repository method return;
- JPA entity được map sang DTO;
- remote call bắt đầu;
- request thread không dùng CPU;
- another method annotation kết thúc nếu nó chỉ join same physical transaction.

Lock release ở commit/rollback. Vì vậy remote p99 latency trực tiếp kéo dài lock
p99 và queue của duplicate commands.

## Different-row starvation

Row lock contention không phải điều kiện bắt buộc. Nếu mỗi request lock một order
khác rồi gọi cùng slow dependency, mọi connection vẫn active.

Qualitative capacity relation:

```text
in-use connections
  ≈ transaction arrival rate × average transaction duration
```

Transaction duration tăng theo remote wait làm required concurrency tăng dù query
rate/CPU PostgreSQL không đổi. Pool chỉ là queue/capacity boundary, không tạo thêm
database throughput.

## Same-row amplification

Trên một hot payment:

1. A giữ lock và gọi remote.
2. B..N borrow connections rồi block ở `FOR UPDATE`.
3. Chỉ một request có useful remote work.
4. Waiters làm pool cạn.
5. Khi A xong, requests tuần tự acquire lock nhưng có thể vẫn gọi remote dù state
   đã đổi nếu code không revalidate sớm.

Lock waiter phải reload/check status ngay sau acquire và return stored/no-op result
nếu command đã terminal. Nếu không, duplicated expensive work kéo dài queue.

## Pool acquisition timeout là symptom

U có thể nhận:

```text
SQLTransientConnectionException
  -> CannotGetJdbcConnectionException
  -> HTTP 5xx/timeout
```

Không có SQL của U để tune. Query vốn ngắn nhưng không được cấp connection.

Tăng acquisition timeout:

- tăng request/thread wait;
- có thể vượt upstream deadline;
- không giải phóng active connections;
- làm queue dài hơn.

Timeout ngắn hơn giúp fail-fast/containment nhưng không sửa root boundary.

## Tăng pool size có thể làm xấu hơn

Pool lớn hơn cho phép nhiều long transactions hơn:

- nhiều PostgreSQL backends/memory;
- nhiều row-lock waiters;
- nhiều remote requests;
- transaction snapshots lâu hơn;
- recovery burst lớn hơn khi dependency trở lại.

Trong multi-instance deployment:

```text
instances × maximumPoolSize
```

có thể vượt database connection budget. Autoscaling theo request latency đôi khi
thêm instances/pools đúng lúc database đã quá tải, tạo positive feedback.

Pool size phải dựa trên database capacity và measured workload; boundary/backpressure
mới xử lý duration/concurrency.

## Executor coupling

Transaction-owning request thread có thể:

- submit risk task;
- giữ DB connection;
- block chờ future;
- risk executor queue đầy;
- thêm request threads tiếp tục borrow connections rồi chờ.

Nếu risk task lại cần cùng request executor hoặc database connection, resource
dependency graph còn nguy hiểm hơn. Không block trong transaction trên work có
queue/capacity độc lập.

Bulkhead giới hạn remote concurrency trước khi database transaction bắt đầu. Khi
bulkhead full, reject/queue bounded mà không giữ JDBC connection.

## Timeout budget

Một overall deadline `D` cần bao gồm:

```text
admission/bulkhead wait
+ remote connect/read
+ DB pool acquisition
+ lock/statement work
+ commit/rollback cleanup
+ response margin
<= D
```

Không cộng các timeout độc lập lớn hơn caller deadline. Mỗi phase nhận remaining
budget, và phase mới không bắt đầu nếu không đủ thời gian hoàn tất an toàn.

Spring transaction timeout không phải general-purpose interrupt cho arbitrary
HTTP/future wait. Remote client và executor wait cần deadline/cancellation riêng.

## Rollback và timeout behavior

Nếu remote timeout ném unchecked exception ra khỏi transaction boundary:

- Spring rollback;
- PostgreSQL release row lock;
- Hikari nhận connection lại;
- waiter tiếp theo tiến lên.

Nhưng cleanup chỉ xảy ra sau remote wait kết thúc/được interrupt. Một HTTP client
không có bounded connect/read timeout có thể giữ transaction rất lâu.

Nếu code catch timeout rồi tiếp tục/commit intermediate state, failure contract
phải explicit. Case khuyến nghị không bắt đầu transaction trước remote wait.

## Client cancellation

Upstream client timeout không tự bảo đảm server work dừng. Nếu server không
propagate cancellation/deadline:

- remote call tiếp tục;
- transaction/connection vẫn bị giữ;
- response bị bỏ nhưng resource cost còn;
- retry request có thể đến song song.

Cancellation phải trigger cleanup nhưng không được để transaction lửng. Luôn end
bằng commit hoặc rollback trong bounded path.

## Fixed boundary cần revalidation

Move remote call ra ngoài transaction tạo một concurrency window:

```text
read snapshot v12
call risk(snapshot v12)
another actor changes payment to CANCELLED/v13
commit phase reloads v13
```

Không được apply decision v12 mù quáng. Short commit transaction phải check:

- payment vẫn `RISK_PENDING`;
- current version/amount/customer match assessed subject;
- decision chưa expired;
- command/idempotency state;
- no terminal result already exists.

Nếu mismatch, return stale/no-op hoặc chạy orchestration lại ngoài transaction theo
bounded policy.

## Async workflow

Khi remote latency dài hoặc cần durable retry:

1. Tx-1 transition `RISK_PENDING` và insert outbox event.
2. Commit/release connection.
3. Worker call remote ngoài DB transaction.
4. Tx-2 conditionally apply decision nếu state/version còn match.
5. Store outcome/idempotency for redelivery.

Workflow không giữ transaction qua message/network boundary. Nó đổi synchronous
latency và cần state-machine/recovery complexity.

## Readiness và cascading operations

Pool starvation có thể làm:

- DB-backed readiness fail;
- orchestrator restart healthy-but-overloaded pods;
- new pods mở thêm pools;
- migrations/admin/reconciliation thiếu connection;
- retries tăng load.

Không dùng một special admin pool như cách che service failure. Nếu có reserved
operational capacity, nó phải nhỏ, isolated và nằm trong database connection
budget; business traffic vẫn cần backpressure.

## Scope: starvation, không phải leak/deadlock

- **Leak:** connection không được return do cleanup bug.
- **Starvation:** connections đang được giữ hợp lệ nhưng quá lâu.
- **Deadlock:** wait-for cycle cần victim/ordering.
- **Pool exhaustion:** không có idle connection cho borrower.

Metrics có thể giống nhau ở bề mặt. Transaction age, lock graph, thread state và
remote latency phân biệt root cause.

## Multi-instance implications

JVM semaphore trên một instance chỉ giới hạn local load. Cross-instance protection
cần:

- per-instance bulkhead cộng global dependency capacity planning;
- load balancer/admission control;
- database `max_connections` budget;
- keyed queue/state machine khi cần serialize aggregate;
- metrics aggregated across instances.

JVM lock không release PostgreSQL lock và không ngăn node khác borrow connection.

## Observability

Correlate:

- Hikari active/idle/pending/max và acquisition timeout;
- connection usage/transaction duration histogram;
- remote latency, timeout, bulkhead active/queued/rejected;
- executor active/queue/rejection;
- PostgreSQL transaction age, `idle in transaction`, lock waits/blockers;
- request deadline remaining;
- order/command ID đã redact/hash phù hợp;
- instance count × configured pool capacity.

Alert trên leading indicators như rising usage duration/pending count, không chờ
pool timeout rate tăng mới phản ứng.
