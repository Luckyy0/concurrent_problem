# Bí Kíp Hóa Giải: Snapshot, Validation và Cầm Khóa (Giải pháp)

## 1. Mục tiêu thiết kế (Mục tiêu cốt lõi)

Trước khi gõ phím, phải trả lời bằng được câu hỏi cốt tử về nghiệp vụ (business) này:

```text
Cái Quyết Định xét duyệt phải ĐÚNG LÝ theo:
  - Cái Luật nằm ở lúc tui đọc nó ra?
  - Hay Cái Luật nằm ở khoảnh khắc tui Ghi Quyết Định xuống?
  - Hay Cái Luật bị ép Đứng Yên Không Đổi cho đến khi Giao Dịch này chốt sổ?
```

Dù sếp chọn đường nào, hệ thống cũng PHẢI LƯU lại Phiên Bản Luật (coherent policy revision) duy nhất, CẤM TIỆT chuyện vay mượn râu ông nọ (snapshot 1) cắm cằm bà kia (snapshot 2).

## 2. Giải pháp 1 — Chụp Một Tấm Ảnh Đóng Băng Dùng Tới Bến (Đọc một immutable policy snapshot)

Nếu Luật Công ty cho phép: "Lúc Đọc ra thấy đúng là duyệt luôn, mặc kệ ai sửa sau đó", thì hãy Đọc ĐÚNG MỘT LẦN rồi nhét vào một Object Kín Cổng Cao Tường (value object) truyền đi khắp nơi:

```java
public record RefundPolicySnapshot(
    UUID merchantId,
    BigDecimal autoRefundLimit,
    boolean active,
    long revision
) {
    boolean allows(BigDecimal amount) {
        return active && amount.compareTo(autoRefundLimit) <= 0;
    }
}
```

Tầng Kho Chứa (Repository) trả về đối tượng và nhét thẳng vào Record:

```java
@Repository
public class RefundPolicyReader {
    private final JdbcTemplate jdbc;

    public RefundPolicyReader(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public RefundPolicySnapshot read(UUID merchantId) {
        return jdbc.queryForObject(
            """
            select merchant_id,
                   auto_refund_limit,
                   active,
                   revision
              from merchant_refund_policy
             where merchant_id = ?
            """,
            (rs, rowNum) -> new RefundPolicySnapshot(
                rs.getObject("merchant_id", UUID.class),
                rs.getBigDecimal("auto_refund_limit"),
                rs.getBoolean("active"),
                rs.getLong("revision")
            ),
            merchantId
        );
    }
}
```

Tầng Dịch Vụ CẤM KHÔNG ĐƯỢC Gọi Query lần 2:

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public RefundResult decide(
    UUID commandId,
    UUID merchantId,
    BigDecimal amount
) {
    // ĐỌC 1 LẦN DUY NHẤT!
    RefundPolicySnapshot policy = policyReader.read(merchantId);

    if (!policy.allows(amount)) {
        return RefundResult.manualReview();
    }

    localRules.validate(amount); // Có ngâm vài giây cũng chả sao

    // GHI SỔ CÙNG DỮ LIỆU ĐÓ
    RefundDecision saved = decisions.save(
        RefundDecision.approved(
            commandId,
            merchantId,
            amount,
            policy.autoRefundLimit(),
            policy.revision()
        )
    );
    return RefundResult.approved(saved.getId());
}
```

Bằng chứng duyệt khớp 100% với Phiên bản Luật đã đọc. Bất quá có ai đó (Concurrent update) sửa Luật thành Bản mới sau lưng thì kệ họ; Cách này đéo hứa hẹn chuyện "Luật phải mới nhất lúc tui chốt sổ" (latest at decision commit).

> **Nói ngắn gọn:** Đọc 1 lần là dẹp tan nạn Trộn Ảnh. Nhưng phải có Kho Lịch Sử Luật Bất Khả Xâm Phạm (immutable policy history) thì cái Phán Quyết Cũ mèm mới có chỗ mà bấu víu (audit) lâu dài.

## 3. Giải pháp 2 — Biến Lịch Sử Thành Bất Tử (Versioned policy history)

Lưu có đúng 1 Dòng Mới Nhất (Mutable current row) là Tội Ác, vì đéo thể nào lật lại Phiên bản cũ. Phải tách Bảng ra cho chúng Bất Tử (immutable):

```sql
-- KHO LƯU BẤT TỬ
create table merchant_refund_policy_version (
    merchant_id uuid not null,
    revision bigint not null,
    auto_refund_limit numeric(19, 2) not null,
    active boolean not null,
    created_at timestamptz not null,
    primary key (merchant_id, revision),
    check (auto_refund_limit >= 0)
);

-- CÂY KIM CHỈ NAM (Trỏ Tới Bản Mới Nhất)
create table merchant_refund_policy_current (
    merchant_id uuid primary key,
    revision bigint not null,
    foreign key (merchant_id, revision)
        references merchant_refund_policy_version(merchant_id, revision)
);

-- ÉP RÀNG BUỘC KHI GHI SỔ
alter table refund_decision
    add constraint fk_refund_decision_policy_version
    foreign key (merchant_id, policy_revision)
    references merchant_refund_policy_version(merchant_id, revision);
```

Sếp Admin khi Update Luật mới sẽ Ghi Thêm 1 dòng Lịch Sử rồi Bẻ Kim Chỉ Nam:

```sql
begin;

insert into merchant_refund_policy_version(
    merchant_id,
    revision,
    auto_refund_limit,
    active,
    created_at
)
values (:merchantId, :nextRevision, :newLimit, true, clock_timestamp());

update merchant_refund_policy_current
   set revision = :nextRevision
 where merchant_id = :merchantId
   and revision = :expectedRevision;

-- Cập nhật thành công phải lòi ra affected row = 1
commit;
```

Còn App đọc Luật thì xài lệnh Tích Hợp (Join):

```sql
select v.merchant_id,
       v.revision,
       v.auto_refund_limit,
       v.active
from merchant_refund_policy_current c
join merchant_refund_policy_version v
  on v.merchant_id = c.merchant_id
 and v.revision = c.revision
where c.merchant_id = :merchantId;
```

Cái Khóa Ngoại (Foreign key) trói cứng không cho ai Ghi Quyết Định xài Số Phiên Bản Bậy Bạ. Luật Cũ đéo bị Ghi Đè, nên 10 năm sau Audit dò lại Phiên Bản `7` vẫn nguyên vẹn hình hài dù Mũi Kim hiện tại đang là Phiên Bản `8`.
Nếu thao tác Dời Kim trả về `affected-row = 0`, Sếp Admin mất cờ (thua optimistic conflict) và phải tự làm lại giao dịch (retry toàn transaction) với revision mới.

## 4. Giải pháp 3 — Móc Chốt Lệnh Cửa Cuối (Conditional final validation)

Nếu Hợp đồng yêu cầu "Lúc cắm bút Ghi Sổ, Luật ĐÓ phải chưa Bị Ai Đụng", hãy nhét cái điều kiện Đuôi (predicate) vào thẳng Lệnh GHI (`INSERT ... SELECT`):

```sql
insert into refund_decision(
    id,
    command_id,
    merchant_id,
    amount,
    outcome,
    evaluated_limit,
    policy_revision
)
select :decisionId,
       :commandId,
       p.merchant_id,
       :amount,
       'APPROVED',
       p.auto_refund_limit,
       p.revision
from merchant_refund_policy p
where p.merchant_id = :merchantId
  and p.revision = :expectedRevision -- CÒN ĐÚNG PHIÊN BẢN CŨ THÌ MỚI INSERT
  and p.active
  and :amount <= p.auto_refund_limit;
```

Nhét vô Spring Repository:

```java
public interface RefundDecisionCommands {

    @Modifying
    @Query(
        value = """
            insert into refund_decision(
                id, command_id, merchant_id, amount, outcome,
                evaluated_limit, policy_revision
            )
            select :decisionId, :commandId, p.merchant_id, :amount,
                   'APPROVED', p.auto_refund_limit, p.revision
              from merchant_refund_policy p
             where p.merchant_id = :merchantId
               and p.revision = :expectedRevision
               and p.active
               and :amount <= p.auto_refund_limit
            """,
        nativeQuery = true
    )
    int insertIfPolicyStillAllows(
        UUID decisionId,
        UUID commandId,
        UUID merchantId,
        BigDecimal amount,
        long expectedRevision
    );
}
```

Tại Tầng Dịch Vụ:

```java
@Transactional
public RefundResult decideValidated(
    UUID commandId,
    UUID merchantId,
    BigDecimal amount
) {
    RefundPolicySnapshot policy = policyReader.read(merchantId);
    if (!policy.allows(amount)) {
        return RefundResult.manualReview();
    }

    localRules.validate(amount); // Tốn cả thanh xuân

    UUID decisionId = UUID.randomUUID();

    // Vừa CHÈN vừa THẨM ĐỊNH LẠI
    int inserted = commands.insertIfPolicyStillAllows(
        decisionId,
        commandId,
        merchantId,
        amount,
        policy.revision()
    );
    if (inserted == 0) { // NẾU HỤT CHÂN THÌ DỪNG LẠI NGAY!
        throw new PolicyChangedException(merchantId, policy.revision());
    }
    return RefundResult.approved(decisionId);
}
```

Ra `affected-row = 1` là êm xuôi (Luật vẫn nguyên vẹn ngay lúc Lệnh thực thi). Ra `0` tức là Luật bị tắt, Số tiền quá lố, hoặc Đứa Nào Vừa Nâng Phiên Bản xong; Lúc này Giao Dịch bị đá Rollback, báo văng Exception để bên ngoài bắt Đầu Transaction mới mà làm lại (retry).

Cái `command_id` Unique chỉ để ngăn chặn bấm Submit liên tục (duplicate delivery). Văng lỗi Unique phải phân biệt rõ ràng với Lỗi Hụt Chân Phiên Bản và xử lý theo idempotency contract.

Hạt sạn nhỏ: Lệnh `INSERT ... SELECT` này xác nhận trạng thái lúc câu Lệnh Chạy (statement snapshot). Kẻ xấu vẫn CÓ THỂ chốt đè (commit) Ngay Sau Câu Lệnh Đó Nhưng TRƯỚC KHI Bọc Giao Dịch Này Kịp Chốt Sổ. Nếu Hợp đồng cấm tiệt luôn khe hở nhỏ xíu này -> Đi lấy Ổ Khóa Dòng (Row Lock) mà xài.

## 5. Giải pháp 4 — Nắm Đầu Ổ Khóa Đọc `FOR SHARE`

Ổ Khóa đọc `FOR SHARE` của PostgreSQL siết chặt cái Dòng Dữ Liệu đó (row-level lock) tới tận lúc Giao dịch Chốt Sổ hay Rollback, khiến thằng B Cập Nhật Luật (UPDATE/DELETE) vỡ mồm rụng răng Đứng Nhìn:

```sql
select merchant_id, auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId
for share;
```

Dùng tuyệt chiêu khóa Bi Quan (Pessimistic read) của Spring Data:

```java
public interface LockedPolicyRepository
    extends JpaRepository<MerchantRefundPolicy, UUID> {

    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("""
        select p
          from MerchantRefundPolicy p
         where p.merchantId = :merchantId
        """)
    Optional<MerchantRefundPolicy> findForDecision(UUID merchantId);
}
```

Với PostgreSQL, nhớ căng mắt xem Code nó sinh ra SQL có Dính Cụm `FOR SHARE` chưa; xài mẹ nó lệnh `nativeQuery = true` (hoặc test) cho chắc cú ăn ngủ khỏi lo.

```java
@Transactional
public RefundResult decideWhilePolicyLocked(
    UUID commandId,
    UUID merchantId,
    BigDecimal amount
) {
    // KHÓA ĐẦU NÓ LẠI!
    MerchantRefundPolicy policy = lockedPolicies
        .findForDecision(merchantId)
        .orElseThrow(PolicyNotFoundException::new);

    if (!policy.allows(amount)) {
        return RefundResult.manualReview();
    }

    // YÊN TÂM CHƠI CÁC TRÒ Ở ĐÂY VÌ LUẬT KHÔNG THỂ BỊ ĐỔI

    RefundDecision saved = decisions.save(
        RefundDecision.approved(
            commandId,
            merchantId,
            amount,
            policy.getAutoRefundLimit(),
            policy.getRevision()
        )
    );
    return RefundResult.approved(saved.getId());
}
```

Cách cuộc chơi diễn ra:

1. Lính A ẵm Cục Khóa Share Row.
2. Sếp B xách dao nhào vô đòi UPDATE -> Đứng Xếp Hàng!
3. Lính A chậm rãi nhét Quyết Định (INSERT) rồi Chốt (commit) hoặc Bỏ (rollback).
4. Khóa của A rơi xuống; Sếp B mới được lao vô làm tiếp (nếu chưa quá giờ - timeout/fail).

Luật thép: KHÔNG ĐƯỢC NHÉT các lệnh ngâm (Gọi HTTP, Gọi API I/O) vào cái Giao Dịch Đang Móc Khóa này. Có Khóa là phải Set thời gian chết (`lock_timeout`) rõ ràng và chuẩn bị Tinh thần Báo Lỗi để Tự Làm Lại. Nếu phải Khóa Nhiều Dòng, Nhớ Xếp Hàng Khóa theo ĐÚNG 1 TRÌNH TỰ (để giảm thiểu Bóp Cổ Nhau - Deadlock).

## 6. Giải pháp 5 — Nâng Khiên `REPEATABLE READ`

Nếu Hợp đồng nài nỉ xin Bức Ảnh Xuyên Suốt (stable transaction view):

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public RefundResult decideWithStableSnapshot(...) {
    RefundPolicySnapshot first = policyReader.read(merchantId);
    localRules.validate(amount);
    RefundPolicySnapshot second = policyReader.read(merchantId);

    if (first.revision() != second.revision()) {
        throw new IllegalStateException("Hợp đồng Tấm Ảnh Bị Phá Vỡ Rồi!"); // Sẽ chả bao giờ xảy ra!
    }
    // Ghi Sổ bằng cục Data lấy từ Tấm Ảnh Xuyên Suốt này.
}
```

Bảo Chứng: Hai phát SELECT đéo bao giờ khác nhau. Bắt buộc Mở SQL Soi (effective isolation) xem cái Bọc Vật Lý dưới DB có ăn được Mức Cô Lập này Không (Nhiều lúc thằng Giao Dịch lồng `REQUIRED` bên trong nó Bú Liếm cái Mức Cũ Rích của thằng Ngoài Cùng là Toang).

Trúng Đổi Lại (Trade-off):

- Bịt được Lỗi Đọc Không Lặp Lại MÀ ĐÉO CẦN ÔM KHÓA DÒNG (read row lock).
- NHƯNG Tấm Ảnh có thể Bị Ố Vàng Hôi Thiu (stale) so với Thằng Chốt mới.
- Giao dịch rề rà sẽ Giam Cầm mấy cái Cục Xóa Rác/Tài Nguyên ngâm giấm dưới DB lâu hơn.
- Viết Luồng Đa Luồng (Flow phức tạp) đụng độ (write conflicts/serialization failures) thì vẫn Bắt buộc Code Lệnh Làm Lại (retry).
- NÓ KHÔNG CHỮA ĐƯỢC CÁI TỘI LƯU LẠI VẾT NHƠ AUDIT KÉM (vẫn phải có Lịch sử Bất tử).

## 7. Giải pháp 6 — Làm Lại Đàng Hoàng Kẻo Hỏng (Bounded retry ở transaction mới)

Toàn bộ Cỗ Máy Cố Đấm Ăn Xôi (Retry) PHẢI LÔI RA KHỎI LỚP GIAO DỊCH (transactional worker):

```java
@Service
public class RefundDecisionRetrier {
    private final RefundDecisionAttempt attempt;
    private final Backoff backoff;

    public RefundDecisionRetrier(
        RefundDecisionAttempt attempt,
        Backoff backoff
    ) {
        this.attempt = attempt;
        this.backoff = backoff;
    }

    public RefundResult decide(RefundCommand command) {
        int maxAttempts = 3;
        for (int number = 1; number <= maxAttempts; number++) {
            try {
                return attempt.runInNewTransaction(command);
            } catch (PolicyChangedException | CannotSerializeTransactionException ex) {
                if (number == maxAttempts) {
                    throw ex; // Hết vé, cho văng thật
                }
                backoff.pauseWithJitter(number); // Nghỉ mệt 1 chút hẵng làm
            }
        }
        throw new IllegalStateException("unreachable");
    }
}

@Service
public class RefundDecisionAttempt {
    // ĐẺ GIAO DỊCH MỚI TINH!
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public RefundResult runInNewTransaction(RefundCommand command) {
        // Mở Luật Mới, Tính Lại Hết, Rồi Mới Nhét Có Điều Kiện.
    }
}
```

Trò Nghỉ mệt (`Backoff`) phải CÓ ĐIỂM DỪNG (bounded) và có tai nghe Ngắt Điện (interrupt-aware). Tuyệt Đối cấm ôm Xác Cũ (old entity/decision) qua Kiếp Mới (retry). Lạm dụng trò Thử Lại (retry amplification) ở mấy Cửa Hàng Đông Khách (hot merchant) là Dễ Đứt Cáp DB, nhớ gắn Đo Đạc (metric) và Cổng Trám (admission control).

## 8. Ảo Giác `SERIALIZABLE` - Đừng Tưởng Cứ Nâng Mức Là Đòi Chơi Đồ Tươi (Vì sao `SERIALIZABLE` không có nghĩa “latest”)

Cái mác `SERIALIZABLE` vĩ đại nó hứa hẹn là "Tụi bay chạy sao thì chạy, kết quả cuối cùng Tao sẽ sắp xếp y chang như việc tụi bây Xếp Hàng Từng Đứa chạy (serial order)". NÓ KHÔNG HỀ HỨA "Thằng vô sau Đọc Bắt Buộc phải dòm thấy Cục Dữ Liệu Tươi Nhất Của Thằng Vừa Ghi" (latest).
Ông A hoàn toàn lọt vào Trật Tự Xếp Hàng Chạy Trước (Serialize trước) Ông B. Dù trên Đồng Hồ Thực Tế B Vừa Chốt Xong, Quyết Định Số 7 Của A Và Update Số 8 Của B Cùng Nắm Tay Nhau Đi Vào Bảng Vàng!

Chỉ mang Đồ Thánh `SERIALIZABLE` ra dùng khi Tương tác Ghi/Đọc (read-write dependencies) Rất Phức Tạp, VÀ BẠN SẴN SÀNG ÓI RA MÁU ÔM FULL-TRANSACTION RETRY KHI GẶP MÃ LỖI `40001`. Đừng có Lười Biếng vác Dao Mổ Trâu Nâng Mức Cô Lập (isolation) Lên Cao Tít Chỉ Vì Nhát Tay Không Dám Ngồi Viết Hợp Đồng Thời Điểm Rành Mạch!

## 9. Cân Đo Đong Đếm (Trade-off comparison)

| Bí Kíp Phương Pháp | Lá Chắn Bảo Vệ | Số Phận Kẻ Chậm Chân (Loser behavior) | Áp Lực Hệ Thống (Contention/latency) | Nhiều Máy Chủ? (Multi-instance) |
| --- | --- | --- | --- | --- |
| Đọc 1 Ảnh Duy Nhất | Tính toàn vẹn lúc Đọc (Evaluation) | Không Ai Bị Giết | Nhẹ nhàng | Chơi tuốt (Nếu Audit Lưu Bất Tử) |
| Tách Lịch Sử + FK | Revision Phải Tồn Tại, Dễ Dò Dấu (Audit) | Dời Kim Hụt là Văng (no-op) | Tốn Thêm Ổ Cứng + Bảng | Chơi tốt |
| Gắn Điều Kiện Đuôi | Luật Y Nguyên lúc Ghi Chốt | Trả về Số `0`, Phải làm lại | Nhẹ đến Vừa Phải | Chơi tuốt |
| Trói `FOR SHARE` | Kẻ Đổi Luật Không Thể Chọt Vào | Thằng Cập Nhật Đứng Đợi/Đứt Cáp | Căng Tay Nếu Luật Này Đang Nóng! | Vô tư luôn |
| `REPEATABLE READ` | Cái Bọc Giao Dịch Không Đứt Gãy | Dễ Chết Ở Các Đấu Trường Khác | Tốn Tài Nguyên Máy Chụp/Làm Lại | Ngon lành |
| Khóa Java `JVM` | Chặn Đám Nhau Trong Cùng 1 RAM | Luồng Nội Bộ Bị Ngậm Miệng | Lủng Lỗ Chỗ! Vô Dụng với DB! | **BÓ TAY** |

## 10. Cách Xử Lý Hậu Sự Khi Vỡ Trận (Failure behavior)

- Lính A xé kèo (Rollback): Tờ Giấy Duyệt Tan Biến; Khóa Bị Tháo, Thế giới bình yên.
- Sếp B xé kèo: Luật Phiên Bản Mới Tiêu Tán; Lính A cứ vô tư nhâm nhi Tấm Ảnh Cũ mà Xét.
- Chặn Điều Kiện Báo Số 0 (Mismatch): Đá văng Giao Dịch! Dẹp! Tải Lại Ảnh Mới Ở Giao Dịch Khác.
- Khóa Hết Giờ (Timeout/Deadlock): Toàn bộ Giao dịch hiện tại Lãnh Án Tử (fail)! Quăng mẹ cái Bộ đệm Rác (persistence context) đi, đừng xài lại!
- Tắt Nguồn Bất Tử giữa chừng (Crash sau chốt, Trước Báo Khách): Lôi cái ID Cũ (`command_id`) ra mà gửi lại (replay)! CẤM CHẠY LẠI KHÂU TRỪ TIỀN (refund side effect)!
- Cập Nhật Lịch Sử Chết Giữa Đường (Write fail): Rollback Đẩy Toàn Bộ Cả Cây Kim Lẫn Lịch Sử Xuống Mồ.

## 11. Đúc Kết Chọn Bài Chữa Bệnh (Recommendation cho case này)

Combo Chuẩn Mực Bắt Buộc (Default):

1. Rinh Mẹ Nó cái Đối Tượng `RefundPolicySnapshot` (Đóng đinh 1 lần) làm thước đo (evaluation);
2. Dán nhãn Chép Sổ gồm Phiên Bản, Mức Đã Xét, Kết Quả vào Tờ Phán Quyết (decision);
3. Dựng Bàn Thờ Khóa Ngoại (foreign key) Bảo Vệ Cái Kho Lịch Sử Luật.
4. Gắn Thêm Phanh Phụ (conditional final validation) Nếu Sếp Bắt Trạng Thái Phải Tươi Mới ngay tại Phút Cuối (statement).
5. Chỉ Chơi Bùa Trói `FOR SHARE` Khi Sếp Dí Dao Vào Cổ Ép Rằng Luật Không Được Sai Lệch Một Ly nào Cho Tới Khi Chốt (commit).

Bùa `REPEATABLE READ` cực Hay Nếu Ông Lười Muốn Một Tấm Ảnh Xuyên Suốt Cái Giao Dịch Dài Thòng, NHƯNG NÓ KHÔNG CỨU ĐƯỢC CHUYỆN ÔNG PHẢI VIẾT ĐỀU ĐIỀU KHOẢN VÀ LƯU BẰNG CHỨNG KIỂM TOÁN!

## 12. Danh Sách Kiểm Tra Trước Khi Trình Làng (Production checklist)

### Luật Lệ & Đạo Đức (Semantics)

- [ ] Đã thống nhất với Sếp là Luật Bắt Đầu Đọc, Luật Phút Cuối, hay Luật Lúc Chốt (Commit)?
- [ ] Cái Giấy Duyệt đã chốt có Chỉ Thẳng (Join) vào đúng cái Lịch Sử Bất Tử Không?
- [ ] KHÔNG BỐC GHÉP Râu Ông Nọ Cằm Bà Kia (trộn fields từ nhiều `PolicyView`).
- [ ] Chống Spam Nút Bấm (Duplicate command) và Tính vẹn Toàn Phiên Bản (Policy mutation) Có Được Xử Lý Riêng Biệt Chưa?

### Nghi Thức Chốt Sổ (Transactions)

- [ ] Lệnh Kiểm Tra Mức Cô Lập Đã Soi Thẳng Vào Giao Dịch Vật Lý Chạy Thật Không?
- [ ] Lệnh Gắn Đuôi (Conditional) khi Cập nhật Trả `0` Có Bắn Lỗi Rõ Ràng Chưa?
- [ ] Việc Thử Lại (Retry) Có chịu Nhét Vào Giao Dịch Mới Tinh và Tính Toán Lại Toàn Bộ Chưa?
- [ ] CẤM Lệnh HTTP Gọi Web Ở Trong Vòng Cấm Địa Ôm Khóa (Row lock)!
- [ ] Đã Vẽ Rõ Trật Tự Khóa (Lock order), Đặt `lock_timeout` Và Dọn Rác Deadlock Chưa?

### Đồ Nghề Cấp Cứu (Operations)

- [ ] Có Gắn Máy Đo (Metric) Nhảy Mất Phiên Bản (mismatch), Thử Lại, Nghẽn Khóa và Quá Giờ Chưa?
- [ ] Có Sẵn Lệnh Dò Sổ Kế Toán (Query reconciliation) để Truy Tìm Giấy Duyệt So Với Phiên Bản Luật Chốt Chưa?
- [ ] Lịch Trình Gọi (Trace) Có Lưu Cái Bản Đọc Vào Đầu Và Phiên Bản Chốt Xuống Đít Không?
- [ ] Bài Test Đang Chạy Thật Trên Cơ Thể Của PostgreSQL, KHÔNG PHẢI MẤY TRÒ ẢO MA TỪ CÁI DB ĐỒ CHƠI H2.
- [ ] Cửa Hàng Bị Treo Nóng (Hot merchant contention) Và Áp Lực Bể Kết Nối CSDL Có Đang Bị Theo Dõi Chặt Chẽ Không?
