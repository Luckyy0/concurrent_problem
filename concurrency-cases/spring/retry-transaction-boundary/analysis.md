# Doomed transaction analysis

## Initial state

```text
inventory_item:
  sku       = BOOK-42
  available = 2
  version   = 7

reservation_record:
  no record for command A or B
```

Hai commands có distinct command IDs và mỗi command reserve quantity `1`.

## Versioned SQL

Sau khi load version 7, Hibernate tạo update tương đương:

```sql
update inventory_item
set available = :newAvailable,
    version = 8
where sku = 'BOOK-42'
  and version = 7;
```

Affected-row count là conflict signal:

```text
1 row -> expected version còn current, update thắng
0 row -> version đã đổi, stale write bị từ chối
```

PostgreSQL không hiểu `@Version`; Hibernate thêm predicate và kiểm tra row count.
Spring dịch conflict thành `ObjectOptimisticLockingFailureException` ở abstraction
layer phù hợp.

## Timeline hai actor

| Bước | Command A — Tx-A | Command B — Tx-B |
| --- | --- | --- |
| T0 | Load stock 2, version 7 | |
| T1 | | Load stock 2, version 7 |
| T2 | In-memory reserve -> 1 | In-memory reserve -> 1 |
| T3 | Flush UPDATE expected v7; affected 1 | |
| T4 | Commit stock 1, version 8; record A | |
| T5 | | Flush UPDATE expected v7; affected 0 |
| T6 | | Hibernate raises optimistic conflict; Tx-B doomed |
| T7 | | Broken loop catches, clears context, starts iteration 2 in Tx-B |
| T8 | | Reload/mutate may run, nhưng Tx-B không thể commit |
| T9 | | Outer boundary rollback hoặc phát late exception |

Correct continuation thay T7–T9:

| Bước | Command B |
| --- | --- |
| R1 | Conflict thoát khỏi attempt boundary |
| R2 | Tx-B rollback; persistence context đóng |
| R3 | Bounded backoff/deadline check ngoài transaction |
| R4 | Tx-B2 bắt đầu với persistence context mới |
| R5 | Reload stock 1, version 8; revalidate quantity |
| R6 | UPDATE expected v8 ảnh hưởng 1 row |
| R7 | Commit stock 0, version 9; record B |

## Expected versus broken actual

Expected:

```text
successful commands = A, B
final available      = 0
final version        = 9
reservation records  = 2
each attempt          = independent transaction
```

Broken:

```text
A commits
B counts multiple loop iterations
B iterations share one failed transaction/context
B cannot produce a valid second commit
final available/records do not match reported retry progress
```

Một loop iteration không phải transaction attempt nếu proxy boundary không được
cross lại.

> **Nói ngắn gọn:** retry count đo số lần gọi code; correctness cần đo số clean
> transactions bắt đầu sau khi previous transaction rollback.

## Vì sao transaction bị doomed?

Một failed JPA flush không phải recoverable statement-level result để application
tiếp tục mutate cùng persistence context. Optimistic failure chỉ ra unit of work
được tính từ stale state; transaction phải rollback.

Có hai lớp state:

1. **PostgreSQL transaction:** versioned UPDATE trả row count 0, bản thân SQL không
   nhất thiết đưa PostgreSQL session vào aborted state.
2. **JPA/Hibernate transaction:** optimistic failure làm current unit of work
   không còn hợp lệ để commit và được xử lý như rollback-only.

Với serialization failure/deadlock victim, PostgreSQL còn mạnh hơn: database abort
transaction; statements tiếp theo gặp “current transaction is aborted” cho tới
rollback.

Application policy nên coi cả hai là failed attempt cần full rollback, không cố
phân biệt để reuse current transaction.

## Persistence context không phải cache có thể reset tùy ý

Sau conflict, persistence context có thể chứa:

- `InventoryItem` với available `1`, version/state dựa trên stale version 7;
- `ReservationRecord` mới nhưng chưa commit;
- queued/failed actions trong Hibernate action queue;
- entity lifecycle state không còn đồng bộ với database.

`EntityManager.clear()` detach managed entities, nhưng:

- không rollback JDBC transaction;
- không reset rollback-only;
- không tạo Spring transaction synchronization mới;
- không undo external side effects;
- không bảo đảm provider cho phép tiếp tục sau failed flush.

`refresh()` cũng không biến failed unit of work thành valid unit. Correct cleanup
là rollback và close attempt context.

## Isolation ảnh hưởng reload nhưng không sửa boundary

Ở PostgreSQL `READ COMMITTED`, một query mới trong cùng transaction có thể nhìn
committed version 8 sau khi context được clear. Nhưng JPA transaction vẫn doomed,
nên fresh-looking data không tạo committable attempt.

Ở `REPEATABLE READ`, transaction snapshot còn không nhìn thấy version mới theo cách
một retry cần. Nâng/hạ isolation không thay requirement: transaction mới phải tạo
fresh snapshot.

## Row lock behavior tại versioned UPDATE

Optimistic read không giữ row lock xuyên business computation. Khi UPDATE chạy:

- updater acquire row-level lock;
- nếu A đang update, B có thể chờ A commit/rollback;
- sau A commit, PostgreSQL re-evaluate B predicate;
- `version = 7` không còn match, affected-row count là 0;
- lock release khi transaction B rollback.

Không có lost update: stale write bị từ chối. Lỗi SPR-006 nằm ở recovery path sau
conflict, không nằm ở optimistic detector.

## Conflict xuất hiện ở đâu?

Không có explicit `flush()`, target method có thể return trước khi SQL chạy.
Transaction interceptor flush/commit sau target invocation; method-local
`try/catch` không bắt được exception.

Có explicit `flush()`, catch thấy conflict sớm hơn nhưng vẫn ở bên trong outer
transaction. Muốn retry đúng, exception phải tiếp tục đi qua transaction
interceptor để rollback hoàn tất, rồi mới được retry coordinator catch.

Correct exception path:

```text
attempt target
  -> flush conflict
  -> transaction interceptor catches
  -> rollback/close context
  -> rethrow
  -> retry coordinator catches
  -> classify/backoff
  -> next proxy call creates new Tx
```

## Advisor ordering

Khi `@Retryable` và `@Transactional` cùng áp dụng, proxy/advisor chain quyết định
semantics:

```text
Correct:
RetryInterceptor
  -> TransactionInterceptor
       -> target attempt

Broken:
TransactionInterceptor
  -> RetryInterceptor
       -> target attempt(s)
```

Annotations nằm cạnh nhau không làm ordering hiển nhiên. Có thể cấu hình order,
nhưng separate coordinator/worker beans dễ review và test hơn; object boundary làm
mỗi retry call đi qua transaction proxy.

## Propagation và outer transaction

Attempt worker dùng `REQUIRED` tạo transaction mới khi coordinator non-
transactional. Nếu một caller vô tình mở outer transaction, worker sẽ join và lại
mất attempt boundary.

Các guardrails:

- coordinator entry không transactional;
- worker có thể dùng `REQUIRES_NEW` nếu independent attempt semantics được chấp
  nhận và outer caller thực sự có thể tồn tại;
- hoặc dùng `TransactionTemplate` với explicit propagation;
- architecture/integration test xác nhận physical transaction identity mỗi attempt.

`REQUIRES_NEW` có cost: suspend outer context, dùng thêm connection và independent
commit. Đừng thêm nó mà không phân tích atomicity.

## Retry classification

Candidate retryable failures:

- optimistic version conflict;
- PostgreSQL serialization failure, SQLSTATE `40001`;
- deadlock victim, SQLSTATE `40P01`, nếu operation safe;
- selected transient connection/lock failures theo explicit policy.

Non-retryable:

- insufficient stock trên freshly loaded state;
- invalid quantity;
- duplicate command với stored terminal result;
- constraint nói business invariant không thể thỏa;
- programming/mapping error;
- deadline/cancellation.

Classification dựa trên most specific cause/SQLSTATE, không dùng catch-all
`RuntimeException`.

## Backoff, jitter và retry amplification

Immediate retry trên hot SKU làm losers quay lại cùng lúc, tạo thêm conflicts.
Policy cần:

- finite `maxAttempts`;
- operation deadline bao phủ cả attempts;
- exponential hoặc adaptive backoff;
- jitter để desynchronize actors;
- cancellation/interruption propagation;
- metrics theo attempt và final outcome.

Backoff phải nằm ngoài transaction để không giữ connection, snapshot hoặc locks
trong lúc chờ. Không có một duration phổ quát; tune từ latency budget và observed
contention, không copy một con số như correctness constant.

## Domain revalidation

Retry không replay stale decision. Attempt mới reload và hỏi lại:

```text
stock còn đủ không?
command đã có terminal record chưa?
pricing/policy version còn phù hợp không?
deadline còn không?
```

Nếu command A lấy đơn vị cuối cùng, command B retry phải trả
`InsufficientStock`, không ép decrement vì attempt đầu từng pass validation.

## External side effect và idempotency

Không gửi email, charge payment hoặc publish non-transactional message trước
attempt commit rồi lặp lại chúng. Database retry rollback không undo external side
effect.

Dùng command ID/database uniqueness cho duplicate delivery và outbox cho
post-commit message khi phù hợp. Idempotency và optimistic retry là hai controls
khác nhau:

- idempotency ngăn cùng command apply nhiều lần;
- version conflict ngăn distinct concurrent mutations ghi đè nhau.

## Crash behavior

- Crash trong attempt trước commit: connection close làm transaction rollback.
- Crash sau commit trước response: command record/idempotency giúp xác định outcome.
- Crash trong backoff: không transaction nào nên đang mở.
- Retry worker restart phải đọc durable command/aggregate state, không memory-only
  attempt counter nếu workflow cần durable retry.

## Multi-instance

JVM lock hoặc in-memory retry counter trên App A không coordinate App B. PostgreSQL
version predicate là cross-instance conflict detector. Retry budget có thể local
cho synchronous request, nhưng global load protection cần admission control,
partitioning hoặc queue strategy phù hợp.

## Observability

Ghi nhận:

- operation/command ID, aggregate key và attempt number;
- loaded/expected version và conflict type;
- transaction completion status/identity trong diagnostics;
- time outside transaction dành cho backoff;
- attempts exhausted, deadline exceeded và final domain outcome;
- conflict rate theo hot key;
- `UnexpectedRollbackException` hoặc aborted-transaction follow-up errors.

Không coi “attempt 2 started” là bằng chứng clean retry. Evidence tốt là previous
rollback completed, new transaction began, version được reload và business result
committed đúng.
