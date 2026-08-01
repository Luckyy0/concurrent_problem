# Solutions

Để giải quyết vấn đề chuyển đổi trạng thái đồng thời (State Transition Race Condition), hệ thống cần đảm bảo tính nguyên tử (atomicity) cho thao tác **kiểm tra trạng thái cũ và cập nhật trạng thái mới**.

Dưới đây là 3 phương pháp phổ biến và an toàn nhất:

## Solution 1: Optimistic Locking với `@Version` (Khuyên dùng)

Sử dụng cơ chế khóa lạc quan (Optimistic Locking) do JPA/Hibernate cung cấp. Hibernate sẽ tự động kiểm tra version trước khi cập nhật.

### Java / Spring Code
```java
// 1. Cập nhật Entity
@Entity
@Table(name = "authorizations")
public class Authorization {
    @Id
    private String id;
    
    // Thêm annotation @Version
    @Version
    private Long version;
    
    // ... các trường khác
}

// 2. Service Code
@Service
@RequiredArgsConstructor
public class AuthorizationService {
    // ...
    @Transactional
    public void capture(String authId, BigDecimal amount) {
        Authorization auth = authorizationRepository.findById(authId).orElseThrow();
        
        if (!"AUTHORIZED".equals(auth.getStatus())) {
            throw new IllegalStateException("Already processed");
        }
        
        auth.setStatus("CAPTURED");
        // Hibernate sinh ra SQL: UPDATE authorizations SET status='CAPTURED', version=version+1 WHERE id=? AND version=?
        authorizationRepository.save(auth);
        
        // ... (Cập nhật Account cũng nên dùng Optimistic Lock hoặc Conditional Update)
    }
}
```

**Ưu điểm:**
- Dễ cài đặt, code Java sạch sẽ (JPA lo phần SQL).
- Hiệu suất tốt khi ít tranh chấp (no database locks held for reads).
**Nhược điểm:**
- Ném ra `ObjectOptimisticLockingFailureException`. Ứng dụng phải catch exception này hoặc để nó ném ra cho client (cần cơ chế Retry nếu hợp lý, nhưng với state machine một chiều thì chỉ cần báo lỗi "Giao dịch đã được xử lý bởi luồng khác").

## Solution 2: Pessimistic Locking (SELECT ... FOR UPDATE)

Sử dụng khóa bi quan để khóa hẳn bản ghi `Authorization` ngay khi đọc, không cho phép transaction khác can thiệp cho đến khi transaction hiện tại commit.

### Java / Spring Code
```java
// 1. Repository
public interface AuthorizationRepository extends JpaRepository<Authorization, String> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Authorization a WHERE a.id = :id")
    Optional<Authorization> findByIdForUpdate(@Param("id") String id);
}

// 2. Service Code
@Service
@RequiredArgsConstructor
public class AuthorizationService {
    @Transactional
    public void capture(String authId) {
        // T2, T3 sẽ bị block ở đây cho đến khi T1 commit
        Authorization auth = authorizationRepository.findByIdForUpdate(authId).orElseThrow();
        
        if (!"AUTHORIZED".equals(auth.getStatus())) {
            // Khi T2 được unblock, T1 đã commit trạng thái là CAPTURED, T2 sẽ rơi vào đây an toàn.
            return; // Idempotent return (hoặc throw exception tùy business)
        }
        
        auth.setStatus("CAPTURED");
        authorizationRepository.save(auth);
        
        // ... (Cập nhật Account)
    }
}
```

**Ưu điểm:**
- Xử lý mượt mà, không sinh ra Exception bắt buộc phải xử lý thủ công (như Retry).
**Nhược điểm:**
- Gây nghẽn hệ thống (database contention) nếu giữ khóa quá lâu.
- Dễ dẫn đến Deadlock nếu logic cập nhật tài khoản (Account) không có thứ tự nhất quán.

## Solution 3: Conditional Update (State Machine in SQL)

Không dùng Hibernate để save entity toàn phần, mà sử dụng câu lệnh UPDATE có điều kiện (Predicate) trực tiếp bằng JPQL/Native SQL. (Đây là kỹ thuật *Compare-and-Swap*).

### Java / Spring Code
```java
// 1. Repository
public interface AuthorizationRepository extends JpaRepository<Authorization, String> {
    @Modifying
    @Query("UPDATE Authorization a SET a.status = :newStatus WHERE a.id = :id AND a.status = :oldStatus")
    int updateStatusConditionally(@Param("id") String id, 
                                  @Param("oldStatus") String oldStatus, 
                                  @Param("newStatus") String newStatus);
}

// 2. Service Code
@Service
@RequiredArgsConstructor
public class AuthorizationService {
    @Transactional
    public void capture(String authId) {
        // Trả về số dòng (affected rows) thực tế được cập nhật
        int updated = authorizationRepository.updateStatusConditionally(authId, "AUTHORIZED", "CAPTURED");
        
        if (updated == 0) {
            // Nếu updated == 0, chứng tỏ record không tồn tại hoặc trạng thái không còn là AUTHORIZED.
            // Điều này nghĩa là luồng khác đã thắng cuộc đua (race).
            throw new IllegalStateException("State transition failed. Authorization might have been processed concurrently.");
        }
        
        // Chỉ luồng cập nhật status thành công mới được đi tiếp để cập nhật Account
        // ... (Cập nhật Account an toàn)
    }
}
```

**Tại sao Invariant được bảo vệ?**
- SQL `UPDATE ... WHERE status = 'AUTHORIZED'` là một atomic operation trên database.
- Trong quá trình đánh giá (evaluate) mệnh đề `WHERE`, PostgreSQL sẽ lấy Row Lock (do là lệnh UPDATE). Nếu T1 đang giữ khóa, T2 sẽ phải chờ. Khi T1 commit, T2 tiếp tục và nhận ra `status` đã thành `'CAPTURED'`, nên điều kiện `WHERE status = 'AUTHORIZED'` sai. T2 sẽ kết thúc với `affected rows = 0`.
- Giải pháp này chặn đứng TOCTOU ở ngay lớp Cơ sở dữ liệu.

## Trade-off Comparison

| Tiêu chí | Solution 1: `@Version` | Solution 2: `FOR UPDATE` | Solution 3: `Conditional UPDATE` |
|---|---|---|---|
| **Độ phức tạp code** | Thấp (JPA tự lo) | Thấp (Thêm Annotation) | Trung bình (Viết JPQL) |
| **Performance** | Tốt nhất (Không khóa khi đọc) | Kém nhất (Lock DB) | Rất tốt (Chỉ lock khi ghi) |
| **Bảo vệ State Machine** | Có | Có | Có |
| **Xử lý Error/Idempotency**| Throw Exception, cần tự catch | Tự động tuần tự hóa (Serialize) | Trả về 0 affected rows, dễ kiểm soát flow |

**Khuyến nghị:** Đối với hệ thống tài chính xử lý State Machine (Trạng thái đơn hướng), **Solution 3 (Conditional UPDATE)** thường là tối ưu nhất vì nó gọn nhẹ, hiệu suất cao và cho phép điều hướng logic idempotency dễ dàng (nếu `affected rows == 0` thì báo thành công mà không làm gì thêm). Nếu logic đi kèm phức tạp và cần đọc nhiều thông tin từ DB, **Solution 2 (Pessimistic Locking)** được ưu tiên để đảm bảo tính an toàn tuyết đối.
