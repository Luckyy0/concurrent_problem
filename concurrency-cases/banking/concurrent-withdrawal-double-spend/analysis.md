# Deep Dive Analysis

## 1. Initial State
- `account_id = 1001` có `available_balance = 1,000,000`.
- Isolation level: `READ COMMITTED` (mặc định của PostgreSQL).
- Hai yêu cầu API đồng thời yêu cầu rút 800,000 VND.

## 2. Root Cause By Layer

### Application Layer (Java/Spring)
- **State in Memory**: Entity `Account` được nạp vào persistence context của Hibernate. Java thread giữ trạng thái bộ nhớ này. Phép toán `subtract` thực hiện hoàn toàn trong bộ nhớ (`JVM`).
- **Absence of Concurrency Control**: JPA không tự động sử dụng khoá (`lock`). Gọi `save()` chỉ flush thay đổi dạng `UPDATE table SET col = new_value WHERE id = id`. 

### Database Layer (PostgreSQL & MVCC)
- Dưới mức cách ly `READ COMMITTED`, mỗi câu lệnh `SELECT` (như `findById`) sẽ đọc dữ liệu đã commit tại thời điểm câu lệnh đó bắt đầu chạy. Do cả hai transaction đều chưa commit, `SELECT` của T2 sẽ đọc được giá trị cũ (1,000,000).
- Khi T1 gọi `UPDATE`, nó sẽ lấy `Row Exclusive Lock` trên dòng dữ liệu. 
- Khi T2 gọi `UPDATE`, nó phải đợi (block) cho đến khi T1 hoàn thành (commit hoặc rollback).
- Khi T1 commit, T2 được tiếp tục. Nó đánh giá lại dòng đã được T1 update. Tuy nhiên, vì JPA gen ra câu lệnh dạng `UPDATE accounts SET available_balance = 200000 WHERE id = 1001` (ghi đè bằng giá trị tuyệt đối đã tính sẵn trong RAM), T2 vẫn update thành `200000`, ghi đè (lost update) lên kết quả của T1.

## 3. Expected vs Actual Behavior
- **Expected**: Yêu cầu 1 trừ thành công (dư 200k). Yêu cầu 2 bị từ chối `Insufficient funds` (ném Exception và rollback).
- **Actual**: Cả hai yêu cầu đều trừ thành công. Số dư lưu lại trong database là 200k (mất bản cập nhật). Double spending xảy ra.

## 4. Multi-instance Behavior
Vấn đề này vẫn xảy ra nguyên vẹn khi có nhiều instance của ứng dụng chạy song song (Horizontal scaling). Việc dùng các cơ chế lock trên JVM (như từ khoá `synchronized` hay `ReentrantLock` trong Java) sẽ **vô dụng** vì hai luồng xử lý có thể nằm trên hai máy chủ khác nhau, không chia sẻ bộ nhớ. Do đó, bài toán đồng bộ bắt buộc phải được giải quyết tại tầng Database (hoặc dùng Distributed Lock).

## 5. Recovery
- Việc khôi phục (recovery) rất khó khăn. Cần phải kiểm tra lại `ledger` (nếu có lưu) để tính toán số tiền thực sự bị trừ.
- Nếu tiền đã được chuyển sang tài khoản ngân hàng khác (chuyển khoản liên ngân hàng), việc thu hồi (rollback nghiệp vụ) phụ thuộc vào quy trình vận hành (Operation/Reconciliation).
