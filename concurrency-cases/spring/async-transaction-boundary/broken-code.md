# Broken code

## Outer transaction gọi async processor ngay lập tức

```java
@Service
public class BrokenOrderPlacementService {
    private final OrderRepository orders;
    private final AsyncOrderProcessor asyncProcessor;
    private final PlacementProbe probe;

    @Transactional
    public long place(PlaceOrderCommand command) {
        Order order = orders.saveAndFlush(Order.pending(command.customerId()));

        asyncProcessor.process(order.getId());
        probe.afterAsyncDispatch();

        if (command.failAfterDispatch()) {
            throw new IllegalStateException("simulated outer rollback");
        }
        return order.getId();
    }
}
```

```java
@Service
public class AsyncOrderProcessor {
    private final OrderRepository orders;
    private final DispatchAttemptRepository attempts;

    @Async("orderExecutor")
    @Transactional
    public CompletableFuture<Void> process(long orderId) {
        Order order = orders.findById(orderId)
                .orElseThrow(() -> new IllegalStateException("order not found"));
        attempts.save(new DispatchAttempt(order.getId(), "STARTED"));
        return CompletableFuture.completedFuture(null);
    }
}
```

`saveAndFlush` gửi INSERT tới PostgreSQL nhưng outer transaction chưa commit.
Async worker mở transaction riêng; tại `READ COMMITTED`, nó không thấy uncommitted
row của caller.

## Variant commit orphan side effect

Nếu processor nhận DTO/orderId và ghi audit mà không query order, worker transaction
có thể commit `DispatchAttempt`. Sau đó caller ném exception và rollback order.
Database còn attempt tham chiếu logical ID không có committed order (hoặc foreign
key làm worker block/fail tùy schema/lock timing).

> **Nói ngắn gọn:** missing read và orphan write là hai mặt của cùng lỗi—hai
>transaction độc lập chạy không có commit ordering contract.

## Transaction context thực tế

```text
request-thread: Tx-Order [insert order ........ rollback/commit]
worker-thread:             Tx-Async [query/write ... commit]
```

`@Transactional` trên async method tạo transaction mới trên worker; nó không join
Tx-Order. Bỏ annotation thì worker repository calls vẫn có thể tự tạo transaction
ngắn, giống fragmentation ở SPR-001.

## Những cách sửa chưa đủ

- Truyền managed `Order` entity vào async method: entity detached/lazy state và
  không mang transaction theo.
- Dùng `TaskDecorator` copy `TransactionSynchronizationManager`: transaction,
  connection và EntityManager không an toàn để dùng xuyên thread.
- Gọi `future.join()` trong outer transaction: có thể block chờ worker đang cần
  row chưa commit, tăng nguy cơ starvation/deadlock-like wait.
- Chỉ thêm `REQUIRES_NEW` trên async method: xác nhận transaction độc lập, không
  tạo after-commit ordering.
- Catch async exception trong caller: caller thường đã return/commit trước khi
  exception xuất hiện.
- Dùng unbounded executor để “chạy nhanh hơn”: không sửa commit race và tạo overload.

## Điều kiện tái hiện

Outer transaction chưa commit; async method chạy trên thread khác; worker phụ
thuộc state/side effect của outer transaction; không có after-commit/outbox handoff.
