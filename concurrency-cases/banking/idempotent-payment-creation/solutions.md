# Solutions

## Overview of Solutions

Để giải quyết trọn vẹn vấn đề Idempotent Creation, chúng ta cần:
1. Đảm bảo tính nguyên tử (atomic) ở tầng Database.
2. Quản lý trạng thái "Đang xử lý" (In-progress state) để các retry request không gây side-effect mà trả về đúng thông điệp.
3. Tách biệt network call ra khỏi database transaction để tối ưu resources.

## Solution 1: Unique Constraint + Separate Claim Table (Recommended)

Giải pháp này sử dụng một bảng riêng biệt `idempotency_keys` để quản lý vòng đời của khóa (atomic key claim). Điều này tách biệt logic đảm bảo an toàn đồng thời khỏi dữ liệu nghiệp vụ, và cho phép lưu trữ HTTP response để `response replay`.

### SQL Schema Changes

```sql
CREATE TABLE idempotency_keys (
    key VARCHAR(255) PRIMARY KEY,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    locked_at TIMESTAMP,
    status VARCHAR(20) NOT NULL, -- IN_PROGRESS, COMPLETED, FAILED
    response_body TEXT,          -- Lưu response payload
    response_code INT            -- Lưu HTTP status code
);

-- Vẫn nên duy trì UNIQUE constraint ở bảng chính như lớp phòng thủ cuối cùng
CREATE TABLE payment_records_safe (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(50) NOT NULL
);
```

### Java Implementation

Chúng ta tách quy trình làm 3 bước: Claim Key (New Transaction), Execute Business Logic (No DB Transaction for Network call), Save Result (New Transaction).

```java
@Service
@RequiredArgsConstructor
public class IdempotentPaymentService {

    private final IdempotencyKeyRepository idempotencyRepository;
    private final PaymentRecordRepository paymentRepository;
    private final PaymentProcessorClient paymentProcessor;
    private final TransactionTemplate transactionTemplate;

    public ResponseEntity<String> processPayment(String idempotencyKey, BigDecimal amount, String currency) {
        
        // BƯỚC 1: Cố gắng Claim (chiếm hữu) Idempotency Key
        IdempotencyKey claim = transactionTemplate.execute(status -> {
            try {
                IdempotencyKey newKey = new IdempotencyKey();
                newKey.setKey(idempotencyKey);
                newKey.setStatus("IN_PROGRESS");
                newKey.setLockedAt(LocalDateTime.now());
                // Insert atomic. Sẽ ném ra DataIntegrityViolationException nếu key đã tồn tại
                return idempotencyRepository.saveAndFlush(newKey);
            } catch (DataIntegrityViolationException ex) {
                // Key đã tồn tại, transaction này rollback
                status.setRollbackOnly();
                return null;
            }
        });

        // Nếu claim = null, nghĩa là một thread/request khác đã (hoặc đang) xử lý key này
        if (claim == null) {
            return handleDuplicateKey(idempotencyKey);
        }

        // BƯỚC 2: Thực thi nghiệp vụ. Không nằm trong khối @Transactional nào.
        // Điều này đảm bảo database connection không bị giữ trong lúc chờ external API.
        boolean chargeResult;
        String responseBody;
        int responseCode;
        
        try {
            chargeResult = paymentProcessor.charge(amount, currency);
            
            // BƯỚC 3: Cập nhật Payment và Idempotency trạng thái (Transaction độc lập)
            transactionTemplate.executeWithoutResult(status -> {
                PaymentRecord newPayment = new PaymentRecord();
                newPayment.setIdempotencyKey(idempotencyKey);
                newPayment.setAmount(amount);
                newPayment.setCurrency(currency);
                newPayment.setStatus(chargeResult ? "COMPLETED" : "FAILED");
                paymentRepository.save(newPayment);
                
                IdempotencyKey keyRecord = idempotencyRepository.findById(idempotencyKey).orElseThrow();
                keyRecord.setStatus("COMPLETED");
                keyRecord.setResponseBody("Payment processed: " + newPayment.getId());
                keyRecord.setResponseCode(chargeResult ? 200 : 400);
                idempotencyRepository.save(keyRecord);
            });
            
            responseCode = chargeResult ? 200 : 400;
            responseBody = "Payment processed";

        } catch (Exception e) {
            // Đánh dấu Key FAILED để client có thể retry lại sau
            transactionTemplate.executeWithoutResult(status -> {
                IdempotencyKey keyRecord = idempotencyRepository.findById(idempotencyKey).orElseThrow();
                keyRecord.setStatus("FAILED");
                idempotencyRepository.save(keyRecord);
            });
            throw e;
        }

        return ResponseEntity.status(responseCode).body(responseBody);
    }

    private ResponseEntity<String> handleDuplicateKey(String idempotencyKey) {
        IdempotencyKey existingKey = idempotencyRepository.findById(idempotencyKey)
            .orElseThrow(() -> new IllegalStateException("Key must exist here"));
            
        if ("IN_PROGRESS".equals(existingKey.getStatus())) {
            // Request đầu tiên vẫn đang chạy
            return ResponseEntity.status(HttpStatus.CONFLICT).body("Request is currently processing");
        } else if ("COMPLETED".equals(existingKey.getStatus())) {
            // Response Replay
            return ResponseEntity.status(existingKey.getResponseCode()).body(existingKey.getResponseBody());
        } else {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Previous request failed");
        }
    }
}
```

## Why Invariant is Protected
1. **Atomic Key Claim**: Thao tác `saveAndFlush` (tương đương SQL `INSERT`) cho một khóa chính (`key`) là nguyên tử trong PostgreSQL. Thread thứ 2 cố gắng insert cùng `key` sẽ bị DB block và trả về lỗi Duplicate Key (nhờ DataIntegrityViolationException).
2. **In-progress State Management**: Giải quyết được trường hợp request 2 tới khi request 1 đang `PENDING`. Thay vì mù quáng trả về lỗi hay tạo mới, nó biết chính xác đang `IN_PROGRESS` và trả về HTTP 409 Conflict (Client có thể retry tự động).
3. **Response Replay**: Bảng `idempotency_keys` lưu lại exact HTTP response. Do đó, request retry nhận lại đúng data như request gốc thành công.
4. **No long-running transaction**: Call ra ngoài (`paymentProcessor.charge`) được thực hiện ngoài block transaction, giúp pool không bị cạn kiệt dưới tải cao.

## Trade-off Comparison

| Tiêu chí | Check-then-act (Broken) | Solution 1 (Idempotency Key Table) |
|----------|-------------------------|------------------------------------|
| Data Integrity | Thất bại dưới concurrent | Đảm bảo an toàn tuyệt đối |
| Connection Pool | Bị giữ (held) lâu, dễ cạn kiệt | Được giải phóng trong quá trình Network IO |
| Retry Handling | Trả về trạng thái sai lầm | Phục hồi chính xác response cũ (Replay) |
| Độ trễ Database | Nhanh hơn (do gộp 1 tx) | Chậm hơn chút (chia làm 2-3 short transactions) |
| Khối lượng lưu trữ | Ít | Phải lưu thêm payload vào bảng `idempotency_keys` (cần job dọn dẹp) |

## Error Handling & Edge Cases
- **Garbage Collection**: Cần có một cron job định kỳ dọn dẹp các `Idempotency-Key` cũ (ví dụ > 30 ngày) để tránh bảng phình to.
- **Dangling In-Progress**: Nếu server crash ngay lúc đang gọi external API (Step 2), khóa sẽ kẹt ở trạng thái `IN_PROGRESS`. Cần một cơ chế quét các khóa `IN_PROGRESS` quá hạn (timeout) và chuyển sang `FAILED` để cho phép retry.
