# Tiền điều kiện & Mã nguồn lỗi

## Database Schema (PostgreSQL)

```sql
CREATE TABLE account (
    id VARCHAR(50) PRIMARY KEY,
    balance DECIMAL(19, 4) NOT NULL CHECK (balance >= 0)
);

-- Bảng lưu trữ yêu cầu chuyển khoản để theo dõi idempotency
CREATE TABLE transfer_request (
    id BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(100) NOT NULL, -- THIẾU UNIQUE CONSTRAINT!
    from_account VARCHAR(50) NOT NULL,
    to_account VARCHAR(50) NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ledger_entry (
    id BIGSERIAL PRIMARY KEY,
    transfer_request_id BIGINT REFERENCES transfer_request(id),
    account_id VARCHAR(50) NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    type VARCHAR(10) NOT NULL, -- 'CREDIT' hoặc 'DEBIT'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Broken Java/Spring Code

```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final TransferRequestRepository requestRepository;
    private final AccountRepository accountRepository;
    private final LedgerEntryRepository ledgerRepository;

    @Transactional
    public TransferResponse executeTransfer(String idempotencyKey, String fromAcc, String toAcc, BigDecimal amount) {
        // ANTI-PATTERN: Check-Then-Act (exists + insert) không an toàn trong môi trường concurrent
        if (requestRepository.existsByIdempotencyKey(idempotencyKey)) {
            return new TransferResponse("ALREADY_PROCESSED");
        }

        // Tạo bản ghi transfer request
        TransferRequest request = new TransferRequest(idempotencyKey, fromAcc, toAcc, amount, "COMPLETED");
        requestRepository.save(request);

        // Trừ tiền người gửi
        Account from = accountRepository.findByIdForUpdate(fromAcc)
            .orElseThrow(() -> new IllegalArgumentException("Account not found"));
        from.setBalance(from.getBalance().subtract(amount));
        accountRepository.save(from);

        // Cộng tiền người nhận
        Account to = accountRepository.findByIdForUpdate(toAcc)
            .orElseThrow(() -> new IllegalArgumentException("Account not found"));
        to.setBalance(to.getBalance().add(amount));
        accountRepository.save(to);

        // Ghi nhận sổ cái
        ledgerRepository.save(new LedgerEntry(request.getId(), fromAcc, amount.negate(), "DEBIT"));
        ledgerRepository.save(new LedgerEntry(request.getId(), toAcc, amount, "CREDIT"));

        return new TransferResponse("SUCCESS");
    }
}
```

## Anti-Patterns Identified
1. **Check-Then-Act without DB Locking (exists+insert)**: Sử dụng lệnh đọc (`exists`) ở cấp độ ứng dụng để ra quyết định ghi (`insert`) mà không có bất kỳ cơ chế khóa (`lock`) hoặc ràng buộc (`constraint`) nào bảo vệ.
2. **Missing Unique Constraint**: Bảng `transfer_request` thiếu `UNIQUE(idempotency_key)`, cho phép database lưu trữ nhiều dòng có cùng một mã.
3. **Application-level Idempotency**: Tin tưởng hoàn toàn vào code Java để kiểm soát tính duy nhất thay vì ủy thác cho database, nơi có khả năng phân giải `concurrency` tốt nhất cho dữ liệu phân tán.
4. **Implicit Transaction Boundaries**: Dùng `@Transactional` chung cho cả quá trình, dẫn đến việc database `lock` các row `account` trong thời gian dài hơn mức cần thiết nếu có nhiều tác vụ phụ đi kèm.
5. **No Idempotency Status Returning**: Nếu trả về `ALREADY_PROCESSED`, client không biết giao dịch cũ đã thành công hay thất bại (ở đây mặc định coi là thành công nhưng thực tế có thể request trước đó đang ở trạng thái `FAILED`).

## Timeline Showing the Bug

Giả sử số dư ban đầu: `1001` = 1,000,000 VND. `amount` = 500,000 VND. Cùng `idempotencyKey` = `tx-999`.

| Thời gian | Thread 1 (Client Original) | Thread 2 (Client Retry) | Trạng thái Database |
|-----------|----------------------------|-------------------------|---------------------|
| T1 | `existsByIdempotencyKey("tx-999")` -> `false` | | Chưa có record `tx-999` |
| T2 | | `existsByIdempotencyKey("tx-999")` -> `false` | Chưa có record `tx-999` |
| T3 | `save(TransferRequest)` | | Insert 1 dòng `tx-999` |
| T4 | | `save(TransferRequest)` | Insert thêm 1 dòng `tx-999` |
| T5 | Trừ balance `1001` đi 500k (còn 500k) | | Balance = 500k |
| T6 | | Trừ balance `1001` đi 500k (còn 0k) | Balance = 0k |
| T7 | `commit` | | Account bị trừ 500k |
| T8 | | `commit` | Account bị trừ tiếp 500k |

## Observability Symptoms
- **Metrics**: Tăng đột biến số lượng giao dịch thành công trên cùng một tài khoản trong khoảng thời gian rất ngắn (< 50ms).
- **Logs**:
  ```log
  [http-nio-8080-exec-1] INFO TransferService - Processing transfer tx-999
  [http-nio-8080-exec-2] INFO TransferService - Processing transfer tx-999
  ```
- **Database**: Bảng `transfer_request` chứa các bản ghi có chung `idempotency_key`.
