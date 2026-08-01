# Phân tích sự cố đồng thời (Concurrency Analysis)

## 1. Trạng thái ban đầu (Initial State)

- Bảng `payments`: `id = UUID-1`, `payment_code = 'PAY-777'`, `status = 'PENDING'`.
- Bảng `payment_events`: Rỗng đối với `payment_id = UUID-1`.
- Cổng thanh toán (PSP) kích hoạt gửi 3 callback (do retry và độ trễ mạng):
  - Request A: event_id `evt_01`, status `AUTHORIZED`
  - Request B: event_id `evt_02`, status `CAPTURED`
  - Request C: event_id `evt_01`, status `AUTHORIZED` (thử lại của A)

## 2. Diễn biến thời gian xen kẽ (Interleaving Timeline)

Kịch bản dưới đây mô tả lúc Request B (đến sau nhưng được xử lý nhanh) và Request C (bị trễ/retry) gây ra lỗi thoái lui trạng thái (Status Regression).

| Thời gian | Thread 1 (Xử lý Request B: CAPTURED) | Thread 2 (Xử lý Request C: AUTHORIZED - retry) | PostgreSQL Behavior (MVCC / Locks) |
| --- | --- | --- | --- |
| t1 | Bắt đầu transaction (T1) | | Mở TX T1 |
| t2 | | Bắt đầu transaction (T2) | Mở TX T2 |
| t3 | `INSERT INTO payment_events (evt_02)` | | Cấp row lock cho bản ghi event mới (T1) |
| t4 | | `INSERT INTO payment_events (evt_01)` | Cấp row lock cho bản ghi event mới (T2). Không lỗi vì không có Unique Constraint |
| t5 | `SELECT * FROM payments WHERE id='UUID-1'`<br>*(Trạng thái lấy được: PENDING)* | | Lấy snapshot của T1. |
| t6 | | `SELECT * FROM payments WHERE id='UUID-1'`<br>*(Trạng thái lấy được: PENDING)* | Lấy snapshot của T2. |
| t7 | `UPDATE payments SET status='CAPTURED'` | | Cấp lock trên bản ghi payment `UUID-1` cho T1. Ghi thành công. |
| t8 | Gọi `triggerFulfillment()` (API call) | | |
| t9 | API call thành công | `UPDATE payments SET status='AUTHORIZED'` | T2 bị block, chờ T1 release lock trên bản ghi payment. |
| t10 | Commit T1 | | T1 giải phóng lock. |
| t11 | | Thực thi UPDATE của T2. | T2 lấy được lock. Ghi đè `status` thành `AUTHORIZED` (Lost Update / Regression). |
| t12 | | Commit T2 | T2 commit thành công. |

## 3. Mong đợi và Thực tế (Expected vs Actual)

- **Mong đợi (Expected):** Request C (thử lại) phải bị từ chối do `evt_01` đã được xử lý (hoặc đã lỗi thời). Request B phải cập nhật trạng thái lên `CAPTURED`. Trạng thái cuối cùng là `CAPTURED`. Logic fulfillment chỉ chạy đúng 1 lần.
- **Thực tế (Actual):** Cả 3 request đều có thể được xử lý thành công. Trạng thái cuối cùng bị lùi về `AUTHORIZED`. Logic fulfillment có thể chạy nhiều lần nếu có nhiều retry cho trạng thái `CAPTURED`.

## 4. Nguyên nhân gốc rễ (Root Cause by Layer)

- **Application Layer (Java/Spring):** Không thực thi mô hình trạng thái đơn điệu (Monotonic State Machine) bằng code (như kiểm tra if-else hợp lệ trước khi update). Không áp dụng Idempotency Key deduplication để ngăn callback lặp lại. Đặt side-effect (API call) vào trong `@Transactional`.
- **Database Layer (PostgreSQL):** Không cấu hình Unique Constraint trên cột `provider_event_id` tại bảng `payment_events` để tận dụng DB phát hiện trùng lặp. Phụ thuộc vào transaction mặc định (Read Committed) khiến hiện tượng Lost Update xảy ra khi hai thread cùng đọc trạng thái cũ và ghi đè trạng thái mới lên nhau.

## 5. Hành vi Multi-instance

Khi hệ thống có nhiều pod/instance (chẳng hạn trên Kubernetes), các yêu cầu callback có thể được định tuyến ngẫu nhiên đến các instance khác nhau. Do không có cơ chế khóa phân tán hoặc khóa cấp cơ sở dữ liệu đúng cách, các instance này sẽ cạnh tranh trực tiếp trên database. Dữ liệu lỗi vẫn xảy ra y hệt trong kịch bản một instance với nhiều threads.

## 6. Lộ trình khôi phục (Recovery Timeline)

Khi xảy ra lỗi trên Production, việc khôi phục rất khó khăn:
1. Phải rà soát log và đối chiếu với file đối soát từ đối tác (Provider) để tìm ra các giao dịch bị sai trạng thái.
2. Cập nhật lại thủ công bằng SQL script các bản ghi về trạng thái đúng (`CAPTURED`).
3. Khắc phục hậu quả của việc `triggerFulfillment()` lặp lại (ví dụ: liên hệ khách hàng/bộ phận vận hành kho để thu hồi hàng, hoặc chịu lỗ).
