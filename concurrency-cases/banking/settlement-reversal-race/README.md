# BANK-009: Settlement, Reversal, and Expiry Race (Race Condition khi Quyết toán, Hủy và Hết hạn Hold)

## 1. Business Scenario (Kịch bản nghiệp vụ)

Trong hệ thống thanh toán qua thẻ, vòng đời của một giao dịch thường đi qua 2 bước chính:
1. **Authorization (Cấp phép):** Giữ (hold) một khoản tiền trên tài khoản khách hàng để đảm bảo khả năng thanh toán. Khoản tiền này nằm trong `hold_balance`, làm giảm `available_balance` nhưng chưa trừ `ledger_balance`.
2. **Clearing / Settlement (Quyết toán / Ghi nhận thực tế):** Khấu trừ khoản tiền đã hold khỏi `ledger_balance` khi merchant yêu cầu (Capture), hoặc giải tỏa khoản hold khi khách hàng/merchant yêu cầu hủy (Reversal), hoặc khi khoản hold hết hạn (Expiry).

**Tình huống tranh chấp (Contention Point):**
Authorization `AUTH-555` đang giữ 1,000,000 VND với trạng thái `AUTHORIZED`. Đột ngột, 3 sự kiện xảy ra đồng thời và cùng tác động lên `AUTH-555`:
- **Capture (Quyết toán):** Hệ thống của merchant gửi yêu cầu capture 1,000,000 VND để hoàn tất giao dịch.
- **Reversal (Hủy giao dịch):** Khách hàng nhấn nút hủy giao dịch trên ứng dụng do đợi quá lâu.
- **Expiry (Hết hạn):** Job quét tự động của hệ thống (cron job) phát hiện khoản hold đã qua 24h và tiến hành giải tỏa.

## 2. Invariants (Bất biến hệ thống)

1. **State Machine Invariant (Bất biến máy trạng thái):** Trạng thái của Authorization chỉ được chuyển từ `AUTHORIZED` sang đúng **MỘT** trong ba trạng thái kết thúc: `CAPTURED`, `REVERSED`, hoặc `EXPIRED`. Máy trạng thái là đơn hướng (monotonic workflow), không được phép chuyển đổi giữa các trạng thái kết thúc.
2. **Balance Integrity (Toàn vẹn số dư):** Tổng số tiền giải tỏa từ `hold_balance` phải bằng chính xác số tiền đã hold ban đầu, bất kể giao dịch kết thúc theo nhánh nào.
3. **Ledger Equation:** 
   - Nếu Capture: `ledger_balance` giảm 1,000,000 VND, `hold_balance` giảm 1,000,000 VND.
   - Nếu Reversal/Expiry: `ledger_balance` không đổi, `hold_balance` giảm 1,000,000 VND.
4. **Idempotency (Tính lũy đẳng):** Nếu một thao tác chuyển trạng thái thành công, các thao tác đồng thời khác cố gắng chuyển trạng thái phải bị từ chối hợp lệ (handled gracefully) mà không làm hỏng dữ liệu hoặc ném ra exception không mong muốn cho client.

## 3. Production Consequences (Hệ quả trên môi trường Production)

Nếu không kiểm soát đồng thời tốt, hệ thống có thể gặp các sự cố nghiêm trọng:
- **Double Release / Double Deduction:** Giao dịch vừa bị capture vừa bị reverse, dẫn đến `hold_balance` bị giảm 2 lần (thành số âm), hoặc `ledger_balance` bị trừ sai lệch. Hệ thống hạch toán mất cân đối nghiêm trọng.
- **Inconsistent State:** Record `Authorization` có trạng thái `CAPTURED` do luồng Capture ghi đè (lost update), nhưng lịch sử giao dịch (Ledger entries) lại ghi nhận là đã hoàn tiền (Reversed).
- **Phàn nàn từ khách hàng / Merchant:** Khách hàng báo đã hủy nhưng vẫn bị trừ tiền, hoặc merchant không nhận được tiền do hệ thống ưu tiên luồng Expiry. Tốn nhiều nhân lực vận hành (Ops/Reconciliation) để tra soát thủ công.

## 4. Applicability & Scope Boundary (Phạm vi áp dụng)

**Áp dụng cho:**
- Core Banking (Thẻ tín dụng / Ghi nợ).
- Payment Gateways / Ví điện tử có cơ chế Hold/Capture.
- Bất kỳ hệ thống phân tán nào áp dụng mẫu Monotonic State Machine (ví dụ: trạng thái đơn hàng: `PENDING` -> `SHIPPED` / `CANCELLED`).

**Giới hạn phạm vi (Scope Boundary):**
- Giới hạn trong vòng đời của một Authorization đơn lẻ.
- Không bao gồm việc đối soát chéo giữa các hệ thống (Cross-system reconciliation) (Xem [BANK-010]).
- Giả định hệ thống Database hỗ trợ ACID (PostgreSQL).

## 5. Terminology (Thuật ngữ)

| Tiếng Anh | Ngữ nghĩa / Vai trò trong ngữ cảnh |
|---|---|
| **Authorization** | Cấp phép giao dịch, tạm giữ (hold) một khoản tiền trên tài khoản. |
| **Capture / Settlement** | Quyết toán giao dịch, chuyển từ tiền giữ tạm thời sang trừ thực tế. |
| **Reversal / Void** | Hủy cấp phép, giải tỏa khoản tiền đã giữ (thường do yêu cầu trước khi capture). |
| **Expiry** | Hết hạn cấp phép (thường 24h hoặc 7 ngày), hệ thống tự động giải tỏa tiền. |
| **Monotonic Workflow** | Luồng công việc một chiều, trạng thái chỉ tiến lên, không lùi lại. |
| **Guarded State Transition** | Chuyển đổi trạng thái có bảo vệ (chỉ thực hiện nếu thỏa mãn điều kiện trạng thái trước đó). |

## 6. Navigation (Điều hướng)

- [broken-code.md](./broken-code.md): Mã nguồn lỗi và phân tích anti-pattern.
- [analysis.md](./analysis.md): Phân tích chi tiết quá trình race condition theo thời gian (timeline).
- [solutions.md](./solutions.md): Giải pháp an toàn với các phương pháp locking và state machine trong SQL.
- [experiments.md](./experiments.md): Mã kiểm thử tự động chứng minh lỗi và xác nhận giải pháp.

**Liên kết liên quan:**
- [BANK-006: Payment Callback Duplication](../payment-callback-duplication/README.md)
- [BANK-008: Authorization vs Available Balance](../authorization-available-balance/README.md)
- [Ledger Balances and Holds](../../concepts/ledger-balances-and-holds.md)
- [Idempotency and Uniqueness](../../concepts/idempotency-and-uniqueness.md)
- [Concurrency Testing](../../concepts/concurrency-testing.md)
