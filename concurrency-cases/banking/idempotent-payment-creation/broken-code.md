# Broken Code & Anti-patterns

## Schema Definition
Bảng `payment_records` lưu trữ thông tin giao dịch. Lỗi thiết kế phổ biến là thiếu unique constraint trên cột `idempotency_key`.

```sql
CREATE TABLE payment_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    idempotency_key VARCHAR(255) NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(50) NOT NULL, -- PENDING, COMPLETED, FAILED
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
-- ANTI-PATTERN: Thiếu UNIQUE INDEX trên idempotency_key
CREATE INDEX idx_idempotency ON payment_records(idempotency_key);
```

## Broken Java Implementation

Mã nguồn Spring Boot service dưới đây sử dụng pattern `check-then-act` phổ biến nhưng đầy rủi ro.

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentRecordRepository paymentRepository;
    private final PaymentProcessorClient paymentProcessor;

    @Transactional
    public PaymentResponse createPayment(String idempotencyKey, BigDecimal amount, String currency) {
        // ANTI-PATTERN 1: Check-then-act không an toàn
        Optional<PaymentRecord> existingPayment = paymentRepository.findByIdempotencyKey(idempotencyKey);
        
        if (existingPayment.isPresent()) {
            // ANTI-PATTERN 2: Chỉ kiểm tra tồn tại mà không tái tạo chính xác HTTP response cũ
            PaymentRecord record = existingPayment.get();
            return new PaymentResponse(record.getId(), record.getStatus(), "Recovered from existing record");
        }

        // Tạo bản ghi thanh toán mới với trạng thái PENDING
        PaymentRecord newPayment = new PaymentRecord();
        newPayment.setIdempotencyKey(idempotencyKey);
        newPayment.setAmount(amount);
        newPayment.setCurrency(currency);
        newPayment.setStatus("PENDING");
        
        // ANTI-PATTERN 3: Save tại đây chưa commit xuống DB cho các thread khác thấy
        paymentRepository.save(newPayment);

        // Gọi API của đối tác bên ngoài (Network I/O) trong một Transaction đang mở
        // ANTI-PATTERN 4: Network call trong database transaction làm tăng thời gian giữ lock/connection
        boolean processorResult = paymentProcessor.charge(amount, currency);
        
        if (processorResult) {
            newPayment.setStatus("COMPLETED");
        } else {
            newPayment.setStatus("FAILED");
        }
        
        paymentRepository.save(newPayment);
        
        return new PaymentResponse(newPayment.getId(), newPayment.getStatus(), "Created successfully");
    }
}
```

## Anti-patterns Explained

1. **Check-then-act Race Condition**: 
   Thread A gọi `findByIdempotencyKey` và trả về empty. Thread B (một request retry do timeout) cũng gọi `findByIdempotencyKey` gần như cùng lúc và cũng trả về empty. Cả 2 thread đều đi tiếp xuống dưới và tạo ra 2 bản ghi thanh toán.
2. **Missing Unique Constraint**: 
   Database không có `UNIQUE INDEX` trên cột `idempotency_key`. Do đó, DB sẽ cho phép cả Thread A và Thread B thực hiện lệnh `INSERT` thành công.
3. **Network Call Inside Transaction**: 
   Gọi hệ thống external (`paymentProcessor.charge`) bên trong khối `@Transactional`. Việc này có thể mất vài giây nếu mạng chậm, làm transaction bị treo dài, ngâm connection trong connection pool và tăng rủi ro race condition.
4. **Incomplete Idempotency Response**: 
   Khi phát hiện record đã tồn tại, ứng dụng chỉ trả về trạng thái từ database. Nếu request A đang gọi external API (bản ghi đang là `PENDING`), request B tới và thấy `PENDING`, nó trả về cho client. Client sẽ không biết giao dịch cuối cùng có thành công hay không.
5. **Lack of "In-progress" Tracking**: 
   Không có cơ chế nào để đánh dấu rằng một `idempotencyKey` đang được xử lý bởi một thread khác (atomic key claim).

## Timeline showing the Bug

| Thời gian (T) | Thread A (Request 1) | Thread B (Request 2 - Retry) | Trạng thái DB (MVCC) |
|---------------|----------------------|------------------------------|----------------------|
| T1 | `findByIdempotencyKey("key-1")` -> Empty | | Chưa có dữ liệu |
| T2 | | `findByIdempotencyKey("key-1")` -> Empty | Chưa có dữ liệu |
| T3 | `save(newPayment)` (Trạng thái PENDING) | | Ghi tạm thời vào TX-A |
| T4 | `paymentProcessor.charge(...)` (Network I/O - Mất 2s) | `save(newPayment)` (Trạng thái PENDING) | Ghi tạm thời vào TX-B |
| T5 | (Đang đợi API đối tác) | `paymentProcessor.charge(...)` (Network I/O) | |
| T6 | API đối tác trả về OK | API đối tác trả về OK | |
| T7 | `setStatus("COMPLETED")`, Transaction A Commit | | Bản ghi 1 hiển thị |
| T8 | | `setStatus("COMPLETED")`, Transaction B Commit | Cả 2 bản ghi hiển thị (Duplicate) |

## Observability Symptoms
- Client báo cáo bị trừ tiền 2 lần (2 transactions xuất hiện trên sao kê ngân hàng).
- Logs hiển thị 2 requests có chung `idempotencyKey` chạy song song, cả 2 đều thực hiện logic charge tiền.
- Bảng `payment_records` xuất hiện nhiều dòng có cùng `idempotency_key`.
