# BANK-006: Duplicate and Out-of-Order Payment Callback (Callback bị trùng lặp và sai thứ tự)

## 1. Tình huống nghiệp vụ (Business Scenario)

Hệ thống cổng thanh toán (Payment Gateway) tích hợp với các nhà cung cấp dịch vụ thanh toán (Payment Provider/PSP). Khi một giao dịch thanh toán thay đổi trạng thái tại PSP (ví dụ: từ `PENDING` sang `AUTHORIZED` rồi `CAPTURED`), PSP sẽ gửi webhook/callback về hệ thống để cập nhật trạng thái tương ứng.

Trong môi trường phân tán, việc giao tiếp mạng không đảm bảo độ tin cậy tuyệt đối. PSP thường áp dụng cơ chế thử lại (retry) khi không nhận được phản hồi thành công (HTTP 200 OK) từ hệ thống. Hậu quả là:
1. **Duplicate Callback**: Một sự kiện trạng thái (ví dụ: `AUTHORIZED`) có thể được gửi nhiều lần.
2. **Out-of-Order Callback**: Do độ trễ mạng hoặc cơ chế hàng đợi của PSP, sự kiện xảy ra sau (ví dụ: `CAPTURED`) lại đến hệ thống *trước* sự kiện xảy ra trước (`AUTHORIZED`).

Trường hợp điển hình cho giao dịch `PAY-777`:
1. Callback A: `status=AUTHORIZED` (đến trước)
2. Callback B: `status=CAPTURED` (đến sau)
3. Callback C: `status=AUTHORIZED` (lặp lại do provider retry, đến sau cùng)

Hệ thống phải xử lý để: Chấp nhận A, chấp nhận B (tiến trạng thái lên `CAPTURED`), và từ chối/bỏ qua C (vì `CAPTURED` là trạng thái cuối cùng hoặc cao hơn `AUTHORIZED`).

## 2. Chủ thể (Actors) & Bất biến (Invariants)

**Actors:**
- **Provider (PSP):** Gửi các HTTPS POST request chứa thông tin trạng thái mới nhất của giao dịch thanh toán.
- **PaymentService:** Dịch vụ nhận và xử lý callback, cập nhật trạng thái giao dịch trong database.

**Invariants (Bất biến hệ thống):**
1. **Monotonic State Machine (Máy trạng thái đơn điệu):** Trạng thái giao dịch chỉ được phép tiến lên theo chiều logic nghiệp vụ (vd: `PENDING` -> `AUTHORIZED` -> `CAPTURED`). Không bao giờ được phép lùi trạng thái (regression).
2. **Exactly-Once Processing:** Mỗi sự kiện callback mang tính logic (xác định bởi một ID sự kiện duy nhất từ provider) chỉ được xử lý thành công một lần. Các bản sao (duplicate) phải bị bỏ qua mà không gây ra lỗi nghiệp vụ (Idempotency).
3. **Consistency:** Bất kể thứ tự nhận được từ mạng, trạng thái cuối cùng của giao dịch trong database phải phản ánh trạng thái mới nhất hợp lệ từ Provider.

## 3. Điểm tranh chấp (Contention Point)

Nhiều thread HTTP worker cùng nhận các callback request cho cùng một `payment_id` gần như đồng thời và cùng cố gắng đọc, đánh giá và ghi đè trạng thái `status` của bản ghi `Payment` trong CSDL PostgreSQL.

## 4. Hậu quả trên môi trường Production (Production Consequences)

- **Trạng thái thụt lùi (Status Regression):** Giao dịch đã thành công (`CAPTURED`) bị chuyển ngược về `AUTHORIZED` hoặc `PENDING`, khiến hệ thống không ghi nhận doanh thu hoặc không thực hiện giao hàng cho khách.
- **Giao hàng kép (Duplicate Fulfillment):** Khi sự kiện `CAPTURED` bị xử lý trùng lặp, nếu hệ thống kích hoạt logic giao hàng (fulfillment) dựa trên sự thay đổi trạng thái, khách hàng có thể nhận được hàng nhiều lần cho một lần thanh toán.
- **Lỗi đối soát (Reconciliation Failures):** Dữ liệu trạng thái và lịch sử giao dịch không khớp với đối tác, gây tốn nhiều thời gian xử lý thủ công (manual operation).

## 5. Phạm vi (Scope Boundary)

- **Bao gồm:** Xử lý sự kiện webhook từ đối tác bên ngoài trực tiếp qua HTTP REST endpoints, sử dụng cơ sở dữ liệu quan hệ (PostgreSQL) để duy trì trạng thái và tính lũy đẳng (idempotency).
- **Không bao gồm:** Cơ chế hàng đợi trung gian như Kafka hay RabbitMQ (xem các case MSG-*), hoặc logic xử lý thanh toán outbound.

## 6. Liên kết (Navigation & References)

- [BANK-005: Idempotent Payment Creation](../idempotent-payment-creation/README.md)
- [DB-006: Unique Constraint Concurrency](../../postgresql/unique-constraint-concurrency/README.md)
- [Concepts: Idempotency and Uniqueness](../../concepts/idempotency-and-uniqueness.md)
- [Concepts: Concurrency Testing](../../concepts/concurrency-testing.md)

## 7. Thuật ngữ (Terminology)

| Thuật ngữ | English Term | Mô tả |
| --- | --- | --- |
| Lũy đẳng | Idempotency | Thuộc tính của một thao tác: thực hiện nhiều lần mang lại kết quả giống như thực hiện một lần. |
| Máy trạng thái đơn điệu | Monotonic State Machine | Máy trạng thái chỉ cho phép chuyển đổi một chiều theo quy tắc nhất định, không cho phép đi lùi. |
| Thử lại | Retry | Cơ chế gọi lại một tác vụ (thường là qua mạng) khi lần gọi trước thất bại hoặc timeout. |
| Callback/Webhook | Callback/Webhook | HTTP request do hệ thống bên ngoài gọi vào để thông báo sự kiện. |
