# Solutions: Concurrent Settlement and Reconciliation Workers

## 1. Correct Approach

Để giải quyết triệt để, chúng ta áp dụng 3 kỹ thuật:
1. **Work Claiming with `FOR UPDATE SKIP LOCKED`**: Đảm bảo các worker nhận các dòng quyết toán khác nhau.
2. **Idempotent Adjustment**: Áp dụng Unique Constraint tại Database để chặn tạo bút toán điều chỉnh 2 lần.
3. **Atomic Balance Update**: Sử dụng câu lệnh UPDATE SQL an toàn thay vì Read-Modify-Write.
4. **Stable Cutoff Timestamp**: Thay vì truyền `LocalDateTime.now()` để tổng hợp report, sử dụng một cơ chế chốt phiên (Session/Batch ID) hoặc chỉ lấy những giao dịch đã thực sự an toàn. (Trong ví dụ này, ta tập trung vào worker resolution).

## 2. SQL Schema Updates

```sql
-- Thêm ràng buộc duy nhất để đảm bảo mỗi settlement_line chỉ có 1 điều chỉnh
ALTER TABLE ledger_entries 
ADD CONSTRAINT uq_ledger_settlement_adj 
UNIQUE (settlement_id, type);
```

## 3. Java/Spring Solution

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.dao.DataIntegrityViolationException;

import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
public class SecureReconciliationService {

    private final SettlementLineRepository settlementRepo;
    private final LedgerEntryRepository ledgerRepo;
    private final AccountBalanceRepository accountRepo;

    @Transactional
    public void processPendingSettlementsSecurely() {
        // SOLUTION 1: FOR UPDATE SKIP LOCKED
        // PostgreSQL: SELECT * FROM settlement_lines WHERE status='PENDING' 
        // LIMIT 50 FOR UPDATE SKIP LOCKED
        List<SettlementLine> lockedLines = settlementRepo.findPendingForUpdateSkipLocked(50);

        for (SettlementLine line : lockedLines) {
            try {
                reconcileLineSecurely(line);
            } catch (Exception e) {
                // Ghi log lỗi cho từng dòng, không để fail toàn bộ batch
                // Lưu ý: Nếu muốn an toàn transaction cho từng dòng, 
                // thì reconcileLineSecurely nên ở một class khác với @Transactional(REQUIRES_NEW)
            }
        }
    }

    // Yêu cầu REQUIRES_NEW để mỗi dòng quyết toán là một transaction độc lập,
    // lỗi của dòng này không rollback dòng khác, và release lock sớm.
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void reconcileLineSecurely(SettlementLine line) {
        boolean hasDiscrepancy = checkDiscrepancy(line);

        if (hasDiscrepancy) {
            // SOLUTION 2: Idempotent Key - Dùng UUID nhưng đã được db chặn bởi Unique Constraint.
            // Có thể dùng trực tiếp settlementId làm tham chiếu chặn lặp.
            LedgerEntry adjustment = new LedgerEntry();
            adjustment.setId(UUID.randomUUID().toString());
            adjustment.setAccountId("SYSTEM_ACCOUNT");
            adjustment.setSettlementId(line.getId());
            adjustment.setAmount(line.getAmount());
            adjustment.setType("ADJUSTMENT");
            
            try {
                ledgerRepo.saveAndFlush(adjustment); // Force push SQL để bắt DataIntegrityViolationException
            } catch (DataIntegrityViolationException ex) {
                // Đã có worker khác ghi adjustment (trong trường hợp fallback)
                throw new IllegalStateException("Double adjustment detected for " + line.getId());
            }

            // SOLUTION 3: Atomic Balance Update
            // UPDATE account_balances SET balance = balance + :amount WHERE id = :id
            int updated = accountRepo.addBalance("SYSTEM_ACCOUNT", line.getAmount());
            if (updated == 0) {
                throw new IllegalStateException("Account not found");
            }

            // SOLUTION 4: Conditional Update (Affected rows check)
            // UPDATE settlement_lines SET status = 'DISCREPANCY' WHERE id = :id AND status = 'PENDING'
            int statusUpdated = settlementRepo.updateStatusCondition(line.getId(), "PENDING", "DISCREPANCY");
            if (statusUpdated == 0) {
                throw new IllegalStateException("Line " + line.getId() + " already processed");
            }
        } else {
            settlementRepo.updateStatusCondition(line.getId(), "PENDING", "RECONCILED");
        }
    }

    private boolean checkDiscrepancy(SettlementLine line) {
        // Logic nghiệp vụ
        return true; 
    }
}
```

### SettlementLineRepository.java
```java
public interface SettlementLineRepository extends JpaRepository<SettlementLine, String> {

    @Query(value = "SELECT * FROM settlement_lines WHERE status = 'PENDING' LIMIT :limit FOR UPDATE SKIP LOCKED", nativeQuery = true)
    List<SettlementLine> findPendingForUpdateSkipLocked(@Param("limit") int limit);

    @Modifying
    @Query("UPDATE SettlementLine s SET s.status = :newStatus WHERE s.id = :id AND s.status = :oldStatus")
    int updateStatusCondition(@Param("id") String id, @Param("oldStatus") String oldStatus, @Param("newStatus") String newStatus);
}
```

## 4. Why Invariant is Protected

1. **Công việc duy nhất**: Lệnh `FOR UPDATE SKIP LOCKED` thiết lập Exclusive Row Lock lên các dòng được trả về. Các worker khác chạy lệnh này sẽ tự động bỏ qua (skip) các dòng đang bị khoá. Đảm bảo 100% không có sự giao nhau (overlap) giữa các worker.
2. **Luỹ đẳng (Idempotency)**: Nếu bằng một cách nào đó (vd như retry ở tầng cao hơn), 2 luồng cùng nhắm vào một `settlement_id`, thì `UNIQUE (settlement_id, type)` ở bảng `ledger_entries` sẽ chặn lại bằng lỗi Database, bảo vệ số dư.
3. **Thay đổi số dư an toàn**: Lệnh `SET balance = balance + :amount` giao hoàn toàn việc khóa dòng và tính toán cho Database, kết hợp với MVCC để đảm bảo dù nhiều worker xử lý các `settlement_lines` khác nhau nhưng tác động vào cùng `SYSTEM_ACCOUNT` thì dữ liệu vẫn tính đúng.

## 5. Trade-off Comparison

| Giải pháp | Ưu điểm | Nhược điểm |
|-----------|---------|------------|
| `SKIP LOCKED` Work Queue | Dễ implement, sử dụng chính RDBMS có sẵn, hiệu năng tốt với lượng dữ liệu vừa/lớn. Đảm bảo tính ACID chặt chẽ. | RDBMS trở thành nút thắt cổ chai (bottleneck) nếu scale quá lớn (>10k workers). Cần DB support (PostgreSQL, MySQL 8+, Oracle). |
| Redis/RabbitMQ Queue | Tách biệt luồng nhận việc, mở rộng (scale) vô hạn. | Cần thêm hạ tầng (infrastructure). Phức tạp hoá việc giữ tính nhất quán giữa Message Broker và Database. |

## 6. Error Handling

- Trong vòng lặp `for (SettlementLine line : lockedLines)`, nếu gọi qua service khác bắt buộc phải bọc bằng `REQUIRES_NEW`. Điều này giúp `SKIP LOCKED` nhả khoá sớm cho từng dòng hoàn thành, tránh việc giữ khoá toàn bộ 50 dòng gây ảnh hưởng tới các truy vấn khác.
- Catch `DataIntegrityViolationException`: Khi gặp ngoại lệ này, tức là ràng buộc luỹ đẳng đã bị vi phạm, hệ thống bỏ qua an toàn và tiếp tục dòng tiếp theo.
