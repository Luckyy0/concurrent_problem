# Giải pháp (Solutions)

Có ba hướng tiếp cận chính để giải quyết vấn đề lost update: Atomic Delta Update, Optimistic Locking, và Pessimistic Locking. Đối với bài toán tính số dư ngân hàng độc lập (không cần validation phức tạp như kiểm tra số dư > 0), **Atomic Delta Update** là giải pháp tốt nhất về mặt hiệu suất.

## 1. Solution 1: Atomic Delta Update (Khuyên dùng)
Giao việc tính toán số dư cho cơ sở dữ liệu xử lý trực tiếp. 

```java
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.data.jpa.repository.JpaRepository;
import java.math.BigDecimal;

public interface AccountRepository extends JpaRepository<Account, Long> {

    // Trực tiếp trừ/cộng tại Database
    @Modifying
    @Query("UPDATE Account a SET a.balance = a.balance + :amount WHERE a.id = :id")
    int addBalance(@Param("id") Long id, @Param("amount") BigDecimal amount);
}
```

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
        int affectedRows = accountRepository.addBalance(accountId, amount);
        if (affectedRows == 0) {
            throw new EntityNotFoundException("Account not found");
        }
    }
}
```
**Tại sao bảo vệ được invariant:** Dưới PostgreSQL, câu lệnh `UPDATE ... SET balance = balance + amount` sẽ thực hiện lấy dòng cuối cùng (latest row version) ngay tại thời điểm thực thi. Dù có bị block bởi T1, khi unblock nó vẫn sẽ áp dụng phép toán cộng/trừ lên giá trị đã được commit gần nhất (của T1).

## 2. Solution 2: Optimistic Locking
Sử dụng trường `@Version` ở tầng entity.

```sql
ALTER TABLE accounts ADD COLUMN version BIGINT DEFAULT 0 NOT NULL;
```

```java
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Version;
import java.math.BigDecimal;

@Entity
public class Account {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String accountNumber;
    
    private BigDecimal balance;
    
    @Version
    private Long version;
    
    // Getters and setters
}
```
**Tại sao bảo vệ được invariant:** Khi T2 cố gắng ghi với giá trị `version` cũ, database báo không update được dòng nào (affected rows = 0), Hibernate sẽ ném ra `ObjectOptimisticLockingFailureException`. Ứng dụng sau đó cần phải triển khai cơ chế `retry` thủ công (thông qua `@Retryable` của Spring, kết hợp mở transaction mới) để tải lại số dư mới và thực hiện lại.
**Trade-off:** Chịu overhead của việc xử lý exception và retry khi contention cao. Phù hợp nếu logic thay đổi số dư phức tạp hơn hoặc cần kiểm tra điều kiện (ví dụ `balance >= 0`).

## 3. Solution 3: Pessimistic Locking (FOR UPDATE)
Sử dụng khóa bi quan tại thời điểm `SELECT`.

```java
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface AccountRepository extends JpaRepository<Account, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);
}
```
**Tại sao bảo vệ được invariant:** `SELECT ... FOR UPDATE` khiến T2 phải chờ ngay từ bước đọc dữ liệu đầu tiên. Khi T2 được unblock, nó sẽ đọc được giá trị `7,000,000` của T1 vừa commit, do đó tính toán trên bộ nhớ của T2 chính xác, không xảy ra lost update.
**Trade-off:** Dễ gây ra tình trạng lock dài, tốn connection pool và có nguy cơ deadlock nếu các thread cập nhật nhiều tài khoản không theo thứ tự nhất quán.
