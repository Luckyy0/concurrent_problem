# Giải pháp (Solutions)

Để giải quyết vấn đề đồng thời, ta cần tuần tự hóa (serialize) các transaction tác động lên cùng một tài khoản. Việc tính toán `available_balance` dựa trên bảng `authorization_hold` là cần thiết, nhưng cần một "vật chủ" để khóa (lock). Thực thể tốt nhất để lock chính là bản ghi `account`.

## Cách 1: Sử dụng Pessimistic Write Lock (`FOR UPDATE`) trên Account

Đây là giải pháp đơn giản và đáng tin cậy. Dù không thay đổi `account` trực tiếp, ta khóa dòng tài khoản này để ngăn chặn bất kỳ luồng nào khác thay đổi `posted_balance` hoặc tạo `authorization_hold` mới đối với tài khoản này.

### Cập nhật Repository
```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") Long id);
}
```

### Mã nguồn Service đã sửa
```java
@Service
@RequiredArgsConstructor
public class AuthorizationService {

    private final AccountRepository accountRepository;
    private final AuthorizationHoldRepository holdRepository;

    @Transactional
    public String authorizeTransaction(Long accountId, BigDecimal amount) {
        // 1. Lock tài khoản bằng SELECT ... FOR UPDATE
        Account account = accountRepository.findByIdForUpdate(accountId)
            .orElseThrow(() -> new EntityNotFoundException("Account not found"));

        // 2. Tính toán active holds (an toàn vì không ai có thể tạo hold mới cho tài khoản này lúc này)
        LocalDateTime now = LocalDateTime.now();
        List<AuthorizationHold> activeHolds = holdRepository
            .findByAccountIdAndStatusAndExpiresAtAfter(accountId, "ACTIVE", now);

        BigDecimal totalHolds = activeHolds.stream()
            .map(AuthorizationHold::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        // 3. Tính và kiểm tra số dư
        BigDecimal availableBalance = account.getPostedBalance().subtract(totalHolds);

        if (availableBalance.compareTo(amount) < 0) {
            throw new InsufficientFundsException("Not enough available balance");
        }

        // 4. Tạo hold mới (sẽ nằm trong cùng transaction với lock trên account)
        AuthorizationHold newHold = new AuthorizationHold();
        newHold.setId(UUID.randomUUID().toString());
        newHold.setAccountId(accountId);
        newHold.setAmount(amount);
        newHold.setStatus("ACTIVE");
        newHold.setExpiresAt(now.plusDays(3));
        
        holdRepository.save(newHold);

        return newHold.getId();
    }
}
```

### Tại sao giải pháp này bảo vệ Invariant?
Bằng cách dùng `SELECT ... FOR UPDATE` trên bảng `account`, Transaction B bị chặn lại ở bước 1 cho đến khi Transaction A commit hoặc rollback. 
Khi Tx B lấy được lock và tiếp tục chạy, nó sẽ đọc được bản ghi `authorization_hold` do Tx A vừa commit. Tổng `totalHolds` sẽ bao gồm cả hold của Tx A. Số dư khả dụng sẽ được tính chính xác, và Tx B sẽ bị từ chối nếu không đủ tiền.

## Cách 2: Maintain `available_balance` như một Column và Dùng Optimistic / Pessimistic Lock

Cách 1 phải `SUM` liên tục qua bảng holds. Để tối ưu, người ta thường duy trì cột `available_balance` trong bảng `account`.
Khi tạo hold:
```sql
UPDATE account 
SET available_balance = available_balance - :amount 
WHERE id = :accountId AND available_balance >= :amount;
```
**Ưu điểm**:
- Hiệu năng rất cao vì không cần `SUM` holds, chỉ cần conditional UPDATE.
- DB tự động xử lý row lock.

**Nhược điểm (Trade-offs)**:
- `authorization_hold` có thời gian hết hạn (`expiry`). Nếu dùng cột `available_balance`, khi một hold hết hạn mà chưa được capture, tiền phải được trả về. Ta phải dùng các job định kỳ (cron job) chạy ngầm để quét các hold hết hạn và cộng ngược tiền lại vào `available_balance` (Compensating Transaction).
- Phức tạp hơn trong việc đồng bộ giữa bảng `account` và `authorization_hold` nếu có sự cố.

### Bảng So sánh Trade-off

| Tiêu chí | Tính toán động + Lock `Account` (Cách 1) | Maintain cột `available_balance` (Cách 2) |
| --- | --- | --- |
| **Độ phức tạp** | Thấp. Code dễ hiểu, dễ maintain. | Cao. Phải có job xử lý expiry. |
| **Hiệu năng** | Thấp hơn do phải SELECT bảng Hold. Lock kéo dài suốt vòng đời transaction. | Rất cao, tận dụng Conditional Update. |
| **Khả năng gặp bão hòa (Contention)** | Cao đối với tài khoản có nhiều giao dịch (VD: tài khoản merchant/doanh nghiệp). | Thấp hơn nhờ lock thời gian ngắn tại lệnh UPDATE. |
| **Data Integrity** | Tốt nhất, dữ liệu luôn Single Source of Truth. | Dễ lệch nếu xử lý sự kiện hết hạn bị miss (VD: Kafka/Job fail). |

Khuyến nghị: Với hệ thống ngân hàng cá nhân thông thường, **Cách 1** (Pessimistic Lock) là lựa chọn ổn định. Với hệ thống core banking cường độ cao, người ta sẽ dùng Event Sourcing hoặc **Cách 2**.
