# Thiết kế tiếp nhận hoàn tiền an toàn

## 1. Mục tiêu thiết kế

Giải pháp cần đồng thời bảo đảm:

1. cùng một ý định gửi lại chỉ có một kết quả;
2. các ý định khác nhau không dùng quá số tiền đã thu;
3. tiền đang chờ nhà cung cấp xử lý vẫn chiếm hạn mức;
4. bảng tổng hợp, refund, bút toán và outbox cùng chốt hoặc cùng hoàn tác;
5. hoàn tất hoặc giải phóng một khoản chỉ xảy ra một lần;
6. lỗi cơ sở dữ liệu không bị trả về như một kết quả nghiệp vụ;
7. cơ chế vẫn đúng khi có nhiều máy chủ ứng dụng.

Giải pháp đề xuất dùng JDBC cho các câu ghi cần biết chính xác số dòng ảnh hưởng
và dữ liệu `RETURNING`. JPA vẫn có thể dùng cho phần đọc hoặc các thực thể không
nằm trên đường tranh chấp.

## 2. Lược đồ dữ liệu

### Giao dịch đã thu và bảng tổng hợp hạn mức

```sql
CREATE TABLE payment_charge (
    charge_id UUID PRIMARY KEY,
    merchant_id UUID NOT NULL,
    captured_amount BIGINT NOT NULL,
    allocated_refund_amount BIGINT NOT NULL DEFAULT 0,
    completed_refund_amount BIGINT NOT NULL DEFAULT 0,
    currency CHAR(3) NOT NULL,
    revision BIGINT NOT NULL DEFAULT 0,
    captured_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    CHECK (captured_amount > 0),
    CHECK (allocated_refund_amount >= 0),
    CHECK (completed_refund_amount >= 0),
    CHECK (completed_refund_amount <= allocated_refund_amount),
    CHECK (allocated_refund_amount <= captured_amount)
);

CREATE INDEX payment_charge_merchant_idx
    ON payment_charge (merchant_id, charge_id);
```

Tiền dùng đơn vị nhỏ nhất của loại tiền và lưu bằng số nguyên. Quy tắc số chữ số,
loại tiền không có đơn vị lẻ và chuyển đổi ngoại tệ phải do miền thanh toán quy
định; không dùng `double`.

### Yêu cầu và kết quả lũy đẳng

```sql
CREATE TABLE refund_request (
    request_id UUID PRIMARY KEY,
    merchant_id UUID NOT NULL,
    idempotency_key VARCHAR(200) NOT NULL,
    request_fingerprint CHAR(64) NOT NULL,
    charge_id UUID NOT NULL,
    amount BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    outcome VARCHAR(30) NOT NULL,
    result_refund_id UUID,
    response_code VARCHAR(40),
    created_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    UNIQUE (merchant_id, idempotency_key),
    UNIQUE (result_refund_id),
    CHECK (amount > 0),
    CHECK (outcome IN (
        'PROCESSING',
        'ACCEPTED',
        'LIMIT_EXCEEDED',
        'CHARGE_NOT_FOUND',
        'CURRENCY_MISMATCH'
    )),
    CHECK (
        (outcome = 'PROCESSING'
            AND completed_at IS NULL
            AND response_code IS NULL
            AND result_refund_id IS NULL)
        OR
        (outcome = 'ACCEPTED'
            AND completed_at IS NOT NULL
            AND response_code = 'ACCEPTED'
            AND result_refund_id IS NOT NULL)
        OR
        (outcome IN (
                'LIMIT_EXCEEDED',
                'CHARGE_NOT_FOUND',
                'CURRENCY_MISMATCH'
            )
            AND completed_at IS NOT NULL
            AND response_code IS NOT NULL
            AND result_refund_id IS NULL)
    )
);
```

`PROCESSING` chỉ tồn tại bên trong giao dịch đang chạy. Dịch vụ không được chốt
giao dịch khi dòng vẫn ở trạng thái này.

### Khoản hoàn và máy trạng thái

```sql
CREATE TABLE refund (
    refund_id UUID PRIMARY KEY,
    request_id UUID NOT NULL UNIQUE
        REFERENCES refund_request (request_id),
    charge_id UUID NOT NULL
        REFERENCES payment_charge (charge_id),
    merchant_id UUID NOT NULL,
    amount BIGINT NOT NULL,
    currency CHAR(3) NOT NULL,
    status VARCHAR(30) NOT NULL,
    provider_refund_id VARCHAR(200),
    created_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL,
    UNIQUE (merchant_id, provider_refund_id),
    CHECK (amount > 0),
    CHECK (status IN (
        'PENDING_PROVIDER',
        'SUCCEEDED',
        'FAILED_RELEASED'
    )),
    CHECK (
        (status = 'PENDING_PROVIDER' AND completed_at IS NULL)
        OR
        (status IN ('SUCCEEDED', 'FAILED_RELEASED')
            AND completed_at IS NOT NULL)
    )
);
```

Ràng buộc duy nhất với `provider_refund_id` cho phép nhiều giá trị `NULL` trong
PostgreSQL, nhưng không cho hai khoản hoàn đã biết cùng mã nhà cung cấp.

### Sổ lịch sử vận hành

```sql
CREATE TABLE refund_ledger_entry (
    entry_id UUID PRIMARY KEY,
    charge_id UUID NOT NULL
        REFERENCES payment_charge (charge_id),
    refund_id UUID NOT NULL
        REFERENCES refund (refund_id),
    entry_type VARCHAR(30) NOT NULL,
    allocation_delta BIGINT NOT NULL,
    completion_delta BIGINT NOT NULL,
    charge_revision BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    UNIQUE (refund_id, entry_type),
    CHECK (
        (entry_type = 'REFUND_ALLOCATED'
            AND allocation_delta > 0
            AND completion_delta = 0)
        OR
        (entry_type = 'REFUND_SUCCEEDED'
            AND allocation_delta = 0
            AND completion_delta > 0)
        OR
        (entry_type = 'REFUND_RELEASED'
            AND allocation_delta < 0
            AND completion_delta = 0)
    )
);
```

Đây là sổ lịch sử cho hạn mức hoàn tiền, không phải tuyên bố rằng hệ thống đã có
sổ kế toán bút toán kép hoàn chỉnh.

### Outbox

```sql
CREATE TABLE outbox_event (
    event_id UUID PRIMARY KEY,
    aggregate_type VARCHAR(50) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(80) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    published_at TIMESTAMPTZ,
    UNIQUE (aggregate_type, aggregate_id, event_type)
);
```

Với case này, `aggregate_id` là `refund_id`; sự kiện tiếp nhận là
`REFUND_REQUESTED`.

## 3. Hợp đồng lệnh và kết quả

```java
public record RefundCommand(
        UUID requestId,
        UUID merchantId,
        String idempotencyKey,
        String requestFingerprint,
        UUID chargeId,
        long amount,
        String currency,
        String reasonCode
) {
    public RefundCommand {
        Objects.requireNonNull(requestId);
        Objects.requireNonNull(merchantId);
        Objects.requireNonNull(idempotencyKey);
        Objects.requireNonNull(requestFingerprint);
        Objects.requireNonNull(chargeId);
        Objects.requireNonNull(currency);
        Objects.requireNonNull(reasonCode);
        if (idempotencyKey.isBlank() || idempotencyKey.length() > 200) {
            throw new IllegalArgumentException("invalid idempotency key");
        }
        if (amount <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }
    }
}

public sealed interface RefundResult {
    record Accepted(UUID requestId, UUID refundId) implements RefundResult {}
    record LimitExceeded(UUID requestId) implements RefundResult {}
    record ChargeNotFound(UUID requestId) implements RefundResult {}
    record CurrencyMismatch(UUID requestId) implements RefundResult {}
    record Busy(UUID requestId, String reason) implements RefundResult {}
}
```

`requestFingerprint` được tính từ biểu diễn chuẩn hóa của `merchantId`,
`chargeId`, `amount`, `currency` và `reasonCode`. Không đưa thời gian nhận request
hoặc giá trị ngẫu nhiên vào dấu vân tay.

## 4. Chiếm khóa lũy đẳng

```java
@Repository
public class RefundRequestDao {
    private final NamedParameterJdbcTemplate jdbc;

    public RefundRequestDao(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public boolean tryClaim(RefundCommand command, Instant now) {
        int changed = jdbc.update("""
                INSERT INTO refund_request (
                    request_id,
                    merchant_id,
                    idempotency_key,
                    request_fingerprint,
                    charge_id,
                    amount,
                    currency,
                    outcome,
                    created_at
                ) VALUES (
                    :requestId,
                    :merchantId,
                    :idempotencyKey,
                    :fingerprint,
                    :chargeId,
                    :amount,
                    :currency,
                    'PROCESSING',
                    :createdAt
                )
                ON CONFLICT (merchant_id, idempotency_key) DO NOTHING
                """, new MapSqlParameterSource()
                .addValue("requestId", command.requestId())
                .addValue("merchantId", command.merchantId())
                .addValue("idempotencyKey", command.idempotencyKey())
                .addValue("fingerprint", command.requestFingerprint())
                .addValue("chargeId", command.chargeId())
                .addValue("amount", command.amount())
                .addValue("currency", command.currency())
                .addValue("createdAt", Timestamp.from(now)));
        return changed == 1;
    }

    public StoredRefundRequest requireExisting(
            UUID merchantId,
            String idempotencyKey
    ) {
        return jdbc.queryForObject("""
                SELECT request_id,
                       merchant_id,
                       idempotency_key,
                       request_fingerprint,
                       charge_id,
                       amount,
                       currency,
                       outcome,
                       result_refund_id,
                       response_code
                FROM refund_request
                WHERE merchant_id = :merchantId
                  AND idempotency_key = :idempotencyKey
                """, Map.of(
                "merchantId", merchantId,
                "idempotencyKey", idempotencyKey
        ), STORED_REQUEST_MAPPER);
    }

    public void completeAccepted(
            UUID requestId,
            UUID refundId,
            Instant completedAt
    ) {
        requireOne(jdbc.update("""
                UPDATE refund_request
                SET outcome = 'ACCEPTED',
                    result_refund_id = :refundId,
                    response_code = 'ACCEPTED',
                    completed_at = :completedAt
                WHERE request_id = :requestId
                  AND outcome = 'PROCESSING'
                """, Map.of(
                "requestId", requestId,
                "refundId", refundId,
                "completedAt", Timestamp.from(completedAt)
        )));
    }

    public void completeRejected(
            UUID requestId,
            String outcome,
            Instant completedAt
    ) {
        Set<String> allowed = Set.of(
                "LIMIT_EXCEEDED",
                "CHARGE_NOT_FOUND",
                "CURRENCY_MISMATCH"
        );
        if (!allowed.contains(outcome)) {
            throw new IllegalArgumentException("unsupported outcome");
        }
        requireOne(jdbc.update("""
                UPDATE refund_request
                SET outcome = :outcome,
                    response_code = :outcome,
                    completed_at = :completedAt
                WHERE request_id = :requestId
                  AND outcome = 'PROCESSING'
                """, Map.of(
                "requestId", requestId,
                "outcome", outcome,
                "completedAt", Timestamp.from(completedAt)
        )));
    }

    private static void requireOne(int changed) {
        if (changed != 1) {
            throw new IllegalStateException(
                    "expected one refund request row, got " + changed
            );
        }
    }
}
```

`StoredRefundRequest.requireSameRequest(command)` phải so sánh dấu vân tay trước
khi `toResult()` phát lại phản hồi. Nếu dấu vân tay khác, ném lỗi
`IdempotencyKeyReusedException`.

## 5. Kiểm tra danh tính giao dịch

Không dùng phép đọc này để quyết định hạn mức. Nó chỉ xác nhận dữ liệu ổn định:

```java
public record ChargeIdentity(
        UUID chargeId,
        UUID merchantId,
        String currency
) {}

@Repository
public class ChargeIdentityDao {
    private final NamedParameterJdbcTemplate jdbc;

    public Optional<ChargeIdentity> find(UUID chargeId, UUID merchantId) {
        return jdbc.query("""
                SELECT charge_id, merchant_id, currency
                FROM payment_charge
                WHERE charge_id = :chargeId
                  AND merchant_id = :merchantId
                """, Map.of(
                "chargeId", chargeId,
                "merchantId", merchantId
        ), (rs, rowNum) -> new ChargeIdentity(
                rs.getObject("charge_id", UUID.class),
                rs.getObject("merchant_id", UUID.class),
                rs.getString("currency")
        )).stream().findFirst();
    }
}
```

Giao dịch thanh toán đã thu không bị xóa vật lý. Nếu nghiệp vụ cho phép vô hiệu
hóa quyền hoàn, thêm trạng thái đó vào chính điều kiện cập nhật ở bước tiếp theo.

## 6. Cập nhật hạn mức nguyên tử

```java
public record RefundCapacityAfterAllocation(
        long capturedAmount,
        long allocatedAmount,
        long completedAmount,
        long revision
) {}

@Repository
public class RefundCapacityDao {
    private final NamedParameterJdbcTemplate jdbc;

    public Optional<RefundCapacityAfterAllocation> tryAllocate(
            RefundCommand command
    ) {
        List<RefundCapacityAfterAllocation> rows = jdbc.query("""
                UPDATE payment_charge
                SET allocated_refund_amount =
                        allocated_refund_amount + :amount,
                    revision = revision + 1,
                    updated_at = CURRENT_TIMESTAMP
                WHERE charge_id = :chargeId
                  AND merchant_id = :merchantId
                  AND currency = :currency
                  AND allocated_refund_amount + :amount <= captured_amount
                RETURNING captured_amount,
                          allocated_refund_amount,
                          completed_refund_amount,
                          revision
                """, new MapSqlParameterSource()
                .addValue("chargeId", command.chargeId())
                .addValue("merchantId", command.merchantId())
                .addValue("currency", command.currency())
                .addValue("amount", command.amount()),
                (rs, rowNum) -> new RefundCapacityAfterAllocation(
                        rs.getLong("captured_amount"),
                        rs.getLong("allocated_refund_amount"),
                        rs.getLong("completed_refund_amount"),
                        rs.getLong("revision")
                ));

        if (rows.size() > 1) {
            throw new IllegalStateException(
                    "single charge update returned multiple rows"
            );
        }
        return rows.stream().findFirst();
    }
}
```

`amount > 0` đã được hợp đồng lệnh kiểm tra và lược đồ tiếp tục phòng thủ. Dịch
vụ chỉ diễn giải tập rỗng là `LIMIT_EXCEEDED` sau khi đã xác nhận giao dịch tồn
tại và loại tiền đúng.

## 7. Ghi refund, bút toán và outbox

```java
@Repository
public class RefundWriteDao {
    private final NamedParameterJdbcTemplate jdbc;

    public void insertPending(
            UUID refundId,
            RefundCommand command,
            Instant now
    ) {
        int changed = jdbc.update("""
                INSERT INTO refund (
                    refund_id,
                    request_id,
                    charge_id,
                    merchant_id,
                    amount,
                    currency,
                    status,
                    created_at,
                    updated_at
                ) VALUES (
                    :refundId,
                    :requestId,
                    :chargeId,
                    :merchantId,
                    :amount,
                    :currency,
                    'PENDING_PROVIDER',
                    :now,
                    :now
                )
                """, new MapSqlParameterSource()
                .addValue("refundId", refundId)
                .addValue("requestId", command.requestId())
                .addValue("chargeId", command.chargeId())
                .addValue("merchantId", command.merchantId())
                .addValue("amount", command.amount())
                .addValue("currency", command.currency())
                .addValue("now", Timestamp.from(now)));
        requireOne(changed);
    }

    public void appendAllocated(
            UUID entryId,
            UUID refundId,
            RefundCommand command,
            long revision,
            Instant now
    ) {
        requireOne(jdbc.update("""
                INSERT INTO refund_ledger_entry (
                    entry_id,
                    charge_id,
                    refund_id,
                    entry_type,
                    allocation_delta,
                    completion_delta,
                    charge_revision,
                    created_at
                ) VALUES (
                    :entryId,
                    :chargeId,
                    :refundId,
                    'REFUND_ALLOCATED',
                    :amount,
                    0,
                    :revision,
                    :now
                )
                """, new MapSqlParameterSource()
                .addValue("entryId", entryId)
                .addValue("chargeId", command.chargeId())
                .addValue("refundId", refundId)
                .addValue("amount", command.amount())
                .addValue("revision", revision)
                .addValue("now", Timestamp.from(now))));
    }

    private static void requireOne(int changed) {
        if (changed != 1) {
            throw new IllegalStateException(
                    "expected one row, got " + changed
            );
        }
    }
}
```

Outbox được chèn bằng JDBC trong cùng giao dịch. Payload chứa `refund_id`,
`charge_id`, số tiền, loại tiền và mã lý do; không chứa dữ liệu bí mật không cần
thiết. Ràng buộc `(aggregate_type, aggregate_id, event_type)` là lớp phòng thủ chống tạo hai lệnh
`REFUND_REQUESTED` cho cùng refund.

## 8. Giao dịch tiếp nhận hoàn tiền

```java
@Service
public class RefundAcceptanceTx {
    private final RefundRequestDao requests;
    private final ChargeIdentityDao chargeIdentities;
    private final RefundCapacityDao capacity;
    private final RefundWriteDao refunds;
    private final RefundOutboxDao outbox;
    private final Clock clock;

    public RefundAcceptanceTx(
            RefundRequestDao requests,
            ChargeIdentityDao chargeIdentities,
            RefundCapacityDao capacity,
            RefundWriteDao refunds,
            RefundOutboxDao outbox,
            Clock clock
    ) {
        this.requests = requests;
        this.chargeIdentities = chargeIdentities;
        this.capacity = capacity;
        this.refunds = refunds;
        this.outbox = outbox;
        this.clock = clock;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public RefundResult accept(RefundCommand command) {
        Instant now = clock.instant();

        if (!requests.tryClaim(command, now)) {
            StoredRefundRequest existing = requests.requireExisting(
                    command.merchantId(),
                    command.idempotencyKey()
            );
            existing.requireSameRequest(command);
            return existing.toResult();
        }

        Optional<ChargeIdentity> identity = chargeIdentities.find(
                command.chargeId(),
                command.merchantId()
        );
        if (identity.isEmpty()) {
            requests.completeRejected(
                    command.requestId(),
                    "CHARGE_NOT_FOUND",
                    now
            );
            return new RefundResult.ChargeNotFound(command.requestId());
        }
        if (!identity.orElseThrow().currency().equals(command.currency())) {
            requests.completeRejected(
                    command.requestId(),
                    "CURRENCY_MISMATCH",
                    now
            );
            return new RefundResult.CurrencyMismatch(command.requestId());
        }

        Optional<RefundCapacityAfterAllocation> changed =
                capacity.tryAllocate(command);
        if (changed.isEmpty()) {
            requests.completeRejected(
                    command.requestId(),
                    "LIMIT_EXCEEDED",
                    now
            );
            return new RefundResult.LimitExceeded(command.requestId());
        }

        RefundCapacityAfterAllocation after = changed.orElseThrow();
        UUID refundId = UUID.randomUUID();
        refunds.insertPending(refundId, command, now);
        refunds.appendAllocated(
                UUID.randomUUID(),
                refundId,
                command,
                after.revision(),
                now
        );
        outbox.insertRefundRequested(
                UUID.randomUUID(),
                refundId,
                command,
                now
        );
        requests.completeAccepted(command.requestId(), refundId, now);

        return new RefundResult.Accepted(command.requestId(), refundId);
    }
}
```

Vì các DAO dùng cùng `DataSource` dưới `@Transactional`, mọi lần ghi tham gia một
giao dịch PostgreSQL. Không bắt ngoại lệ ràng buộc rồi tiếp tục dùng giao dịch đã
bị đánh dấu lỗi.

## 9. Vì sao giao dịch tiếp nhận hoạt động

### Cùng một khóa

Ràng buộc duy nhất cho một bên chiếm khóa. Bên còn lại không chạy cập nhật hạn
mức; nó kiểm tra dấu vân tay và phát lại kết quả bền vững.

### Hai khóa khác nhau

Hai dòng `refund_request` không xung đột, nhưng hai câu cập nhật cùng dòng
`payment_charge` được PostgreSQL tuần tự hóa. Điều kiện hạn mức được kiểm tra lại
trên giá trị mới nhất.

### Lỗi ở giữa

Nếu chèn bút toán hoặc outbox lỗi, giao dịch hoàn tác cả khoản phân bổ và quyền
khóa lũy đẳng. Không tồn tại trạng thái nửa chừng đã chốt.

### Kết quả từ chối

`LIMIT_EXCEEDED` được lưu trong dòng request. Việc gửi lại cùng khóa không đánh
giá hạn mức lần nữa và không bất ngờ đổi kết quả.

## 10. Bộ điều phối ngoài giao dịch

```java
@Component
public class RefundCoordinator {
    private final RefundAcceptanceTx worker;
    private final SqlStateClassifier classifier;

    public RefundCoordinator(
            RefundAcceptanceTx worker,
            SqlStateClassifier classifier
    ) {
        this.worker = worker;
        this.classifier = classifier;
    }

    public RefundResult accept(RefundCommand command) {
        if (TransactionSynchronizationManager
                .isActualTransactionActive()) {
            throw new IllegalStateException(
                    "coordinator must run outside a transaction"
            );
        }

        try {
            return worker.accept(command);
        } catch (RuntimeException failure) {
            return classifier.classify(failure)
                    .map(kind -> new RefundResult.Busy(
                            command.requestId(),
                            kind.name()
                    ))
                    .orElseThrow(() -> failure);
        }
    }
}
```

Nếu chính sách cho phép thử lại lỗi bế tắc hoặc lỗi tuần tự hóa, bộ điều phối tạo
một lần gọi mới vào `worker` với cùng command. Không thử lại bên trong giao dịch
đã thất bại.

## 11. Giới hạn thời gian chờ

Đặt `lock_timeout` và `statement_timeout` theo phạm vi giao dịch bằng
`SET LOCAL` hoặc `set_config(..., true)`. Giá trị phải đến từ cấu hình vận hành,
không chép một con số cứng từ tài liệu.

Phân loại tối thiểu:

| SQLSTATE | Ý nghĩa | Cách xử lý |
| --- | --- | --- |
| `55P03` | Không lấy được khóa trong giới hạn | Hoàn tác, báo bận hoặc thử lại có giới hạn |
| `40P01` | Bế tắc | Giao dịch đã bị hủy; thử lại ngoài giao dịch nếu chính sách cho phép |
| `40001` | Lỗi tuần tự hóa | Thử lại toàn bộ giao dịch nếu dùng mức cô lập phù hợp |
| `23505` | Vi phạm duy nhất ngoài đường đã dự kiến | Hoàn tác và điều tra hoặc phân loại đúng ràng buộc |

Không ánh xạ các lỗi này thành `LIMIT_EXCEEDED`.

## 12. Phát lệnh sang nhà cung cấp

Tiến trình outbox thực hiện sau khi giao dịch tiếp nhận chốt:

```text
đọc REFUND_REQUESTED chưa gửi
→ gọi provider với provider_idempotency_key = refund_id
→ ghi nhận kết quả hoặc trạng thái chưa rõ
→ đánh dấu outbox theo quy tắc giao lại an toàn
```

Nếu nhà cung cấp hỗ trợ khóa lũy đẳng, luôn dùng cùng `refund_id` cho mọi lần gửi
lại. Nếu không hỗ trợ, cần tra cứu theo tham chiếu ổn định trước khi gửi lại; chỉ
dùng outbox không thể tự ngăn nhà cung cấp thực hiện hai lần sau lỗi mạng.

## 13. Hoàn tất khoản hoàn đúng một lần

```java
public record PendingRefund(
        UUID refundId,
        UUID chargeId,
        long amount
) {}

@Repository
public class RefundTransitionDao {
    private final NamedParameterJdbcTemplate jdbc;

    public Optional<PendingRefund> tryMarkSucceeded(
            UUID refundId,
            String providerRefundId,
            Instant now
    ) {
        return jdbc.query("""
                UPDATE refund
                SET status = 'SUCCEEDED',
                    provider_refund_id = :providerRefundId,
                    completed_at = :now,
                    updated_at = :now
                WHERE refund_id = :refundId
                  AND status = 'PENDING_PROVIDER'
                RETURNING refund_id, charge_id, amount
                """, Map.of(
                "refundId", refundId,
                "providerRefundId", providerRefundId,
                "now", Timestamp.from(now)
        ), (rs, rowNum) -> new PendingRefund(
                rs.getObject("refund_id", UUID.class),
                rs.getObject("charge_id", UUID.class),
                rs.getLong("amount")
        )).stream().findFirst();
    }
}
```

Trong cùng giao dịch, sau khi chuyển trạng thái trả một dòng:

```sql
UPDATE payment_charge
SET completed_refund_amount = completed_refund_amount + :amount,
    revision = revision + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE charge_id = :chargeId
  AND completed_refund_amount + :amount <= allocated_refund_amount
RETURNING revision;
```

Sau đó thêm `REFUND_SUCCEEDED` với `completion_delta = :amount`. Nếu cập nhật
tổng hợp hoặc chèn bút toán không thành công, ném ngoại lệ để hoàn tác cả chuyển
trạng thái.

Khi `tryMarkSucceeded` trả rỗng, đọc trạng thái hiện có:

- đã `SUCCEEDED` với cùng mã nhà cung cấp: coi là phát lại;
- đã `FAILED_RELEASED`: không tự đổi ngược; chuyển sang quy trình xử lý thứ tự
  thông báo của BANK-006;
- không tồn tại: lỗi dữ liệu hoặc callback không hợp lệ.

## 14. Giải phóng khoản bị từ chối đúng một lần

Giao dịch giải phóng bắt đầu bằng:

```sql
UPDATE refund
SET status = 'FAILED_RELEASED',
    completed_at = :now,
    updated_at = :now
WHERE refund_id = :refundId
  AND status = 'PENDING_PROVIDER'
RETURNING charge_id, amount;
```

Nếu nhận một dòng, tiếp tục:

```sql
UPDATE payment_charge
SET allocated_refund_amount = allocated_refund_amount - :amount,
    revision = revision + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE charge_id = :chargeId
  AND allocated_refund_amount - :amount >= completed_refund_amount
RETURNING revision;
```

Cuối cùng thêm bút toán:

```sql
INSERT INTO refund_ledger_entry (
    entry_id,
    charge_id,
    refund_id,
    entry_type,
    allocation_delta,
    completion_delta,
    charge_revision,
    created_at
) VALUES (
    :entryId,
    :chargeId,
    :refundId,
    'REFUND_RELEASED',
    -:amount,
    0,
    :revision,
    :now
);
```

Hai thông báo lỗi trùng cùng tranh dòng `refund`; chỉ một bên chuyển được trạng
thái. Không xóa refund hoặc bút toán phân bổ cũ.

Chỉ giải phóng khi nhà cung cấp đã từ chối dứt khoát hoặc quy trình tra cứu xác
nhận chưa hoàn tiền. Kết quả mạng chưa rõ phải giữ trạng thái chờ.

## 15. Hành vi của bên thua

| Tình huống | Hành vi đúng |
| --- | --- |
| Trùng khóa, cùng nội dung | Phát lại `refund_id` hoặc kết quả từ chối |
| Trùng khóa, khác nội dung | Từ chối `IDEMPOTENCY_KEY_REUSED` |
| Khác khóa nhưng hết hạn mức | Lưu và trả `LIMIT_EXCEEDED` |
| Khác khóa, vẫn đủ hạn mức | Được chấp nhận sau khi chờ khóa |
| Callback trùng cùng kết quả | Phát lại trạng thái, không cập nhật số tiền |
| Callback xung đột trạng thái | Không đổi ngược; chuyển sang xử lý theo máy trạng thái nhà cung cấp |
| Hết thời gian chờ khóa | Hoàn tác và trả lỗi kỹ thuật |

## 16. Phương án thay thế: `SELECT ... FOR UPDATE`

Có thể khóa dòng charge, đọc số tiền rồi quyết định:

```sql
SELECT captured_amount, allocated_refund_amount
FROM payment_charge
WHERE charge_id = :chargeId
FOR UPDATE;
```

Cách này phù hợp khi quyết định cần nhiều quy tắc không thể diễn đạt rõ trong một
câu cập nhật. Đổi lại, nó cần thêm lượt trao đổi và dễ giữ khóa lâu nếu Java làm
nhiều việc. Vẫn phải dùng khóa lũy đẳng, ledger, outbox và giao dịch ngắn.

## 17. Phương án thay thế: JPA `@Version`

Khóa lạc quan phát hiện hai lần ghi dựa trên cùng phiên bản. Nó phù hợp khi xung
đột hiếm và ứng dụng có chính sách thử lại tốt. Mỗi lần thử lại phải đọc lại,
kiểm tra lại hạn mức và chạy trong giao dịch mới với cùng khóa lũy đẳng.

Không gọi nhà cung cấp trong lần giao dịch có thể bị thử lại. Khi nhiều yêu cầu
cùng tranh một giao dịch nóng, số xung đột và công việc bị bỏ có thể tăng.

## 18. Các lựa chọn không thay thế bất biến

### `SERIALIZABLE`

Mức cô lập này có thể phát hiện một số chuỗi thực thi không tuần tự hóa được,
nhưng ứng dụng phải thử lại lỗi `40001`. Ràng buộc duy nhất và các ràng buộc số
tiền vẫn cần để bảo vệ dữ liệu và diễn đạt ý định.

### Khóa phân tán

Khóa phân tán thêm vòng đời thuê khóa, tạm dừng tiến trình và chủ cũ tiếp tục
chạy. Nó không thay thế điều kiện trong PostgreSQL, nhất là khi dữ liệu có thẩm
quyền đã nằm tại PostgreSQL.

### Hàng đợi theo `charge_id`

Hàng đợi có thể tạo áp lực ngược và giảm tranh chấp. Thông điệp vẫn có thể được
giao lại, consumer có thể chạy song song khi cấu hình sai, và tác vụ vận hành có
thể ghi trực tiếp. Giữ các lớp phòng thủ cơ sở dữ liệu.

### `synchronized`

Chỉ bảo vệ một JVM và không tồn tại sau khởi động lại. Nó không phải ranh giới có
thẩm quyền cho hệ thống nhiều máy chủ.

## 19. So sánh định tính

| Cách | Điểm mạnh | Điểm cần cân nhắc |
| --- | --- | --- |
| `UPDATE` có điều kiện | Một điểm quyết định rõ, làm việc trên giá trị mới nhất | Điều kiện phải diễn đạt được trong SQL; cần phân loại kết quả rỗng |
| `FOR UPDATE` | Dễ đọc nhiều dữ liệu rồi quyết định | Thêm lượt đọc và nguy cơ giữ khóa lâu |
| `@Version` | Tích hợp tự nhiên với JPA | Cần thử lại toàn bộ và có thể lãng phí công việc khi tranh chấp cao |
| `SERIALIZABLE` | Bảo vệ rộng cho giao dịch phức tạp | Có lỗi tuần tự hóa cần thử lại; không thay ràng buộc miền |
| Hàng đợi | Giảm tải và tạo thứ tự vận hành | Không tự chống giao lại hoặc đường ghi ngoài hàng đợi |

## 20. Đối soát định kỳ

```sql
SELECT c.charge_id,
       c.allocated_refund_amount,
       COALESCE(SUM(e.allocation_delta), 0) AS ledger_allocated,
       c.completed_refund_amount,
       COALESCE(SUM(e.completion_delta), 0) AS ledger_completed
FROM payment_charge c
LEFT JOIN refund_ledger_entry e ON e.charge_id = c.charge_id
GROUP BY c.charge_id,
         c.allocated_refund_amount,
         c.completed_refund_amount
HAVING c.allocated_refund_amount <> COALESCE(SUM(e.allocation_delta), 0)
    OR c.completed_refund_amount <> COALESCE(SUM(e.completion_delta), 0);
```

Thêm kiểm tra:

```sql
SELECT r.refund_id
FROM refund r
LEFT JOIN refund_ledger_entry a
       ON a.refund_id = r.refund_id
      AND a.entry_type = 'REFUND_ALLOCATED'
LEFT JOIN outbox_event o
       ON o.aggregate_id = r.refund_id
      AND o.event_type = 'REFUND_REQUESTED'
WHERE a.entry_id IS NULL
   OR o.event_id IS NULL;
```

Các truy vấn phải không trả dòng. Khi có chênh lệch, giữ bằng chứng, xác định
đường ghi gây lỗi và sửa bằng quy trình có kiểm soát; không sửa số tổng hợp âm
thầm.

## 21. Danh sách kiểm tra triển khai

- [ ] Số tiền dùng số nguyên theo đơn vị nhỏ nhất; không dùng `double`.
- [ ] Khóa lũy đẳng có phạm vi cửa hàng và ràng buộc duy nhất.
- [ ] Dấu vân tay bao gồm mọi trường làm thay đổi ý nghĩa lệnh.
- [ ] Cùng khóa, cùng nội dung phát lại cả kết quả thành công lẫn từ chối.
- [ ] Cùng khóa, khác nội dung bị từ chối rõ ràng.
- [ ] Hạn mức được tăng bằng một `UPDATE` chênh lệch có điều kiện.
- [ ] Khoản đang chờ cũng nằm trong `allocated_refund_amount`.
- [ ] Refund, projection, ledger, outbox và kết quả request cùng giao dịch.
- [ ] Không gọi nhà cung cấp trong giao dịch giữ khóa charge.
- [ ] Outbox dùng `refund_id` ổn định khi gửi lại.
- [ ] Hoàn tất và giải phóng dùng chuyển trạng thái có điều kiện.
- [ ] Giải phóng tạo bút toán bù, không xóa lịch sử.
- [ ] Lỗi khóa và bế tắc không bị đổi thành lỗi hạn mức.
- [ ] Thử lại dùng giao dịch mới nhưng giữ nguyên khóa lũy đẳng.
- [ ] Có giới hạn chờ khóa từ cấu hình vận hành.
- [ ] Có đối soát projection với ledger và cảnh báo khoản chờ quá lâu.
- [ ] Test chạy trên PostgreSQL thật bằng Testcontainers.
