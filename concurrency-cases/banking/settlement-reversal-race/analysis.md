# Analysis of Concurrent Execution

Tài liệu này phân tích chi tiết diễn biến của hệ thống khi 3 luồng: `Capture` (Thread 1), `Reverse` (Thread 2), và `Expire` (Thread 3) chạy đồng thời trên mã nguồn lỗi.

## 1. Initial State (Trạng thái ban đầu)

- **Account `ACC-001`**: `ledger_balance` = 5,000,000; `hold_balance` = 1,000,000.
- **Authorization `AUTH-555`**: `status` = 'AUTHORIZED', `amount` = 1,000,000.
- PostgreSQL Isolation Level: `READ COMMITTED`.

## 2. Interleaving Timeline (Dòng thời gian thực thi xen kẽ)

Dưới đây là một kịch bản tranh chấp điển hình (Race Condition) dẫn đến **Double Release** và **Inconsistent State**.

| Time | Thread 1 (Capture) | Thread 2 (Reverse) | Thread 3 (Expire) | PostgreSQL (MVCC / Locks) |
|---|---|---|---|---|
| `T0` | Bắt đầu Transaction (T1) | Bắt đầu Transaction (T2) | Bắt đầu Transaction (T3) | Tạo snapshot cho 3 TX. |
| `T1` | `findById('AUTH-555')` <br> `status` = 'AUTHORIZED' | | | T1 đọc bản ghi gốc (v1). |
| `T2` | | `findById('AUTH-555')` <br> `status` = 'AUTHORIZED' | | T2 đọc bản ghi gốc (v1). |
| `T3` | | | `findById('AUTH-555')` <br> `status` = 'AUTHORIZED' | T3 đọc bản ghi gốc (v1). |
| `T4` | `if (status == AUTHORIZED)` => TRUE | `if (status == AUTHORIZED)` => TRUE | `if (status == AUTHORIZED)` => TRUE | Business logic TOCTOU bị khai thác. Cả 3 đều cho phép tiếp tục. |
| `T5` | `auth.setStatus('CAPTURED')` <br> `save(auth)` (UPDATE auth) | | | `UPDATE authorizations` <br> Lấy Row-Level Lock trên `AUTH-555`. Tạo (v2). |
| `T6` | | `auth.setStatus('REVERSED')` <br> `save(auth)` | | `UPDATE authorizations` <br> **BLOCK** chờ T1 nhả lock. |
| `T7` | | | `auth.setStatus('EXPIRED')` <br> `save(auth)` | `UPDATE authorizations` <br> **BLOCK** chờ lock (vào queue). |
| `T8` | Đọc Account `ACC-001` <br> `hold`=1M, `ledger`=5M | | | Đọc Account (v1). |
| `T9` | Cập nhật Account: <br> `hold`=0, `ledger`=4M <br> `save(account)` | | | Lấy Row-Level Lock trên `ACC-001`. Cập nhật Account thành (v2). |
| `T10`| **COMMIT (T1)** | | | T1 nhả toàn bộ lock. `AUTH-555`=(v2, CAPTURED), `ACC-001`=(v2, hold=0, ledger=4M). |
| `T11`| | *Unblocked!* Tiếp tục UPDATE | | Do `READ COMMITTED`, T2 đánh giá lại (Re-evaluate) UPDATE trên dòng `AUTH-555` mới (v2). Câu lệnh UPDATE không có điều kiện WHERE status, nên nó ghi đè trạng thái thành 'REVERSED'. Tạo (v3). |
| `T12`| | Đọc Account `ACC-001` <br> Nhận bản mới nhất (v2): `hold`=0, `ledger`=4M | | T2 đọc Account (v2) (do READ COMMITTED đọc bản đã commit). |
| `T13`| | Cập nhật Account: <br> `hold` = 0 - 1M = -1M <br> `save(account)` | | Lấy Row Lock trên `ACC-001`. Tạo bản (v3). |
| `T14`| | **COMMIT (T2)** | | T2 nhả lock. `AUTH-555`=(v3, REVERSED), `ACC-001`=(v3, hold=-1M, ledger=4M). |
| `T15`| | | *Unblocked!* Tiếp tục UPDATE | Tương tự T2, T3 ghi đè trạng thái thành 'EXPIRED'. Tạo (v4). |
| `T16`| | | Đọc Account (v3) <br> `hold`=-1M | |
| `T17`| | | Cập nhật Account: <br> `hold` = -1M - 1M = -2M <br> `save(account)` | Cập nhật Account thành (v4). |
| `T18`| | | **COMMIT (T3)** | Dữ liệu cuối cùng được lưu trữ. |

## 3. Expected vs Actual Outcome

**Expected (Kết quả mong đợi):**
Chỉ MỘT luồng được thực hiện thành công. Giả sử Capture (T1) nhanh nhất:
- `authorizations.status` = 'CAPTURED'
- `accounts.hold_balance` = 0
- `accounts.ledger_balance` = 4,000,000
- T2 và T3 phải bị từ chối (Exception hoặc return sớm).

**Actual (Kết quả thực tế sau T18):**
- `authorizations.status` = 'EXPIRED' (Do T3 ghi đè cuối cùng - Lost Update trạng thái).
- `accounts.hold_balance` = -2,000,000 (Bị trừ 3 lần).
- `accounts.ledger_balance` = 4,000,000 (Trừ 1 lần bởi Capture).

## 4. Root Cause by Layer (Nguyên nhân gốc rễ)

- **Application Layer (Java):** Kiểm tra điều kiện và thực thi không nằm trong một khối nguyên tử (atomic block). JPA/Hibernate `save()` chỉ map thành `UPDATE ... WHERE id = ?`, không quan tâm trạng thái gốc có bị thay đổi bởi transaction khác hay không.
- **Database Layer (PostgreSQL):** Isolation Level `READ COMMITTED` chống Dirty Read, nhưng không chống được Non-Repeatable Read và Lost Update. Khi một Transaction block bị đánh thức (unblocked), nó sẽ áp dụng câu lệnh UPDATE lên version mới nhất của dữ liệu, bỏ qua mọi logic kiểm tra (`if`) đã thực thi ở tầng Java trước đó.

## 5. Multi-instance Behavior (Hành vi trong môi trường đa Server)

Nếu hệ thống deploy nhiều instance của ứng dụng:
- Vấn đề càng trầm trọng hơn vì không thể dùng các cơ chế khóa trong bộ nhớ cục bộ (như `ReentrantLock` hay `synchronized` trong Java).
- Database trở thành điểm đồng bộ duy nhất, do đó việc giải quyết bằng Locking ở DB hoặc SQL có tính nguyên tử là bắt buộc.
