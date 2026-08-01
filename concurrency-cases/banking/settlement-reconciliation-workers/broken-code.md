# Broken Code: Concurrent Settlement and Reconciliation Workers

## 1. Schema
Cấu trúc cơ sở dữ liệu lưu trữ quyết toán (`settlement_lines`) và sổ cái nội bộ (`ledger_entries`).

```sql
CREATE TABLE account_balances (
    account_id VARCHAR(50) PRIMARY KEY,
    balance DECIMAL(15, 2) NOT NULL DEFAULT 0.00
);

CREATE TABLE settlement_lines (
    id VARCHAR(50) PRIMARY KEY,
    partner_id VARCHAR(50) NOT NULL,
    amount DECIMAL(15, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING', -- PENDING, RECONCILED, DISCREPANCY
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE ledger_entries (
    id VARCHAR(50) PRIMARY KEY,
    account_id VARCHAR(50) NOT NULL,
    settlement_id VARCHAR(50),
    amount DECIMAL(15, 2) NOT NULL,
    type VARCHAR(20) NOT NULL, -- PAYMENT, ADJUSTMENT
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

## 2. Broken Java Code

Đoạn mã sau thực thi tiến trình worker quét các dòng quyết toán chưa xử lý, so khớp với sổ cái nội bộ (Ledger) và điều chỉnh số dư nếu phát hiện sai lệch.

```java
@Service
@RequiredArgsConstructor
public class ReconciliationWorkerService {

    private final SettlementLineRepository settlementRepo;
    private final LedgerEntryRepository ledgerRepo;
    private final AccountBalanceRepository accountRepo;

    // Chạy batch job, nhiều instance/worker có thể chạy đồng thời
    @Transactional
    public void processPendingSettlements() {
        // ANTI-PATTERN 1: Lấy danh sách cần xử lý mà không có Lock. 
        // Nhiều worker sẽ lấy cùng một tập kết quả.
        List<SettlementLine> pendingLines = settlementRepo.findTop50ByStatus("PENDING");

        for (SettlementLine line : pendingLines) {
            reconcileLine(line);
        }
    }

    private void reconcileLine(SettlementLine line) {
        // Giả sử logic kiểm tra phát hiện sai lệch (discrepancy)
        boolean hasDiscrepancy = checkDiscrepancy(line);

        if (hasDiscrepancy) {
            // ANTI-PATTERN 2: Tạo bút toán điều chỉnh mà không kiểm tra Idempotency
            LedgerEntry adjustment = new LedgerEntry();
            adjustment.setId(UUID.randomUUID().toString());
            adjustment.setAccountId("SYSTEM_ACCOUNT");
            adjustment.setSettlementId(line.getId());
            adjustment.setAmount(line.getAmount());
            adjustment.setType("ADJUSTMENT");
            ledgerRepo.save(adjustment);

            // ANTI-PATTERN 3: Cập nhật số dư trực tiếp (Read-Modify-Write)
            AccountBalance balance = accountRepo.findById("SYSTEM_ACCOUNT").orElseThrow();
            balance.setBalance(balance.getBalance().add(line.getAmount()));
            accountRepo.save(balance);

            line.setStatus("DISCREPANCY");
        } else {
            line.setStatus("RECONCILED");
        }

        // ANTI-PATTERN 4: Chỉ cập nhật trạng thái đơn giản, không kiểm tra version hay affected rows.
        settlementRepo.save(line);
    }
    
    // Tổng hợp cuối ngày
    @Transactional(readOnly = true)
    public BigDecimal calculateEndOfDayBalance(LocalDateTime cutoffTime) {
        // ANTI-PATTERN 5: Cutoff time không an toàn với In-flight transactions.
        // Có thể lỡ các transaction bắt đầu trước cutoffTime nhưng commit sau cutoffTime.
        return ledgerRepo.sumAmountByCreatedAtBefore(cutoffTime);
    }
}
```

## 3. Anti-patterns

1. **Unsafe Work Claiming**: Việc gọi `findTop50ByStatus("PENDING")` không hề có cơ chế lock hoặc khoá dòng. Nếu Worker A và Worker B cùng chạy vào một thời điểm, cả 2 sẽ nhận được chính xác 50 dòng giống nhau để xử lý.
2. **Lack of Idempotency on Adjustments**: Khi tạo bút toán điều chỉnh (`adjustment`), hệ thống tạo một UUID mới hoàn toàn (`UUID.randomUUID()`). Không có cơ chế nhận diện để tránh tạo 2 bút toán cho cùng một `settlementId`. Điều này dẫn đến lỗi *Double Adjustment*.
3. **Read-Modify-Write on Balances**: Việc cập nhật `AccountBalance` bằng cách `GET -> ADD -> SAVE` mà không có khoá sẽ gây ra race condition mất dữ liệu (Lost Update) ở mức database khi có concurrency.
4. **Blind Overwrite on Status**: Lưu trạng thái (`settlementRepo.save(line)`) là một thao tác ghi đè (overwrite). Nó không kiểm tra xem trạng thái của `line` hiện tại có phải còn là `PENDING` hay không (thiếu optimistic locking).
5. **Moving Cutoff Problem**: Câu truy vấn `created_at < cutoffTime` trong môi trường MVCC sẽ bỏ sót những giao dịch (transactions) đã sinh ra `created_at` nhỏ hơn `cutoffTime` nhưng tại thời điểm truy vấn, transaction đó chưa `commit`.

## 4. Preconditions

- Hệ thống đang chạy từ 2 worker trở lên (hoặc nhiều server nodes).
- Bảng `settlement_lines` có dữ liệu `PENDING`.
- Cấu hình database không áp dụng isolation level `SERIALIZABLE` (mặc định PostgreSQL là `READ COMMITTED`).

## 5. Timeline Showing the Bug (Double Reconciliation)

| Time | Worker A | Worker B | Database / System State |
| ---- | -------- | -------- | ----------------------- |
| T1 | `SELECT * FROM settlement_lines WHERE status='PENDING'` | | Lấy dòng `S1001` (Amount 500) |
| T2 | | `SELECT * FROM settlement_lines WHERE status='PENDING'` | Lấy dòng `S1001` (Amount 500) |
| T3 | `hasDiscrepancy == true` | | Worker A phát hiện lệch |
| T4 | | `hasDiscrepancy == true` | Worker B cũng phát hiện lệch |
| T5 | `INSERT INTO ledger_entries (id, type) VALUES ('uuid_A', 'ADJUSTMENT')` | | Tạo bút toán A |
| T6 | `UPDATE account_balances SET balance = balance + 500` | | Số dư = +500 |
| T7 | | `INSERT INTO ledger_entries (id, type) VALUES ('uuid_B', 'ADJUSTMENT')`| Tạo bút toán B |
| T8 | | `UPDATE account_balances SET balance = balance + 500` | Số dư = +1000 (SAI!) |
| T9 | `UPDATE settlement_lines SET status='DISCREPANCY'` | | `S1001` cập nhật bởi A |
| T10| | `UPDATE settlement_lines SET status='DISCREPANCY'` | `S1001` cập nhật bởi B |
| T11| `COMMIT` | `COMMIT` | Dữ liệu hỏng (Double Adjustment) |

## 6. Observability Symptoms

- Lỗi bất thường về số dư (Balance) của khách hàng hoặc tài khoản hệ thống (System Account) cao hơn thực tế do bị cộng kép.
- Có nhiều `ledger_entries` loại `ADJUSTMENT` trỏ về cùng một `settlementId`.
- Cảnh báo lệch sổ cái với đối tác (do `calculateEndOfDayBalance` báo số liệu khác với tổng giao dịch trong ngày do `moving cutoff`).
