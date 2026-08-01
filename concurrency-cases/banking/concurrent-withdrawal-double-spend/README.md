# BANK-001: Concurrent Withdrawal and Double Spending

## 1. Business Scenario (Kịch bản nghiệp vụ)
Trong các hệ thống ngân hàng hoặc ví điện tử, thao tác rút tiền (withdrawal) hoặc thanh toán là nghiệp vụ cốt lõi. Khách hàng có một số dư khả dụng (`available balance`). Khi một yêu cầu rút tiền diễn ra, hệ thống cần kiểm tra xem số dư có đủ để thực hiện giao dịch hay không (check-then-debit). 
Nếu một tài khoản (ví dụ: ID `1001`) có số dư khả dụng là 1,000,000 VND, và cùng lúc có hai yêu cầu rút tiền 800,000 VND từ hai máy chủ (hoặc hai luồng - `thread`) khác nhau, hệ thống phải đảm bảo chỉ một yêu cầu thành công, yêu cầu còn lại phải bị từ chối do không đủ số dư.

## 2. Actors & Invariants (Tác nhân & Bất biến)
- **Actors**: 
  - Khách hàng (User/Client) gửi yêu cầu rút tiền.
  - Các node xử lý (`application instances` hoặc `threads`) nhận và xử lý yêu cầu đồng thời.
  - Hệ quản trị cơ sở dữ liệu (PostgreSQL) lưu trữ trạng thái tài khoản.
- **Invariants** (Quy tắc bất biến):
  - `available_balance >= 0` tại mọi thời điểm (sau khi commit `transaction`).
  - Tổng số tiền bị trừ trên tài khoản không được vượt quá số dư trước khi thực hiện các giao dịch trừ tiền đó.

## 3. Contention Point (Điểm tranh chấp)
Bản ghi (row) trong bảng `accounts` tương ứng với `account_id = 1001`. Cả hai `transaction` đều cố gắng đọc `available_balance` và cập nhật giá trị mới cho cùng một bản ghi.

## 4. Production Consequences (Hậu quả trên Production)
- **Double Spending**: Khách hàng rút được nhiều tiền hơn số dư họ có.
- **Financial Loss**: Ngân hàng hoặc công ty ví điện tử bị thất thoát tiền (tiền đã được chuyển ra ngoài hệ thống hoặc thanh toán thành công cho đối tác).
- **Data Inconsistency**: Sổ cái (`ledger`) không khớp với thực tế giao dịch.

## 5. Scope Boundary (Phạm vi)
- **Bao gồm**: An toàn dữ liệu khi có nhiều truy cập đồng thời rút tiền trên cùng một tài khoản (`Concurrent funds safety`).
- **Không bao gồm**: Ngăn chặn yêu cầu trùng lặp (`duplicate request prevention` / `idempotency`). Chúng ta giả định đây là hai yêu cầu rút tiền hoàn toàn hợp lệ và khác nhau (khác `transaction_id` hoặc `reference_id`), nhưng nhắm vào cùng một nguồn tiền.

## 6. Terminology (Thuật ngữ)

| English Term | Tiếng Việt | Ý nghĩa |
|--------------|------------|---------|
| Double spending | Tiêu tiền hai lần | Lỗi khi một số tiền bị chi dùng cho nhiều hơn một giao dịch do xử lý đồng thời. |
| Check-then-debit | Kiểm tra rồi trừ tiền | Mô hình (anti-pattern nếu không lock) kiểm tra điều kiện trước, rồi hành động. |
| Available balance | Số dư khả dụng | Số tiền thực sự có thể sử dụng (đã trừ đi các khoản `holds`). |
| Ledger | Sổ cái | Sổ ghi chép lịch sử giao dịch tăng/giảm. |

## 7. Navigation
- **Prerequisites**: 
  - [DB-001: Lost Update MVCC](../../postgresql/lost-update-mvcc/README.md)
  - [LOCK-003: Pessimistic Write / FOR UPDATE](../../locking/pessimistic-write-for-update/README.md)
  - [LOCK-004: Conditional Atomic Update](../../locking/conditional-atomic-update/README.md)
- **References**:
  - [C-LEDGER: Ledger Balances and Holds](../../concepts/ledger-balances-and-holds.md)
  - [C-TEST: Concurrency Testing](../../concepts/concurrency-testing.md)
