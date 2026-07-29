# Thread boundary, commit ordering và failure modes

## Trạng thái ban đầu

Order ID 42 chưa tồn tại ở committed database state. Request thread mở Tx-Order,
INSERT order và flush; worker chưa bắt đầu.

## Interleaving 1: async reader không thấy order

| Bước | Request thread / Tx-Order | Worker thread / Tx-Async | PostgreSQL committed state |
| --- | --- | --- | --- |
| 1 | INSERT order 42, flush | | chưa có 42 |
| 2 | submit async task | | chưa có 42 |
| 3 | dừng trước commit | bắt đầu transaction riêng | chưa có 42 |
| 4 | | `findById(42)` → empty | chưa có 42 |
| 5 | commit sau đó | worker đã fail | có 42 nhưng work mất |

PostgreSQL `READ COMMITTED` không đọc uncommitted INSERT của transaction khác.
Flush chỉ gửi SQL; nó không tạo external visibility.

## Interleaving 2: async side effect sống sót outer rollback

| Bước | Request thread | Worker thread | Kết quả |
| --- | --- | --- | --- |
| 1 | insert order trong Tx-Order | | uncommitted order |
| 2 | submit DTO/id | ghi audit/attempt trong Tx-Async | transaction độc lập |
| 3 | | commit attempt | side effect durable |
| 4 | ném exception, rollback order | | orphan attempt |

Foreign key có thể làm worker block hoặc fail thay vì commit, nhưng không biến
thiết kế thành atomic. Behavior phụ thuộc schema/lock timing và vẫn không có
commit-order contract.

> **Nói ngắn gọn:** async task có thể chạy quá sớm khi caller sẽ commit, hoặc chạy
>“quá thành công” khi caller cuối cùng rollback.

## Expected và actual

| Khía cạnh | Mong đợi | Broken flow |
| --- | --- | --- |
| Ordering | Async dispatch sau order commit | Dispatch trước terminal outcome |
| Visibility | Worker luôn đọc được committed order | Worker có thể thấy missing row |
| Rollback | Không side effect nếu order rollback | Worker transaction có thể commit |
| Context | Mỗi thread có boundary rõ | Developer tưởng worker join caller |
| Failure | Outcome được quan sát/retry | Exception có thể chỉ nằm trong future/log |
| Durability | Theo requirement explicit | Local queue có crash gap |

## Root cause theo layer

### Spring transaction context

Transaction manager bind resource như connection/EntityManager với request thread.
Executor thread không kế thừa binding. `@Transactional` trên async method chạy khi
worker invoke target và tạo/join transaction của worker thread, không phải caller.

### Async proxy

`@Async` cũng dựa vào Spring proxy; self-invocation có thể khiến method chạy đồng
bộ (SPR-001). Khi call đúng proxy, async interceptor submit work và caller tiếp
tục; không có implicit completion/commit dependency.

### Hibernate/JPA

Managed entity, lazy proxy và persistence context không an toàn để chia sẻ qua
thread. Truyền immutable event data/ID; worker load aggregate lại trong transaction
của nó sau commit.

### PostgreSQL

MVCC bảo vệ isolation giữa transactions, không suy luận causal relationship từ
order ID được truyền qua executor. Database chỉ biết commit/rollback của từng
transaction riêng.

## After-commit semantics

`@TransactionalEventListener(phase = AFTER_COMMIT)` chỉ invoke listener sau
successful commit; rollback không invoke. Listener mặc định không chạy nếu event
được publish ngoài transaction, trừ khi bật `fallbackExecution`—không nên bật mù
vì làm yếu contract.

Để database write trong listener rõ ràng, listener sau commit nên gọi một async
bean khác; worker method mở transaction mới. Không tiếp tục ghi bằng resource của
transaction vừa commit trên cùng thread.

## Failure, cancellation và executor overload

- Commit đã xong thì executor rejection không thể rollback order.
- Local after-commit event có thể mất nếu process crash trước/during handoff.
- `CompletableFuture.cancel(true)` là best-effort; worker I/O/transaction có thể
  vẫn commit.
- Void `@Async` exception cần `AsyncUncaughtExceptionHandler`; future-returning
  method cần caller/monitor quan sát exceptional completion.
- Bounded executor và rejection policy là bắt buộc; xem JVM-008.
- Worker retry cần idempotency vì caller/event có thể được phát lại ở durable flow.

## Context propagation

`TaskDecorator` phù hợp copy MDC, trace hoặc security context nếu policy cho phép.
Không copy `TransactionSynchronizationManager`, EntityManager hoặc JDBC connection
sang worker. Trace context propagation không thay đổi database atomicity.

## Multi-instance và durability

After-commit local listener đúng commit ordering trong node nhưng không durable.
Node crash sau commit làm work biến mất; scaling không giúp recover task. Nếu work
phải eventually execute, ghi outbox record trong Tx-Order và để relay/consumer
xử lý idempotently (`MSG-007`).

## Observability

Theo dõi order commit → async scheduled → started → succeeded/failed timestamps,
executor rejection/queue age, orphan/missing-order outcome, late completion sau
cancel và outbox lag nếu dùng durable flow. Correlation bằng immutable event ID.

Kiến thức nền: [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md).
