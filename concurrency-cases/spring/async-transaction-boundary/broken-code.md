# Code lỗi — Giao việc bất đồng bộ sai thời điểm

## Giao dịch bên ngoài (Outer transaction) gọi bộ xử lý async ngay lập tức

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

Trong đoạn mã trên, `saveAndFlush` gửi lệnh `INSERT` tới PostgreSQL nhưng giao dịch bên ngoài chưa thực sự được commit. Worker bất đồng bộ sẽ mở một giao dịch riêng; ở mức độ cô lập `READ COMMITTED`, nó sẽ không thể nhìn thấy dòng dữ liệu chưa được commit của caller. Kết quả là hàm `process` ném ra ngoại lệ "order not found".

## Biến thể: Ghi nhận tác động mồ côi (Orphan side effect)

Nếu bộ xử lý nhận vào một DTO hoặc trực tiếp nhận mã `orderId` và tiến hành ghi bản ghi kiểm toán (audit) mà không cần truy vấn lại bảng đơn hàng, giao dịch của worker có thể sẽ commit thành công bản ghi `DispatchAttempt`. Ngay sau đó, caller gặp lỗi và rollback đơn hàng.
Lúc này, cơ sở dữ liệu sẽ chứa một bản ghi `DispatchAttempt` tham chiếu đến một ID logic mà không hề tồn tại đơn hàng thực sự (nếu không có khóa ngoại, hoặc khóa ngoại làm worker bị chặn lại tùy thuộc vào cấu trúc và thời điểm khóa).

> **Nói ngắn gọn:** Hiện tượng đọc hụt (missing read) và ghi mồ côi (orphan write) chỉ là hai mặt của cùng một vấn đề — hai giao dịch độc lập chạy song song mà không có khế ước thống nhất về thứ tự commit.

## Bối cảnh giao dịch trên thực tế

```text
request-thread: Tx-Order [insert order ........ rollback/commit]
worker-thread:             Tx-Async [query/write ... commit]
```

Annotation `@Transactional` trên hàm async tạo ra một giao dịch mới trên luồng worker; nó hoàn toàn không tham gia (join) vào `Tx-Order`. Ngay cả khi bạn bỏ annotation này, các thao tác gọi kho dữ liệu (repository calls) của worker vẫn có thể tự tạo ra các giao dịch ngắn, dẫn đến tình trạng phân mảnh (fragmentation) giống như bài toán SPR-001.

## Những cách sửa chữa chưa đủ hoặc sai lầm

- **Truyền thực thể `Order` (managed entity) vào hàm async**: Thực thể lúc này sẽ ở trạng thái tách rời (detached) hoặc tải trễ (lazy state) và hoàn toàn không mang theo giao dịch gốc.
- **Dùng `TaskDecorator` để sao chép `TransactionSynchronizationManager`**: Việc chia sẻ giao dịch, kết nối và `EntityManager` xuyên qua các luồng là một hành động nguy hiểm và không an toàn.
- **Gọi `future.join()` trong giao dịch bên ngoài**: Có thể làm luồng hiện tại bị chặn (block) để chờ một worker đang cần chính dòng dữ liệu chưa được commit kia, làm tăng nguy cơ tắc nghẽn (starvation) hoặc treo hệ thống (deadlock-like wait).
- **Chỉ thêm `REQUIRES_NEW` trên hàm async**: Điều này chỉ xác nhận rằng giao dịch là độc lập, chứ không hề tạo ra thứ tự sau-commit (after-commit ordering).
- **Bắt ngoại lệ async (catch exception) trong caller**: Thông thường caller đã kết thúc và commit xong xuôi từ trước khi ngoại lệ của hàm async kịp xuất hiện.
- **Dùng bộ thực thi không giới hạn (unbounded executor) để "chạy nhanh hơn"**: Cách này không giải quyết được vấn đề chạy đua commit (commit race) mà còn tạo ra nguy cơ quá tải hệ thống.

## Điều kiện để tái hiện lỗi

Giao dịch bên ngoài chưa commit; hàm bất đồng bộ chạy trên một luồng khác; worker phụ thuộc vào trạng thái hoặc thực thi các tác động bên ngoài (side effect) của giao dịch gốc; không có cơ chế chuyển giao an toàn sau-commit (after-commit/outbox handoff).
