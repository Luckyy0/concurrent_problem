# Thiết kế bảo vệ giới hạn coupon/voucher

## 1. Mục tiêu thiết kế

Giải pháp chính bảo vệ đồng thời ba quy tắc:

```text
một chương trình + một checkout → tối đa một lần sử dụng
một chương trình             → không vượt giới hạn toàn cục
một chương trình + một khách → không vượt giới hạn mỗi khách hàng
```

Mỗi quy tắc có một cơ chế ở PostgreSQL, nhưng mọi thay đổi cùng nằm trong một
giao dịch. Nếu bất kỳ giới hạn nào không còn thỏa, không bộ đếm hay lịch sử nào
được chốt một phần.

## 2. Lược đồ dữ liệu

```sql
CREATE TABLE promotion (
    promotion_id UUID PRIMARY KEY,
    code VARCHAR(40) NOT NULL,
    status VARCHAR(20) NOT NULL,
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ NOT NULL,
    global_limit BIGINT,
    per_user_limit INTEGER,
    redeemed_count BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT uk_promotion_code UNIQUE (code),
    CONSTRAINT ck_promotion_status
        CHECK (status IN ('DRAFT', 'ACTIVE', 'PAUSED', 'ENDED')),
    CONSTRAINT ck_promotion_window
        CHECK (starts_at < ends_at),
    CONSTRAINT ck_promotion_global_limit
        CHECK (global_limit IS NULL OR global_limit > 0),
    CONSTRAINT ck_promotion_per_user_limit
        CHECK (per_user_limit IS NULL OR per_user_limit > 0),
    CONSTRAINT ck_promotion_redeemed_count
        CHECK (
            redeemed_count >= 0
            AND (
                global_limit IS NULL
                OR redeemed_count <= global_limit
            )
        )
);

CREATE TABLE promotion_user_usage (
    promotion_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    used_count INTEGER NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT pk_promotion_user_usage
        PRIMARY KEY (promotion_id, customer_id),
    CONSTRAINT fk_usage_promotion
        FOREIGN KEY (promotion_id)
        REFERENCES promotion(promotion_id),
    CONSTRAINT ck_promotion_user_used_count
        CHECK (used_count > 0)
);

CREATE TABLE promotion_redemption (
    redemption_id UUID PRIMARY KEY,
    promotion_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    checkout_id UUID NOT NULL,
    request_fingerprint CHAR(64) NOT NULL,
    status VARCHAR(16) NOT NULL,
    discount_amount NUMERIC(19, 2),
    currency CHAR(3),
    created_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,

    CONSTRAINT fk_redemption_promotion
        FOREIGN KEY (promotion_id)
        REFERENCES promotion(promotion_id),
    CONSTRAINT uk_redemption_promotion_checkout
        UNIQUE (promotion_id, checkout_id),
    CONSTRAINT ck_redemption_status
        CHECK (
            (
                status = 'PROCESSING'
                AND discount_amount IS NULL
                AND currency IS NULL
                AND completed_at IS NULL
            )
            OR
            (
                status = 'APPLIED'
                AND discount_amount > 0
                AND currency IS NOT NULL
                AND completed_at IS NOT NULL
            )
        )
);

CREATE INDEX ix_redemption_promotion_customer
    ON promotion_redemption (promotion_id, customer_id)
    WHERE status = 'APPLIED';
```

`ck_promotion_redeemed_count` là lớp phòng thủ cuối cùng. Nó không thay thế câu
cập nhật có điều kiện vì hai giao dịch cùng ghi một giá trị tuyệt đối vẫn có thể
vượt số lịch sử mà không làm bộ đếm vượt `global_limit`.

`promotion_user_usage` không lặp lại `per_user_limit`. Giao dịch lấy giới hạn từ
dòng `promotion` đang bị khóa, nên việc quản trị thay đổi giới hạn cũng phải chờ
và không tạo hai nguồn cấu hình khác nhau.

## 3. Lệnh và kết quả

```java
public record RedeemPromotionCommand(
    UUID promotionId,
    UUID customerId,
    UUID checkoutId,
    long cartVersion,
    UUID quoteId
) {
    public RedeemPromotionCommand {
        Objects.requireNonNull(promotionId);
        Objects.requireNonNull(customerId);
        Objects.requireNonNull(checkoutId);
        Objects.requireNonNull(quoteId);
        if (cartVersion < 0) {
            throw new IllegalArgumentException(
                "cartVersion must not be negative"
            );
        }
    }
}

public record RedemptionResult(
    UUID redemptionId,
    BigDecimal discountAmount,
    String currency,
    boolean replayed
) {
}
```

`customerId` phải lấy từ danh tính đã xác thực hoặc checkout có thẩm quyền, không
tin một giá trị tùy ý trong nội dung HTTP. Số tiền đủ điều kiện và mức giảm được
đọc lại từ dữ liệu giá tại máy chủ; phía gọi không tự quyết định số tiền giảm.

## 4. Dấu vân tay của lần sử dụng mã

```java
@Component
public class PromotionFingerprint {

    public String calculate(RedeemPromotionCommand command) {
        String canonical = String.join(
            "\n",
            "schema=v1",
            "promotionId=" + command.promotionId(),
            "customerId=" + command.customerId(),
            "checkoutId=" + command.checkoutId(),
            "cartVersion=" + command.cartVersion(),
            "quoteId=" + command.quoteId()
        );

        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            return HexFormat.of().formatHex(
                digest.digest(canonical.getBytes(StandardCharsets.UTF_8))
            );
        } catch (NoSuchAlgorithmException impossible) {
            throw new IllegalStateException("SHA-256 is unavailable", impossible);
        }
    }
}
```

Dấu vân tay giúp phân biệt lần gửi lại hợp lệ với việc dùng cùng checkout cho
nội dung khác. Nó không bảo vệ giới hạn toàn cục hay giới hạn mỗi người; hai
checkout khác nhau vẫn có hai dấu vân tay hợp lệ và cùng tranh hạn mức.

## 5. Chiếm quyền trên một checkout

```java
@Repository
public class RedemptionStore {

    private final NamedParameterJdbcTemplate jdbc;

    public RedemptionStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<UUID> tryClaim(
        RedeemPromotionCommand command,
        String fingerprint
    ) {
        UUID redemptionId = UUID.randomUUID();
        MapSqlParameterSource parameters = new MapSqlParameterSource()
            .addValue("redemptionId", redemptionId)
            .addValue("promotionId", command.promotionId())
            .addValue("customerId", command.customerId())
            .addValue("checkoutId", command.checkoutId())
            .addValue("fingerprint", fingerprint);

        List<UUID> ids = jdbc.query(
            """
            INSERT INTO promotion_redemption (
                redemption_id,
                promotion_id,
                customer_id,
                checkout_id,
                request_fingerprint,
                status,
                created_at
            )
            VALUES (
                :redemptionId,
                :promotionId,
                :customerId,
                :checkoutId,
                :fingerprint,
                'PROCESSING',
                CURRENT_TIMESTAMP
            )
            ON CONFLICT (promotion_id, checkout_id) DO NOTHING
            RETURNING redemption_id
            """,
            parameters,
            (rs, rowNumber) -> rs.getObject("redemption_id", UUID.class)
        );

        return ids.stream().findFirst();
    }

    public StoredRedemption findExisting(
        UUID promotionId,
        UUID checkoutId
    ) {
        return jdbc.queryForObject(
            """
            SELECT redemption_id,
                   customer_id,
                   request_fingerprint,
                   status,
                   discount_amount,
                   currency
            FROM promotion_redemption
            WHERE promotion_id = :promotionId
              AND checkout_id = :checkoutId
            """,
            Map.of(
                "promotionId", promotionId,
                "checkoutId", checkoutId
            ),
            (rs, rowNumber) -> new StoredRedemption(
                rs.getObject("redemption_id", UUID.class),
                rs.getObject("customer_id", UUID.class),
                rs.getString("request_fingerprint"),
                RedemptionStatus.valueOf(rs.getString("status")),
                rs.getBigDecimal("discount_amount"),
                rs.getString("currency")
            )
        );
    }

    public void complete(
        UUID redemptionId,
        Money discount
    ) {
        int changed = jdbc.update(
            """
            UPDATE promotion_redemption
            SET status = 'APPLIED',
                discount_amount = :amount,
                currency = :currency,
                completed_at = CURRENT_TIMESTAMP
            WHERE redemption_id = :redemptionId
              AND status = 'PROCESSING'
            """,
            new MapSqlParameterSource()
                .addValue("redemptionId", redemptionId)
                .addValue("amount", discount.amount())
                .addValue("currency", discount.currency())
        );

        if (changed != 1) {
            throw new IllegalStateException(
                "Redemption was not completed exactly once"
            );
        }
    }
}
```

Nếu hai giao dịch chèn cùng khóa, bên đến sau có thể chờ bên trước. Ở
`READ COMMITTED`, sau khi `DO NOTHING` trả về rỗng, câu `SELECT` kế tiếp lấy ảnh
chụp mới và thấy bản ghi vừa chốt. Nếu bên trước hoàn tác, bên đang chờ có thể
chèn thành công.

## 6. Tăng giới hạn toàn cục

```java
public record PromotionCapacity(
    Integer perUserLimit,
    long redeemedCount
) {
}
```

```java
@Repository
public class PromotionCounterStore {

    private final NamedParameterJdbcTemplate jdbc;

    public PromotionCounterStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<PromotionCapacity> incrementIfAvailable(
        UUID promotionId
    ) {
        List<PromotionCapacity> rows = jdbc.query(
            """
            UPDATE promotion
            SET redeemed_count = redeemed_count + 1,
                updated_at = CURRENT_TIMESTAMP
            WHERE promotion_id = :promotionId
              AND status = 'ACTIVE'
              AND starts_at <= CURRENT_TIMESTAMP
              AND ends_at > CURRENT_TIMESTAMP
              AND (
                  global_limit IS NULL
                  OR redeemed_count < global_limit
              )
            RETURNING per_user_limit, redeemed_count
            """,
            Map.of("promotionId", promotionId),
            (rs, rowNumber) -> new PromotionCapacity(
                rs.getObject("per_user_limit", Integer.class),
                rs.getLong("redeemed_count")
            )
        );

        return rows.stream().findFirst();
    }
}
```

Có đúng một dòng trả về nghĩa là chương trình tồn tại, đang hoạt động và đã dành
được một lượt toàn cục. Không có dòng trả về chỉ có nghĩa là không thể áp dụng;
nếu cần lý do chi tiết, đọc lại sau khi giao dịch này đã hoàn tác.

## 7. Tăng giới hạn mỗi khách hàng

```java
@Repository
public class PromotionUserUsageStore {

    private final NamedParameterJdbcTemplate jdbc;

    public PromotionUserUsageStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<Integer> incrementIfAvailable(
        UUID promotionId,
        UUID customerId,
        Integer perUserLimit
    ) {
        MapSqlParameterSource parameters = new MapSqlParameterSource()
            .addValue("promotionId", promotionId)
            .addValue("customerId", customerId)
            .addValue("perUserLimit", perUserLimit);

        List<Integer> counts = jdbc.query(
            """
            INSERT INTO promotion_user_usage (
                promotion_id,
                customer_id,
                used_count,
                updated_at
            )
            VALUES (
                :promotionId,
                :customerId,
                1,
                CURRENT_TIMESTAMP
            )
            ON CONFLICT (promotion_id, customer_id) DO UPDATE
            SET used_count = promotion_user_usage.used_count + 1,
                updated_at = EXCLUDED.updated_at
            WHERE :perUserLimit IS NULL
               OR promotion_user_usage.used_count < :perUserLimit
            RETURNING used_count
            """,
            parameters,
            (rs, rowNumber) -> rs.getInt("used_count")
        );

        return counts.stream().findFirst();
    }
}
```

Không tách thành `SELECT count` rồi `INSERT`/`UPDATE`. Câu upsert tự phân xử cả
dòng chưa tồn tại lẫn dòng đã có, và điều kiện được kiểm tra trong lúc PostgreSQL
đang giữ khóa phù hợp.

Nếu `perUserLimit` là `null`, câu lệnh vẫn tăng bộ đếm để phục vụ đối soát nhưng
không từ chối theo khách hàng. Nếu không cần theo dõi trong trường hợp này, có
thể bỏ qua bảng người dùng; phải giữ cách tính đối soát nhất quán với lựa chọn đó.

## 8. Giao dịch áp dụng mã

```java
@Service
public class PromotionRedemptionTransaction {

    private final RedemptionStore redemptions;
    private final PromotionCounterStore promotionCounters;
    private final PromotionUserUsageStore userUsage;
    private final PromotionPricingPolicy pricingPolicy;

    public PromotionRedemptionTransaction(
        RedemptionStore redemptions,
        PromotionCounterStore promotionCounters,
        PromotionUserUsageStore userUsage,
        PromotionPricingPolicy pricingPolicy
    ) {
        this.redemptions = redemptions;
        this.promotionCounters = promotionCounters;
        this.userUsage = userUsage;
        this.pricingPolicy = pricingPolicy;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public RedemptionResult execute(
        RedeemPromotionCommand command,
        String fingerprint
    ) {
        Optional<UUID> claimed = redemptions.tryClaim(
            command,
            fingerprint
        );

        if (claimed.isEmpty()) {
            return replay(command, fingerprint);
        }

        UUID redemptionId = claimed.orElseThrow();

        PromotionCapacity capacity = promotionCounters
            .incrementIfAvailable(command.promotionId())
            .orElseThrow(PromotionUnavailableException::new);

        userUsage.incrementIfAvailable(
            command.promotionId(),
            command.customerId(),
            capacity.perUserLimit()
        ).orElseThrow(PerUserLimitReachedException::new);

        Money discount = pricingPolicy.calculate(command);
        redemptions.complete(redemptionId, discount);

        return new RedemptionResult(
            redemptionId,
            discount.amount(),
            discount.currency(),
            false
        );
    }

    private RedemptionResult replay(
        RedeemPromotionCommand command,
        String fingerprint
    ) {
        StoredRedemption existing = redemptions.findExisting(
            command.promotionId(),
            command.checkoutId()
        );

        if (
            !existing.customerId().equals(command.customerId())
                || !existing.requestFingerprint().equals(fingerprint)
        ) {
            throw new RedemptionKeyReusedException();
        }

        if (existing.status() == RedemptionStatus.PROCESSING) {
            throw new RedemptionStillProcessingException();
        }

        return new RedemptionResult(
            existing.redemptionId(),
            existing.discountAmount(),
            existing.currency(),
            true
        );
    }
}
```

`PromotionUnavailableException` và `PerUserLimitReachedException` là ngoại lệ
thời gian chạy. Chúng phải đi ra khỏi Spring proxy để giao dịch hoàn tác. Bộ điều
khiển hoặc lớp ứng dụng bên ngoài mới chuyển chúng thành phản hồi nghiệp vụ.

`pricingPolicy.calculate` chỉ đọc dữ liệu cùng cơ sở dữ liệu hoặc tính toán trong
bộ nhớ; không gọi dịch vụ từ xa khi dòng `promotion` còn bị khóa.

## 9. Lớp ứng dụng ngoài giao dịch

```java
@Service
public class PromotionApplicationService {

    private final PromotionFingerprint fingerprints;
    private final PromotionRedemptionTransaction transaction;

    public PromotionApplicationService(
        PromotionFingerprint fingerprints,
        PromotionRedemptionTransaction transaction
    ) {
        this.fingerprints = fingerprints;
        this.transaction = transaction;
    }

    public RedemptionResult redeem(RedeemPromotionCommand command) {
        String fingerprint = fingerprints.calculate(command);
        return transaction.execute(command, fingerprint);
    }
}
```

Tách bean giúp lời gọi đi qua Spring proxy. Nếu một phương thức trong cùng lớp tự
gọi phương thức `@Transactional`, proxy có thể bị bỏ qua.

Một `@RestControllerAdvice` có thể ánh xạ:

```text
PromotionUnavailableException   → 409 PROMOTION_UNAVAILABLE
PerUserLimitReachedException     → 409 PER_USER_LIMIT_REACHED
RedemptionKeyReusedException     → 409 REDEMPTION_KEY_REUSED
RedemptionStillProcessingException → 409 hoặc 425 theo hợp đồng
```

Nếu cần phân biệt hết hạn, tạm dừng và hết lượt toàn cục, lớp ngoài giao dịch đọc
lại `promotion`. Kết quả đọc đó chỉ dùng để giải thích, không thay đổi quyết định
đã được câu `UPDATE` bảo vệ.

## 10. Vì sao các bất biến được bảo vệ

### Một checkout chỉ có một lịch sử

Chỉ mục duy nhất phân xử mọi máy chủ. Bên thua không được tăng bất kỳ bộ đếm nào;
nó chỉ đọc và phát lại kết quả của bên thắng.

### Giới hạn toàn cục không bị vượt

Phép kiểm tra và tăng nằm trong một câu `UPDATE`. Bên đến sau chờ khóa dòng, rồi
PostgreSQL kiểm tra lại điều kiện trên bộ đếm đã chốt mới nhất.

### Giới hạn mỗi khách hàng không bị vượt

Khóa chính của `promotion_user_usage` phân xử lần dùng đầu tiên. Các lần tiếp
theo tăng trong `ON CONFLICT DO UPDATE` với điều kiện giới hạn ngay tại thời điểm
ghi.

### Không có thay đổi dở dang

Nếu giới hạn người dùng thất bại sau khi bộ đếm toàn cục đã tăng, ngoại lệ làm
hoàn tác cả phép tăng, bản ghi chiếm quyền và mọi thay đổi khác. Chỉ
`promotion_redemption` ở trạng thái `APPLIED` mới được chốt.

## 11. Hành vi khi lỗi

| Tình huống | Điều xảy ra trong PostgreSQL | Kết quả |
| --- | --- | --- |
| Bên giữ lượt toàn cục chốt | Bên chờ đánh giá lại điều kiện | Một bên có thể nhận `0` dòng |
| Bên giữ lượt toàn cục hoàn tác | Thay đổi bộ đếm biến mất | Bên chờ có thể thành công |
| Người dùng hết lượt | Upsert trả `0` dòng | Toàn giao dịch hoàn tác |
| Lỗi `23505` ngoài xung đột dự kiến | Giao dịch bị lỗi | Phân loại theo đúng tên ràng buộc |
| Hết thời gian chờ khóa | Không được coi là hết lượt | Hoàn tác, thử lại có giới hạn |
| Mất kết nối quanh `COMMIT` | Kết quả chưa rõ | Tra cứu bằng cùng checkout |
| Tiến trình sập trước chốt | PostgreSQL hoàn tác giao dịch | Không để lại `PROCESSING` bền vững |

## 12. Chính sách thử lại

- Không thử lại `GLOBAL_LIMIT_REACHED`, `PER_USER_LIMIT_REACHED` hoặc mã không
  hoạt động.
- Có thể thử lại `55P03`, `40P01`, `40001` và một số lỗi kết nối nếu còn thời
  gian tổng.
- Mỗi lần thử mở giao dịch mới và dùng cùng `checkout_id` cùng dấu vân tay.
- Có giới hạn số lần, khoảng lùi và độ lệch ngẫu nhiên để tránh bão thử lại.
- Không thử lại bên trong giao dịch đang giữ thêm khóa giỏ hàng hoặc đơn hàng.
- Với kết quả chốt chưa rõ, tra cứu trước khi quyết định chạy lại.

## 13. Trường hợp mỗi người chỉ được dùng một lần

Nếu `per_user_limit` luôn bằng `1`, lược đồ có thể đơn giản hơn:

```sql
ALTER TABLE promotion_redemption
ADD CONSTRAINT uk_redemption_promotion_customer
UNIQUE (promotion_id, customer_id);
```

Ràng buộc này mạnh và dễ đối soát hơn bộ đếm. Tuy nhiên, nó không biểu diễn giới
hạn `2`, `3` hoặc chính sách thay đổi theo chương trình. Không tạo nhiều “ô lượt”
bằng dữ liệu ngầm mà không có quy tắc cấp và thu hồi rõ ràng.

## 14. Các phương án thay thế

### Khóa bi quan dòng chương trình và dòng người dùng

`SELECT ... FOR UPDATE` rồi kiểm tra trong Java có thể đúng nếu dòng người dùng
được tạo trước, mọi đường ghi lấy khóa theo cùng thứ tự và giao dịch ngắn. Cách
này linh hoạt cho quy tắc phức tạp nhưng cần nhiều câu SQL và dễ giữ khóa lâu.

### Khóa lạc quan bằng `@Version`

Phát hiện hai giao dịch cùng sửa `promotion`, nhưng bên thua phải chạy lại toàn
bộ quyết định. Phù hợp khi xung đột hiếm; không phù hợp làm lựa chọn mặc định cho
một mã nóng ở cuối chiến dịch. Vẫn cần bảo vệ riêng dòng người dùng chưa tồn tại.

### Mức cô lập `SERIALIZABLE`

Có thể hủy một giao dịch khi chuỗi đọc–đếm–chèn không thể tuần tự hóa. Nó cần
thử lại toàn bộ giao dịch và không thay thế ràng buộc duy nhất rõ ràng. Phù hợp
khi bất biến là một vị từ phức tạp không thể đưa về bộ đếm có thẩm quyền.

### Cấp sẵn các lượt dưới dạng token

Có thể tạo trước các dòng token và cho mỗi lần sử dụng chiếm một token bằng
`FOR UPDATE SKIP LOCKED`. Cách này tránh một bộ đếm duy nhất nhưng tăng số dòng,
chi phí dọn dẹp và quy tắc thu hồi. Giới hạn mỗi khách hàng vẫn cần bảo vệ riêng.

### Khóa phân tán

Không cần thiết khi toàn bộ bất biến nằm trong một PostgreSQL. Hết hạn thuê hoặc
phân vùng mạng có thể cho hai chủ cùng chạy; cơ sở dữ liệu vẫn phải có ràng buộc
cuối cùng.

## 15. So sánh định tính

| Cách | Tính đúng đắn | Tranh chấp | Hành vi bên thua | Độ phức tạp vận hành |
| --- | --- | --- | --- | --- |
| Bộ đếm có điều kiện + ràng buộc duy nhất | Trực tiếp cho ba bất biến | Dòng chương trình có thể nóng | Chờ rồi nhận `0` dòng hoặc phát lại | Trung bình |
| `FOR UPDATE` | Đúng nếu khóa đủ dòng và đúng thứ tự | Giữ khóa qua phần quyết định Java | Chờ, hết thời gian hoặc bế tắc | Trung bình–cao |
| `@Version` | Phát hiện xung đột trên dòng có phiên bản | Nhiều lần thử lại khi mã nóng | Ngoại lệ xung đột | Trung bình |
| `SERIALIZABLE` | Bảo vệ vị từ rộng | Có thể tăng lỗi `40001` | Thử lại toàn giao dịch | Cao hơn |
| Token cấp sẵn | Chính xác nếu vòng đời token đúng | Phân tán trên nhiều dòng | Không lấy được token | Cao |
| Khóa JVM | Không bảo vệ nhiều máy chủ | Chỉ cục bộ | Chờ trong một tiến trình | Không đáp ứng bất biến |

## 16. Chuyển đổi dữ liệu đang chạy

Trước khi thêm ràng buộc và tin bộ đếm:

1. tìm checkout có nhiều lịch sử;
2. tìm khách hàng vượt giới hạn;
3. so sánh `redeemed_count` với lịch sử `APPLIED`;
4. chọn lịch sử chuẩn và xử lý các đơn đã hưởng ưu đãi theo chính sách nghiệp vụ;
5. sửa bộ đếm từ dữ liệu lịch sử có thẩm quyền;
6. tạo ràng buộc duy nhất và `CHECK`;
7. triển khai câu ghi mới;
8. tiếp tục đối soát sau khi phát hành.

Không tự động xóa ưu đãi trên đơn đã thanh toán chỉ để làm sạch dữ liệu. Đây có
thể là thay đổi tài chính cần kiểm toán và quyết định nghiệp vụ.

## 17. Danh sách kiểm tra triển khai

- [ ] Migration có đúng tên các ràng buộc trong tài liệu.
- [ ] Cùng checkout được chiếm quyền bằng một câu `INSERT` nguyên tử.
- [ ] Giới hạn toàn cục nằm trong mệnh đề `WHERE` của phép tăng.
- [ ] Dòng người dùng chưa tồn tại được xử lý bằng upsert nguyên tử.
- [ ] Mọi đường ghi lấy khóa theo cùng thứ tự.
- [ ] Ngoại lệ giới hạn đi ra khỏi Spring proxy để hoàn tác giao dịch.
- [ ] Không có thực thể JPA cũ ghi đè sau câu SQL trực tiếp.
- [ ] Không gọi mạng khi đang giữ khóa bộ đếm.
- [ ] Lần thử lại dùng cùng checkout và dấu vân tay.
- [ ] Lỗi kỹ thuật và từ chối nghiệp vụ được ánh xạ khác nhau.
- [ ] Có truy vấn đối soát bộ đếm với lịch sử.
- [ ] Thời gian chờ và mức tranh chấp trên dòng chương trình được theo dõi.
