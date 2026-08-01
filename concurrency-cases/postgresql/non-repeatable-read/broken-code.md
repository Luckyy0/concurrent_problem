# Phân Tích Mã Nguồn Lỗi: Quyết định dựa trên hai lần đọc (Broken two-read refund decision)

## 1. Thực thể Chính sách Hiện Tại (Current policy entity)

```java
@Entity
@Table(name = "merchant_refund_policy")
public class MerchantRefundPolicy {
    @Id
    @Column(name = "merchant_id", nullable = false)
    private UUID merchantId;

    @Column(name = "auto_refund_limit", nullable = false, precision = 19, scale = 2)
    private BigDecimal autoRefundLimit;

    @Column(nullable = false)
    private boolean active;

    @Column(nullable = false)
    private long revision;

    protected MerchantRefundPolicy() {
    }

    public void changeLimit(BigDecimal newLimit) {
        if (newLimit.signum() < 0) {
            throw new IllegalArgumentException("newLimit must be non-negative");
        }
        autoRefundLimit = newLimit;
        revision++;
    }

    public boolean allows(BigDecimal amount) {
        return active && amount.compareTo(autoRefundLimit) <= 0;
    }

    public BigDecimal getAutoRefundLimit() {
        return autoRefundLimit;
    }

    public long getRevision() {
        return revision;
    }
}
```

Thuộc tính `revision` ở đây đóng vai trò là phiên bản của quy tắc kinh doanh (business policy revision), không phải là annotation `@Version` của JPA dùng cho optimistic locking. Chúng ta không bàn đến trường hợp xung đột ghi (lost update) giữa các quản trị viên trong ngữ cảnh này.

## 2. Truy vấn Projection luôn gọi SQL (Scalar projection)

```java
public interface PolicyView {
    BigDecimal getAutoRefundLimit();
    boolean getActive();
    long getRevision();
}

public interface MerchantRefundPolicyRepository
    extends JpaRepository<MerchantRefundPolicy, UUID> {

    @Query(
        value = """
            select auto_refund_limit as autoRefundLimit,
                   active as active,
                   revision as revision
              from merchant_refund_policy
             where merchant_id = :merchantId
            """,
        nativeQuery = true
    )
    Optional<PolicyView> readPolicy(UUID merchantId);
}
```

Sử dụng native query với scalar projection sẽ không tạo ra một thực thể được quản lý (managed entity) bởi Hibernate. Mỗi lần gọi phương thức `readPolicy()`, hệ thống sẽ thực thi một lệnh SELECT trực tiếp xuống cơ sở dữ liệu. Điều này dẫn đến việc PostgreSQL cấp một statement snapshot mới cho mỗi lần gọi.

## 3. Thực thể Quyết Định Hoàn Tiền (Decision entity)

```java
@Entity
@Table(
    name = "refund_decision",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_refund_decision_command",
        columnNames = "command_id"
    )
)
public class RefundDecision {
    @Id
    private UUID id;

    @Column(name = "command_id", nullable = false)
    private UUID commandId;

    @Column(name = "merchant_id", nullable = false)
    private UUID merchantId;

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private RefundOutcome outcome;

    @Column(name = "evaluated_limit", nullable = false, precision = 19, scale = 2)
    private BigDecimal evaluatedLimit;

    @Column(name = "policy_revision", nullable = false)
    private long policyRevision; // Điểm tiềm ẩn lỗi bất nhất phiên bản

    protected RefundDecision() {
    }

    public static RefundDecision approved(
        UUID commandId,
        UUID merchantId,
        BigDecimal amount,
        BigDecimal evaluatedLimit,
        long policyRevision
    ) {
        RefundDecision decision = new RefundDecision();
        decision.id = UUID.randomUUID();
        decision.commandId = commandId;
        decision.merchantId = merchantId;
        decision.amount = amount;
        decision.outcome = RefundOutcome.APPROVED;
        decision.evaluatedLimit = evaluatedLimit;
        decision.policyRevision = policyRevision;
        return decision;
    }

    public UUID getId() {
        return id;
    }
}
```

Ràng buộc duy nhất (UniqueConstraint) trên `command_id` chỉ nhằm mục đích ngăn chặn xử lý trùng lặp (idempotency), nó KHÔNG đảm bảo tính toàn vẹn của phiên bản chính sách (policy consistency) được liên kết.

## 4. Cấu trúc Tầng Dịch Vụ chứa lỗi (Broken service)

```java
@Service
public class RefundDecisionService {
    private final MerchantRefundPolicyRepository policies;
    private final RefundDecisionRepository decisions;
    private final LocalRefundRules localRules;

    public RefundDecisionService(
        MerchantRefundPolicyRepository policies,
        RefundDecisionRepository decisions,
        LocalRefundRules localRules
    ) {
        this.policies = policies;
        this.decisions = decisions;
        this.localRules = localRules;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public RefundResult decide(
        UUID commandId,
        UUID merchantId,
        BigDecimal amount
    ) {
        // LẦN ĐỌC 1: Lấy chính sách để xét duyệt
        PolicyView eligibility = policies.readPolicy(merchantId)
            .orElseThrow(PolicyNotFoundException::new);

        if (!eligibility.getActive()
            || amount.compareTo(eligibility.getAutoRefundLimit()) > 0) {
            return RefundResult.manualReview();
        }

        localRules.validate(amount); // Logic tính toán có thể kéo dài thời gian xử lý

        // LẦN ĐỌC 2: Đọc lại chính sách để ghi audit log
        PolicyView audit = policies.readPolicy(merchantId)
            .orElseThrow(PolicyNotFoundException::new);

        // GHI NHẬN: Mâu thuẫn dữ liệu giữa hai lần đọc
        RefundDecision saved = decisions.save(
            RefundDecision.approved(
                commandId,
                merchantId,
                amount,
                eligibility.getAutoRefundLimit(), // Lấy từ Lần đọc 1
                audit.getRevision()               // Lấy từ Lần đọc 2
            )
        );
        return RefundResult.approved(saved.getId());
    }
}
```

Cấu trúc mã này thường xuất hiện sau các đợt tái cấu trúc (refactor) khi lập trình viên muốn đảm bảo thông tin audit là "mới nhất". Lỗi phát sinh do việc kết hợp kết quả đánh giá từ Snapshot 1 với số phiên bản từ Snapshot 2 mà không có cơ chế kiểm tra tính đồng nhất.

> **Ghi chú quan trọng:** Annotation `@Transactional` đảm bảo tính nguyên tử cho thao tác commit/rollback, nhưng với mức `READ_COMMITTED`, nó không đảm bảo các câu lệnh SELECT bên trong sẽ sử dụng chung một snapshot.

## 5. Giao dịch cập nhật đồng thời (Concurrent policy update)

```java
@Service
public class PolicyAdministrationService {
    private final MerchantRefundPolicyRepository policies;

    public PolicyAdministrationService(
        MerchantRefundPolicyRepository policies
    ) {
        this.policies = policies;
    }

    @Transactional
    public void lowerLimit(UUID merchantId, BigDecimal newLimit) {
        MerchantRefundPolicy policy = policies.findById(merchantId)
            .orElseThrow(PolicyNotFoundException::new);
        policy.changeLimit(newLimit);
    }
}
```

Lệnh UPDATE được sinh ra:

```sql
update merchant_refund_policy
   set active = true,
       auto_refund_limit = 50.00,
       revision = 8
 where merchant_id = '...';
```

Lệnh UPDATE này sẽ chiếm khóa cấp dòng (row lock) cho đến khi commit. Tuy nhiên, luồng xét duyệt sử dụng lệnh SELECT thông thường nên PostgreSQL sử dụng MVCC để trả về snapshot trước khi sửa đổi, tránh bị block và không phải chờ giao dịch cập nhật hoàn tất.

## 6. Lệnh SQL tương đương (Equivalent SQL representation)

```sql
begin isolation level read committed;

select auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId;
-- Kết quả trả về: 100.00, true, 7

-- << Giao dịch cập nhật (session B) cập nhật thành version 8 và commit tại đây >>

select auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId;
-- Kết quả trả về từ snapshot mới: 50.00, true, 8

insert into refund_decision(
    id,
    command_id,
    merchant_id,
    amount,
    outcome,
    evaluated_limit,
    policy_revision
)
values (
    :id,
    :commandId,
    :merchantId,
    80.00,
    'APPROVED',
    100.00, -- Lấy từ lệnh đọc 1
    8       -- Lấy từ lệnh đọc 2 (Gây bất nhất dữ liệu)
);

commit;
```

## 7. Hành vi của JPA First-Level Cache

Nếu đoạn mã không sử dụng native query mà gọi phương thức `findById()` hai lần trong cùng một giao dịch, Hibernate sẽ trả về thực thể Java từ First-Level Cache (Identity Map) mà không thực thi SQL lần hai. Trong trường hợp đó, hệ thống sẽ tiếp tục sử dụng Phiên bản 7.

Tuy nhiên, cơ chế cache sẽ bị bỏ qua và SQL được thực thi nếu:

- Sử dụng native query hoặc scalar projection (như trong ví dụ này).
- Gọi `EntityManager.refresh(...)` để ép tải lại dữ liệu.
- Gọi `EntityManager.clear()` trước khi đọc lần hai.
- Thực hiện truy vấn trong một transaction/thread khác.
- Sử dụng JDBC trực tiếp xen vào giữa giao dịch.

Nguyên tắc: Không nên phụ thuộc vào First-Level Cache như một giải pháp bảo vệ dữ liệu khỏi các vấn đề tranh chấp tương tranh (concurrency anomalies).

## 8. Điều kiện tái hiện lỗi (Preconditions for reproduction)

1. Hai luồng xét duyệt và quản trị sử dụng các kết nối/giao dịch độc lập.
2. Luồng xét duyệt hoạt động ở mức cô lập `READ COMMITTED`.
3. Hai lần gọi hàm đọc của luồng xét duyệt thực sự kích hoạt hai lệnh SELECT độc lập xuống cơ sở dữ liệu.
4. Luồng quản trị thực hiện commit ngay giữa hai lệnh SELECT của luồng xét duyệt.
5. Logic xét duyệt lưu trữ kết quả đánh giá từ phiên bản cũ.
6. Lệnh ghi (INSERT) không sử dụng khóa (lock) hoặc kiểm tra lại (validate) phiên bản.
7. Unit Test không được bọc trong một outer transaction duy nhất (nếu có).

## 9. Các giải pháp chưa triệt để (Inadequate workarounds)

### Chỉ bổ sung `@Transactional`

Giao dịch đã được thiết lập, nhưng mức `READ_COMMITTED` vẫn áp dụng hành vi cấp statement snapshot riêng biệt.

### Loại bỏ lần đọc thứ hai

Việc này giúp dữ liệu ghi nhận nhất quán trong bộ nhớ (coherent in-memory decision), nhưng cần đảm bảo yêu cầu nghiệp vụ chấp nhận việc lưu quyết định dựa trên chính sách cũ sau khi chính sách mới đã được cập nhật. 

### Sử dụng khóa `synchronized`

Khóa này chỉ có tác dụng trong phạm vi một máy ảo Java (JVM). Trong môi trường phân tán với nhiều instance (hoặc khi quản trị viên cập nhật trực tiếp qua công cụ quản lý cơ sở dữ liệu), khóa cục bộ hoàn toàn vô hiệu.

### Sử dụng phương thức `flush()` sớm

Lệnh flush() không thay đổi hành vi của policy snapshot, cũng không tự động bổ sung điều kiện ràng buộc phiên bản vào lệnh INSERT.

### Retry bên trong cùng một giao dịch

Khi xảy ra lỗi cần retry, tiến trình phải khởi tạo một giao dịch MỚI. Việc thực hiện truy vấn lại bên trong giao dịch hiện tại (đặc biệt nếu dùng `REPEATABLE READ`) sẽ tiếp tục trả về snapshot cũ từ thời điểm bắt đầu giao dịch, không giải quyết được vấn đề dữ liệu bị lỗi thời.
