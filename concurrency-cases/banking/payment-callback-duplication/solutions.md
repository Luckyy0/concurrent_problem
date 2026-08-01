# Giải pháp (Solutions)

Để giải quyết vấn đề duplicate và out-of-order callback, chúng ta cần kết hợp nhiều kỹ thuật ở cả cấp cơ sở dữ liệu và ứng dụng.

## 1. Cập nhật Lược đồ (Schema Updates)

Áp dụng Constraint để xử lý lỗi ngay từ tầng Database.

```sql
-- 1. Thêm Unique Constraint để chặn Duplicate Callback ngay từ lúc Insert
ALTER TABLE payment_events 
ADD CONSTRAINT uk_provider_event_id UNIQUE (provider_event_id);

-- (Tùy chọn) 2. Bảng định nghĩa State Machine để quản lý độ ưu tiên trạng thái
-- PENDING=1, AUTHORIZED=2, CAPTURED=3, FAILED=4
```

## 2. Giải pháp 1: Lũy đẳng + Máy trạng thái với Conditional Update (Khuyên dùng)

Giải pháp này sử dụng **Idempotency Key** thông qua Unique Constraint để chặn callback trùng lặp, và **Conditional Update (affected rows)** kết hợp với trọng số trạng thái để chặn các callback sai thứ tự (Out-of-Order).

### Thiết kế State Machine (Monotonicity)

Chúng ta có thể gán cấp độ ưu tiên cho trạng thái:
- `PENDING` (0)
- `AUTHORIZED` (1)
- `CAPTURED` (2)
- `FAILED` (2)

Trạng thái chỉ có thể chuyển từ mức thấp lên mức cao, không được chuyển ngược.

### Mã nguồn Java (Java Code)

```java
@Service
@RequiredArgsConstructor
public class PaymentCallbackService {

    private final PaymentRepository paymentRepository;
    private final PaymentEventRepository eventRepository;
    private final FulfillmentService fulfillmentService;

    // Sử dụng Map tĩnh để biểu diễn cấp độ ưu tiên trạng thái
    private static final Map<String, Integer> STATE_WEIGHTS = Map.of(
        "PENDING", 0,
        "AUTHORIZED", 1,
        "CAPTURED", 2,
        "FAILED", 2
    );

    @Transactional
    public void processCallback(CallbackRequest request) {
        // 1. Kiểm tra Lũy đẳng (Idempotency) dựa trên DB Unique Constraint
        try {
            PaymentEvent event = new PaymentEvent();
            event.setPaymentId(request.getPaymentId());
            event.setProviderEventId(request.getEventId());
            event.setEventType(request.getStatus());
            eventRepository.saveAndFlush(event); // Bắt lỗi UniqueConstraintViolation ngay lập tức
        } catch (DataIntegrityViolationException ex) {
            // Callback đã được xử lý trước đó. Bỏ qua an toàn.
            log.info("Duplicate callback ignored: {}", request.getEventId());
            return;
        }

        // 2. Conditional Update để đảm bảo Monotonic State (Chặn Out-of-Order)
        Payment payment = paymentRepository.findById(request.getPaymentId())
            .orElseThrow(() -> new EntityNotFoundException("Payment not found"));

        int currentWeight = STATE_WEIGHTS.getOrDefault(payment.getStatus(), -1);
        int targetWeight = STATE_WEIGHTS.getOrDefault(request.getStatus(), -1);

        if (targetWeight <= currentWeight) {
            // Out-of-Order event hoặc event không hợp lệ, bỏ qua cập nhật trạng thái
            log.warn("Out of order state transition ignored. Current: {}, Target: {}", 
                     payment.getStatus(), request.getStatus());
            return; // Đã lưu event log ở trên nhưng không đổi trạng thái payment
        }

        // Cập nhật bằng JPQL sử dụng optimistic/conditional update
        int updatedRows = paymentRepository.updateStatusConditionally(
                payment.getId(), 
                request.getStatus(), 
                payment.getStatus() // Cần đúng trạng thái cũ để tránh Lost Update
        );

        if (updatedRows == 0) {
            // Bản ghi đã bị thread khác cập nhật, throw Exception để transaction rollback và PSP tự retry
            throw new OptimisticLockingFailureException("Payment status modified concurrently");
        }

        // 3. Sử dụng Outbox Pattern hoặc Event Publisher cho Side-effect (KHÔNG gọi API trực tiếp)
        if ("CAPTURED".equals(request.getStatus())) {
            // Lưu message vào bảng Outbox cùng transaction, worker sẽ đọc và gửi sau
            fulfillmentService.scheduleFulfillment(payment.getId()); 
        }
    }
}
```

### Spring Data JPA Repository

```java
public interface PaymentRepository extends JpaRepository<Payment, UUID> {
    
    @Modifying
    @Query("UPDATE Payment p SET p.status = :newStatus, p.updatedAt = CURRENT_TIMESTAMP " +
           "WHERE p.id = :id AND p.status = :oldStatus")
    int updateStatusConditionally(
        @Param("id") UUID id, 
        @Param("newStatus") String newStatus, 
        @Param("oldStatus") String oldStatus
    );
}
```

## 3. Tại sao giải pháp này bảo vệ được Bất biến (Why Invariants are Protected)

1. **Exactly-Once Processing:** Nếu 2 callback giống hệt nhau (`provider_event_id` giống nhau) tới cùng lúc, câu lệnh `INSERT` vào bảng `payment_events` sẽ văng lỗi `DataIntegrityViolationException` cho thread thứ hai (nhờ Unique Constraint). Thread này bắt lỗi và thoát êm đẹp (idempotent).
2. **Monotonic State Machine (Out-of-Order Handling):** Nếu callback `AUTHORIZED` đến sau callback `CAPTURED`, logic mã nguồn so sánh trọng số (`targetWeight <= currentWeight`) và từ chối cập nhật, bảo vệ trạng thái cao hơn hiện tại.
3. **Concurrency Control:** Việc sử dụng câu lệnh `UPDATE ... WHERE id = :id AND status = :oldStatus` đảm bảo khóa row-level nguyên tử (atomic). Nếu một thread khác đã thay đổi trạng thái, affected row sẽ là 0, ngăn chặn lỗi Lost Update.

## 4. Xử lý Lỗi và Trade-offs

- **Cấu hình Retry:** Nếu trả về lỗi `OptimisticLockingFailureException`, transaction của Spring sẽ rollback toàn bộ (gồm cả insert event log). Cổng thanh toán (PSP) nhận được HTTP 500 sẽ retry lại. Ở lần retry sau, trạng thái DB đã ổn định, request có thể bị loại bởi logic Out-of-Order hoặc thành công.
- **Tách biệt Side-effect:** Không gọi API bên thứ ba (`triggerFulfillment`) trong `@Transactional`. Thay vào đó, sử dụng Pattern Transactional Outbox (ghi yêu cầu gọi hàm vào một bảng trong DB cùng một transaction, và một schedule/worker khác sẽ xử lý gửi lời gọi đi). Việc này tránh cạn kiệt Connection Pool.
