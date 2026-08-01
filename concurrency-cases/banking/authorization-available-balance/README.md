# BANK-008: Available versus Posted Balance During Authorization

## Tóm tắt Business Scenario
Trong hệ thống thẻ và thanh toán, khi người dùng thực hiện một giao dịch (ví dụ: quẹt thẻ tại máy POS), số tiền không bị trừ ngay lập tức khỏi tài khoản. Thay vào đó, hệ thống sẽ thực hiện một **ủy quyền (authorization)** để giữ lại số tiền đó (tạo một `authorization hold`). 
Số dư đã ghi nhận (`posted balance`) không thay đổi cho đến khi giao dịch được thanh toán chính thức (`clearing/capture`). Tuy nhiên, số dư khả dụng (`available balance`) phải giảm đi tương ứng với các khoản hold đang chờ xử lý (`active holds`).

Nếu hệ thống không tính toán và bảo vệ số dư khả dụng một cách chính xác khi có nhiều yêu cầu ủy quyền đồng thời, tài khoản có thể bị giữ số tiền vượt quá số dư thực tế, dẫn đến rủi ro thấu chi (overdraft) không mong muốn đối với ngân hàng.

**Ví dụ:**
Tài khoản 4004 có số dư đã ghi nhận (`posted_balance`) = 2,000,000 VND. 
- Auth A: giữ 1,500,000 VND cho giao dịch thanh toán vé máy bay.
- Auth B: giữ 1,200,000 VND cho giao dịch mua sắm trực tuyến.

Nếu cả hai giao dịch được duyệt do điều kiện kiểm tra `available_balance` bị lỗi race condition, tổng số tiền bị giữ (holds) sẽ là 2,700,000 VND, vượt quá 2,000,000 VND.

## Invariants (Quy tắc bất biến)
1. **Quản lý Hạn mức**: `available_balance = posted_balance - SUM(active_holds)`.
2. **Ngăn chặn Thấu chi (Non-negative Available Balance)**: Tại bất kỳ thời điểm nào, `available_balance >= 0` (giả sử không có hạn mức thấu chi).
3. **Tính Toàn vẹn (Integrity)**: Tổng số tiền của các `active holds` không bao giờ được vượt quá `posted_balance`.

## Điểm tranh chấp (Contention Point)
- Việc tính toán `available_balance` và lưu một bản ghi `authorization_hold` mới.
- Khác với giao dịch rút tiền ngay lập tức, `authorization hold` tạo ra một record tạm thời và thời gian hết hạn (`expiry`). Race condition xảy ra khi nhiều thread đọc cùng một danh sách `active holds` (hoặc cùng một `available_balance` nếu nó được denormalized) trước khi các bản ghi hold mới được commit.

## Production Consequences
- **Thấu chi không mong muốn**: Khách hàng tiêu nhiều hơn số tiền mình có, ngân hàng phải chịu rủi ro tín dụng.
- **Không nhất quán về trạng thái**: Số dư bị âm khi tính toán lại (`posted_balance - SUM(active holds) < 0`), hệ thống báo cáo và đối soát gặp lỗi.

## Bảng thuật ngữ (Terminology)

| Tiếng Anh | Tiếng Việt | Mô tả |
| --- | --- | --- |
| Authorization Hold | Khoản giữ tiền ủy quyền | Số tiền bị khóa tạm thời để đảm bảo thanh toán sau này. |
| Posted Balance | Số dư đã ghi nhận | Tổng số dư thực tế tính dựa trên các giao dịch đã hoàn tất (cleared). |
| Available Balance | Số dư khả dụng | Số tiền tối đa khách hàng có thể dùng (`posted_balance` trừ đi các khoản hold). |
| Reservation | Đặt giữ chỗ | Quá trình khóa một nguồn lực (tiền) cho một giao dịch. |
| Expiry | Hết hạn | Thời điểm mà một khoản hold tự động được giải phóng nếu không có yêu cầu thanh toán (capture). |

## Liên kết liên quan (Related Links)
- [BANK-001: Concurrent Withdrawal Double Spend](../concurrent-withdrawal-double-spend/README.md)
- [BANK-007: Ledger Posting Projection](../ledger-posting-projection/README.md)
- [Concepts: Ledger Balances and Holds](../../concepts/ledger-balances-and-holds.md)
- [Concepts: Concurrency Testing](../../concepts/concurrency-testing.md)
