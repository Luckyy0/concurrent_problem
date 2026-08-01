# Phân tích Concurrency & Root Cause

## Initial State
- `account_1001`: `balance` = 1,000,000 VND.
- `account_2002`: `balance` = 0 VND.
- Bảng `transfer_request` rỗng.
- Client A bắt đầu gửi lệnh chuyển khoản với `idempotency_key` = `KEY_001`. Do timeout, Client A gửi lệnh `retry` thứ hai ngay lập tức. Cả 2 request đi vào hệ thống gần như đồng thời (Thread 1 và Thread 2).

## Detailed Interleaving Timeline

Mức độ cô lập (`Isolation Level`) mặc định của PostgreSQL là `READ COMMITTED`.

| Bước | Thread 1 (T1) | Thread 2 (T2) | PostgreSQL Behavior (READ COMMITTED) |
|------|---------------|---------------|---------------------------------------|
| 1 | Bắt đầu Transaction (TX1) | Bắt đầu Transaction (TX2) | Cấp phát transaction ID |
| 2 | `SELECT EXISTS WHERE key='KEY_001'` | | Trả về `false` do bảng rỗng |
| 3 | | `SELECT EXISTS WHERE key='KEY_001'`| Trả về `false` do T1 chưa `commit` |
| 4 | `INSERT INTO transfer_request` | | Ghi tạm dữ liệu vào WAL. Lấy PK ID=1 |
| 5 | | `INSERT INTO transfer_request` | Ghi tạm dữ liệu vào WAL. Lấy PK ID=2 |
| 6 | `SELECT * FROM account WHERE id='1001' FOR UPDATE` | | T1 giữ `Row Exclusive Lock` trên dòng `1001` |
| 7 | `UPDATE account SET balance=500000` | `SELECT * FROM account WHERE id='1001' FOR UPDATE` | T2 bị `Block`, chờ T1 nhả lock |
| 8 | `SELECT * FROM account WHERE id='2002' FOR UPDATE` | *(Đang chờ lock)* | T1 giữ lock trên dòng `2002` |
| 9 | `UPDATE account SET balance=500000` | *(Đang chờ lock)* | |
| 10 | Ghi các `ledger_entry` | *(Đang chờ lock)* | |
| 11 | `COMMIT` TX1 | *(Tiếp tục)* | T1 hoàn tất, giải phóng lock. |
| 12 | | T2 nhận được lock. Đọc `balance` mới (500000) | T2 lấy được snapshot mới nhất do `READ COMMITTED` |
| 13 | | `UPDATE account SET balance=0` | Trừ thêm 500,000 VND |
| 14 | | ... update 2002, insert ledger | |
| 15 | | `COMMIT` TX2 | T2 hoàn tất |

## Expected vs Actual
- **Expected**: Khi T2 thực hiện lệnh `exists`, nó phải bị chặn lại hoặc phát hiện ra T1 đã xử lý `KEY_001`. Nếu T1 thành công, T2 chỉ trả về thông báo đã xử lý mà không thay đổi số dư. Khách hàng bị trừ đúng 500,000 VND.
- **Actual**: Vì PostgreSQL không có `UNIQUE CONSTRAINT` trên cột `idempotency_key`, nó cho phép cả 2 lệnh `INSERT` thành công. Tính chất `READ COMMITTED` khiến T2 nhìn thấy số dư đã bị cập nhật của T1 (500,000) sau khi T1 commit, và tiếp tục trừ thêm (xuống 0). Khách hàng bị trừ tổng cộng 1,000,000 VND.

## Root Cause by Layer

### Application Layer
- Sử dụng mô hình `Check-Then-Act` mà không đảm bảo tính nguyên tử (`atomic`). Khi xử lý đồng thời, trạng thái của hệ thống đã thay đổi giữa thời điểm "kiểm tra" và "thực thi".

### Database Layer
- Thiếu `Constraint`: Lớp phòng thủ cuối cùng để bảo vệ tính nhất quán dữ liệu là các `constraint` ở mức database. Thiếu `UNIQUE(idempotency_key)` vô tình cho phép application ghi đè logic kinh doanh lên các dữ liệu lẽ ra phải không hợp lệ.

## Multi-Instance Behavior
- Nếu triển khai trên nhiều instance (Microservices / Kubernetes pods), tình huống này xảy ra thường xuyên hơn vì request có thể được cân bằng tải (`load balanced`) sang hai pod khác nhau, vượt qua các cơ chế lock nội bộ của JVM như `synchronized` hoặc `ReentrantLock`.
- Giải pháp khóa phân tán (`Distributed Lock` như Redis) có thể làm giảm rủi ro nhưng vẫn tiềm ẩn lỗi nếu khóa hết hạn trước khi database `commit` xong. Việc dựa vào Database Constraints là giải pháp an toàn nhất.

## Recovery
- Để sửa sai, nhân viên vận hành cần truy vấn các `transfer_request` có cùng `idempotency_key` và thời gian tạo gần nhau.
- Sau đó, lập bút toán đảo (`reversal ledger entry`) cho một trong các request bị trùng, hoàn lại tiền cho tài khoản gửi và trừ tiền tài khoản nhận. Việc này tốn nhiều công sức và ảnh hưởng trải nghiệm khách hàng.
