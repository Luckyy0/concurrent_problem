# Giải Pháp: Sử Dụng Snapshot, Validation và Khóa (Solutions)

## 1. Mục tiêu cốt lõi (Design objectives)

Trước khi tiến hành thiết kế, hệ thống cần làm rõ yêu cầu nghiệp vụ về tính đồng nhất của quyết định xét duyệt:

```text
Yêu cầu hợp lệ của quyết định:
  - Dựa trên chính sách ở thời điểm hệ thống ĐỌC để đánh giá?
  - Dựa trên chính sách mới nhất tại thời điểm hệ thống GHI quyết định?
  - Hay chính sách phải ĐƯỢC GIỮ NGUYÊN không đổi từ lúc đọc cho đến khi kết thúc giao dịch?
```

Bất kể tùy chọn nào được đưa ra, hệ thống PHẢI đảm bảo việc ghi nhận duy nhất một Phiên bản chính sách đồng nhất (coherent policy revision), ngăn chặn tuyệt đối tình trạng kết hợp kết quả đánh giá ở snapshot 1 với phiên bản audit ở snapshot 2.

## 2. Giải pháp 1 — Đọc một Immutable Policy Snapshot

Nếu quy định kinh doanh cho phép đánh giá dựa trên trạng thái chính sách tại thời điểm đọc, và không yêu cầu phải là dữ liệu mới nhất khi chốt, thì ứng dụng nên đọc dữ liệu MỘT LẦN DUY NHẤT. Dữ liệu này được đóng gói thành một Value Object không thể thay đổi (immutable):

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

Tầng Repository đóng gói kết quả truy vấn thành đối tượng trên:

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

Tầng Dịch Vụ sẽ chỉ sử dụng duy nhất object này trong suốt quá trình xử lý:

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public RefundResult decide(
    UUID commandId,
    UUID merchantId,
    BigDecimal amount
) {
    // Chỉ đọc thông tin chính sách MỘT LẦN duy nhất
    RefundPolicySnapshot policy = policyReader.read(merchantId);

    if (!policy.allows(amount)) {
        return RefundResult.manualReview();
    }

    localRules.validate(amount); // Quá trình tính toán kéo dài

    // Ghi nhận quyết định dựa trên chính thông tin vừa đọc
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

Hệ thống ghi nhận bằng chứng xét duyệt khớp hoàn toàn với phiên bản chính sách đã truy xuất. Mọi cập nhật đồng thời diễn ra sau lệnh đọc sẽ không tác động đến giao dịch này.

> **Ghi chú:** Đọc dữ liệu một lần giải quyết được triệt để vấn đề trộn lẫn snapshot. Tuy nhiên, nó đòi hỏi hệ thống phải lưu trữ lịch sử chính sách (immutable policy history) để đối soát quyết định dựa trên các phiên bản cũ trong tương lai.

## 3. Giải pháp 2 — Lưu trữ Lịch Sử Chính Sách Bất Biến (Versioned policy history)

Việc duy trì một bản ghi cập nhật trạng thái mới nhất là chưa đủ, hệ thống cần bảo tồn lịch sử qua từng phiên bản:

```sql
-- BẢNG LƯU TRỮ LỊCH SỬ BẤT BIẾN
create table merchant_refund_policy_version (
    merchant_id uuid not null,
    revision bigint not null,
    auto_refund_limit numeric(19, 2) not null,
    active boolean not null,
    created_at timestamptz not null,
    primary key (merchant_id, revision),
    check (auto_refund_limit >= 0)
);

-- BẢNG CON TRỎ (Chỉ định phiên bản hiện hành)
create table merchant_refund_policy_current (
    merchant_id uuid primary key,
    revision bigint not null,
    foreign key (merchant_id, revision)
        references merchant_refund_policy_version(merchant_id, revision)
);

-- RÀNG BUỘC KHÓA NGOẠI KHI GHI QUYẾT ĐỊNH
alter table refund_decision
    add constraint fk_refund_decision_policy_version
    foreign key (merchant_id, policy_revision)
    references merchant_refund_policy_version(merchant_id, revision);
```

Thao tác cập nhật chính sách bao gồm thêm phiên bản mới và chuyển con trỏ:

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

-- Tiến trình hợp lệ khi số dòng ảnh hưởng của UPDATE là 1
commit;
```

Lệnh truy vấn của ứng dụng khi sử dụng Join:

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

Cấu trúc khóa ngoại (Foreign key) đảm bảo tính toàn vẹn của audit log. Nếu thao tác dời con trỏ (cập nhật) không ảnh hưởng tới bản ghi nào (bị tranh chấp optimistic locking), quản trị viên cần khởi tạo lại giao dịch với thông tin cập nhật nhất.

## 4. Giải pháp 3 — Xác Thực Có Điều Kiện Ở Bước Cuối (Conditional final validation)

Nếu nghiệp vụ yêu cầu kiểm soát chính sách không thay đổi ngay tại thời điểm ghi quyết định, hệ thống nên nhúng điều kiện kiểm tra (predicate) trực tiếp vào lệnh lưu (sử dụng `INSERT ... SELECT`):

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
  and p.revision = :expectedRevision -- KIỂM TRA PHIÊN BẢN KHÔNG THAY ĐỔI
  and p.active
  and :amount <= p.auto_refund_limit;
```

Tích hợp vào Spring Repository:

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

Tầng Dịch Vụ:

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

    localRules.validate(amount); // Tính toán bổ sung

    UUID decisionId = UUID.randomUUID();

    // Kết hợp Ghi dữ liệu và Kiểm tra xác thực (Validation)
    int inserted = commands.insertIfPolicyStillAllows(
        decisionId,
        commandId,
        merchantId,
        amount,
        policy.revision()
    );
    if (inserted == 0) { // Hủy giao dịch nếu bản ghi đã thay đổi
        throw new PolicyChangedException(merchantId, policy.revision());
    }
    return RefundResult.approved(decisionId);
}
```

Kết quả `affected-row = 1` chứng tỏ chính sách ổn định trong thời gian lệnh ghi thực thi. Nếu `affected-row = 0`, nguyên nhân có thể do hạn mức, trạng thái active thay đổi hoặc phiên bản đã được nâng lên. Trường hợp này, ứng dụng ném ngoại lệ và giao dịch sẽ bị rollback, sau đó tiến hành thủ tục retry.

Lưu ý: Lệnh `INSERT ... SELECT` phản ánh trạng thái statement snapshot. Khả năng giao dịch quản trị thực hiện commit ngay sau lệnh này nhưng trước khi giao dịch xét duyệt hoàn tất (commit) vẫn tồn tại (tuy cực kỳ thấp). Để loại bỏ hoàn toàn khoảng trống hẹp này, hệ thống cần áp dụng cơ chế Khóa cấp dòng (Row Lock).

## 5. Giải pháp 4 — Khóa Chia Sẻ (Pessimistic Read / `FOR SHARE`)

Sử dụng khóa `FOR SHARE` để đảm bảo bản ghi chính sách bị khóa đối với các hành động chỉnh sửa (UPDATE/DELETE) cho đến khi giao dịch hiện tại hoàn tất:

```sql
select merchant_id, auto_refund_limit, active, revision
from merchant_refund_policy
where merchant_id = :merchantId
for share;
```

Cấu hình khóa bi quan trong Spring Data:

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

Tầng Dịch Vụ:

```java
@Transactional
public RefundResult decideWhilePolicyLocked(
    UUID commandId,
    UUID merchantId,
    BigDecimal amount
) {
    // KHÓA BẢN GHI
    MerchantRefundPolicy policy = lockedPolicies
        .findForDecision(merchantId)
        .orElseThrow(PolicyNotFoundException::new);

    if (!policy.allows(amount)) {
        return RefundResult.manualReview();
    }

    // Đảm bảo chính sách không thể bị thay đổi đồng thời trong đoạn mã này

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

Cơ chế khóa:

1. Luồng xét duyệt thu được khóa cấp dòng `FOR SHARE`.
2. Luồng quản trị rủi ro bị block nếu cố gắng UPDATE cùng bản ghi đó.
3. Luồng xét duyệt chèn kết quả và commit.
4. Luồng quản trị mới tiếp tục (nếu chưa vượt quá `lock_timeout`).

Nguyên tắc bắt buộc: Không thực hiện các tác vụ I/O chậm (như gọi HTTP API) trong khi đang giữ khóa cơ sở dữ liệu. Bắt buộc thiết lập `lock_timeout` rõ ràng và có chiến lược dự phòng khi chờ khóa thất bại.

## 6. Giải pháp 5 — Mức Cô Lập `REPEATABLE READ`

Sử dụng mức cô lập cấp độ giao dịch (stable transaction view):

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public RefundResult decideWithStableSnapshot(...) {
    RefundPolicySnapshot first = policyReader.read(merchantId);
    localRules.validate(amount);
    RefundPolicySnapshot second = policyReader.read(merchantId);

    if (first.revision() != second.revision()) {
        throw new IllegalStateException("Trạng thái bất nhất, lỗi không mong muốn!"); // Điều này sẽ không xảy ra ở mức cô lập này
    }
    // Ghi sổ dựa trên Snapshot ổn định của giao dịch.
}
```

Ưu điểm và Đánh đổi (Trade-off):

- Giải quyết lỗi Đọc Không Lặp Lại mà không cần cấp khóa độc quyền.
- Snapshot cung cấp có thể không phải dữ liệu mới nhất nếu luồng quản trị đã commit bản ghi sau khi snapshot này được cấp.
- Giao dịch kéo dài có thể làm chậm quá trình dọn dẹp các phiên bản ghi đã cũ trong PostgreSQL (Vacuum process).
- Bắt buộc xử lý lỗi do tranh chấp ghi (write conflict) bằng cơ chế Retry nếu cần thực thi trên cùng bản ghi.
- Giải pháp này vẫn đòi hỏi có hệ thống lịch sử lưu trữ để đáp ứng yêu cầu Audit.

## 7. Giải pháp 6 — Áp dụng Cơ chế Bounded Retry

Các tác vụ thử lại (Retry) cần được tách biệt hoàn toàn khỏi ngữ cảnh transaction hiện hữu:

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
                    throw ex; // Hết số lượt thử lại, ném lỗi ra ngoài
                }
                backoff.pauseWithJitter(number); // Nghỉ một chút và thêm ngẫu nhiên thời gian
            }
        }
        throw new IllegalStateException("unreachable");
    }
}

@Service
public class RefundDecisionAttempt {
    // KHỞI TẠO GIAO DỊCH MỚI
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public RefundResult runInNewTransaction(RefundCommand command) {
        // Đọc lại toàn bộ quy tắc, tính toán lại và chèn có điều kiện.
    }
}
```

Giới hạn số lần thử lại để tránh lãng phí tài nguyên và tạo ra hiện tượng "cộng hưởng lỗi" (retry amplification).

## 8. Tại sao `SERIALIZABLE` không đảm bảo "Dữ liệu mới nhất"

Mức cô lập `SERIALIZABLE` cung cấp cam kết: Kết quả thực thi song song tương đương với kết quả của quá trình thực thi các giao dịch đó một cách tuần tự.
Mức này KHÔNG cam kết một lệnh truy vấn luôn đọc được dữ liệu mà giao dịch khác vừa mới commit trong cùng khoảng thời gian đó.

Nếu một tác vụ Đọc/Ghi (Luồng A) và một tác vụ Cập nhật (Luồng B) hoạt động cùng thời điểm, SSI (Serializable Snapshot Isolation) coi kịch bản "Luồng A chạy trước, sau đó Luồng B chạy" là hoàn toàn hợp lệ, cho phép cả hai luồng commit thành công mà không phát hiện anomaly.

Nên sử dụng `SERIALIZABLE` khi cấu trúc tương tác read-write dependency rất phức tạp và phải xử lý mã lỗi `40001` (serialization failure) bằng cách retry.

## 9. Đánh giá và Lựa chọn (Trade-off comparison)

| Chiến Lược Giải Quyết | Mức Độ An Toàn | Rủi Ro Đối Với Cập Nhật | Yêu cầu Tài Nguyên / Độ Trễ | Môi trường phân tán |
| --- | --- | --- | --- | --- |
| Đọc Một Lần (Immutable Snapshot) | Bảo đảm toàn vẹn khi đọc | Không block, không xung đột | Nhẹ | Khả thi (cần lưu audit) |
| Lịch sử và Con trỏ FK | Toàn vẹn lịch sử, dễ dàng Audit | Lỗi tranh chấp khi dịch con trỏ | Cần lưu trữ bảng bổ sung | Khả thi cao |
| Truy vấn Cập Nhật Có Điều Kiện | Xác thực chính xác thời điểm Ghi | Giao dịch xét duyệt có thể phải Retry | Nhẹ nhàng | Khả thi |
| Khóa Pessimistic (`FOR SHARE`) | Tuyệt đối chặn thay đổi đồng thời | Cập nhật bị chờ hoặc timeout | Nguy cơ nghẽn cục bộ | Khả thi |
| `REPEATABLE READ` | Cung cấp View ổn định | Đòi hỏi xử lý write conflict riêng | Khá tốn kém (do Vacuum) | Khả thi |
| Khóa nội bộ (`synchronized`) | An toàn cấp luồng bộ nhớ JVM | Không đồng bộ giữa nhiều máy chủ | Không quản lý DB | **Không Khả Thi** |

## 10. Chiến lược Phục hồi sau Sự cố (Failure behavior)

- Hủy giao dịch (Rollback) bên Xét duyệt: Bản ghi quyết định bị xóa, khóa bị giải phóng.
- Hủy giao dịch bên Quản trị: Phiên bản chính sách không được áp dụng, luồng đọc tiếp tục sử dụng bản hiện tại một cách an toàn.
- Cập nhật có điều kiện trả về `0` (Mismatch): Hủy giao dịch xét duyệt và tự động xử lý Retry với dữ liệu mới.
- Hết hạn khóa (Lock Timeout / Deadlock): Ném ngoại lệ, tiến hành thu hồi dữ liệu. Yêu cầu tránh sử dụng lại context Hibernate cũ.
- Crash giữa chừng (Sau commit DB, chưa trả về Client): Client có thể gửi lại cùng một ID, hệ thống sử dụng cơ chế idempotency để chặn các side-effect dư thừa.
- Lỗi ghi Audit: Toàn bộ thao tác cập nhật phiên bản chính sách và quyết định sẽ cùng bị rollback.

## 11. Khuyến nghị Giải pháp (Recommendation)

Cấu trúc tiếp cận chuẩn mực:

1. Đóng gói kết quả đọc đầu tiên vào đối tượng `RefundPolicySnapshot` duy nhất để sử dụng.
2. Gắn kèm phiên bản chính sách, hạn mức và kết quả vào bản ghi quyết định khi lưu trữ.
3. Thiết lập Khóa ngoại (Foreign key constraint) để tham chiếu đến bảng lưu trữ lịch sử chính sách độc lập.
4. Tích hợp validation (cập nhật có điều kiện) ở bước insert nếu cần xác thực tính cập nhật của chính sách lúc chốt.
5. Chỉ sử dụng khóa bi quan (`FOR SHARE`) cho những trường hợp hệ thống đòi hỏi độ chính xác tuyệt đối mà độ trễ hay tắc nghẽn cục bộ được đánh giá là chấp nhận được.

## 12. Danh sách rà soát triển khai (Production checklist)

### Yêu cầu Nghiệp vụ (Semantics)

- [ ] Quy định cụ thể về thời điểm áp dụng chính sách: Đầu giao dịch hay trước lúc commit?
- [ ] Bản ghi quyết định có liên kết (Foreign key) nhất quán với bảng lịch sử hay không?
- [ ] Kiểm soát việc không sử dụng nhiều thuộc tính từ các nguồn đọc khác nhau cho một thực thể (No view mixing).
- [ ] Quy trình bảo vệ Idempotency đã bao gồm chức năng ngăn chặn thao tác cập nhật trùng lặp?

### Quản lý Giao Dịch (Transactions)

- [ ] Lệnh kiểm tra cấp độ cô lập có xác nhận chính xác các lệnh thực thi ở môi trường Production chưa?
- [ ] Các lệnh Cập nhật Có điều kiện đã có code bắt trường hợp trả về 0 dòng ảnh hưởng (affected-row) chưa?
- [ ] Cơ chế Retry có gọi lại giao dịch vật lý hoàn toàn mới (REQUIRES_NEW) không?
- [ ] KHÔNG xử lý các phương thức HTTP, I/O chậm khi bản ghi đang bị khóa (`FOR SHARE`).
- [ ] Cấu hình thiết lập `lock_timeout` rõ ràng và đảm bảo không xảy ra Deadlock chéo.

### Vận hành và Giám sát (Operations)

- [ ] Xây dựng bảng biểu đo lường tần suất Lệch Phiên Bản (mismatch), Timeout và Retry.
- [ ] Phát triển các truy vấn đối soát (Reconciliation) để xác thực quyết định dựa trên các phiên bản lịch sử.
- [ ] Cung cấp Trace Span đầy đủ từ bước Đọc đầu tiên đến bước Commit cuối cùng.
- [ ] Đảm bảo các bài kiểm thử tương tranh (Concurrency tests) được chạy trên PostgreSQL chứ không phải trên cơ sở dữ liệu in-memory (H2).
- [ ] Liên tục đo lường áp lực Connection Pool và xử lý độ trễ trong các bảng chịu truy cập cao.
