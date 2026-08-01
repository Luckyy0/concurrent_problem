# BANK-004: Duplicate Transfer Request

## Business Scenario
Khách hàng thực hiện một lệnh chuyển khoản (`transfer`) số tiền 500,000 VND từ tài khoản `1001` sang tài khoản `2002`. Trong quá trình thực hiện, do kết nối mạng chập chờn (`network timeout`), phía `client` (mobile app hoặc API gateway) tự động gửi lại (`retry`) yêu cầu chuyển khoản với cùng một mã chống trùng lặp (`idempotency_key`). 
Hệ thống thiếu cơ chế đảm bảo tính `idempotency` ở mức cơ sở dữ liệu (`database`), dẫn đến việc xử lý thành công cả hai yêu cầu. Kết quả là tài khoản `1001` bị trừ 1,000,000 VND và tài khoản `2002` nhận được 1,000,000 VND.

## Actors
- **Client**: Gửi và `retry` yêu cầu chuyển khoản với một `idempotency_key` duy nhất.
- **Transfer API**: Nhận yêu cầu và gọi `TransferService`.
- **Database (PostgreSQL)**: Lưu trữ trạng thái tài khoản (`balance`) và lịch sử giao dịch (`ledger_entry`).

## Invariants (Bất biến)
1. **Idempotency Guarantee**: Một `idempotency_key` chỉ được phép sinh ra đúng **một** tập hợp các giao dịch kế toán (`ledger entries`) và cập nhật số dư (`balance`) tương ứng.
2. **Exactly-Once Processing**: Một yêu cầu chuyển khoản (xác định bởi `idempotency_key`) nếu được gửi nhiều lần thì kết quả cuối cùng phải giống như chỉ được gửi một lần.

## Contention Point
Khối mã (code block) kiểm tra sự tồn tại của `idempotency_key` (`exists`) và sau đó thực hiện ghi nhận giao dịch (`insert` và `update`). Nếu hai `thread` cùng vượt qua bước kiểm tra `exists` trước khi bất kỳ `thread` nào kịp `commit`, cả hai sẽ cùng thực hiện logic chuyển tiền.

## Production Consequences
- Khách hàng bị trừ tiền nhiều lần cho cùng một giao dịch, gây bức xúc và mất niềm tin.
- Hệ thống tạo ra các giao dịch (`ledger entries`) hoàn toàn hợp lệ về mặt chữ ký số/nghiệp vụ nhưng lại là sai sót logic.
- Quá trình khắc phục (`recovery`) đòi hỏi phải `rollback` thủ công hoặc tạo các bút toán đảo (`reversal entries`) có sự can thiệp của vận hành viên (`operator`).

## Applicability
- Hệ thống thanh toán, ví điện tử (`e-wallet`), cổng thanh toán (`payment gateway`).
- Bất kỳ API nào cho phép `client retry` hoặc chịu rủi ro `double-submit` từ giao diện người dùng.

## Scope Boundary
Case study này tập trung giải quyết bài toán chống trùng lặp yêu cầu chuyển khoản (`Duplicate command prevention`) thông qua `idempotency key` và `unique constraint`. Các vấn đề liên quan đến `deadlock` khi cập nhật số dư song song đã được đề cập trong [BANK-001] đến [BANK-003].

## Terminology

| Thuật ngữ tiếng Việt | Canonical English Term | Ý nghĩa trong ngữ cảnh này |
|----------------------|------------------------|---------------------------|
| Chống trùng lặp      | Idempotency            | Đảm bảo nhiều request giống nhau chỉ tạo ra một hiệu ứng duy nhất. |
| Khóa định danh       | Idempotency Key        | Chuỗi duy nhất do client sinh ra cho mỗi giao dịch logic. |
| Bút toán             | Ledger Entry           | Bản ghi lịch sử giao dịch không thể thay đổi (`immutable`). |
| Ràng buộc duy nhất   | Unique Constraint      | Ràng buộc cấp database đảm bảo không có hai bản ghi trùng giá trị trên một cột. |

## Navigation
- [Broken Code & Symptoms](./broken-code.md)
- [Concurrency Analysis](./analysis.md)
- [Solutions](./solutions.md)
- [Experiments](./experiments.md)
