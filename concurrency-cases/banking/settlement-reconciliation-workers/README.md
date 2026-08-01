# BANK-010: Concurrent Settlement and Reconciliation Workers

## 1. Business Scenario
Trong hệ thống tài chính, **đối soát** (`reconciliation`) là quá trình so khớp dữ liệu giao dịch giữa hệ thống nội bộ (Ledger) và đối tác (Gateway/Partner). Quá trình đối soát diễn ra định kỳ (batch) hoặc liên tục thông qua các **worker** chạy song song để tối ưu hiệu suất. 
Khi có sai lệch (`discrepancy`), hệ thống phải thực hiện các bút toán điều chỉnh (`adjustment`) và đánh dấu dòng quyết toán (`settlement line`) là đã đối soát (`reconciled`).

Tuy nhiên, với việc triển khai nhiều worker hoạt động đồng thời mà không có cơ chế phân chia công việc (`work claiming`) và quản lý mốc thời gian (`cutoff timestamp`) an toàn, hệ thống sẽ rơi vào tình trạng cạnh tranh dữ liệu (race condition) và mất tính nhất quán (`inconsistency`).

## 2. Actors
- **Worker A, Worker B**: Các luồng tiến trình chạy song song (background jobs/schedulers) lấy các dòng quyết toán chưa xử lý (`status = 'PENDING'`) để thực hiện đối soát.
- **Worker C**: Một luồng tiến trình khác đối soát tổng hợp vào cuối ngày, phụ thuộc vào mốc thời gian (`cutoff`).
- **Partner System**: Hệ thống của đối tác (cung cấp file settlement hoặc API).
- **Ledger System**: Sổ cái nội bộ lưu trữ số dư (balance) và lịch sử giao dịch (transaction history).

## 3. Invariants (Bất biến)
- **Công việc duy nhất**: Một dòng quyết toán (`settlement_line`) chỉ được xử lý bởi **duy nhất một worker** trong một chu kỳ (không xảy ra xử lý trùng lặp).
- **Điều chỉnh duy nhất (Idempotency)**: Nếu có sai lệch, bút toán điều chỉnh (`adjustment entry`) chỉ được tạo **một lần duy nhất** cho mỗi dòng quyết toán.
- **Cutoff ổn định (Stable Cutoff)**: Việc đối soát theo thời gian phải dựa trên một mốc `cutoff timestamp` cố định, không được bao gồm các giao dịch đang `commit` dở dang (in-flight transactions) sau thời điểm đó.

## 4. Contention Point
- Bảng `settlement_lines`: Nhiều worker cùng `SELECT` các dòng có trạng thái `'PENDING'` và cùng `UPDATE` trạng thái sang `'RECONCILED'`.
- Bảng `ledger_entries` và `account_balances`: Các worker có thể đồng thời tạo ra các bút toán điều chỉnh cộng/trừ số dư nếu phát hiện sai lệch.

## 5. Production Consequences
Nếu không xử lý tốt concurrency:
- **Double Adjustment**: Worker A và Worker B cùng lấy ra dòng `SETTLE-1001` đang lỗi lệch 100k. Cả hai cùng tạo bút toán điều chỉnh 100k, dẫn đến số dư khách hàng bị cộng/trừ 2 lần (200k) — gây tổn thất tài chính nghiêm trọng.
- **Lost Work / Moving Cutoff**: Worker C tổng hợp dữ liệu tại mốc `T`. Tuy nhiên có transaction `TX-999` bắt đầu trước `T` nhưng `commit` sau `T`. Báo cáo tổng hợp bị sai lệch, dẫn đến số liệu đối soát nội bộ và đối tác không khớp nhau kéo dài.

## 6. Applicability
Mô hình lỗi này thường gặp ở:
- Các batch job (Spring Batch, Quartz, Hangfire) chạy ở chế độ multi-node / multi-thread.
- Các hệ thống tích hợp (Integration Hub) cần so khớp file đối soát (Settlement File).
- Hệ thống cần xử lý hàng triệu bản ghi và phải chia nhỏ (partition) dữ liệu ra cho nhiều worker xử lý đồng thời.

## 7. Scope Boundary
- **Bao gồm**: Phân chia công việc (work claiming) an toàn qua cơ chế Lock/Queue ở database; Đảm bảo tính luỹ đẳng (idempotent) của điều chỉnh; Chốt mốc thời gian an toàn cho report/reconciliation.
- **Không bao gồm**: Thuật toán so khớp (matching algorithm) phức tạp như fuzzy logic; Luồng uỷ quyền thời gian thực (real-time authorization).

## 8. Terminology

| Vietnamese (Ý nghĩa) | Canonical English Term | Description |
| ------------------- | ---------------------- | ----------- |
| Đối soát | Reconciliation | Quá trình so khớp dữ liệu giữa 2 hay nhiều nguồn. |
| Dòng quyết toán | Settlement Line | Một bản ghi giao dịch từ đối tác cần đối soát. |
| Điều chỉnh | Adjustment | Hành động tạo bút toán bù trừ khi có sai lệch. |
| Phân chia/nhận việc | Work Claiming | Lấy một phần dữ liệu từ hàng đợi để xử lý độc quyền. |
| Luỹ đẳng | Idempotency | Tính chất cho phép thực hiện một thao tác nhiều lần nhưng kết quả chỉ thay đổi 1 lần. |
| Mốc chốt thời gian | Cutoff Timestamp | Điểm thời gian để chia tách các batch giao dịch. |
| Giao dịch đang chạy | In-flight Transaction | Giao dịch đang được xử lý, chưa hoàn tất (commit/rollback). |

## 9. Related Links
- [BANK-007: Ledger Posting Projection](../ledger-posting-projection/README.md)
- [BANK-009: Settlement Reversal Race](../settlement-reversal-race/README.md)
- [DB-010: Skip Locked Work Queue](../../postgresql/skip-locked-work-queue/README.md)
- [Ledger Balances and Holds](../../concepts/ledger-balances-and-holds.md)
- [Idempotency and Uniqueness](../../concepts/idempotency-and-uniqueness.md)
- [Concurrency Testing](../../concepts/concurrency-testing.md)
