# SPR-007 — Connection pool cạn vì long transactions

## Tóm tắt

`PaymentRiskService.assessAndApprove()` mở transaction, lock payment row bằng
`PESSIMISTIC_WRITE`, rồi gọi remote risk API. Khi remote latency tăng:

- lock holder giữ connection và row lock;
- duplicate requests chờ cùng row lock nhưng vẫn giữ connections;
- requests cho orders khác cũng giữ connections trong remote wait;
- pool hết connection;
- unrelated database traffic timeout khi acquire connection.

Invariant:

```text
Không giữ database connection/transaction/row lock trong lúc chờ remote I/O,
executor capacity hoặc unbounded coordination.
Remote wait phải có timeout/bulkhead riêng.
Database commit phase phải ngắn, bounded và revalidate state đã thay đổi.
Pool saturation phải tạo backpressure/fail-fast trước cascading failure.
```

> **Nói ngắn gọn:** connection pool không chỉ cạn vì query chậm; code không chạm
> database nhưng vẫn nằm trong transaction cũng giữ connection.

## Actors và shared resources

| Thành phần | Vai trò |
| --- | --- |
| Request A | Lock order P-42 rồi chờ remote risk decision |
| Duplicate B | Chờ row lock P-42, giữ connection khác |
| Requests C..N | Lock orders khác rồi cùng chờ remote |
| Unrelated request U | Cần connection cho query ngắn nhưng pool đã full |
| HikariCP | Cấp/thu hồi finite database connections |
| PostgreSQL | Giữ row locks tới commit/rollback |
| Remote risk API | Latency/failure domain độc lập database |

## Shared state và transaction boundary

```text
payment_order:
  id = P-42
  status = RISK_PENDING
  amount = 500
  business_version = 12
```

Broken boundary:

```text
BEGIN
  -> SELECT ... FOR UPDATE
  -> remote risk call / future wait
  -> UPDATE APPROVED
COMMIT
```

Recommended boundary:

```text
short snapshot read -> close connection
remote risk call with deadline/bulkhead
BEGIN short commit transaction
  -> lock/reload
  -> revalidate version/status/decision subject
  -> update or reject stale decision
COMMIT
```

## Contention point

Có hai finite resources:

1. row lock trên `payment_order`;
2. database connection trong application pool.

Requests cho cùng order tạo lock queue. Requests cho khác orders không conflict
row nhưng vẫn có thể chiếm toàn pool trong remote wait. Vì vậy pool starvation
không cần database deadlock.

## Expected và actual

| | Transaction duration | Pool | Unrelated query |
| --- | --- | --- | --- |
| Expected | Chỉ bao quanh short DB work | Còn headroom | Hoàn tất bình thường |
| Broken | Bằng DB + remote/executor wait | Active=max, pending tăng | Acquisition timeout |

Pool acquisition timeout của U là downstream symptom; tăng pool size chỉ dời điểm
sụp và có thể chuyển overload sang PostgreSQL.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| connection pool | Tập hữu hạn JDBC connections được tái sử dụng |
| pool starvation | Work cần connection nhưng mọi connection đều bị giữ |
| transaction duration | Từ begin/first acquisition đến commit/rollback |
| lock duration | Từ lock acquisition đến transaction end |
| pending acquisition | Thread đang chờ mượn connection |
| backpressure | Giới hạn/đẩy ngược load trước khi resource collapse |
| bulkhead | Giới hạn concurrency của một dependency để cô lập failure |
| timeout budget | Phân bổ overall deadline cho pool, lock, remote và cleanup |
| `idle in transaction` | Backend không chạy query nhưng transaction vẫn mở |

## Điều hướng

- [Broken long transaction](broken-code.md)
- [Pool-starvation analysis](analysis.md)
- [Short transaction and backpressure solutions](solutions.md)
- [Bounded saturation experiments](experiments.md)
- [PostgreSQL locks](../../concepts/postgresql-locks.md)
- [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- JDBC acquisition timeouts lan sang endpoints không liên quan.
- Row lock queues tăng transaction age và tail latency.
- Request threads/executors bị giữ, gây secondary thread starvation.
- Readiness/health dependencies có thể fail và kích hoạt restart/scale churn.
- Scale-out nhân tổng số connections, làm PostgreSQL quá tải nhanh hơn.
- Timeout retry mù tăng arrival rate và khuếch đại sự cố.

## Hướng sửa khuyến nghị

Tách remote/executor wait khỏi transaction. Lấy immutable snapshot, release
connection, gọi remote với deadline/bulkhead, sau đó mở short transaction để lock
và revalidate trước commit.

Với workflow dài hoặc remote availability thấp, dùng durable state machine/outbox
thay cho synchronous transaction.

## Phạm vi

Case phân tích resource starvation và cascading timeout, không phải deadlock
detection hoặc unknown outcome của remote side effect.
