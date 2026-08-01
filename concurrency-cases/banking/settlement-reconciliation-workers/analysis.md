# Analysis: Concurrent Settlement and Reconciliation Workers

## 1. Initial State

- `account_balances`: `SYSTEM_ACCOUNT` có số dư (`balance`) là 0.
- `settlement_lines`: Có 1 dòng `S-1001` đang ở trạng thái `PENDING`, amount = 1000.
- Worker A và Worker B là các luồng (thread) hoặc ứng dụng độc lập, cùng kích hoạt tiến trình `processPendingSettlements()` gần như đồng thời.

## 2. Detailed Interleaving Timeline

Môi trường: PostgreSQL với Isolation Level `READ COMMITTED`.

| T | Worker A (Tx1) | Worker B (Tx2) | PostgreSQL MVCC / Lock Behavior |
|---|----------------|----------------|---------------------------------|
| 1 | `BEGIN` | | Bắt đầu Tx1 |
| 2 | | `BEGIN` | Bắt đầu Tx2 |
| 3 | `SELECT * FROM settlement_lines WHERE status='PENDING' LIMIT 50;` | | Tx1 đọc Snapshot, thấy `S-1001`. Không có row lock. |
| 4 | | `SELECT * FROM settlement_lines WHERE status='PENDING' LIMIT 50;` | Tx2 đọc Snapshot, thấy `S-1001`. |
| 5 | Phát hiện Discrepancy với `S-1001` | | |
| 6 | | Phát hiện Discrepancy với `S-1001`| |
| 7 | `INSERT INTO ledger_entries VALUES ('uuid_1', 'S-1001', 1000, 'ADJUSTMENT');` | | Tx1 ghi insert thành công (row lock trên dòng mới). |
| 8 | | `INSERT INTO ledger_entries VALUES ('uuid_2', 'S-1001', 1000, 'ADJUSTMENT');` | Tx2 ghi insert thành công (không conflict vì khóa chính UUID khác nhau). |
| 9 | `SELECT balance FROM account_balances WHERE account_id='SYSTEM_ACCOUNT';` | | Tx1 đọc balance = 0 |
| 10| | `SELECT balance FROM account_balances WHERE account_id='SYSTEM_ACCOUNT';` | Tx2 đọc balance = 0 (Tx1 chưa commit) |
| 11| `UPDATE account_balances SET balance=1000 WHERE id='SYSTEM_ACCOUNT';`| | Tx1 lấy Row Lock độc quyền trên `SYSTEM_ACCOUNT`. Ghi số dư = 1000. |
| 12| | `UPDATE account_balances SET balance=1000 WHERE id='SYSTEM_ACCOUNT';`| Tx2 muốn lấy Row Lock trên `SYSTEM_ACCOUNT` nhưng bị chặn bởi Tx1 (Wait). |
| 13| `UPDATE settlement_lines SET status='DISCREPANCY' WHERE id='S-1001';` | | Tx1 lấy Row Lock trên `S-1001`. |
| 14| `COMMIT` | | Tx1 release toàn bộ locks. Tx1 thành công. |
| 15| | (Tx2 được cấp lock) | Tx2 áp dụng UPDATE, ghi đè balance = 1000. (Lỗi Lost Update: Đáng lẽ phải là 2000 hoặc chỉ 1000 nhưng 1 worker thất bại). Thực tế trong SQL RMW nếu sửa lại là `balance = balance + 1000`, Tx2 sẽ tính toán lại dựa trên giá trị commit của Tx1 => Balance = 2000 (Double Adjustment). |
| 16| | `UPDATE settlement_lines SET status='DISCREPANCY' WHERE id='S-1001';` | Tx2 ghi đè status của S-1001 thành `DISCREPANCY` thành công. |
| 17| | `COMMIT` | Tx2 thành công. |

## 3. Expected vs Actual

- **Expected**: `S-1001` chỉ được xử lý bởi một worker. Có đúng 1 `ledger_entries` sinh ra cho `S-1001` (amount = 1000). Số dư của `SYSTEM_ACCOUNT` thay đổi đúng 1000.
- **Actual**: `S-1001` bị xử lý 2 lần. Có 2 `ledger_entries` được sinh ra (`uuid_1` và `uuid_2`). Số dư của `SYSTEM_ACCOUNT` thay đổi 2000 (Bị double).

## 4. Root Cause by Layer

- **Application Layer**: 
  - Thiếu điều phối (coordination) giữa các worker. 
  - Tạo `Idempotency Key` (bằng UUID random) cho các operation không luỹ đẳng.
  - Sử dụng mô hình *Read-Modify-Write* (RMW) không an toàn ở tầng business (`balance = balance.add(1000)`).
- **Data Access Layer**: 
  - Không sử dụng `FOR UPDATE SKIP LOCKED` khi claim task từ hàng đợi (bảng `settlement_lines`).
- **Database Layer**:
  - `READ COMMITTED` không ngăn được dirty reads đối với ý nghĩa nghiệp vụ (worker B không biết worker A đang xử lý dữ liệu) vì transaction A chưa commit.
  - Không có Unique Constraint đảm bảo `settlement_id` + `type` là duy nhất trong bảng `ledger_entries`.

## 5. MVCC / Lock Behavior Detail

Trong PostgreSQL:
- Câu lệnh `SELECT` không có từ khoá khóa (`FOR UPDATE/SHARE`) sẽ sử dụng MVCC snapshot mới nhất được commit. Do đó cả 2 worker cùng nhìn thấy `S-1001` có `status='PENDING'`.
- Khi Insert một dòng với ID mới (UUID.randomUUID), PostgreSQL không coi đây là conflict kể cả khi ý nghĩa nghiệp vụ là trùng lặp.
- Lỗi Lost Update hoặc Double Increment xảy ra tuỳ theo cách ứng dụng viết câu UPDATE. Nếu sử dụng ORM mapping RMW (`balance = 0 + 1000`), kết quả cuối là 1000 (Lost Update). Nếu dùng SQL RMW (`balance = balance + 1000`), kết quả cuối là 2000 (Double Application). Cả hai đều vi phạm bất biến.

## 6. Multi-instance Behavior

Khi triển khai trên K8s (Kubernetes) với nhiều pods, tần suất xảy ra lỗi này cực cao (lên đến 100% đối với các scheduler kích hoạt cùng 1 thời điểm). Các instances đều không nhận biết được hành động của nhau.

## 7. Recovery Timeline (Tái cấu trúc lại dữ liệu)

Khi dữ liệu đã bị lỗi:
1. Phát hiện bằng câu query `SELECT settlement_id, COUNT(*) FROM ledger_entries WHERE type = 'ADJUSTMENT' GROUP BY settlement_id HAVING COUNT(*) > 1`.
2. Tạo kịch bản sửa dữ liệu:
   - Viết bút toán đảo (Reversal Entry) cho các bản ghi bị dư.
   - Trừ đi phần `balance` bị cộng lố tương ứng.
3. Cần downtime hoặc tạm dừng các worker trước khi chạy script vá lỗi để tránh dữ liệu tiếp tục sai lệch.
