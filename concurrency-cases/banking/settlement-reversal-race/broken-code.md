# Schema, Broken Code & Anti-patterns

## 1. Database Schema

Schema cơ bản gồm bảng `accounts` lưu số dư và bảng `authorizations` lưu trạng thái của khoản giữ tiền.

```sql
CREATE TABLE accounts (
    id VARCHAR(50) PRIMARY KEY,
    ledger_balance DECIMAL(19, 4) NOT NULL,
    hold_balance DECIMAL(19, 4) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE authorizations (
    id VARCHAR(50) PRIMARY KEY,
    account_id VARCHAR(50) NOT NULL REFERENCES accounts(id),
    amount DECIMAL(19, 4) NOT NULL,
    status VARCHAR(20) NOT NULL, -- 'AUTHORIZED', 'CAPTURED', 'REVERSED', 'EXPIRED'
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Dữ liệu khởi tạo
INSERT INTO accounts (id, ledger_balance, hold_balance) 
VALUES ('ACC-001', 5000000.00, 1000000.00);

INSERT INTO authorizations (id, account_id, amount, status, expires_at)
VALUES ('AUTH-555', 'ACC-001', 1000000.00, 'AUTHORIZED', NOW() - INTERVAL '1 minute');
```

## 2. Broken Java / Spring Code

Đoạn mã dưới đây mô phỏng một `AuthorizationService` xử lý 3 luồng: `capture`, `reverse`, và `expire`.

```java
@Service
@RequiredArgsConstructor
public class AuthorizationService {

    private final AccountRepository accountRepository;
    private final AuthorizationRepository authorizationRepository;

    @Transactional
    public void capture(String authId) {
        Authorization auth = authorizationRepository.findById(authId).orElseThrow();
        
        // Anti-pattern: Check-then-act without lock
        if (!"AUTHORIZED".equals(auth.getStatus())) {
            throw new IllegalStateException("Cannot capture. Current status: " + auth.getStatus());
        }

        // 1. Cập nhật trạng thái Authorization
        auth.setStatus("CAPTURED");
        authorizationRepository.save(auth);

        // 2. Cập nhật Account (Khấu trừ hold_balance và ledger_balance)
        Account account = accountRepository.findById(auth.getAccountId()).orElseThrow();
        account.setHoldBalance(account.getHoldBalance().subtract(auth.getAmount()));
        account.setLedgerBalance(account.getLedgerBalance().subtract(auth.getAmount()));
        accountRepository.save(account);
    }

    @Transactional
    public void reverse(String authId) {
        Authorization auth = authorizationRepository.findById(authId).orElseThrow();
        
        if (!"AUTHORIZED".equals(auth.getStatus())) {
            throw new IllegalStateException("Cannot reverse. Current status: " + auth.getStatus());
        }

        // 1. Cập nhật trạng thái Authorization
        auth.setStatus("REVERSED");
        authorizationRepository.save(auth);

        // 2. Cập nhật Account (Chỉ giải tỏa hold_balance)
        Account account = accountRepository.findById(auth.getAccountId()).orElseThrow();
        account.setHoldBalance(account.getHoldBalance().subtract(auth.getAmount()));
        accountRepository.save(account);
    }

    @Transactional
    public void expire(String authId) {
        Authorization auth = authorizationRepository.findById(authId).orElseThrow();
        
        if (!"AUTHORIZED".equals(auth.getStatus())) {
            return; // Im lặng bỏ qua nếu đã xử lý
        }
        if (auth.getExpiresAt().isAfter(LocalDateTime.now())) {
            throw new IllegalStateException("Authorization not yet expired");
        }

        // 1. Cập nhật trạng thái Authorization
        auth.setStatus("EXPIRED");
        authorizationRepository.save(auth);

        // 2. Cập nhật Account (Giải tỏa hold_balance)
        Account account = accountRepository.findById(auth.getAccountId()).orElseThrow();
        account.setHoldBalance(account.getHoldBalance().subtract(auth.getAmount()));
        accountRepository.save(account);
    }
}
```

## 3. Anti-patterns

Đoạn mã trên vi phạm nhiều nguyên tắc xử lý đồng thời, đặc biệt là trong môi trường multi-thread hoặc multi-instance:

1. **Time-of-Check to Time-of-Use (TOCTOU):**
   - Ứng dụng đọc trạng thái (`if (!"AUTHORIZED".equals(auth.getStatus()))`) và sau đó quyết định ghi dựa trên trạng thái này. Giữa thời điểm "đọc" và "ghi", một thread khác (VD: `reverse`) có thể đã đọc và tiến hành ghi trước, làm vô hiệu hóa tiền đề của thread hiện tại (VD: `capture`).
   - Hậu quả: Nhiều thread cùng vượt qua bước kiểm tra và cùng thực hiện thay đổi số dư tài khoản.

2. **Non-atomic State Transition:**
   - Việc chuyển trạng thái máy trạng thái (state machine) từ `AUTHORIZED` sang `CAPTURED`/`REVERSED`/`EXPIRED` không được bảo vệ bởi bất kỳ cơ chế khóa (lock) hay kiểm tra đồng thời (optimistic concurrency control) nào.

3. **Lost Update trên Account Balance:**
   - Tương tự như bài toán BANK-008, việc đọc `Account` (`findById`), tính toán trên bộ nhớ ứng dụng (`subtract`), và lưu lại (`save`) dẫn đến hiện tượng ghi đè (lost update) khi 2 thread cùng thay đổi số dư của cùng một tài khoản.
   - Thậm chí tệ hơn, nếu cả `capture` và `reverse` cùng vượt qua bước kiểm tra trạng thái, `hold_balance` sẽ bị trừ 2 lần (Double Release), gây sai lệch dữ liệu tài chính nghiêm trọng.

4. **Thiếu Idempotency an toàn:**
   - Nếu client gọi `capture` hai lần đồng thời, cả hai có thể cùng trừ tiền. Các thao tác không an toàn và không tự động phản hồi (idempotent) chính xác nếu trạng thái đã bị luồng khác thay đổi.

## 4. Preconditions & Observability Symptoms

**Preconditions (Điều kiện tiên quyết để xảy ra lỗi):**
- Mức độ cô lập giao dịch (Transaction Isolation Level) là `READ COMMITTED` (mặc định của PostgreSQL/Spring).
- Ba request (`capture`, `reverse`, `expire`) đến gần như cùng một lúc (cách nhau vài mili-giây) khi khoản hold `AUTH-555` vừa đúng lúc quá hạn.

**Observability Symptoms (Triệu chứng quan sát được trên hệ thống):**
- **Cơ sở dữ liệu:**
  - `authorizations.status` phản ánh trạng thái của luồng `commit` cuối cùng (ví dụ: `EXPIRED`), trong khi thực tế luồng `capture` cũng đã trừ `ledger_balance`.
  - `accounts.hold_balance` âm (ví dụ: `-2,000,000` thay vì `0`).
- **Logs:**
  - Không có exception nào như `ObjectOptimisticLockingFailureException` hay `DeadlockLoserDataAccessException` được ném ra. Mọi transaction đều log "Success".
- **Business Monitoring:** Báo cáo đối soát cuối ngày (Reconciliation) phát hiện chênh lệch giữa tổng số tiền `hold_balance` được giải tỏa và số lượng Authorization kết thúc. Khách hàng khiếu nại tài khoản bị trừ âm bất thường.
