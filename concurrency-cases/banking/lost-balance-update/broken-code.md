# Khuyết điểm mã nguồn (Broken Code)

## 1. Schema
Bảng `accounts` lưu trữ thông tin cơ bản của tài khoản.

```sql
CREATE TABLE accounts (
    id BIGSERIAL PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE NOT NULL,
    balance DECIMAL(19, 4) NOT NULL DEFAULT 0.0000,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

## 2. Broken Java / Spring Code
Đoạn code sau mô phỏng việc cộng/trừ tiền sử dụng pattern Read-Modify-Write (tìm tài khoản, thay đổi số dư ở memory, rồi lưu lại).

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;

@Service
public class AccountService {
    
    private final AccountRepository accountRepository;
    
    public AccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional
    public void updateBalance(Long accountId, BigDecimal amount) {
        // ANTI-PATTERN: Read-modify-write without locks
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new EntityNotFoundException("Account not found"));
            
        // Calculate new balance in application memory
        BigDecimal newBalance = account.getBalance().add(amount);
        
        // Modify entity
        account.setBalance(newBalance);
        
        // Save back to database (Hibernate will emit an UPDATE statement)
        accountRepository.save(account);
    }
}
```

## 3. Anti-patterns
- **Read-Modify-Write Unprotected:** Đọc entity vào memory, sửa đổi và lưu lại mà không có bất kỳ cơ chế khóa (locking) nào.
- **Thiếu Versioning (No Optimistic Locking):** Bảng không có cột `@Version`, do đó Hibernate không thể phát hiện thay đổi đồng thời.
- **Delegating state to Application:** Việc cộng/trừ được tính ở bộ nhớ ứng dụng thay vì ở tầng cơ sở dữ liệu (`balance = balance + ?`).
- **Ghi đè giá trị tuyệt đối:** Câu lệnh sinh ra bởi Hibernate sẽ là `UPDATE accounts SET balance = 4500000.0000 WHERE id = ?`, thay vì sử dụng atomic delta.

## 4. Preconditions
- Mức độ cô lập (Isolation level) mặc định của PostgreSQL là `READ COMMITTED`.
- Hệ thống chịu tải đa luồng (multi-threaded) khi có nhiều request API tới cùng một `accountId`.
- Giá trị ban đầu của tài khoản 2002 là `5,000,000`.

## 5. Observability Symptoms
- Số dư cuối ngày (end-of-day balance) không khớp với tổng lịch sử giao dịch (transaction log/ledger entries).
- Không có exception nào (như `ObjectOptimisticLockingFailureException` hay `PessimisticLockException`) được văng ra trong log. Quá trình xử lý âm thầm sai lệch.
