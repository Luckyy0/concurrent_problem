# Phân tích sự cố (Analysis)

## 1. Initial State
- Bảng `accounts`: Row với `id = 2002` có `balance = 5,000,000`.
- Transaction T1 (Credit): `amount = +2,000,000`
- Transaction T2 (Debit): `amount = -500,000`

## 2. Interleaving Timeline
Quá trình thực thi diễn ra đồng thời như sau dưới `READ COMMITTED`:

| Time | Thread 1 (Credit +2,000,000) | Thread 2 (Debit -500,000) | PostgreSQL Behavior (MVCC / Locks) |
|------|---------------------------|------------------------|------------------------------------|
| T0   | `BEGIN`                   | `BEGIN`                |                                    |
| T1   | `SELECT balance FROM accounts WHERE id = 2002` (Reads: 5,000,000) | | MVCC trả về snapshot mới nhất (5,000,000). |
| T2   |                           | `SELECT balance FROM accounts WHERE id = 2002` (Reads: 5,000,000) | MVCC trả về snapshot mới nhất (5,000,000) vì T1 chưa commit. |
| T3   | Application tính: `newBalance = 5,000,000 + 2,000,000 = 7,000,000` | |
| T4   |                           | Application tính: `newBalance = 5,000,000 - 500,000 = 4,500,000` | |
| T5   | `UPDATE accounts SET balance = 7,000,000 WHERE id = 2002` | | T1 lấy `Row Exclusive Lock` trên dòng 2002 và ghi dữ liệu mới. |
| T6   |                           | `UPDATE accounts SET balance = 4,500,000 WHERE id = 2002` | T2 cố gắng lấy khóa nhưng bị block, chờ T1 nhả khóa (lock wait). |
| T7   | `COMMIT`                  |                        | T1 release lock. T2 được unblock. |
| T8   |                           | (T2 unblocked) Thực thi UPDATE của T2. | T2 ghi đè giá trị mới: `balance = 4,500,000`. |
| T9   |                           | `COMMIT`               | T2 release lock. Cập nhật của T1 bị MẤT. |

## 3. Expected vs Actual
- **Expected:** Số dư phải là `5,000,000 + 2,000,000 - 500,000 = 6,500,000`.
- **Actual:** Số dư là `4,500,000` (Khoản nạp 2,000,000 bị mất hoàn toàn). Hoặc ngược lại nếu T1 commit sau, số dư sẽ là `7,000,000` và khoản phí 500,000 bị mất.

## 4. Root Cause by Layer
- **Application Layer:** Tính toán state mới hoàn toàn độc lập mà không quan tâm đến state database lúc ghi có còn khớp với lúc đọc hay không.
- **Database Layer (PostgreSQL):** Ở chế độ `READ COMMITTED`, `UPDATE` thứ hai chỉ block chứ không abort, và sau khi unblock nó sẽ ghi đè (overwrite) dữ liệu lên snapshot mới, gây ra hiện tượng Lost Update.
- **ORM Layer (Hibernate):** Nếu không dùng `@Version` (optimistic) hoặc `@Lock` (pessimistic), Hibernate mặc định sẽ emit câu lệnh cập nhật toàn bộ trường của entity bằng giá trị memory, không thể bảo vệ khỏi tranh chấp.

## 5. Multi-instance Behavior
Lỗi này xảy ra ngay cả khi có một instance hay nhiều instance, vì cơ chế tranh chấp nằm ở tầng row của database. Khi có nhiều instances (scale-out), xác suất T1 và T2 xảy ra đồng thời càng cao, khiến issue bùng nổ dữ dội hơn.

## 6. Recovery Timeline
Việc khắc phục hậu quả trên production yêu cầu:
1. Đối soát (reconciliation) log (ledger entries) với bảng `accounts` để tìm ra độ lệch.
2. Viết script tính lại số dư đúng cho các tài khoản bị ảnh hưởng.
3. Fix code và deploy bản vá có locking hoặc atomic updates.
