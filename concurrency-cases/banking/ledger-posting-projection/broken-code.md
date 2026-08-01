# Schema & Broken Code

## Database Schema

```sql
CREATE TABLE balance_projection (
    account_id VARCHAR(50) PRIMARY KEY,
    projected_balance DECIMAL(19, 4) NOT NULL DEFAULT 0,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ledger_entry (
    entry_id UUID PRIMARY KEY,
    account_id VARCHAR(50) NOT NULL REFERENCES balance_projection(account_id),
    amount DECIMAL(19, 4) NOT NULL,
    reference_id VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT uq_ledger_reference UNIQUE (reference_id)
);
```

## Broken Java/Spring Code

```java
@Service
@RequiredArgsConstructor
public class LedgerPostingService {

    private final BalanceProjectionRepository balanceRepository;
    private final LedgerEntryRepository ledgerRepository;

    @Transactional
    public void postTransaction(String accountId, BigDecimal amount, String referenceId) {
        // Anti-pattern 1: Read-modify-write trên application layer
        BalanceProjection balance = balanceRepository.findById(accountId)
                .orElseThrow(() -> new AccountNotFoundException(accountId));
        
        // Tạo bút toán (append-only)
        LedgerEntry entry = new LedgerEntry(
                UUID.randomUUID(), 
                accountId, 
                amount, 
                referenceId
        );
        ledgerRepository.save(entry);

        // Tính toán số dư mới trên application
        BigDecimal newBalance = balance.getProjectedBalance().add(amount);
        balance.setProjectedBalance(newBalance);
        balance.setUpdatedAt(LocalDateTime.now());
        
        // Anti-pattern 2: Ghi đè toàn bộ giá trị số dư
        balanceRepository.save(balance);
    }
}
```

## Anti-Patterns

1. **Read-Modify-Write Without Lock**: Code đọc `projected_balance` về memory của Java, cộng/trừ số tiền và lưu lại. Khi chạy đồng thời, các `transaction` sẽ ghi đè kết quả của nhau (`lost update`).
2. **Entity Overwrite**: Việc dùng `balanceRepository.save(balance)` hoặc cơ chế dirty checking mặc định của Hibernate sẽ sinh ra câu lệnh `UPDATE balance_projection SET projected_balance = ? WHERE account_id = ?`, ghi đè giá trị tĩnh thay vì dùng `atomic delta`.
3. **Missing Validation**: Không kiểm tra số dư âm (nếu nghiệp vụ yêu cầu). Mặc dù ledger là nguồn chân lý, nhưng ở cấp độ projection vẫn cần kiểm tra logic nghiệp vụ.
4. **Weak Isolation Dependency**: Tin tưởng vào mức `isolation level` mặc định (thường là `READ COMMITTED` trong PostgreSQL) để bảo vệ data integrity của chuỗi read-modify-write, điều này là sai lầm căn bản.
5. **No Optimistic/Pessimistic Lock**: Không áp dụng bất kỳ cơ chế locking nào (`@Version` hay `FOR UPDATE`) trên bảng `balance_projection`.

## Preconditions
- Tài khoản `3003` tồn tại với `projected_balance = 5000000`.
- Hệ thống cho phép nạp và rút tiền.
- `Transaction` A nạp `2000000`.
- `Transaction` B rút `300000`.

## Timeline: Lost Update on Projection

| Time | Thread A (Nạp +2M) | Thread B (Rút -300K) | Database State (`projected_balance`) |
| :--- | :--- | :--- | :--- |
| T1 | `BEGIN;` | | `5000000` |
| T2 | | `BEGIN;` | `5000000` |
| T3 | `SELECT ... WHERE account_id='3003'` (Đọc: 5M) | | `5000000` |
| T4 | | `SELECT ... WHERE account_id='3003'` (Đọc: 5M) | `5000000` |
| T5 | `INSERT INTO ledger_entry (..., 2000000)` | | `5000000` |
| T6 | | `INSERT INTO ledger_entry (..., -300000)` | `5000000` |
| T7 | `UPDATE ... SET projected_balance = 7000000` | | `7000000` (A chưa commit, lock dòng) |
| T8 | `COMMIT;` | | `7000000` (A commit xong) |
| T9 | | `UPDATE ... SET projected_balance = 4700000` | `4700000` (B ghi đè) |
| T10| | `COMMIT;` | **`4700000`** |

## Observability Symptoms
Khi đối soát (`reconciliation`), hệ thống cảnh báo:
- Tổng ledger cho tài khoản 3003: `5,000,000 + 2,000,000 - 300,000 = 6,700,000`
- Balance projection: `4,700,000`
- Chênh lệch: `2,000,000` (Giá trị của nạp tiền từ Thread A đã bị mất do Thread B ghi đè).
