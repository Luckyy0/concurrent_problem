# Các giải pháp: Xử lý đồng bộ, kích hoạt sau-commit và chuyển giao bền vững (durable handoff)

## Giải pháp 1: Giữ công việc trong cùng giao dịch khi cần tính nguyên tử (atomicity)

Nếu công việc (work) là một thao tác cập nhật cơ sở dữ liệu (database mutation) bắt buộc phải được commit cùng lúc với đơn hàng, tuyệt đối không dùng phương thức bất đồng bộ (async):

```java
@Transactional
public long place(PlaceOrderCommand command) {
    Order order = orders.save(Order.pending(command.customerId()));
    reserveInventory(order, command);
    createRequiredDatabaseState(order);
    return order.getId();
}
```

Nên tránh việc gọi các I/O mạng từ xa (remote I/O) kéo dài bên trong giao dịch nếu có thể. Khái niệm "Cùng transaction" chỉ mang ý nghĩa áp dụng cho các tài nguyên đang được bộ quản lý giao dịch điều phối; việc gửi email hay gọi API HTTP không hề được tự động rollback cùng với PostgreSQL.

## Giải pháp 2: Sự kiện giao dịch (Transactional event) kích hoạt sau-commit rồi gọi một bean async

Nhà phát hành (Publisher) chỉ việc lưu đơn hàng và phát ra một sự kiện bất biến (immutable event) bên trong giao dịch:

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

Trình lắng nghe (Listener) chỉ thực hiện điều phối (dispatch) sau khi giao dịch đã commit thành công trót lọt:

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

Bộ xử lý async (Async processor) phải là một bean khác để đảm bảo lời gọi đi qua async proxy của Spring; nó sẽ mở ra một giao dịch riêng trên luồng worker nếu cần thực hiện ghi dữ liệu:

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

### Vì sao tính đúng đắn cục bộ (invariant) được bảo vệ

- Giao dịch bị rollback sẽ không bao giờ kích hoạt trình lắng nghe sau-commit.
- Worker chỉ được phân phối công việc sau khi đã commit, do đó việc nạp lại đơn hàng chắc chắn sẽ đọc được trạng thái đã commit.
- Sự kiện (event) chỉ chứa các ID bất biến, hoàn toàn không chứa thực thể được quản lý (managed entity).
- Giao dịch của worker có ranh giới hoàn toàn độc lập và được khai báo minh bạch.

> **Nói ngắn gọn:** Giao dịch chỉ làm nhiệm vụ quyết định "có được giao việc hay không"; bộ thực thi (executor) chỉ làm nhiệm vụ quyết định "công việc sau commit sẽ chạy ở luồng nào".

### Các giới hạn

Mặc dù giao dịch đã được commit thành công trước khi chuyển giao công việc (local handoff), nếu executor từ chối, ứng dụng bị lỗi (crash) hay máy chủ tắt (shutdown) thì công việc vẫn có thể bốc hơi mất. Trình lắng nghe bắt buộc phải ghi lại số liệu (metric) và xử lý cẩn thận các trường hợp bị từ chối; tuyệt đối không thể ném ngoại lệ (exception) để đòi rollback một giao dịch vốn đã commit xong.

## Giải pháp 3: Dùng TransactionSynchronization tường minh

Khi bạn không muốn dùng lớp trừu tượng sự kiện (event abstraction):

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

Phương pháp này có cùng nguy cơ rủi ro mất dữ liệu hay quá tải tương tự như cách dùng transactional event nội bộ. Trình lắng nghe sự kiện (Event listener) thường giúp giảm sự phụ thuộc (coupling) tốt hơn; việc đồng bộ hóa giao dịch một cách tường minh (explicit synchronization) chỉ phù hợp cho các đoạn mã chuyển đổi hạ tầng (adapter) nhỏ và được kiểm thử cực kỳ kỹ lưỡng.

## Giải pháp 4: Mẫu Transactional Outbox cho các tác vụ đòi hỏi sự bền vững (durable work)

Bên trong giao dịch lưu đơn hàng (Tx-Order), hãy thực hiện thao tác chèn (insert) đơn hàng đồng thời chèn một bản ghi sự kiện vào bảng outbox trong cùng một giao dịch. Một tiến trình chuyển tiếp (relay) sẽ quét đọc các bản ghi outbox đã được commit, tiến hành phát hành thông điệp (publish message); sau đó các consumer sẽ xử lý theo cơ chế lũy đẳng (idempotently) và tự đánh dấu kết quả xử lý (outcome). Đây là giải pháp tiêu chuẩn khi công việc của bạn không được phép bị bốc hơi nếu tiến trình gặp lỗi (process crash) hoặc cần cơ chế khôi phục phân tán giữa nhiều máy chủ (multi-node recovery).

Trường hợp phân tích này chỉ nói về ranh giới để chọn giải pháp; việc thiết kế cấu trúc bảng (schema), khóa dữ liệu (locking), gửi lại tin nhắn (redelivery) và giao thức inbox/outbox sẽ được đi sâu phát triển tại `MSG-007`.

## Cấu hình Executor và Context

- Phải luôn sử dụng bộ thực thi có giới hạn hàng đợi (bounded executor/queue), đặt tên luồng có ý nghĩa và lập trình sẵn các bộ xử lý từ chối (explicit rejection handler).
- Cần có cơ chế kiểm soát tiếp nhận (Admission control) và giảm tải (load shedding) phù hợp với năng lực của hệ thống đích (downstream capacity).
- `TaskDecorator` chỉ được phép dùng để truyền tải các dữ liệu MDC/trace/security đã được duyệt, và nhất thiết phải được dọn dẹp sạch sẽ trong khối `finally` để tránh bị rò rỉ dữ liệu luồng (thread-local leak).
- Tuyệt đối không truyền lan (propagate) đối tượng transaction hoặc EntityManager.
- Quá trình tắt ứng dụng (Shutdown) phải có hạn định thời gian (deadline), ngưng nhận thêm việc và quan sát kỹ các công việc còn đang chạy dang dở (unfinished tasks).

## Chính sách xử lý lỗi (Failure policy)

- Với hàm async trả về kiểu future: Bắt buộc phải quan sát các ngoại lệ (observe exception), cơ chế thử lại (retry) phải có tính lũy đẳng (idempotency) và giới hạn thời gian (deadline).
- Với hàm trả về kiểu void: Hãy cấu hình `AsyncUncaughtExceptionHandler`, tuyệt đối không dùng giải pháp ghi log sơ sài mặc định.
- Việc tạo cơ chế thử lại cho lỗi không tìm thấy đơn hàng (missing-order) trên một hệ thống thiết kế sai sót không phải là cách fix lỗi; bạn phải sửa lại cơ chế thứ tự commit (commit ordering).
- Giao dịch của worker có bị rollback thì cũng không thể rollback lại đơn hàng đã được commit; vì vậy phần nghiệp vụ (domain) cần cung cấp các trạng thái như `PROCESSING/FAILED` hoặc thiết lập cơ chế thử lại bền vững (durable retry) nếu kết quả trả về thực sự quan trọng.
- Thao tác hủy (Cancellation) chỉ mang tính chất nỗ lực hết mình (best-effort), không bao giờ được xem là một bảo chứng chắc chắn cho việc rollback.

## So sánh các phương án

| Phương án | Thứ tự Commit | Khả năng an toàn sau sự cố (Crash durability) | Độ trễ tác động tới Caller | Độ phức tạp |
| --- | --- | --- | --- | --- |
| Thao tác DB đồng bộ | Cùng giao dịch | Cơ sở dữ liệu đảm bảo an toàn | Cao hơn | Thấp |
| Local async sau khi commit | Tuân thủ nghiêm ngặt | Không bảo đảm lúc bàn giao | Thấp | Vừa phải |
| TransactionSynchronization | Tuân thủ nghiêm ngặt | Không bảo đảm lúc bàn giao | Thấp | Vừa phải/Coupled |
| Transactional outbox | Tuân thủ nghiêm ngặt | Chắc chắn có, nhờ cơ chế chuyển tiếp (relay/retry) | Thấp | Cao hơn |

## Checklist trước khi triển khai (Production)

Các yêu cầu về ranh giới/độ an toàn (boundary/durability) đã được viết ra tài liệu chưa? Sự kiện (event) có tính bất biến không? Đã tách riêng thành một async bean chưa? Giao dịch của worker đã được phân định ranh giới tường minh chưa? Đã dùng bộ thực thi có giới hạn (bounded executor) chưa? Đã có cơ chế cảnh báo/quan sát được hiện tượng từ chối (rejection)/ngoại lệ (exception) chưa? Tuyệt đối không truyền thực thể bị quản lý (managed entity) qua luồng chưa? Các quy trình xử lý đều mang tính lũy đẳng (idempotent processing) chưa? Có đầy đủ Integration tests cho các trường hợp commit, rollback, trạng thái bị đọc hụt (missing-state) và khả năng xử lý rủi ro mất mát dữ liệu lúc hệ thống sập (crash-gap selection) chưa?
