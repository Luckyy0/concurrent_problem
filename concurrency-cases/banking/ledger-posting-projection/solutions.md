# Solutions

## Solution: Atomic Delta Update (In-Database Calculation)

Cách tối ưu và an toàn nhất cho bảng `projection` là ủy quyền việc cộng trừ cho database thông qua `atomic update`. Bằng cách này, ta loại bỏ hoàn toàn read-modify-write trên application layer, đồng thời không cần sử dụng khóa bi quan (`FOR UPDATE`) gây tắc nghẽn quá mức.

### Java Implementation

```java
@Repository
public interface BalanceProjectionRepository extends JpaRepository<BalanceProjection, String> {
    
    // Sử dụng tính năng atomic delta của SQL
    @Modifying
    @Query("UPDATE BalanceProjection b SET b.projectedBalance = b.projectedBalance + :amount, b.updatedAt = CURRENT_TIMESTAMP WHERE b.accountId = :accountId")
    int addBalance(String accountId, BigDecimal amount);
    
    // Nếu có nghiệp vụ kiểm tra số dư không được âm
    @Modifying
    @Query("UPDATE BalanceProjection b SET b.projectedBalance = b.projectedBalance + :amount, b.updatedAt = CURRENT_TIMESTAMP WHERE b.accountId = :accountId AND (b.projectedBalance + :amount) >= 0")
    int addBalanceStrict(String accountId, BigDecimal amount);
}
```

```java
@Service
@RequiredArgsConstructor
public class LedgerPostingService {

    private final BalanceProjectionRepository balanceRepository;
    private final LedgerEntryRepository ledgerRepository;

    @Transactional
    public void postTransaction(String accountId, BigDecimal amount, String referenceId) {
        
        // 1. Ghi nhận bút toán append-only trước
        LedgerEntry entry = new LedgerEntry(
                UUID.randomUUID(), 
                accountId, 
                amount, 
                referenceId
        );
        ledgerRepository.save(entry);

        // 2. Cập nhật projection bằng atomic delta
        int affectedRows = balanceRepository.addBalanceStrict(accountId, amount);
        
        // 3. Kiểm tra affected rows để xác nhận invariant
        if (affectedRows == 0) {
            // Rollback toàn bộ (bao gồm cả ledger entry) nếu không đủ số dư
            throw new InsufficientBalanceException("Account " + accountId + " has insufficient balance or does not exist.");
        }
    }
}
```

### Tại sao giải pháp này bảo vệ Invariant?

Khi sử dụng câu lệnh `UPDATE balance_projection SET projected_balance = projected_balance + :delta WHERE account_id = :accountId`, cơ chế nội bộ của PostgreSQL xử lý như sau (ngay cả ở `READ COMMITTED`):
1. **Tx A** thực thi `UPDATE`, lấy `RowExclusiveLock`, đọc giá trị version mới nhất của dòng (5M), cộng `2M`, lưu `7M`.
2. **Tx B** thực thi `UPDATE` cùng lúc, bị block chờ lock của **Tx A**.
3. Khi **Tx A** commit, **Tx B** được unblock.
4. Ở mức `READ COMMITTED`, **Tx B** sẽ re-evaluate (đọc lại version mới nhất của dòng đã commit bởi Tx A - tức là `7M`).
5. PostgreSQL tự động áp dụng biểu thức `projected_balance = 7,000,000 + (-300,000)`.
6. Kết quả cuối cùng là `6,700,000`, chính xác tuyệt đối mà không bị lost update.

### Trade-off Comparison

| Khía cạnh | Read-Modify-Write (Lỗi) | Atomic Delta (Khuyên dùng) | Optimistic Locking (`@Version`) | Pessimistic Locking (`FOR UPDATE`) |
| :--- | :--- | :--- | :--- | :--- |
| **Tính đúng đắn** | Sai (Lost update) | Chắc chắn đúng | Đúng (nhưng cần `retry`) | Đúng |
| **Hiệu suất / Throughput**| Rất cao (vì sai) | Cao (lock thời gian cực ngắn lúc thực thi UPDATE) | Thấp (do throw exception và phải retry nhiều nếu contention cao) | Thấp nhất (block từ lúc SELECT đến khi COMMIT) |
| **Logic chống âm** | Tính trên app | Chuyển xuống SQL `WHERE` | Tính trên app | Tính trên app |
| **Độ phức tạp code** | Thấp | Rất thấp, gọn gàng | Trung bình (cần `@Retryable`) | Thấp |

### Error Handling & Retries
Với `Atomic Delta` kết hợp điều kiện `WHERE (balance + amount) >= 0`, nếu trả về `affectedRows == 0`, hệ thống cần phân biệt giữa hai trường hợp:
1. `account_id` không tồn tại.
2. Số dư không đủ để trừ.

Có thể thực hiện thêm một câu `SELECT 1 FROM balance_projection WHERE account_id = ?` nếu cần throw message lỗi chính xác (dành cho UX). 
Không cần cấu hình `Retry` cho việc `lost update` vì cơ chế atomic delta không throw `OptimisticLockException`, nó xử lý serialize ngay trong engine của DB. `Retry` chỉ cần thiết nếu DB trả về `deadlock`.
