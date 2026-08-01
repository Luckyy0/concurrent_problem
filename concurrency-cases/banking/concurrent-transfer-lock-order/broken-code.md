# BANK-003: Broken Code & Anti-patterns

## Sơ đồ dữ liệu (Schema)
```sql
CREATE TABLE account (
    id BIGSERIAL PRIMARY KEY,
    balance DECIMAL(15, 2) NOT NULL CHECK (balance >= 0)
);
```

## Mã nguồn Java/Spring (Broken Code)

```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final AccountRepository accountRepository;

    @Transactional
    public void transfer(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        // Lấy tài khoản nguồn và đích, dùng Pessimistic Lock để an toàn
        Account fromAccount = accountRepository.findByIdForUpdate(fromAccountId)
            .orElseThrow(() -> new IllegalArgumentException("Account not found"));
            
        Account toAccount = accountRepository.findByIdForUpdate(toAccountId)
            .orElseThrow(() -> new IllegalArgumentException("Account not found"));

        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException("Not enough balance");
        }

        // Thực hiện trừ tiền và cộng tiền
        fromAccount.setBalance(fromAccount.getBalance().subtract(amount));
        toAccount.setBalance(toAccount.getBalance().add(amount));

        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
    }
}
```

```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);
}
```

## Các Anti-patterns (Anti-patterns)

1. **Khóa bản ghi tùy ý (Arbitrary Lock Order)**
   Mã nguồn thực hiện `lock` dựa trên tham số truyền vào: `fromAccountId` trước, rồi tới `toAccountId`. Đây là nguyên nhân trực tiếp gây ra `deadlock` nếu 2 chiều chuyển tiền diễn ra đồng thời.
2. **Không xử lý Retry khi gặp Deadlock (Missing Retry Mechanism)**
   Khi PostgreSQL báo lỗi `deadlock` và `rollback` transaction, hệ thống không tự động `retry` thao tác. Người dùng phải chịu lỗi trực tiếp.
3. **Gọi Repository.save() thừa thải trong Hibernate (Redundant save() call)**
   Trong Spring Data JPA, khi đối tượng nằm trong một `transaction` và đang được quản lý (managed entity), thao tác `save()` ở cuối là không cần thiết. `flush` sẽ tự cập nhật. Tuy nhiên, nó không gây bug sai số liệu, nhưng thể hiện sự thiếu hiểu biết về vòng đời `entity`.
4. **Logic kiểm tra số dư tại Application (Application-level Invariant Check)**
   Tuy có `CHECK (balance >= 0)` ở DB, mã lại đang dựa vào check ở Java. Khi dùng `FOR UPDATE` thì an toàn, nhưng nếu ai đó quên `FOR UPDATE`, việc check tại Java sẽ vô nghĩa (TOC-TOU).
5. **Gắn chặt với thời gian phản hồi của mạng (Network Roundtrip Bottleneck)**
   Việc fetch lần lượt: lấy A (1 roundtrip), lấy B (1 roundtrip), sau đó update A, update B. Tổng cộng quá nhiều network roundtrip giữ `lock` lâu, làm tăng cửa sổ cạnh tranh (contention window) và tỷ lệ `deadlock`.

## Điều kiện tiên quyết để xảy ra lỗi (Preconditions)
1. Có ít nhất hai `thread` hoặc hai lời gọi API thực thi đồng thời.
2. `Thread 1` chuyển tiền từ tài khoản X sang tài khoản Y.
3. `Thread 2` chuyển tiền từ tài khoản Y sang tài khoản X.
4. Quá trình bắt đầu `transaction` và thao tác `fetch` đầu tiên của hai `thread` xảy ra gần như cùng lúc, đủ gần để tạo ra sự giao nhau (interleaving) trong thứ tự khóa.

## Dấu hiệu nhận biết lỗi (Observability Symptoms)
- Hệ thống văng lỗi `org.springframework.dao.CannotAcquireLockException`.
- Trong log của PostgreSQL sẽ xuất hiện lỗi `ERROR: deadlock detected` kèm theo chi tiết thông tin về các tiến trình (PID) đang cản trở lẫn nhau.
- Biểu đồ theo dõi lỗi (Error rate) của endpoint chuyển khoản tăng đột biến (spikes) trong các sự kiện có khối lượng truy cập cao.
