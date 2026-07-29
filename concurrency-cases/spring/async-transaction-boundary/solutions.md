# Solutions: synchronous, after-commit và durable handoff

## Giải pháp 1: giữ work trong transaction khi cần atomicity

Nếu work là database mutation phải cùng commit với order, không dùng async:

```java
@Transactional
public long place(PlaceOrderCommand command) {
    Order order = orders.save(Order.pending(command.customerId()));
    reserveInventory(order, command);
    createRequiredDatabaseState(order);
    return order.getId();
}
```

Không gọi remote I/O dài trong transaction nếu có thể tránh. “Cùng transaction”
chỉ áp dụng resource được transaction manager điều phối; email/HTTP không rollback
cùng PostgreSQL.

## Giải pháp 2: transactional event sau commit rồi gọi async bean

Publisher chỉ ghi order và publish immutable event trong transaction:

```java
@Service
public class OrderPlacementService {
    private final OrderRepository orders;
    private final ApplicationEventPublisher events;

    @Transactional
    public long place(PlaceOrderCommand command) {
        Order order = orders.save(Order.pending(command.customerId()));
        events.publishEvent(new OrderPlacedEvent(order.getId()));
        return order.getId();
    }
}

public record OrderPlacedEvent(long orderId) {}
```

Listener chỉ dispatch sau successful commit:

```java
@Component
public class OrderPlacedAfterCommitListener {
    private final CommittedOrderProcessor processor;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterCommit(OrderPlacedEvent event) {
        processor.process(event);
    }
}
```

Async processor là bean khác nên call đi qua async proxy; nó mở transaction riêng
trên worker nếu cần database writes:

```java
@Service
public class CommittedOrderProcessor {

    @Async("orderExecutor")
    @Transactional
    public CompletableFuture<Void> process(OrderPlacedEvent event) {
        processCommittedOrder(event.orderId());
        return CompletableFuture.completedFuture(null);
    }
}
```

### Vì sao invariant local được bảo vệ

- Rollback không kích hoạt after-commit listener.
- Worker chỉ được submit sau commit, nên load order thấy committed state.
- Event chứa ID immutable, không chứa managed entity.
- Worker transaction có boundary riêng và explicit.

> **Nói ngắn gọn:** transaction quyết định “có được giao việc hay không”; executor
>chỉ quyết định “việc sau commit chạy ở thread nào”.

### Giới hạn

Commit đã thành công trước local handoff. Executor reject, process crash hoặc node
shutdown có thể làm work mất. Listener phải ghi metric/handle rejection; không thể
ném exception để rollback transaction đã commit.

## Giải pháp 3: TransactionSynchronization explicit

Khi không muốn event abstraction:

```java
TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                processor.process(new OrderPlacedEvent(orderId));
            }
        }
);
```

Cùng durability gap và overload concerns như local transactional event. Event
listener thường tách coupling tốt hơn; explicit synchronization phù hợp adapter
hạ tầng nhỏ và được test kỹ.

## Giải pháp 4: transactional outbox cho durable work

Trong Tx-Order, insert order và outbox row cùng transaction. Relay đọc committed
outbox, publish message; consumer xử lý idempotently và đánh dấu outcome. Đây là
lựa chọn khi work không được mất sau process crash hoặc cần multi-node recovery.

Case chỉ nêu selection boundary; schema, locking, redelivery và inbox/outbox
protocol được phát triển tại `MSG-007`.

## Executor và context configuration

- Bounded executor/queue, meaningful thread names, explicit rejection handler.
- Admission/load shedding phù hợp downstream capacity.
- `TaskDecorator` chỉ propagate approved MDC/trace/security context và cleanup
  trong `finally` để tránh thread-local leak.
- Không propagate transaction/EntityManager.
- Shutdown có deadline, ngừng admission và theo dõi unfinished tasks.

## Failure policy

- Future-returning async method: observe exception, retry có idempotency/deadline.
- Void method: cấu hình `AsyncUncaughtExceptionHandler`, không chỉ log mặc định.
- Retry missing-order trong broken design không phải fix; sửa commit ordering.
- Worker transaction rollback không rollback order đã commit; domain cần status
  `PROCESSING/FAILED` hoặc durable retry nếu outcome quan trọng.
- Cancellation best-effort, không được coi là rollback guarantee.

## So sánh

| Phương án | Commit ordering | Crash durability | Latency caller | Complexity |
| --- | --- | --- | --- | --- |
| Synchronous DB work | Cùng transaction | Database durable | Cao hơn | Thấp |
| After-commit + local async | Đúng | Không bảo đảm handoff | Thấp | Vừa |
| TransactionSynchronization | Đúng | Không bảo đảm handoff | Thấp | Vừa/coupled |
| Transactional outbox | Đúng | Có, với relay/retry | Thấp | Cao hơn |

## Production checklist

Boundary/durability requirement documented; event immutable; separate async bean;
worker transaction explicit; bounded executor; rejection/exception observable;
no managed entity across thread; idempotent processing; integration tests cho
commit, rollback, missing-state và crash-gap selection.
