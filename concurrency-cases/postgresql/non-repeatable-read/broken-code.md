# Đoạn Code Thảm Họa: Quyết định dựa trên 2 lần đọc (Broken two-read refund decision)

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

Cái `revision` ở đây chỉ là Đánh Số Phiên Bản Luật lệ kinh doanh thôi (business policy revision), chứ KHÔNG PHẢI là Bùa Hộ Mệnh `@Version` của JPA đâu nhé. Sếp B gọi hàm đổi Luật, cái case này ta không bàn tới vụ các sếp đấm nhau ghi đè (lost update).

## 2. Truy vấn Lấy Lẻ Tẻ Luôn Bắn Lệnh SQL Xuống DB (Scalar projection luôn phát SQL)

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

Cách lấy lẻ tẻ (Projection scalar) kiểu này nó KHÔNG sinh ra một Object xịn (managed entity) được Hibernate chăn dắt. Mỗi lần bạn gọi `readPolicy()` là chắc chắn 100% nó quất thêm 1 câu lệnh SELECT tươi rói thẳng xuống DB, do đó PostgreSQL vui vẻ phát cho bạn thêm một tấm Ảnh mới (statement snapshot mới).

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
    private long policyRevision; // Sẽ nhét Phiên bản dỏm vào đây!

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

Cái Khóa duy nhất (Unique) `command_id` nó chỉ cản được bọn rảnh bấm nút Gửi (submit) 2 lần thôi, chứ nó KHÔNG hề giữ cho cái Chính sách được vẹn toàn (policy consistency). Đây là 2 khái niệm khác biệt!

## 4. Tầng Dịch Vụ Cùi Bắp (Broken service)

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
        // LẦN ĐỌC 1: Lấy Luật để Xét Duyệt
        PolicyView eligibility = policies.readPolicy(merchantId)
            .orElseThrow(PolicyNotFoundException::new);

        if (!eligibility.getActive()
            || amount.compareTo(eligibility.getAutoRefundLimit()) > 0) {
            return RefundResult.manualReview();
        }

        localRules.validate(amount); // Tính toán lằng nhằng tốn thời gian trong RAM

        // LẦN ĐỌC 2: Lại chạy lấy Luật để chép Sổ Audit
        PolicyView audit = policies.readPolicy(merchantId)
            .orElseThrow(PolicyNotFoundException::new);

        // GHI SỔ: MANG RÂU ÔNG NỌ CẮM CẰM BÀ KIA
        RefundDecision saved = decisions.save(
            RefundDecision.approved(
                commandId,
                merchantId,
                amount,
                eligibility.getAutoRefundLimit(), // Lấy từ Lần Đọc 1
                audit.getRevision()               // Ép uổng Lấy từ Lần Đọc 2
            )
        );
        return RefundResult.approved(saved.getId());
    }
}
```

Đoạn Code này nhìn thì ngây ngô nhưng trên Đời Thực hay gặp lắm: Lần Đọc 2 thường mọc ra sau vài đợt sửa Code dọn dẹp (refactor) để lấy mấy cái Audit metadata "mới nhất", hoặc bị lồng đâu đó trong các hàm Helper. Bug chình ình ở cái khâu nhào trộn Quyết định (đã duyệt ở Lần 1) với con số Phiên bản chép Sổ (Lấy ở Lần 2).

> **Nói ngắn gọn:** Cái áo choàng `@Transactional` chỉ bó các cục Ghi (commit/rollback) thành 1 cục, chứ `READ_COMMITTED` KHÔNG HỀ hứa hẹn rằng mọi cú SELECT trong nhà đó đều chụp chung một tấm ảnh (snapshot) đâu nhé!

## 5. Sếp B Tạt Ngang Đổi Luật (Concurrent policy update)

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

Lệnh Ghi của B bắn xuống SQL:

```sql
update merchant_refund_policy
   set active = true,
       auto_refund_limit = 50.00,
       revision = 8
 where merchant_id = '...';
```

Lệnh UPDATE này hốt cái Ổ Khóa Dòng (row lock) chờ B chốt. Trớ trêu thay Ông A chỉ toàn SELECT chay (plain SELECT) nên được CSDL phát cho Tấm Ảnh Cũ (committed version) để đọc cho rảnh nợ thay vì phải Xếp Hàng Đợi B chốt xong.

## 6. Mổ Xẻ Đoạn SQL Tương Đương Dưới DB (SQL tương đương)

```sql
begin isolation level read committed;

select auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId;
-- Thấy: 100.00, true, 7

-- << Bụp! Ông B (session B) lôi cổ sửa thành version 8 và chốt sổ ngay chỗ này >>

select auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId;
-- Thấy MỚI TOANH: 50.00, true, 8

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
    100.00, -- Lấy từ Cú Đọc 1
    8       -- Lấy từ Cú Đọc 2 (Tạo ra bằng chứng Giả Mạo!)
);

commit;
```

## 7. Cú Lừa Bộ Đệm Hibernate (JPA first-level cache có thể che anomaly)

Lỡ như Code không xài Native query mà xài `findById()` bình thường 2 lần trong cùng một Giao Dịch, Hibernate thông minh sẽ ném lại cái Object Java cũ mèm nó còn lưu ở RAM mà không gọi SQL. Lúc đó Code sẽ thấy Phiên bản `7`, tưởng là xịn! Đó là hành vi của Bộ đệm (identity-map behavior), chứ không phải do Bức Ảnh `READ COMMITTED` xịn xò của PostgreSQL giúp bạn đâu!

Đừng mừng vội, lỡ bạn rớ vô các thao tác này thì DB sẽ rít lên và gọi SQL liền:

- Viết query lấy lẻ tẻ (scalar/native projection) như bài này.
- Gọi `EntityManager.refresh(...)` ép tải lại.
- Dọn sạch thùng rác `EntityManager.clear()` rồi mới Đọc.
- Chọc vô lấy ở một Giao Dịch/Thread khác (persistence context khác).
- Dùng Lệnh JDBC chay chen ngang Giao Dịch.

Nguyên tắc: Đừng bao giờ biến cái Bộ đệm Cấp 1 (first-level cache) mỏng manh thành Cái Áo Giáp Bảo Vệ Lỗi Tranh Chấp (isolation contract)!

## 8. Điều Kiện Để Lỗi Nở Hoa (Preconditions tái hiện)

1. Cả A và B đều xài Giao Dịch / Kết nối (connections) độc lập riêng rẽ.
2. Mức Cô Lập (Effective isolation) của A là PostgreSQL `READ COMMITTED` mặc định.
3. Cả 2 Lần gọi Đọc của A chắc chắn phải kích hoạt 2 phát SELECT xuống CSDL.
4. Ông B lanh chanh Chốt (commit) ngay giữa Phát SELECT 1 và Phát SELECT 2 của Ông A.
5. Code của Ông A ôm khư khư kết quả xét duyệt từ Phiên bản 7.
6. Khi ghi Sổ (INSERT), Code không thèm khóa (lock) cũng không thẩm định lại (validate) cái phiên bản Luật.
7. Bạn không bọc bài Test này dưới 1 cái Giao Dịch Ảo (outer test transaction).

## 9. Đừng Bôi Thuốc Đỏ Nửa Mùa (Những cách sửa chưa đủ)

### CHỈ đắp thêm bùa `@Transactional`

Đã đắp từ đời nào rồi sếp ơi! Nhưng nó vẫn xài chế độ phát nhiều Tấm Ảnh (statement snapshots).

### CHỈ Bóp đi lần Đọc 2 (Xóa SELECT #2)

Cách này giúp đưa ra 1 Cái Giấy Chép Sổ đồng nhất trong RAM (coherent in-memory decision), nhưng bạn phải chắc chắn là Lưu Đủ bằng chứng và thống nhất với Sếp: Luật Đã Sửa (concurrent update) thì Quyết Định cũ vác ra ghi vẫn hợp lệ nhé! (Kẻo đối soát chửi).

### Xài cái ổ khóa Ram Java `synchronized`

Tuyệt chiêu vô dụng! Nó chỉ bịt mồm được mấy cái luồng (threads) chạy chung trên 1 cục Máy Chủ (JVM). Nếu Admin Sếp B đăng nhập từ Máy Chủ khác, hay chọc thẳng tool vô Database thì ổ khóa này phế võ công.

### Bắt Ép Ghi Sớm Bằng `flush()`

Lệnh Xả (Flush) cái INSERT xuống nó không làm cho Ảnh Chụp (policy snapshot) đứng yên, và cũng chẳng gắn thêm cái điều kiện rà soát (revision predicate) nào cho bạn đâu.

### Ráng Thử Lại Trong Cùng 1 Giao Dịch

Khi thất bại phải đẻ ra 1 Giao Dịch MỚI TINH tươm rồi làm lại. Nếu bạn lì lợm chạy Lệnh Đọc lại bên trong Cùng Giao Dịch (ví dụ xài `REPEATABLE READ`) thì nó vẫn quẳng lại cho bạn cái Bức Ảnh Cũ Mèm từ đợt chạy đầu tiên thôi!
