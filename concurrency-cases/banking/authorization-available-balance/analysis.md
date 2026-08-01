# Phân tích nguyên nhân gốc rễ (Root Cause Analysis)

## Trạng thái ban đầu (Initial State)
- **Tài khoản (Account)**: `id` = 4004, `posted_balance` = 2,000,000.
- **Hold (AuthorizationHold)**: Chưa có khoản hold nào (Total Holds = 0).
- **Yêu cầu (Requests)**:
  - Giao dịch A (Tx A): Yêu cầu giữ 1,500,000.
  - Giao dịch B (Tx B): Yêu cầu giữ 1,200,000.

## Interleaving Timeline

| Thời gian | Tx A (Thread 1) | Tx B (Thread 2) | PostgreSQL / Database State (Read Committed) |
| --- | --- | --- | --- |
| t1 | `BEGIN` | | |
| t2 | | `BEGIN` | |
| t3 | `SELECT * FROM account WHERE id=4004` | | Trả về `posted_balance = 2,000,000` |
| t4 | | `SELECT * FROM account WHERE id=4004` | Trả về `posted_balance = 2,000,000` |
| t5 | `SELECT * FROM authorization_hold WHERE account_id=4004 AND status='ACTIVE'` | | Trả về danh sách rỗng (Total = 0) |
| t6 | | `SELECT * FROM authorization_hold WHERE account_id=4004 AND status='ACTIVE'` | Trả về danh sách rỗng (Total = 0) |
| t7 | Tính: `avail = 2M - 0 = 2M`. `2M >= 1.5M` (OK) | | |
| t8 | | Tính: `avail = 2M - 0 = 2M`. `2M >= 1.2M` (OK) | |
| t9 | `INSERT INTO authorization_hold (id='H1', amount=1.5M)` | | Ghi log vào WAL, chưa commit. |
| t10| | `INSERT INTO authorization_hold (id='H2', amount=1.2M)` | Ghi log vào WAL, chưa commit. Không bị block do chèn ID khác nhau. |
| t11| `COMMIT` | | Row `H1` được hiển thị cho các transaction khác. |
| t12| | `COMMIT` | Row `H2` được hiển thị. |

## Expected vs Actual
- **Kỳ vọng (Expected)**: Một trong hai giao dịch phải bị từ chối với `InsufficientFundsException` do tổng hold (1,500,000 + 1,200,000 = 2,700,000) lớn hơn số dư 2,000,000.
- **Thực tế (Actual)**: Cả hai giao dịch đều được duyệt. Có 2 bản ghi hold mới được tạo. Khách hàng đã được phép tiêu 2,700,000 trong khi chỉ có 2,000,000.

## Root Cause theo từng layer

### Tầng Ứng dụng (Java/Spring)
Ứng dụng thực hiện check-then-act mà không có cơ chế đồng bộ hóa. Việc kiểm tra điều kiện logic dựa trên kết quả đọc từ hai bảng khác nhau (`account` và `authorization_hold`) và sau đó thay đổi trạng thái hệ thống bằng cách ghi một bản ghi mới. Điều này vi phạm tính nguyên tử (atomicity) của logical transaction.

### Tầng Database (PostgreSQL & MVCC)
Dưới mức cô lập (isolation level) mặc định là `Read Committed`, PostgreSQL cho phép mỗi câu truy vấn đọc những dữ liệu đã được commit trước khi câu truy vấn đó bắt đầu. Tx A và Tx B đều không thấy sự hiện diện của bản ghi mới do phía bên kia đang chèn (phantom inserts). Hơn nữa, vì cả hai chèn vào hai bản ghi có ID khác nhau (UUID), PostgreSQL hoàn toàn không phát hiện bất kỳ xung đột nào (no write conflict). 

Dù có nâng lên `Repeatable Read`, lỗi này vẫn có thể xảy ra trong một số cấu hình nếu hệ quản trị CSDL sử dụng snapshot isolation, bởi việc ghi vào một bảng không tạo ra conflict trực tiếp với việc đọc ở bảng đó (write skew / phantom read anomaly). Chỉ với `Serializable` thì PostgreSQL mới có thể phát hiện read/write dependencies và abort một transaction, nhưng hiệu năng sẽ rất kém.

## Giải pháp phục hồi
Khi sự cố đã xảy ra (khách hàng tiêu lố tiền):
- **Phục hồi**: Nếu giao dịch sau đó tới bước capture, tài khoản có thể bị âm tiền (bằng cách trừ thẳng vào `posted_balance`). Ngân hàng sẽ phải thu hồi nợ.
- Tự động huỷ (cancel) một trong các uỷ quyền nếu hệ thống hỗ trợ việc kiểm tra sau cấp phép (post-authorization review), tuy nhiên điều này ảnh hưởng nghiêm trọng đến trải nghiệm người dùng (giao dịch tại POS bị từ chối sau khi đã báo thành công).
