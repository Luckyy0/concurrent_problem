# Giải pháp & Khắc phục

## Giải pháp 1: Unique Constraint + Exception Handling (Đề xuất)

Đây là phương pháp robust và đáng tin cậy nhất. Bằng cách ủy thác việc kiểm tra trùng lặp cho Database thông qua `UNIQUE CONSTRAINT`, chúng ta loại bỏ hoàn toàn rủi ro từ `Check-Then-Act` trong môi trường đa luồng (`multi-threaded`) và đa máy chủ (`multi-instance`).

### 1. Cập nhật Database Schema
Thêm ràng buộc duy nhất vào bảng `transfer_request`:
```sql
ALTER TABLE transfer_request
ADD CONSTRAINT uq_transfer_request_idempotency_key UNIQUE (idempotency_key);
```

### 2. Cập nhật mã Java

```java
@Service
@RequiredArgsConstructor
public class TransferService {

    private final TransferRequestRepository requestRepository;
    private final AccountRepository accountRepository;
    private final LedgerEntryRepository ledgerRepository;

    @Transactional
    public TransferResponse executeTransfer(String idempotencyKey, String fromAcc, String toAcc, BigDecimal amount) {
        
        TransferRequest request = new TransferRequest(idempotencyKey, fromAcc, toAcc, amount, "COMPLETED");
        
        try {
            // Chèn bản ghi trước.
            // Flush ngay lập tức để ép database kiểm tra Unique Constraint.
            requestRepository.saveAndFlush(request);
        } catch (DataIntegrityViolationException e) {
            // Ràng buộc duy nhất bị vi phạm, nghĩa là request đã được xử lý (hoặc đang được xử lý)
            return new TransferResponse("ALREADY_PROCESSED");
        }

        // Nếu insert thành công, hệ thống đảm bảo chỉ có 1 thread được đi tiếp
        Account from = accountRepository.findByIdForUpdate(fromAcc)
            .orElseThrow(() -> new IllegalArgumentException("Account not found"));
        from.setBalance(from.getBalance().subtract(amount));
        accountRepository.save(from);

        Account to = accountRepository.findByIdForUpdate(toAcc)
            .orElseThrow(() -> new IllegalArgumentException("Account not found"));
        to.setBalance(to.getBalance().add(amount));
        accountRepository.save(to);

        ledgerRepository.save(new LedgerEntry(request.getId(), fromAcc, amount.negate(), "DEBIT"));
        ledgerRepository.save(new LedgerEntry(request.getId(), toAcc, amount, "CREDIT"));

        return new TransferResponse("SUCCESS");
    }
}
```

### Tại sao Invariant được bảo vệ?
- Khi hai thread cùng gọi `saveAndFlush`, PostgreSQL sẽ cố gắng `INSERT` 2 row. Thread đến sau sẽ bị PostgreSQL báo lỗi vi phạm `UNIQUE CONSTRAINT` (`SQLState 23505`).
- Spring Data JPA chuyển đổi lỗi này thành `DataIntegrityViolationException`.
- Khối `catch` sẽ bắt lỗi này và trả về `ALREADY_PROCESSED`, ngăn chặn việc thực hiện tiếp logic trừ tiền.
- Bất biến (`Invariant`) được giữ vững vì chỉ có duy nhất một bản ghi được tạo ra trong `transfer_request`, tương ứng với một lần cập nhật số dư.

### Trade-offs
- **Pros**: An toàn tuyệt đối 100% không phụ thuộc vào Application Logic, không cần Distributed Lock.
- **Cons**: Bắn ra exception có thể chậm một chút về hiệu năng ở JVM (stack trace generation), nhưng tần suất trùng lặp thấp nên hoàn toàn chấp nhận được.

---

## Giải pháp 2: Conditional Insert với `ON CONFLICT` (PostgreSQL)

Thay vì để exception bay ra, ta có thể chủ động báo cho PostgreSQL bỏ qua nếu trùng lặp. Tuy nhiên, Spring Data JPA không hỗ trợ trực tiếp `ON CONFLICT`, ta cần dùng `@Modifying` query.

### SQL / Java Code

```java
public interface TransferRequestRepository extends JpaRepository<TransferRequest, Long> {

    @Modifying
    @Query(value = """
        INSERT INTO transfer_request (idempotency_key, from_account, to_account, amount, status)
        VALUES (:key, :fromAcc, :toAcc, :amount, 'COMPLETED')
        ON CONFLICT (idempotency_key) DO NOTHING
        """, nativeQuery = true)
    int insertIfNotExists(@Param("key") String key, 
                          @Param("fromAcc") String fromAcc, 
                          @Param("toAcc") String toAcc, 
                          @Param("amount") BigDecimal amount);
}
```

```java
@Transactional
public TransferResponse executeTransfer(String idempotencyKey, String fromAcc, String toAcc, BigDecimal amount) {
    
    int affectedRows = requestRepository.insertIfNotExists(idempotencyKey, fromAcc, toAcc, amount);
    
    // Nếu affectedRows == 0, bản ghi đã tồn tại -> Trùng lặp
    if (affectedRows == 0) {
        return new TransferResponse("ALREADY_PROCESSED");
    }

    // Thực hiện tiếp bước trừ tiền như bình thường...
}
```

### Trade-offs so với Giải pháp 1
- **Pros**: Hiệu năng cao hơn vì không sinh ra Exception trên JVM. Mã rõ ràng về mặt logic xử lý luồng.
- **Cons**: Cú pháp `ON CONFLICT` là đặc thù của PostgreSQL. Phải viết Native SQL, mất đi sự linh hoạt của Hibernate/JPA khi map Entity. Khó khăn khi cần lấy ra Generated ID của bản ghi vừa chèn nếu muốn dùng ID đó lưu vào sổ cái (`ledger_entry`). (Cần dùng thêm `RETURNING id` nhưng xử lý map về JPA phức tạp hơn).
