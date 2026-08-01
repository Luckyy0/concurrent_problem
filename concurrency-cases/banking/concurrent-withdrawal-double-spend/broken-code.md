# Broken Implementation

## 1. Schema (PostgreSQL)

```sql
CREATE TABLE accounts (
    id BIGINT PRIMARY KEY,
    available_balance DECIMAL(19, 4) NOT NULL
);

INSERT INTO accounts (id, available_balance) VALUES (1001, 1000000);
```

## 2. Broken Java / Spring Boot Code

Đoạn mã sau sử dụng Spring Data JPA và bị lỗi race condition kinh điển "check-then-act" (cụ thể là "check-then-debit").

```java
@Entity
@Table(name = "accounts")
public class Account {
    @Id
    private Long id;
    private BigDecimal availableBalance;

    // Getters and Setters omitted
}

@Service
public class AccountService {
    
    private final AccountRepository accountRepository;

    public AccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional
    public void withdraw(Long accountId, BigDecimal amount) {
        // Anti-pattern: Read without lock
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new RuntimeException("Account not found"));

        // Check condition
        if (account.getAvailableBalance().compareTo(amount) < 0) {
            throw new RuntimeException("Insufficient funds");
        }

        // Action (Debit)
        account.setAvailableBalance(account.getAvailableBalance().subtract(amount));
        
        // Save (Implicitly done by JPA dirty checking on commit, or explicit save)
        accountRepository.save(account);
    }
}
```

## 3. Anti-patterns

1. **Check-then-act without synchronization/lock**: Đọc `availableBalance` vào bộ nhớ ứng dụng, kiểm tra logic, rồi mới lưu lại. Giữa lúc đọc và lúc ghi, bản ghi trong database có thể đã bị thread khác thay đổi.
2. **Lost Update**: JPA mặc định sẽ tạo ra câu lệnh `UPDATE accounts SET available_balance = ? WHERE id = 1001`. Nếu hai transaction cùng đọc được giá trị 1,000,000, cả hai đều thỏa mãn điều kiện `>= 800,000`, và sau đó cùng update thành `200,000`. Kết quả cuối cùng là `200,000`, mất đi 800,000 của giao dịch kia.
3. **Missing DB Constraint**: Database không có ràng buộc `CHECK (available_balance >= 0)`. Dù đây không cản được lost update (nếu update đè giá trị dương), nó thiếu lớp phòng thủ cuối cùng ở tầng DB.
4. **Lack of Versioning/Optimistic Locking**: Không có cột `@Version` nào được khai báo để Hibernate tự phát hiện thay đổi xung đột (Optimistic Lock).
5. **Memory-level State Manipulation**: Phép trừ `.subtract()` thực hiện hoàn toàn trong RAM (JVM), làm mất khả năng để Database tự phân giải xung đột thông qua update nguyên tử.

## 4. Preconditions (Điều kiện tiền quyết)
- Hai hoặc nhiều `thread` (hoặc node) thực thi phương thức `withdraw` cùng một lúc.
- Isolation level mặc định của PostgreSQL là `READ COMMITTED`.
- Transaction không sử dụng `Pessimistic Lock` (FOR UPDATE) hoặc `Optimistic Lock` (@Version).

## 5. Timeline of the Bug (Trình tự lỗi)

| Thời gian | Thread 1 (Rút 800k) | Thread 2 (Rút 800k) | Kết quả / Database (PostgreSQL) |
|-----------|---------------------|---------------------|---------------------------------|
| T1 | Bắt đầu `Transaction 1` | | |
| T2 | | Bắt đầu `Transaction 2` | |
| T3 | `findById(1001)` -> 1,000k | | DB không lock dòng dữ liệu. |
| T4 | | `findById(1001)` -> 1,000k | DB không lock dòng dữ liệu. |
| T5 | `1000k >= 800k` (Pass) | | |
| T6 | | `1000k >= 800k` (Pass) | Cả 2 luồng đều qua cửa kiểm tra. |
| T7 | `available = 200k` | | Bộ nhớ T1 lưu giá trị 200k. |
| T8 | | `available = 200k` | Bộ nhớ T2 lưu giá trị 200k. |
| T9 | `UPDATE ... SET = 200k` | | T1 lấy Row Lock, ghi 200k. |
| T10 | Commit `Transaction 1` | | T1 hoàn thành. |
| T11 | | `UPDATE ... SET = 200k` | T2 ghi đè 200k. |
| T12 | | Commit `Transaction 2` | DB: `available_balance` = 200k |

> Hậu quả: Tài khoản bị trừ 1,600,000 VND thực tế, nhưng balance trong DB chỉ giảm 800,000 VND (Lost Update).

## 6. Observability Symptoms
- Số lượng lịch sử giao dịch (ví dụ bản ghi `ledger_entries` được insert) không khớp với số dư hiện tại của `account`.
- Log ứng dụng không có Exception nào. Mọi thứ báo thành công (`HTTP 200`). Khách hàng không phản hồi, cho đến khi đối soát tài chính (`reconciliation`) phát hiện ra thâm hụt.
