# BANK-003: Phân tích kỹ thuật (Analysis)

## Trạng thái ban đầu (Initial State)
- `Account A (ID=100)`: balance = 2,000,000
- `Account B (ID=200)`: balance = 2,000,000
- **Transaction T1**: User A chuyển 1,000,000 cho B (`from=100, to=200`).
- **Transaction T2**: User B chuyển 500,000 cho A (`from=200, to=100`).

## Diễn biến thời gian (Interleaving Timeline)

| Thời gian (Time) | Transaction T1 (A -> B) | Transaction T2 (B -> A) | PostgreSQL Lock Behavior / MVCC |
|------------------|-------------------------|-------------------------|---------------------------------|
| `t1` | `BEGIN` | | |
| `t2` | | `BEGIN` | |
| `t3` | `SELECT * FROM account WHERE id=100 FOR UPDATE;` | | T1 chiếm được `RowExclusiveLock` trên row ID=100 |
| `t4` | | `SELECT * FROM account WHERE id=200 FOR UPDATE;` | T2 chiếm được `RowExclusiveLock` trên row ID=200 |
| `t5` | Thực hiện trừ tiền A (Java RAM) | | |
| `t6` | | Thực hiện trừ tiền B (Java RAM) | |
| `t7` | `SELECT * FROM account WHERE id=200 FOR UPDATE;` | | T1 bị **BLOCK** vì row ID=200 đang bị T2 khóa. |
| `t8` | (Blocked) | `SELECT * FROM account WHERE id=100 FOR UPDATE;` | T2 bị **BLOCK** vì row ID=100 đang bị T1 khóa. |
| `t9` | (Blocked) | (Blocked) | PostgreSQL deadlock detector chạy (kiểm tra sau `deadlock_timeout`, mặc định 1s) |
| `t10` | | | Postgres phát hiện chu trình phụ thuộc (T1 chờ T2, T2 chờ T1) |
| `t11` | **ERROR: deadlock detected** | | Postgres chủ động `rollback` T1 để giải phóng khóa. |
| `t12` | `ROLLBACK` (T1 thất bại) | T2 hết block, lấy được khóa trên ID=100 | T2 lấy được `RowExclusiveLock` trên row ID=100 |
| `t13` | | Cập nhật số dư (RAM) | |
| `t14` | | `UPDATE account SET balance=...` (Flush) | |
| `t15` | | `COMMIT` | T2 hoàn thành thành công. |

## Kết quả Mong đợi vs Thực tế (Expected vs Actual)
- **Mong đợi**: Cả hai giao dịch cùng thành công tuần tự. Hoặc T1 xong trước rồi tới T2, số dư cuối cùng: A = 1,500,000, B = 2,500,000.
- **Thực tế**: `Deadlock` khiến một giao dịch bị hủy bỏ (T1 thất bại). Số dư cuối cùng bị ảnh hưởng bởi chỉ giao dịch T2: A = 2,500,000, B = 1,500,000. Lệnh của A không được thực hiện, ảnh hưởng xấu tới trải nghiệm người dùng.

## Nguyên nhân gốc rễ (Root Cause Analysis)
Nguyên nhân gốc rễ nằm ở **Application Layer** đã chỉ định yêu cầu hệ thống lấy khóa độc quyền (`Pessimistic Lock`) theo một thứ tự động, phụ thuộc hoàn toàn vào tham số nghiệp vụ (người gửi -> người nhận). 
- Ở tầng cơ sở dữ liệu (`Database Layer`), PostgreSQL thực thi đúng nhiệm vụ: nó cấp quyền khóa trên từng dòng một khi có yêu cầu. Khi phát hiện tình huống chờ vòng tròn (circular wait), nó phải `rollback` một trong hai `transaction` để bảo vệ hệ thống không bị treo cứng (liveness).

## Hành vi trong hệ thống nhiều máy chủ (Multi-instance Behavior)
Vấn đề hoàn toàn giống hệt dù bạn chạy 1 `instance` hay 100 `instances` của ứng dụng Spring Boot. Lý do là cơ chế `lock` được quản lý tập trung tại PostgreSQL. Deadlock vẫn sẽ xảy ra nếu thứ tự truy cập bản ghi tại cơ sở dữ liệu không đồng nhất.

## Recovery Timeline (Quá trình phục hồi)
Khi một giao dịch bị PostgreSQL ép `rollback`, `connection` sẽ được giải phóng. Tuy nhiên, nếu ứng dụng không có cơ chế `retry`, giao dịch đó coi như mất hoàn toàn. Để khôi phục tự động, ta cần cơ chế `Deadlock Retry` (bọc `transaction` trong một logic thử lại), hoặc phải sửa triệt để nguyên nhân tạo ra `deadlock` bằng cách sắp xếp **Lock Order**.
