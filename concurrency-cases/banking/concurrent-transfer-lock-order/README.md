# BANK-003: Chuyển khoản đồng thời và Deadlock do thứ tự Lock (Concurrent Transfer and Deterministic Lock Order)

## Tóm tắt (Overview)
Trong hệ thống ngân hàng, nghiệp vụ chuyển khoản (transfer) yêu cầu trừ tiền ở tài khoản nguồn (debit) và cộng tiền vào tài khoản đích (credit) trong cùng một `transaction`. Khi hai người dùng thực hiện chuyển khoản cho nhau cùng một lúc (A chuyển cho B, và B chuyển cho A), nếu hệ thống không sắp xếp thứ tự lấy `lock` trên các tài khoản, một tình trạng `deadlock` (khóa chéo) sẽ xảy ra tại mức cơ sở dữ liệu (database level). 

## Kịch bản nghiệp vụ (Business Scenario)
- **Tài khoản A** đang có 2,000,000 VND.
- **Tài khoản B** đang có 2,000,000 VND.
- **User A** thực hiện lệnh chuyển 1,000,000 VND sang tài khoản B.
- Cùng thời điểm đó, **User B** thực hiện lệnh chuyển 500,000 VND sang tài khoản A.
- Nếu `transaction` của User A khóa tài khoản A trước, và `transaction` của User B khóa tài khoản B trước, cả hai sẽ chờ nhau để khóa tài khoản còn lại, dẫn đến `deadlock`.

## Các diễn viên (Actors)
- **User A**: Khách hàng khởi tạo giao dịch chuyển khoản từ A sang B.
- **User B**: Khách hàng khởi tạo giao dịch chuyển khoản từ B sang A.
- **Transfer Service**: Application layer xử lý logic chuyển khoản.
- **Database (PostgreSQL)**: Hệ quản trị cơ sở dữ liệu lưu trữ số dư và quản lý `lock`.

## Bất biến (Invariants)
1. **Conservation of Money**: Tổng số dư của A và B trước và sau giao dịch phải không đổi (trừ khi có phí giao dịch, nhưng ở đây giả định phí = 0).
2. **No Deadlock**: Các giao dịch đồng thời không được phép kẹt vô hạn. Hệ thống phải đảm bảo tính khả dụng (liveness).
3. **Atomic State Mutation**: Giao dịch debit và credit phải thành công cùng nhau hoặc thất bại cùng nhau (atomic).

## Điểm tranh chấp (Contention Point)
- Cùng một lúc, 2 `transaction` tranh chấp quyền `lock` độc quyền (`FOR UPDATE`) trên 2 bản ghi (row) giống nhau trong bảng `account` nhưng theo chiều ngược nhau.

## Hậu quả trên Production (Production Consequences)
- **Transaction Rollback**: PostgreSQL sẽ phát hiện `deadlock` và tự động `rollback` một trong hai `transaction`.
- **User Experience**: Người dùng sẽ nhận được thông báo lỗi chung chung (HTTP 500) và giao dịch bị hủy bỏ, yêu cầu họ phải thử lại thủ công.
- **System Degradation**: Nếu có nhiều luồng chuyển khoản đan chéo trong các tài khoản tập trung (hot accounts), tỷ lệ `deadlock` tăng cao làm cạn kiệt `connection pool` tạm thời và giảm `throughput` của toàn hệ thống.

## Phạm vi áp dụng (Applicability & Scope Boundary)
- **Áp dụng**: Các hệ thống thực hiện thao tác trên nhiều bản ghi (multi-row transaction) yêu cầu `pessimistic lock` hoặc ghi đè đồng thời.
- **Nằm ngoài phạm vi**: Vấn đề trùng lặp lệnh chuyển khoản (duplicate transfer commands) sẽ được xử lý trong BANK-004. Ở đây giả định mỗi lệnh là duy nhất và hợp lệ.

## Thuật ngữ (Terminology)
| Thuật ngữ tiếng Anh | Định nghĩa / Ngữ cảnh |
|---------------------|----------------------|
| `transaction` | Một chuỗi các thao tác thao tác cơ sở dữ liệu được thực thi như một đơn vị nguyên tử. |
| `deadlock` | Tình trạng khóa chéo khi hai hay nhiều `transaction` chờ đợi lẫn nhau vô thời hạn. |
| `lock order` | Thứ tự mà các tài nguyên (row/bản ghi) được khóa trong một `transaction`. |
| `deterministic` | Có tính tất định, luôn cho ra cùng một kết quả/thứ tự không đổi với cùng đầu vào. |
| `rollback` | Hoàn tác lại toàn bộ các thay đổi của một `transaction` khi xảy ra lỗi. |
| `pessimistic lock` | Cơ chế khóa bi quan (`FOR UPDATE`), ngăn chặn các luồng khác thay đổi dữ liệu cho đến khi `transaction` kết thúc. |

## Điều hướng (Navigation)
- [Phân tích mã lỗi (broken-code.md)](./broken-code.md)
- [Phân tích luồng thực thi (analysis.md)](./analysis.md)
- [Giải pháp (solutions.md)](./solutions.md)
- [Thực nghiệm (experiments.md)](./experiments.md)

**Related Links:**
- [BANK-001: Concurrent Withdrawal and Double Spend](../concurrent-withdrawal-double-spend/README.md)
- [DB-008: Opposite Row Order Deadlock](../../postgresql/opposite-row-order-deadlock/README.md)
- [LOCK-003: Pessimistic Write / FOR UPDATE](../../locking/pessimistic-write-for-update/README.md)
