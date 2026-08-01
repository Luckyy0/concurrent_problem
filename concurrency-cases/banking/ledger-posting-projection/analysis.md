# Analysis

## Initial State
- Bảng `balance_projection`: Tài khoản `3003`, `projected_balance = 5,000,000`.
- Bảng `ledger_entry`: Có bút toán khởi tạo `5,000,000`.

## Detailed Interleaving Timeline & Database Behavior

Quá trình xảy ra dưới mức `READ COMMITTED` mặc định của PostgreSQL.

| Time | Transaction A (+2,000,000) | Transaction B (-300,000) | PostgreSQL MVCC / Lock Behavior |
| :--- | :--- | :--- | :--- |
| T1 | `BEGIN` | | Bắt đầu Tx A. |
| T2 | `SELECT * FROM balance_projection WHERE account_id = '3003'` | | Đọc snapshot hiện tại. `projected_balance = 5,000,000`. |
| T3 | | `BEGIN` | Bắt đầu Tx B. |
| T4 | | `SELECT * FROM balance_projection WHERE account_id = '3003'` | Đọc snapshot hiện tại. `projected_balance = 5,000,000`. |
| T5 | `INSERT INTO ledger_entry ... (+2M)` | | Chèn dòng thành công. |
| T6 | | `INSERT INTO ledger_entry ... (-300K)` | Chèn dòng thành công. Không conflict với T5 vì PK/UQ khác nhau. |
| T7 | `UPDATE balance_projection SET projected_balance = 7000000 WHERE account_id = '3003'` | | Tx A lấy được `RowExclusiveLock` trên dòng tài khoản `3003`. Ghi `7,000,000`. |
| T8 | | `UPDATE balance_projection SET projected_balance = 4700000 WHERE account_id = '3003'` | Tx B bị block chờ `RowExclusiveLock` được giải phóng bởi Tx A. |
| T9 | `COMMIT` | | Tx A commit. Giải phóng lock. |
| T10 | | (Tx B tiếp tục) | Tx B nhận được lock, tự động re-evaluate (tính lại) điều kiện WHERE. Do WHERE chỉ check `account_id` (không check số dư cũ), điều kiện vẫn thỏa mãn. Tx B áp dụng giá trị `4,700,000` đè lên bản ghi mới của A. |
| T11 | | `COMMIT` | Tx B commit. Số dư cuối cùng là `4,700,000`. |

## Expected vs Actual
- **Expected**: Cả hai bút toán `ledger_entry` đều được ghi. Số dư `projected_balance` được cộng dồn đúng bằng `5,000,000 + 2,000,000 - 300,000 = 6,700,000`.
- **Actual**: Cả hai bút toán `ledger_entry` đều được ghi thành công, nhưng `projected_balance = 4,700,000`. `Invariant` bị phá vỡ.

## Root Cause by Layer

### 1. Application Layer (Java/Spring)
Việc sử dụng Spring Data JPA để fetch `Entity` về Java memory, thực hiện phép toán `add`/`subtract` rồi gọi `repository.save()` là một anti-pattern trong xử lý đồng thời. Ứng dụng coi object trên memory là trạng thái duy nhất.

### 2. ORM Layer (Hibernate)
Hibernate mặc định thực hiện dirty checking và sinh ra câu lệnh UPDATE toàn bộ các cột với giá trị trực tiếp từ object: `UPDATE table SET column = absolute_value`. Hibernate không tự động dịch các phép toán cộng trừ sang cú pháp SQL `column = column + delta`.

### 3. Database Layer (PostgreSQL)
Mức `READ COMMITTED` trong PostgreSQL không ngăn chặn `lost update`. Khi Tx B bị block và sau đó wake up, nó đánh giá lại điều kiện `WHERE` (chỉ là `account_id = '3003'`) và áp dụng thay đổi (với giá trị tuyệt đối đã tính sẵn ở application) mà không biết rằng dữ liệu đã bị thay đổi bởi Tx A.

## Multi-instance Behavior
Vấn đề càng nghiêm trọng hơn trong môi trường phân tán (multi-instance). Việc dùng các cơ chế synchronized tại JVM hay `ReentrantLock` chỉ có tác dụng trong 1 node, hoàn toàn vô dụng khi hai yêu cầu được route tới hai instance khác nhau.

## Recovery Timeline (Khắc phục sự cố)
Vì `ledger_entry` (source of truth) là append-only và không bị ảnh hưởng, việc khắc phục hoàn toàn khả thi:
1. Tạm dừng các giao dịch vào tài khoản `3003`.
2. Chạy script tính tổng lại từ `ledger`: `UPDATE balance_projection SET projected_balance = (SELECT SUM(amount) FROM ledger_entry WHERE account_id = '3003') WHERE account_id = '3003'`.
3. Mở lại giao dịch.
*Tuy nhiên, việc này tốn kém công sức vận hành và ảnh hưởng trải nghiệm người dùng.*
