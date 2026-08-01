# BANK-007: Concurrent Ledger Posting and Balance Projection

## Business Scenario
Trong hệ thống tài chính cốt lõi (core banking), **sổ cái** (`ledger`) là nguồn chân lý (source of truth). Các giao dịch (transaction) được lưu trữ dưới dạng các bút toán `append-only` (chỉ thêm mới, không bao giờ UPDATE hay DELETE). Tuy nhiên, để hệ thống có thể truy vấn số dư nhanh chóng mà không cần tính tổng hàng triệu dòng bút toán mỗi lần, hệ thống duy trì một **bảng chiếu** (`balance_projection`). Bảng này lưu trữ số dư hiện tại của tài khoản (`projected_balance`).

Kịch bản: Tài khoản `3003` đang có số dư chiếu là `5,000,000 VND`. Hệ thống nhận được hai giao dịch đồng thời:
- **Giao dịch A**: Nạp `+2,000,000 VND` vào tài khoản 3003.
- **Giao dịch B**: Trừ `-300,000 VND` từ tài khoản 3003.

Yêu cầu là ghi nhận cả hai bút toán vào sổ cái và cập nhật bảng chiếu để số dư cuối cùng là `6,700,000 VND`.

## Invariants (Bất biến)
1. **Ledger Integrity**: `ledger_entry` chỉ được phép `INSERT`. Bất kỳ thay đổi tài chính nào cũng phải được ghi lại thông qua một bút toán kép (double entry) hoặc bút toán đơn tùy cấu trúc. (Trong bài toán này ta đơn giản hóa với bút toán đơn trên tài khoản khách hàng).
2. **Projection Consistency**: Số dư trong `balance_projection` tại bất kỳ thời điểm nào phải bằng tổng (`SUM`) của tất cả các `amount` trong `ledger_entry` của tài khoản đó.
   `SELECT projected_balance FROM balance_projection WHERE account_id = '3003' == SELECT SUM(amount) FROM ledger_entry WHERE account_id = '3003'`

## Contention Point
Điểm tranh chấp xảy ra khi nhiều `thread` đồng thời cập nhật `balance_projection`. Nếu hệ thống sử dụng mô hình read-modify-write trên application layer để tính toán số dư mới, `lost update` (mất mát cập nhật) sẽ xảy ra.

## Production Consequences
- Các bút toán trong `ledger_entry` được lưu đầy đủ (không bị mất), do đó tiền của khách hàng về mặt lý thuyết vẫn chính xác nếu tính toán lại.
- Tuy nhiên, `projected_balance` bị sai lệch. Khách hàng nhìn thấy số dư sai trên ứng dụng, có thể dẫn đến việc khách hàng không thể rút tiền (nếu số dư thấp hơn thực tế) hoặc rút lố (nếu số dư cao hơn thực tế).
- Chi phí khắc phục (reconciliation) rất lớn vì cần phải quét lại toàn bộ dữ liệu để đồng bộ lại bảng chiếu.

## Applicability
- Bất kỳ hệ thống nào sử dụng Event Sourcing hoặc Append-Only Logs có kèm theo Read Model / Projection cập nhật đồng bộ trong cùng một `transaction`.
- Hệ thống tính điểm thưởng (loyalty points), ví điện tử, quản lý kho hàng (inventory ledger).

## Scope Boundary
- Case này tập trung vào tính nhất quán giữa durable posting (ghi bút toán) và projection (cập nhật số dư).
- Các khái niệm về `holds` (giữ tiền tạm thời) được bàn luận tại [BANK-008](../holds/README.md).
- Vấn đề trừ tiền dưới 0 đã được bàn ở [BANK-002](../lost-balance-update/README.md).

## Terminology

| Tiếng Anh | Tiếng Việt | Ý nghĩa trong ngữ cảnh |
| --- | --- | --- |
| Ledger entry | Bút toán sổ cái | Bản ghi append-only đại diện cho một thay đổi tài chính. |
| Append-only | Chỉ thêm mới | Tính chất của bảng dữ liệu không bao giờ bị sửa đổi. |
| Balance projection | Bảng chiếu số dư | Bảng lưu trữ trạng thái hiện tại (số dư) được tính toán từ các sự kiện. |
| Atomic delta | Cập nhật chênh lệch nguyên tử | Kỹ thuật cập nhật số dư bằng phép cộng/trừ trực tiếp trong SQL (`balance = balance + delta`). |
| Auditability | Khả năng kiểm toán | Đảm bảo mọi thay đổi số dư đều có dấu vết rõ ràng qua ledger. |

## Navigation
- [BANK-002: Lost Balance Update](../lost-balance-update/README.md)
- [DB-006: Unique Constraint Concurrency](../../postgresql/unique-constraint-concurrency/README.md)
- [LOCK-004: Conditional Atomic Update](../../locking/conditional-atomic-update/README.md)
- [Concepts: Ledger Balances and Holds](../../concepts/ledger-balances-and-holds.md)
- [Concepts: Idempotency and Uniqueness](../../concepts/idempotency-and-uniqueness.md)
- [Concepts: Concurrency Testing](../../concepts/concurrency-testing.md)
