# BANK-003: Giải pháp (Solutions)

## Cách tiếp cận: Deterministic Lock Ordering (Sắp xếp thứ tự khóa tất định)
Để ngăn chặn hoàn toàn hiện tượng `deadlock` do thứ tự truy cập, ta phải đảm bảo rằng mọi `transaction` trong hệ thống luôn lấy `lock` trên các bản ghi theo cùng một thứ tự. Cách đơn giản và hiệu quả nhất là sắp xếp các ID tài khoản (ví dụ: từ nhỏ đến lớn) trước khi thực hiện truy vấn `FOR UPDATE`.

## Mã nguồn Java/Spring sửa lỗi (Corrected Code)

```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        if (fromAccountId.equals(toAccountId)) {
            throw new IllegalArgumentException("Cannot transfer to the same account");
        }

        // 1. Xác định thứ tự Lock (Deterministic Lock Order)
        Long firstLockId = Math.min(fromAccountId, toAccountId);
        Long secondLockId = Math.max(fromAccountId, toAccountId);

        // 2. Lấy lock theo thứ tự ID nhỏ trước, lớn sau
        Account firstAccount = accountRepository.findByIdForUpdate(firstLockId)
            .orElseThrow(() -> new IllegalArgumentException("Account not found"));
            
        Account secondAccount = accountRepository.findByIdForUpdate(secondLockId)
            .orElseThrow(() -> new IllegalArgumentException("Account not found"));

        // 3. Map lại biến cho đúng ngữ nghĩa nghiệp vụ
        Account fromAccount = firstLockId.equals(fromAccountId) ? firstAccount : secondAccount;
        Account toAccount = firstLockId.equals(fromAccountId) ? secondAccount : firstAccount;

        // 4. Kiểm tra số dư và thực hiện chuyển đổi
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException("Not enough balance");
        }

        fromAccount.setBalance(fromAccount.getBalance().subtract(amount));
        toAccount.setBalance(toAccount.getBalance().add(amount));

        // Không cần gọi .save() do dirty checking của Hibernate sẽ tự flush khi commit
    }
}
```

### Cách cải tiến hơn (Optimized SQL Solution)
Thay vì hai lệnh `SELECT ... FOR UPDATE` riêng biệt làm tăng số lần giao tiếp mạng (network roundtrips), ta có thể gộp lại thành 1 câu query với toán tử `IN` có sắp xếp ngầm (hoặc cần Native Query để đảm bảo `ORDER BY`):

```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    
    // Spring Data JPA IN clause with FOR UPDATE. 
    // Mặc định PostgreSQL khi quyét Index hoặc lấy theo IN có thể không đảm bảo thứ tự lock tuyệt đối,
    // nên cần sử dụng một truy vấn Native hoặc ORDER BY cụ thể.
    @Query(value = "SELECT * FROM account WHERE id IN (:id1, :id2) ORDER BY id FOR UPDATE", nativeQuery = true)
    List<Account> findBothForUpdate(@Param("id1") Long id1, @Param("id2") Long id2);
}
```
*Lưu ý:* Khi sử dụng `SELECT ... WHERE id IN (...) ORDER BY id FOR UPDATE`, PostgreSQL sẽ thực hiện sắp xếp trước khi lấy khóa, điều này đảm bảo tuyệt đối an toàn khỏi `deadlock` đảo ngược.

## Tại sao bất biến (Invariant) được bảo vệ?
- Bất biến **Conservation of Money** được bảo vệ bởi tính chất nguyên tử (`atomic`) của `transaction`.
- Bất biến **No Deadlock** được đảm bảo bởi vì dù User A chuyển cho B, hay User B chuyển cho A, `transaction` luôn cố gắng khóa tài khoản có ID nhỏ hơn trước (ví dụ ID=100). Do đó:
  - T1 và T2 cùng tranh nhau khóa ID=100. 
  - Chỉ một người lấy được `lock`. Người còn lại phải chờ. 
  - Người lấy được khóa (ví dụ T1) sẽ tiếp tục tiến tới khóa ID=200. Do T2 đang chờ ở ID=100 nên ID=200 đang tự do, T1 dễ dàng lấy được và hoàn thành `transaction`. Không có tình huống khóa chéo.

## Đánh giá Trade-off
| Đặc điểm | Cách tiếp cận | Đánh giá |
|----------|---------------|----------|
| **Hiệu suất (Performance)** | Không thay đổi so với cách cũ (vẫn tốn chi phí lock). Tuy nhiên loại bỏ được overhead khi database phải rollback do deadlock. | Ưu điểm |
| **Tính dễ hiểu (Readability)** | Cần thêm logic sắp xếp ID trước khi khóa, có thể gây khó hiểu cho người mới (cần có comment giải thích rõ lý do phải làm thế). | Nhược điểm nhẹ |
| **Bảo trì (Maintenance)** | Khá dễ bảo trì đối với 2 bảng. Nhưng nếu transaction dính dáng đến 3-4 tài khoản (split payment), việc viết logic sắp xếp bằng Java sẽ phức tạp hơn. | Nhược điểm |

## Xử lý lỗi nâng cao (Error Handling & Retry)
Mặc dù Deterministic Lock Order giải quyết được `deadlock` giữa 2 tài khoản, trong một hệ thống phức tạp, tài khoản có thể bị `deadlock` với một batch process (ví dụ tính lãi cuối tháng). Do đó, bọc `transaction` bằng cơ chế `Retry` vẫn luôn là Best Practice:

```java
@Retryable(
    value = { CannotAcquireLockException.class, DeadlockLoserDataAccessException.class },
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, maxDelay = 500)
)
@Transactional
public void transferWithRetry(...) {
    // transfer logic
}
```
*Lưu ý:* Khi sử dụng `@Retryable`, phương thức phải được gọi từ ngoài một `Bean` khác (hoặc tự tiêm lại chính mình qua proxy) thì Spring mới chặn (intercept) và thực hiện retry đúng cách.
