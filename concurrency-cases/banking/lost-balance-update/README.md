# BANK-002: Lost Account Balance Update

## 1. Business Scenario
Hệ thống ngân hàng xử lý các giao dịch cộng và trừ tiền từ tài khoản. Trong tình huống này, Tài khoản 2002 hiện có số dư là 5,000,000 VND. Xảy ra đồng thời hai giao dịch:
- **T1 (Credit)**: Khách hàng nạp thêm +2,000,000 VND.
- **T2 (Debit)**: Hệ thống trừ phí hàng tháng -500,000 VND.

Nếu hệ thống không xử lý concurrency đúng cách, một trong hai cập nhật có thể bị ghi đè, dẫn đến số dư cuối cùng sai lệch nhưng trông có vẻ hợp lệ (plausible nhưng incorrect). 

## 2. Invariants (Bất biến hệ thống)
- Số dư tài khoản phải phản ánh chính xác tổng của tất cả các khoản ghi nợ (debit) và ghi có (credit).
- `Balance_Final = Balance_Initial + Sum(Credit) - Sum(Debit)`
- Việc tính toán số dư không được phép bỏ sót bất kỳ cập nhật nào.

## 3. Contention Point
Điểm tranh chấp là dòng dữ liệu (row) tương ứng với Tài khoản 2002 trong bảng `accounts`. Cả hai thread T1 và T2 đều cố gắng đọc trạng thái hiện tại, tính toán số dư mới trên bộ nhớ (in-memory), và ghi xuống cơ sở dữ liệu (`UPDATE`).

## 4. Production Consequences
- **Tổn thất tài chính:** Ngân hàng hoặc khách hàng có thể mất tiền do số dư không chính xác.
- **Mất niềm tin:** Giao dịch thành công nhưng số dư không thay đổi tương ứng khiến khách hàng khiếu nại.
- **Khó truy vết:** Các bản ghi transaction (ledger entries) có thể vẫn lưu đầy đủ, nhưng số dư tài khoản thì bị sai, dẫn đến việc báo cáo đối soát cuối ngày (reconciliation) bị lệch và tốn nhiều công sức khắc phục.

## 5. Applicability & Scope Boundary
- **Áp dụng cho:** Tất cả các hệ thống sử dụng kiến trúc read-modify-write không có cơ chế khóa thích hợp (e.g. ví điện tử, ngân hàng, điểm thưởng).
- **Scope boundary:** Tập trung vào hiện tượng "lost update" làm sai lệch balance. Việc kiểm tra số dư âm (overspending) được xử lý trong case **BANK-001**.

## 6. Terminology
| Thuật ngữ | Tiếng Anh (Canonical) | Ý nghĩa |
|----------|-----------------------|---------|
| Cập nhật bị mất | Lost Update | Xảy ra khi hai transaction cùng đọc một giá trị, sửa đổi độc lập và ghi đè lên nhau. |
| Khóa lạc quan | Optimistic Lock | Cơ chế sử dụng version field để kiểm tra xem dữ liệu có bị thay đổi trước khi cập nhật hay không. |
| Khóa bi quan | Pessimistic Lock | Cơ chế yêu cầu database khóa bản ghi (e.g., `FOR UPDATE`) ngay từ lúc đọc để ngăn các transaction khác can thiệp. |
| Cập nhật nguyên tử | Atomic Update / Delta | Giao phó việc cộng/trừ số dư cho database bằng câu lệnh `UPDATE ... SET balance = balance + :amount`. |

## 7. Navigation
- **Tiếp theo:** [Broken Code](./broken-code.md), [Analysis](./analysis.md), [Solutions](./solutions.md), [Experiments](./experiments.md)
- **Kiến thức tiên quyết:** [DB-001](../../postgresql/lost-update-mvcc/README.md), [LOCK-001](../../locking/optimistic-version-conflict/README.md), [LOCK-004](../../locking/conditional-atomic-update/README.md)
