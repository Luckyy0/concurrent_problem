# Solutions

Có hai hướng giải quyết chính cho bài toán này, mỗi hướng có ưu/nhược điểm riêng. Cả hai giải pháp đều đảm bảo bất biến (invariant) `available_balance >= 0`.

## Solution 1: Pessimistic Locking (FOR UPDATE)

Sử dụng khoá bi quan (pessimistic lock) ở tầng Database để đảm bảo tại một thời điểm, chỉ có một `transaction` được đọc và thay đổi bản ghi.

### Code Implementation

**Repository:**
```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints({@QueryHint(name = "javax.persistence.lock.timeout", value = "3000")})
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);
}
```

**Service:**
```java
@Service
public class AccountService {
    
    private final AccountRepository accountRepository;

    public AccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional
    public void withdrawPessimistic(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findByIdForUpdate(accountId)
            .orElseThrow(() -> new RuntimeException("Account not found"));

        if (account.getAvailableBalance().compareTo(amount) < 0) {
            throw new RuntimeException("Insufficient funds");
        }

        account.setAvailableBalance(account.getAvailableBalance().subtract(amount));
        // Hibernate sẽ tự sinh UPDATE khi commit transaction
    }
}
```

### Tại sao giải pháp này bảo vệ Invariant?
- Câu lệnh sinh ra là `SELECT ... FOR UPDATE`. PostgreSQL sẽ tạo một `Row Exclusive Lock` trên bản ghi ngay từ lúc đọc.
- Nếu Thread 2 đến sau (dù chỉ vài mili-giây), nó sẽ bị block (chờ) tại câu lệnh `SELECT FOR UPDATE` cho đến khi Thread 1 commit.
- Sau khi Thread 1 commit, Thread 2 mới được lấy lock và đọc dữ liệu. Lúc này nó sẽ đọc được giá trị mới là `200,000`. Khi check điều kiện `200k < 800k`, nó sẽ ném ra `RuntimeException("Insufficient funds")`.

---

## Solution 2: Conditional Update (Atomic SQL Update)

Sử dụng nguyên tắc update nguyên tử của Database kèm điều kiện (predicate).

### Code Implementation

**Repository:**
```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    
    @Modifying
    @Query("UPDATE Account a SET a.availableBalance = a.availableBalance - :amount " +
           "WHERE a.id = :id AND a.availableBalance >= :amount")
    int debitBalance(@Param("id") Long id, @Param("amount") BigDecimal amount);
}
```

**Service:**
```java
@Service
public class AccountService {
    
    private final AccountRepository accountRepository;

    public AccountService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional
    public void withdrawAtomic(Long accountId, BigDecimal amount) {
        int updatedRows = accountRepository.debitBalance(accountId, amount);
        
        if (updatedRows == 0) {
            throw new RuntimeException("Insufficient funds or account not found");
        }
        // Giao dịch thành công. Invariant an toàn.
    }
}
```

### Tại sao giải pháp này bảo vệ Invariant?
- Không cần đọc (`SELECT`) dữ liệu vào Java RAM rồi mới tính toán. 
- Mọi thứ được phó thác cho câu lệnh nguyên tử của PostgreSQL. Dưới mức `READ COMMITTED`, các câu lệnh `UPDATE` sẽ tự block lẫn nhau nếu trỏ vào cùng một bản ghi. Thread 1 thực thi `UPDATE` trước. Thread 2 bị block.
- Khi Thread 1 commit, Thread 2 tiếp tục. PostgreSQL sẽ tự động đánh giá lại điều kiện `WHERE a.availableBalance >= :amount` trên **phiên bản dữ liệu mới nhất** đã commit. 
- Do `available_balance` lúc này là 200,000, điều kiện `>= 800,000` trở thành sai, nên số dòng bị ảnh hưởng (`affected rows`) = 0.
- Mã Java kiểm tra `updatedRows == 0` và từ chối giao dịch thứ hai bằng Exception, kích hoạt rollback.

---

## 3. Trade-off Comparison (So sánh)

| Tiêu chí | Solution 1: Pessimistic Lock | Solution 2: Conditional Update |
|----------|-----------------------------|--------------------------------|
| **Hiệu năng (Performance)** | Thấp hơn (phải giữ lock từ lúc SELECT đến lúc commit). | Tốt hơn (chỉ lock trong tích tắc khi chạy UPDATE). |
| **Logic nghiệp vụ** | Dễ triển khai nếu nghiệp vụ cần tính toán phức tạp dựa trên state của tài khoản. | Khó triển khai nếu logic nghiệp vụ phức tạp, cần join/check nhiều bảng. |
| **Deadlock risk** | Dễ xảy ra nếu có transaction khác cũng lấy lock nhiều tài khoản nhưng không theo thứ tự. | Rất thấp, do vòng đời lock rất ngắn. |
| **Khuyến nghị** | Dùng khi cần đọc dữ liệu, tính toán phức tạp trước khi ghi, hoặc nhiều steps. | Ưu tiên dùng cho các update đơn lẻ, đếm số/trừ tiền trực tiếp và logic đơn giản. |
