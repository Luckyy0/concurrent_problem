# Cấu trúc Database và Code lỗi (Broken Code)

## Schema

```sql
CREATE TABLE account (
    id BIGINT PRIMARY KEY,
    account_no VARCHAR(20) UNIQUE NOT NULL,
    posted_balance DECIMAL(15, 2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE authorization_hold (
    id VARCHAR(36) PRIMARY KEY,
    account_id BIGINT NOT NULL REFERENCES account(id),
    amount DECIMAL(15, 2) NOT NULL,
    status VARCHAR(20) NOT NULL, -- ACTIVE, CAPTURED, EXPIRED, CANCELLED
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_active_holds ON authorization_hold(account_id, status, expires_at);
```

## Java / Spring Boot Code (Lỗi)

```java
@Service
@RequiredArgsConstructor
public class AuthorizationService {

    private final AccountRepository accountRepository;
    private final AuthorizationHoldRepository holdRepository;

    @Transactional
    public String authorizeTransaction(Long accountId, BigDecimal amount) {
        // 1. Đọc account
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new EntityNotFoundException("Account not found"));

        // 2. Tính toán các khoản hold đang active
        LocalDateTime now = LocalDateTime.now();
        List<AuthorizationHold> activeHolds = holdRepository
            .findByAccountIdAndStatusAndExpiresAtAfter(accountId, "ACTIVE", now);

        BigDecimal totalHolds = activeHolds.stream()
            .map(AuthorizationHold::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        // 3. Tính số dư khả dụng
        BigDecimal availableBalance = account.getPostedBalance().subtract(totalHolds);

        // 4. Kiểm tra số dư
        if (availableBalance.compareTo(amount) < 0) {
            throw new InsufficientFundsException("Not enough available balance");
        }

        // 5. Tạo hold mới
        AuthorizationHold newHold = new AuthorizationHold();
        newHold.setId(UUID.randomUUID().toString());
        newHold.setAccountId(accountId);
        newHold.setAmount(amount);
        newHold.setStatus("ACTIVE");
        newHold.setExpiresAt(now.plusDays(3)); // Hold trong 3 ngày
        
        holdRepository.save(newHold);

        return newHold.getId();
    }
}
```

## Anti-patterns trong Code
1. **Read-Modify-Write qua nhiều thực thể**: Đọc `account`, đọc `authorization_hold` (tính sum), kiểm tra logic, rồi ghi mới một `authorization_hold`. Toàn bộ quá trình này không có lock, dẫn đến việc nhiều transaction đồng thời tính toán `available_balance` dựa trên cùng một trạng thái snapshot.
2. **Không khóa Account**: Việc kiểm tra số dư phụ thuộc vào trạng thái tài khoản. Dù không cập nhật trực tiếp `account`, nhưng logic nghiệp vụ yêu cầu account không được thay đổi (hoặc các hold mới không được thêm vào) trong quá trình tính toán.
3. **Tính toán `available_balance` động thiếu bảo vệ**: Phantom reads có thể xảy ra khi đọc `activeHolds` vì không có predicate lock ở mức bảng. Nếu một transaction khác thêm một hold mới, transaction hiện tại sẽ không thấy nếu nó chạy ở `Read Committed` (và dù ở `Serializable`, nó có thể dẫn đến lỗi Serialization).
4. **Race Condition trên Insert**: Transaction A và B cùng tính ra `availableBalance` đủ, sau đó cùng thực hiện Insert `newHold`. Không có conflict báo về từ Database vì Insert vào bảng `authorization_hold` với ID khác nhau không vi phạm khóa chính.
5. **Thiếu cơ chế idempotency**: Không có `idempotency_key` được kiểm tra. Nếu một request bị timeout và client retry, nó có thể tạo ra nhiều hold cho cùng một giao dịch.

## Dấu hiệu nhận biết (Observability Symptoms)
- **Tăng đột biến số dư âm**: Khi đối soát giữa `posted_balance` và tổng `hold`, phát hiện khách hàng xài quá số tiền họ có.
- **Log**: Không có exception liên quan đến Database hay Concurrency. Log ứng dụng cho thấy các transaction đồng thời đều báo `availableBalance` đủ và sinh ra ID mới thành công.
- **Metrics**: Tỉ lệ `InsufficientFundsException` thấp một cách bất thường trong thời điểm có lưu lượng giao dịch cao trên cùng một tài khoản.
