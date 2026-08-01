# Lựa chọn propagation chính xác và sự đánh đổi (trade-offs)

## Giải pháp 1: trạng thái thành công dùng REQUIRED cùng transaction ngoài

```java
@Service
public class PaymentAuditService {
    private final PaymentAuditRepository audits;

    @Transactional(propagation = Propagation.REQUIRED)
    public void recordPaymentCompleted(long orderId, UUID operationId) {
        audits.save(new PaymentAudit(operationId, orderId, "PAYMENT_COMPLETED"));
    }
}
```

Checkout bên ngoài gọi bean proxy trong Tx-O. Audit tham gia cùng transaction vật lý;
lỗi ở khối ngoài sẽ rollback cả order và audit. ID thao tác duy nhất giúp chống việc thử lại bị trùng lặp (duplicate retry).

Nếu audit chỉ là dữ kiện nội bộ trong cơ sở dữ liệu và đi cùng kết quả nghiệp vụ, đây là lựa chọn
đơn giản và dễ chứng minh nhất.

> **Nói ngắn gọn:** dữ liệu cùng nói về một kết quả nên thường nằm chung trong một
> quyết định commit.

## Giải pháp 2: xuất bản sự kiện thành công sau khi commit hoặc qua outbox

Nếu consumer chỉ cần chạy sau khi checkout thành công, hãy xuất bản domain event trong
Tx-O và dùng after-commit listener cho các luồng xử lý cục bộ, không yêu cầu lưu trữ bền vững (non-durable handoff) (SPR-002).

Nếu sự kiện thành công/audit không được phép mất khi có sự cố (crash), hãy chèn dòng outbox cùng với Tx-O.
Tiến trình relay sẽ xuất bản sự kiện sau khi commit, và consumer phải có tính lũy đẳng (idempotent). Không dùng `REQUIRES_NEW` để giả
lập tính nguyên tử (atomicity) giữa database và broker.

## Giải pháp 3: REQUIRES_NEW cho bản ghi thử nghiệm độc lập

Việc commit độc lập là hợp lệ khi bản ghi mô tả sự thật độc lập:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void recordAttempt(UUID operationId, long orderId, String stage) {
    attempts.insertIfAbsent(operationId, orderId, stage, Instant.now());
}
```

Tên/trạng thái là `ATTEMPT_STARTED` hoặc `OUTER_FAILED`, không phải là
`PAYMENT_COMPLETED`. Bảng không phụ thuộc vào dữ liệu chưa commit của khối ngoài; phương thức này ngắn,
có tính lũy đẳng và không truy cập các dòng (rows) mà khối ngoài đang giữ lock.

Sự đánh đổi: tốn thêm connection, lỗi/timeout diễn ra độc lập, có thể làm nghẽn pool. Chính sách xử lý lỗi của audit
phải rõ ràng: có làm khối ngoài thất bại không hay chỉ dùng để ghi nhận metric/cảnh báo?

## Giải pháp 4: kết quả tùy chọn dự kiến không dùng ngoại lệ rollback

```java
@Transactional(propagation = Propagation.REQUIRED)
public RiskOutcome evaluate(long orderId) {
    RiskDecision decision = calculate(orderId);
    return decision.approved()
            ? RiskOutcome.APPROVED
            : RiskOutcome.REJECTED;
}
```

Sự từ chối dự kiến là một giá trị (value); khối ngoài quyết định không đánh dấu đã thanh toán (mark paid) hoặc ném domain
exception để rollback toàn bộ transaction. Lỗi kỹ thuật vẫn sẽ lan truyền (propagate) và
rollback. Không bắt (catch) ngoại lệ runtime từ interceptor bên trong rồi giả vờ như có thể commit được.

## Giải pháp 5: TransactionTemplate làm rõ ranh giới

Khi workflow thật sự cần nhiều transaction vật lý, hãy dùng các
`TransactionTemplate` riêng biệt với cấu hình propagation rõ ràng và đặt tên cho các bước/kết quả. Ví dụ commit
bản ghi thử nghiệm một cách độc lập, sau đó chạy transaction nghiệp vụ. Mã nguồn phải thừa nhận
kết quả từng phần (partial outcome) và có cơ chế đối soát (reconciliation); không gọi toàn bộ workflow là thao tác nguyên tử (atomic).

## Dùng NESTED/savepoint khi phù hợp

Savepoint có thể rollback các thao tác cơ sở dữ liệu tùy chọn bên trong mà vẫn giữ lại transaction ngoài,
nhưng chỉ khi transaction manager/driver hỗ trợ và trạng thái nghiệp vụ sau khi rollback về
savepoint vẫn hợp lệ. Nên kiểm thử tích hợp trên stack PostgreSQL; không thay `REQUIRES_NEW`
bằng `NESTED` chỉ theo tên gọi mà không hiểu ngữ nghĩa vật lý (physical semantics) của nó.

## Những lựa chọn cần tránh

- Dùng `REQUIRES_NEW` cho mọi helper để tránh rollback lan truyền.
- Bắt lỗi `UnexpectedRollbackException` và trả về HTTP 200.
- Dùng `noRollbackFor = Exception.class` quá rộng.
- Phương thức bên trong truy cập dòng (row) mà khối ngoài đang giữ lock.
- Thực hiện I/O mạng từ xa (Remote I/O) bên trong một transaction độc lập kéo dài.
- Thử lại (retry) mà không có ID thao tác/ràng buộc duy nhất.

## So sánh

| Phương án | Commit vật lý | Rollback khối ngoài | Chi phí tài nguyên | Phù hợp |
| --- | --- | --- | --- | --- |
| `REQUIRED` | Một commit chung | Rollback tất cả | Một connection | Cùng kết quả nghiệp vụ |
| `REQUIRES_NEW` | Khối trong độc lập | Khối trong tồn tại | Tốn thêm connection/lock | Bản ghi trung thực, độc lập |
| `NESTED` | Cùng khối ngoài + savepoint | Rollback tất cả | Dùng chung connection thường lệ | Bước DB phụ trợ tùy chọn có hỗ trợ |
| After-commit cục bộ | Sau khi khối ngoài commit | Không phát hành sự kiện | Dùng Executor, không lưu trữ bền vững | Công việc cục bộ xử lý nỗ lực tối đa (best-effort) |
| Outbox | Ghi dòng cùng khối ngoài commit | Không có dòng outbox | Tiến trình relay/lưu trữ | Sự kiện/công việc bền vững |

## Danh sách kiểm tra trên production

Mỗi phương thức có tài liệu ghi rõ mục đích propagation; các dữ kiện thành công (success facts) đi cùng với kết quả commit;
các bảng dùng REQUIRES_NEW độc lập về mặt ngữ nghĩa; các quy tắc rollback rõ ràng; không nuốt (swallow)
trạng thái rollback-only; sử dụng ID thao tác duy nhất; có cấu hình kích thước pool/timeout; có kiểm thử tích hợp (integration tests) cho
rollback ở khối ngoài, lỗi bên trong và UnexpectedRollback; khả năng quan sát (observability) phải liên kết
được transaction vật lý và kết quả của nó.
