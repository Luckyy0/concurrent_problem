# Phân tích mã lỗi (Broken Code Analysis)

## 1. Lược đồ cơ sở dữ liệu (Schema)

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    payment_code VARCHAR(50) NOT NULL UNIQUE,
    amount DECIMAL(19, 4) NOT NULL,
    status VARCHAR(20) NOT NULL, -- PENDING, AUTHORIZED, CAPTURED, FAILED
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE payment_events (
    id UUID PRIMARY KEY,
    payment_id UUID NOT NULL REFERENCES payments(id),
    provider_event_id VARCHAR(100) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    payload JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

## 2. Mã nguồn Java/Spring chứa lỗi (Broken Java/Spring Code)

```java
@Service
@RequiredArgsConstructor
public class PaymentCallbackService {

    private final PaymentRepository paymentRepository;
    private final PaymentEventRepository eventRepository;

    @Transactional
    public void processCallback(CallbackRequest request) {
        // 1. Lưu lại event log (chưa có unique constraint ngăn trùng lặp)
        PaymentEvent event = new PaymentEvent();
        event.setPaymentId(request.getPaymentId());
        event.setProviderEventId(request.getEventId());
        event.setEventType(request.getStatus());
        eventRepository.save(event);

        // 2. Tìm payment
        Payment payment = paymentRepository.findById(request.getPaymentId())
            .orElseThrow(() -> new EntityNotFoundException("Payment not found"));

        // 3. Cập nhật trạng thái trực tiếp
        // Anti-pattern: Không kiểm tra tính hợp lệ của việc chuyển đổi trạng thái
        // Anti-pattern: Thiếu lock đồng thời (pessimistic lock hoặc optimistic concurrency control)
        payment.setStatus(request.getStatus());
        payment.setUpdatedAt(Instant.now());
        
        paymentRepository.save(payment);
        
        // 4. Kích hoạt logic giao hàng nếu trạng thái là CAPTURED
        if ("CAPTURED".equals(request.getStatus())) {
            triggerFulfillment(payment); // Anti-pattern: Gọi side-effect trong transaction
        }
    }
}
```

## 3. Các phản mẫu (Anti-patterns) đã sử dụng

1. **Blind State Overwrite (Ghi đè trạng thái mù quáng):** Mã nguồn lấy trạng thái từ request và gán thẳng cho entity mà không kiểm tra xem việc chuyển đổi từ trạng thái hiện tại sang trạng thái mới có hợp lệ hay không (vd: `CAPTURED` về `AUTHORIZED` là không hợp lệ).
2. **Missing Idempotency Check (Thiếu kiểm tra lũy đẳng):** Hệ thống không kiểm tra xem `provider_event_id` đã được xử lý thành công trước đó hay chưa. Cùng một request được gửi hai lần sẽ tạo ra hai bản ghi log và cập nhật trạng thái hai lần.
3. **Read-Modify-Write without Locking (Đọc-Sửa-Ghi không có khóa):** Sử dụng JPA mặc định (không có khóa bi quan hay lạc quan). Nếu hai callback (ví dụ `AUTHORIZED` và `CAPTURED`) đến cùng lúc, có thể xảy ra race condition (lost update).
4. **Side-effects inside Database Transaction (Thực thi tác vụ phụ trong giao dịch):** Việc gọi `triggerFulfillment(payment)` bên trong `@Transactional` có thể dẫn đến việc gọi third-party API. Nếu API này chậm, database connection bị giữ lâu, gây cạn kiệt connection pool. Nếu API thành công nhưng commit CSDL thất bại, hệ thống bị inconsistent.
5. **No Unique Constraint on Event Logging:** Bảng `payment_events` thiếu ràng buộc `UNIQUE(provider_event_id)` để dựa vào CSDL ngăn chặn duplicate callback ngay ở mức schema.

## 4. Điều kiện tiền đề (Preconditions)

- Giao dịch `PAY-777` đang có trạng thái ban đầu là `PENDING`.
- Cổng thanh toán (PSP) gửi callback `AUTHORIZED` nhưng do mạng chậm, timeout xảy ra. PSP tiếp tục gửi `CAPTURED` khi khách hàng hoàn tất xác thực, và sau đó cơ chế retry của PSP lại gửi lại sự kiện `AUTHORIZED` lần nữa.
- Các yêu cầu HTTP được cân bằng tải đến nhiều instance/thread của hệ thống.

## 5. Dấu hiệu quan sát được (Observability Symptoms)

- **Logs:** Thấy nhiều dòng log xử lý cùng một `provider_event_id` thành công. Log ghi nhận trạng thái chuyển từ `PENDING` -> `CAPTURED` -> `AUTHORIZED`.
- **Database:** Bảng `payments` cho giao dịch `PAY-777` hiển thị trạng thái hiện hành là `AUTHORIZED`, dù trước đó đã có lúc lưu là `CAPTURED`. Bảng `payment_events` có nhiều bản ghi có chung `provider_event_id`.
- **Nghiệp vụ:** Khách hàng bị trừ tiền thành công nhưng đơn hàng trên hệ thống thương mại điện tử vẫn báo "Chưa hoàn tất thanh toán" (do bị revert về `AUTHORIZED`), hoặc đơn hàng được giao nhiều lần.
