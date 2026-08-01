# Transaction mới cho mỗi lần thử (attempt)

## Mục tiêu thiết kế

Chuỗi xử lý (pipeline) retry đúng đắn:

```text
phân loại lỗi có thể retry
  -> rollback transaction bị lỗi (failed transaction) hoàn tất
  -> đóng persistence context
  -> kiểm tra giới hạn thời hạn/ngân sách cho số lần thử
  -> thời gian chờ có giới hạn (bounded backoff) ở bên ngoài transaction
  -> tạo transaction mới
  -> tải lại và kiểm tra lại quy tắc nghiệp vụ
  -> commit hoặc thất bại
```

> **Nói ngắn gọn:** điều phối viên retry (retry coordinator) sở hữu các lần thử (attempts); transaction worker sở hữu đúng một lần thử.

## Giải pháp 1 — Tách biệt retry coordinator và transactional worker

Đây là cấu trúc được khuyến nghị khi sử dụng Spring Retry.

### Worker cho một lần thử (One-attempt worker)

```java
@Service
public class ReservationAttemptService {
    private final InventoryItemRepository inventory;
    private final ReservationRecordRepository reservations;

    public ReservationAttemptService(
        InventoryItemRepository inventory,
        ReservationRecordRepository reservations
    ) {
        this.inventory = inventory;
        this.reservations = reservations;
    }

    @Transactional
    public ReservationResult reserveOnce(
        UUID commandId,
        String sku,
        int quantity
    ) {
        Optional<ReservationRecord> existing =
            reservations.findByCommandId(commandId);

        if (existing.isPresent()) {
            return ReservationResult.replayed(existing.orElseThrow());
        }

        InventoryItem item = inventory.findById(sku)
            .orElseThrow();
        long expectedVersion = item.getVersion();

        item.reserve(quantity); // kiểm tra lại stock trên trạng thái mới
        reservations.save(ReservationRecord.accepted(
            commandId,
            sku,
            quantity
        ));

        inventory.flush(); // làm xuất hiện xung đột ngay trong ranh giới lần thử này

        return ReservationResult.accepted(
            commandId,
            expectedVersion
        );
    }
}
```

Câu lệnh ghi được sinh ra:

```sql
update inventory_item
set available = ?,
    version = ?
where sku = ?
  and version = ?;
```

Khóa command duy nhất (Unique command key) vẫn là yêu cầu bắt buộc:

```sql
alter table reservation_record
    add constraint uk_reservation_command unique (command_id);
```

### Bộ điều phối (coordinator) không nằm trong transaction

```java
@Service
public class ReservationRetryCoordinator {
    private final ReservationAttemptService attempts;

    public ReservationRetryCoordinator(
        ReservationAttemptService attempts
    ) {
        this.attempts = attempts;
    }

    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttemptsExpression =
            "${reservation.retry.max-attempts:4}",
        backoff = @Backoff(
            delayExpression =
                "${reservation.retry.initial-delay-ms:20}",
            maxDelayExpression =
                "${reservation.retry.max-delay-ms:200}",
            multiplierExpression =
                "${reservation.retry.multiplier:2.0}",
            random = true
        )
    )
    public ReservationResult reserve(
        UUID commandId,
        String sku,
        int quantity
    ) {
        return attempts.reserveOnce(commandId, sku, quantity);
    }
}
```

Coordinator không dùng annotation `@Transactional`. Mỗi lần gọi từ `RetryInterceptor` sẽ gọi `attempts` qua một bean proxy khác:

```text
Retry lần thử 1
  -> ReservationAttemptService proxy bắt đầu Tx-1
  -> xung đột (conflict)
  -> proxy rollback Tx-1 và ném lại ngoại lệ
  -> RetryInterceptor bắt (catches)

Retry lần thử 2
  -> proxy bắt đầu Tx-2 với EntityManager mới
  -> tải lại version 8
  -> commit version 9
```

Các giá trị mặc định của thuộc tính chỉ là cấu hình mẫu khi phát triển (development). Các giá trị trên production phải dựa trên thời hạn (deadline) của operation, phân phối xung đột và công suất (capacity) của database.

### Khi cạn kiệt (exhausted) số lần thử

Để lỗi optimistic cuối cùng tiếp tục lan truyền (propagate) hoặc dùng `@Recover` để ánh xạ (map) thành một kết quả ứng dụng ổn định:

```java
@Recover
public ReservationResult exhausted(
    ObjectOptimisticLockingFailureException conflict,
    UUID commandId,
    String sku,
    int quantity
) {
    throw new ReservationContentionException(
        commandId,
        sku,
        conflict
    );
}
```

Khối `@Recover` không được phép báo cáo là thành công (success). Phía gọi (caller) cần biết thao tác (operation) chưa được commit và có thể truy vấn bằng command ID nếu kết quả trả về trước đó không rõ ràng (ambiguous).

## Bảo vệ transaction bên ngoài (outer transaction)

Giải pháp 1 giả định coordinator là ranh giới use-case gốc (root). Nếu một phía gọi (caller) transactional bao bọc coordinator, worker có tính `REQUIRED` sẽ kết nối (join) vào transaction bên ngoài.

Các lựa chọn:

- kiến trúc hệ thống nghiêm cấm phía gọi mang tính transactional và có bài kiểm thử kiến trúc/tích hợp để kiểm chứng;
- sử dụng `REQUIRES_NEW` cho attempt worker khi yêu cầu commit độc lập là đúng đắn;
- sử dụng template dạng lập trình (programmatic) với thiết lập lan truyền tường minh;
- tách tác vụ retry thành một asynchronous command handler mà không phụ thuộc vào transaction bên ngoài.

Thuộc tính `REQUIRES_NEW` đảm bảo (guarantee) một transaction thử nghiệm mới nhưng sẽ làm ngưng (suspend) transaction bên ngoài, việc này có thể tốn thêm connection và tạo commit độc lập. Chỉ nên chọn giải pháp này nếu hợp đồng tính nguyên tử (atomicity contract) của hệ thống cho phép.

## Giải pháp 2 — Vòng lặp coordinator thủ công (Manual coordinator loop) bên ngoài transaction

Khi không sử dụng Spring Retry hoặc khi cần quản lý deadline hoạt động chung (overall operation deadline):

```java
@Service
public class ManualReservationRetryCoordinator {
    private final ReservationAttemptService attempts;
    private final RetryPolicy retryPolicy;
    private final RetryBackoff retryBackoff;

    public ReservationResult reserve(
        UUID commandId,
        String sku,
        int quantity,
        Instant deadline
    ) {
        for (int attempt = 1; ; attempt++) {
            try {
                return attempts.reserveOnce(
                    commandId,
                    sku,
                    quantity
                );
            } catch (ObjectOptimisticLockingFailureException conflict) {
                RetryDecision decision = retryPolicy.decide(
                    attempt,
                    deadline,
                    conflict
                );

                if (decision == RetryDecision.STOP) {
                    throw conflict;
                }

                retryBackoff.pauseInterruptibly(
                    attempt,
                    deadline
                );
            }
        }
    }
}
```

Phần xử lý catch chạy sau khi proxy `reserveOnce()` đã tiến hành rollback. Cơ chế backoff không giữ lại connection hoặc transaction của database. Phương thức `pauseInterruptibly` phải tôn trọng các yêu cầu hủy bỏ (cancellation) và không vượt quá deadline.

Vòng lặp này đúng vì transaction được bao bọc trong bean được gọi (called bean), không bao trùm lấy toàn bộ vòng lặp.

## Giải pháp 3 — Dùng `TransactionTemplate` cho các lần thử (attempt) tường minh

```java
@Service
public class ProgrammaticReservationRetry {
    private final TransactionTemplate attemptTransaction;
    private final ReservationWork work;
    private final RetryPolicy retryPolicy;
    private final RetryBackoff retryBackoff;

    public ProgrammaticReservationRetry(
        PlatformTransactionManager transactionManager,
        ReservationWork work,
        RetryPolicy retryPolicy,
        RetryBackoff retryBackoff
    ) {
        this.attemptTransaction =
            new TransactionTemplate(transactionManager);
        this.attemptTransaction.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW
        );
        this.work = work;
        this.retryPolicy = retryPolicy;
        this.retryBackoff = retryBackoff;
    }

    public ReservationResult reserve(
        ReservationCommand command,
        Instant deadline
    ) {
        for (int attempt = 1; ; attempt++) {
            try {
                return attemptTransaction.execute(status ->
                    work.reserve(command)
                );
            } catch (ObjectOptimisticLockingFailureException conflict) {
                if (!retryPolicy.canRetry(attempt, deadline, conflict)) {
                    throw conflict;
                }
                retryBackoff.pauseInterruptibly(attempt, deadline);
            }
        }
    }
}
```

Lời gọi `execute()` đảm bảo rollback và đóng tài nguyên hoàn tất trước khi ngoại lệ chạy tới khối catch. Dùng `REQUIRES_NEW` làm cho ngữ nghĩa tường minh (explicit) ngay cả khi phía gọi có transaction, nhưng chi phí tốn kém liên quan đến kết nối và quá trình commit độc lập cần phải được chấp nhận.

Nếu có thể chắc chắn phương thức là gốc rễ (root) và không hề có transaction bên ngoài, thì việc dùng `REQUIRED` cũng vẫn đủ để tạo một transaction mới cho mỗi lần gọi `execute()` riêng biệt diễn ra sau khi hoàn thành thao tác trước đó.

## Giải pháp 4 — Sắp xếp thứ tự các advisor tường minh

Có thể cấu hình quá trình advice của chức năng retry nằm bên ngoài advice của transaction. Tuy nhiên, so với việc tách các bean, đây là một lựa chọn khó thể hiện rõ (kém self-evident) hơn:

```text
Thứ tự Retry advisor < Thứ tự Transaction advisor
=> retry interceptor ở lớp bên ngoài
```

Giá trị/thứ tự cụ thể thay đổi tùy thuộc vào cấu hình và thiết lập của framework. Nếu chọn cách này:

- kiểm tra chuỗi advisor thực tế trong các bài kiểm thử tích hợp (integration test);
- xác nhận transaction identity thay đổi sau mỗi lần thử;
- xác nhận quá trình hoàn tất (completion) của lượt trước là rollback trước khi lượt mới được bắt đầu;
- dùng test để ràng buộc cấu hình nhằm tránh bị hồi quy khi nâng cấp (upgrade) version.

Không chỉ tin tưởng vào các unit test trên annotation hay mặc định cho rằng số nhỏ/lớn hơn là xong, mà không kiểm tra proxy ở thời điểm runtime.

## Phân loại lỗi (Failure classification)

Danh sách được phép (Allowlist):

```java
boolean isRetryable(Throwable failure) {
    Throwable specific = mostSpecificCause(failure);

    return failure instanceof ObjectOptimisticLockingFailureException
        || isSqlState(specific, "40001")
        || isSqlState(specific, "40P01");
}
```

Quá trình phân loại SQLSTATE cần phải duyệt chuỗi nguyên nhân một cách có kiểm soát. Không thực hiện retry đối với `InsufficientStockException`, các command trùng lặp (duplicate) đã có lưu sẵn kết quả, đầu vào (input) không hợp lệ, lỗi mapping bug hoặc lệnh hủy (cancellation).

Một lệnh retry do deadlock/serialization cũng bắt buộc phải chạy lại toàn bộ một database unit trong một transaction mới. Tính an toàn đối với các side-effect thuộc hệ thống bên ngoài theo đặc thù nghiệp vụ vẫn luôn cần có sự kết hợp của cơ chế tính lũy đẳng (idempotency)/outbox.

## Kiểm tra lại các điều kiện nghiệp vụ (Business revalidation)

Lần thử (attempt) 2 không tái sử dụng quyết định (không replay) `available = 2`. Nó tải lại:

```java
InventoryItem current = inventory.findById(command.sku())
    .orElseThrow();
current.reserve(command.quantity());
```

Nếu số lượng kho lúc này không đủ, `InsufficientStockException` mới là kết quả cuối cùng (final domain outcome), không phải là một optimistic conflict để tiếp tục retry.

## Kết quả xử lý xung đột (Conflict outcome)

| Tiến trình (Actor) | Detector | Hành vi ngay lúc đó (Immediate behavior) | Hành vi cuối cùng (Final behavior) |
| --- | --- | --- | --- |
| Tiến trình thắng A | Version predicate khớp (matches) | Update/commit | Thành công (Success) |
| Tiến trình thua B lần thử 1 | Số dòng ảnh hưởng = 0 | Thất bại và rollback | Đủ điều kiện retry (Eligible for retry) |
| B lần thử 2 | Tải lại v8, predicate khớp | Update/commit | Thành công (Success) |
| Kẻ thua cạn kiệt (Exhausted loser) | Hết lượt retry/deadline | Dừng | Thông báo lỗi tranh chấp |
| Kẻ thua theo quy tắc nghiệp vụ | Kho hiện tại không đủ | Không retry | Bị từ chối nghiệp vụ (Domain rejection) |

## Các lựa chọn thay thế (Alternatives) khi contention đối với khóa nóng (hot-key) cao

Retry không phải là giải pháp đem lại thông lượng (throughput) tốt cho mọi loại khối lượng công việc (workload):

- quá trình cập nhật trừ giá trị theo mức nguyên tử (atomic conditional decrement) có thể tránh cần đến read/write retry;
- `PESSIMISTIC_WRITE` chuyển xung đột thành dạng hàng đợi bị chặn (blocking queue);
- serialize chuỗi thực thi của các command theo khóa (aggregate key);
- chia nhỏ/phân mảnh (shard/partition) cho khóa nóng;
- kiểm soát và điều phối tải (admission control/backpressure);
- trả thẳng lỗi tranh chấp (contention failure) thay vì cố tình retry ở phía máy chủ.

Mỗi quyết định đều phải đánh đổi (trade-off) về độ trễ, thời gian giữ khóa, mức chịu tải của database và khả năng mở rộng theo chiều ngang (horizontal scaling).
Không có bất cứ bộ đo điểm chuẩn (benchmark) phổ quát nào; phải tự đo tốc độ gây xung đột và thông lượng đạt được cuối cùng trên workload đặc thù của hệ thống.

## Bảng So sánh

| Lựa chọn | Sự phân định ranh giới | An toàn Transaction ngoài | Độ phức tạp | Nhược điểm / Đánh đổi chính |
| --- | --- | --- | --- | --- |
| Tách biệt coordinator/worker | Cao | Cần bảo vệ caller | Thấp-vừa | Khuyến nghị mặc định |
| Vòng lặp thủ công vòng ngoài | Cao | Cần bảo vệ caller | Vừa | Linh hoạt Deadline/phân loại lỗi |
| `TransactionTemplate` NEW | Rất cao | Cao | Vừa | Có phí liên quan commit độc lập/kết nối |
| Sắp xếp thứ tự advisor | Thấp-vừa | Phụ thuộc proxy chain | Vừa-cao | Rất dễ gặp hồi quy cấu hình (regression) |
| Vòng lặp cùng Tx + gọi clear | Sai | Sai | Trông có vẻ thấp | Không thể tạo lập lần retry sạch |

## Danh sách kiểm tra môi trường Production (Production checklist)

### Ranh giới (Boundary)

- [ ] Lệnh catch đối với retry nằm ngoài transaction interceptor đã thất bại.
- [ ] Mỗi lần thử (attempt) sở hữu một định danh (identity) transaction/persistence context mới.
- [ ] Quá trình rollback trước đó đã hoàn tất trước khi vào quá trình backoff.
- [ ] Backoff không giữ database connection.
- [ ] Các giả định (assumptions) trên transaction bên ngoài đều được thực thi/kiểm thử chặt chẽ.

### Chính sách (Policy)

- [ ] Những loại lỗi có thể retry dùng nhóm (allowlist) type/SQLSTATE.
- [ ] Số lượt thử và tổng thời gian khả dụng phải được giới hạn.
- [ ] Backoff phải có tính ngẫu nhiên (jitter) và cho phép gián đoạn (interruption).
- [ ] Lần thử tiếp theo phải kiểm tra lại trạng thái nghiệp vụ.
- [ ] Khi hết số lần thử, trả về trạng thái thất bại (non-success outcome) rõ ràng.

### Dữ liệu và Các thao tác

- [ ] Thuộc tính `@Version` và predicate đã được tích hợp kiểm tra kỹ.
- [ ] Command ID duy nhất được sử dụng xử lý việc truyền tải lặp thông điệp.
- [ ] Tính năng liên quan sự kiện bên ngoài (External side effects) luôn nằm sau commit và đi qua outbox khi phù hợp.
- [ ] Số liệu của quá trình Xung đột/lượt thử/kết quả đều có định danh (dimension) aggregate key an toàn.
- [ ] Đối với dữ liệu khoá nóng luôn có cách dự phòng (fallback) ngoài retry lạc quan.
