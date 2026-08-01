# BANK-005: Idempotent Payment Creation

## Business Scenario
Hệ thống cổng thanh toán (Payment Gateway) cung cấp một API POST `/payments` để tạo và xử lý các giao dịch thanh toán. Các API client (ví dụ: mobile app, web frontend, hoặc các dịch vụ upstream) được yêu cầu cung cấp một header `Idempotency-Key` (thường là UUID) để đảm bảo rằng các retry do lỗi mạng sẽ không tạo ra các khoản charge (trừ tiền) trùng lặp. 
Khi có sự cố mạng chậm trễ, client gửi yêu cầu lần 1 (tạo payment `PROCESSING`), bị timeout, và ngay lập tức gửi lại yêu cầu lần 2 với cùng `Idempotency-Key` `pay-abc-123`. Cả hai yêu cầu đến server gần như đồng thời.

## Actors
- **API Client**: Hệ thống gọi API thanh toán (ví dụ: E-commerce Checkout Service) có cơ chế auto-retry.
- **Payment API**: Ứng dụng Spring Boot cung cấp endpoint thanh toán.
- **Database**: Hệ quản trị cơ sở dữ liệu (PostgreSQL) lưu trữ trạng thái thanh toán.

## Invariants (Bất biến)
1. Một `Idempotency-Key` duy nhất chỉ được phép tạo ra tối đa một bản ghi payment.
2. Các request tiếp theo với cùng `Idempotency-Key` (nếu request trước đã thành công) phải trả về cùng một kết quả (response payload) thay vì xử lý lại.
3. Nếu request đầu tiên đang ở trạng thái xử lý (`PROCESSING`), request thứ hai đến đồng thời phải bị từ chối với HTTP 409 Conflict hoặc chờ cho đến khi request đầu tiên hoàn tất.

## Contention Point
Điểm tranh chấp xảy ra tại bước khởi tạo payment. Hai thread (đại diện cho 2 API request đồng thời) cùng kiểm tra sự tồn tại của `Idempotency-Key` trong database và sau đó cùng tiến hành `INSERT` bản ghi mới nếu chưa tồn tại.

## Production Consequences
- **Khách hàng bị trừ tiền nhiều lần**: Tạo ra duplicate payments cho cùng một đơn hàng, dẫn đến khách hàng bị charge tiền gấp đôi.
- **Reconciliation Issues (Vấn đề đối soát)**: Lệch số liệu giữa hệ thống nội bộ và phía ngân hàng/đối tác.
- **Trải nghiệm tồi tệ**: Quá trình hoàn tiền (refund) mất nhiều thời gian, làm giảm độ tin cậy của dịch vụ.

## Applicability & Scope Boundary
- **Applicability**: Bất kỳ hệ thống phân tán nào phơi bày API cho việc tạo mới resource nhạy cảm (payments, orders, ledger entries) có hỗ trợ retry từ client.
- **Scope Boundary**: Case study này tập trung vào vòng đời API idempotency (bảo vệ ở mức endpoint/database). Nó không bao gồm xử lý duplicate từ webhook/callback của đối tác payment processor (thường giải quyết qua Transaction ID của bên thứ ba).

## Terminology
| Term | Nghĩa Tiếng Việt | Canonical English Term | Ý nghĩa trong bối cảnh này |
|------|----------------|------------------------|---------------------------|
| Idempotency Key | Khóa toàn vẹn | Idempotency Key | Một chuỗi duy nhất client gửi kèm request để định danh một hành động duy nhất, an toàn khi retry. |
| Check-then-act | Kiểm tra rồi thực thi | Check-then-act | Một race condition phổ biến khi kiểm tra trạng thái và hành động trên trạng thái đó không phải là atomic. |
| In-progress state | Trạng thái đang xử lý | In-progress state | Trạng thái khi một request đang được xử lý, cần ngăn chặn các request trùng lặp khác. |
| Response replay | Phát lại phản hồi | Response replay | Trả về chính xác response của request gốc cho các request retry. |

## Navigation
- [Broken Code & Anti-patterns](broken-code.md)
- [Concurrency Analysis](analysis.md)
- [Solutions](solutions.md)
- [Experiments & Tests](experiments.md)
- Related: [DB-006: Unique Constraint Concurrency](../../postgresql/unique-constraint-concurrency/README.md)
- Related: [BANK-004: Duplicate Transfer Request](../duplicate-transfer-request/README.md)
