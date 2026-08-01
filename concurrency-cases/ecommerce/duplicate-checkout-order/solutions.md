# Thiết kế checkout lũy đẳng bằng PostgreSQL

## 1. Mục tiêu thiết kế

Giải pháp chính phải đồng thời bảo đảm:

1. PostgreSQL chỉ cấp một quyền xử lý cho mỗi `(customer_id, idempotency_key)`.
2. Một quyền xử lý chỉ tạo tối đa một đơn hàng.
3. Kết quả checkout và lệnh gửi sang thanh toán cùng chốt với đơn.
4. Lần gửi lại cùng nội dung nhận đúng phản hồi đã lưu.
5. Dùng lại khóa với nội dung khác bị từ chối.
6. Lỗi kỹ thuật trước `COMMIT` không để lại quyền xử lý mồ côi.
7. Không có lời gọi mạng trong giao dịch cơ sở dữ liệu.

## 2. Lược đồ dữ liệu

Migration nên đặt tên rõ cho mọi ràng buộc được dùng để phân loại lỗi:

```sql
CREATE TABLE checkout_request (
    request_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    idempotency_key VARCHAR(80) NOT NULL,
    request_fingerprint CHAR(64) NOT NULL,
    status VARCHAR(16) NOT NULL,
    outcome VARCHAR(16),
    response_status SMALLINT,
    response_body JSONB,
    created_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,

    CONSTRAINT uk_checkout_request_customer_key
        UNIQUE (customer_id, idempotency_key),

    CONSTRAINT ck_checkout_request_status
        CHECK (
            (
                status = 'PROCESSING'
                AND outcome IS NULL
                AND response_status IS NULL
                AND response_body IS NULL
                AND completed_at IS NULL
            )
            OR
            (
                status = 'COMPLETED'
                AND outcome IN ('SUCCEEDED', 'REJECTED')
                AND response_status IS NOT NULL
                AND response_body IS NOT NULL
                AND completed_at IS NOT NULL
            )
        )
);

CREATE TABLE purchase_order (
    order_id UUID PRIMARY KEY,
    checkout_request_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    cart_id UUID NOT NULL,
    total_amount NUMERIC(19, 2) NOT NULL,
    currency CHAR(3) NOT NULL,
    status VARCHAR(30) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT fk_order_checkout_request
        FOREIGN KEY (checkout_request_id)
        REFERENCES checkout_request(request_id),

    CONSTRAINT uk_order_checkout_request
        UNIQUE (checkout_request_id),

    CONSTRAINT ck_order_total_positive
        CHECK (total_amount > 0)
);

CREATE TABLE outbox_event (
    event_id UUID PRIMARY KEY,
    aggregate_type VARCHAR(40) NOT NULL,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(80) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    published_at TIMESTAMPTZ,

    CONSTRAINT uk_outbox_aggregate_event
        UNIQUE (aggregate_type, aggregate_id, event_type)
);

CREATE INDEX ix_outbox_unpublished
    ON outbox_event (created_at)
    WHERE published_at IS NULL;
```

Hai lớp duy nhất có vai trò khác nhau:

- `uk_checkout_request_customer_key` phân xử một mã checkout;
- `uk_order_checkout_request` bảo đảm một quyền xử lý không tạo hai đơn, kể cả
  khi mã Java bị gọi lại sai cách.

Ràng buộc outbox tạo tối đa một lệnh thanh toán logic cho một đơn. Việc giao
thông điệp vẫn có thể lặp lại và phía nhận phải chống lặp riêng.

Không dựa vào `@UniqueConstraint` để tạo lược đồ trên môi trường thật. Flyway
hoặc Liquibase phải triển khai DDL, và quy trình khởi động phải kiểm tra lược đồ
không bị lệch.

## 3. Hợp đồng đầu vào

```java
public record CheckoutCommand(
    UUID cartId,
    long cartVersion,
    UUID quoteId,
    UUID shippingAddressId
) {
}

public record CheckoutResult(
    int httpStatus,
    JsonNode responseBody,
    boolean replayed
) {
}
```

Giá tiền không lấy trực tiếp từ phía gọi. Dịch vụ đọc giỏ hàng và báo giá có
thẩm quyền dựa trên các mã trong lệnh. `customerId` lấy từ danh tính đã xác thực,
không lấy từ trường tùy ý trong JSON.

Bộ điều khiển yêu cầu khóa và kiểm tra định dạng trước khi mở giao dịch:

```java
@RestController
@RequestMapping("/v1/checkouts")
public class CheckoutController {

    private final CheckoutApplicationService checkoutService;

    public CheckoutController(CheckoutApplicationService checkoutService) {
        this.checkoutService = checkoutService;
    }

    @PostMapping
    public ResponseEntity<JsonNode> checkout(
        @AuthenticationPrincipal CustomerPrincipal principal,
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody @Valid CheckoutCommand command
    ) {
        IdempotencyKey key = IdempotencyKey.parse(idempotencyKey);
        CheckoutResult result = checkoutService.checkout(
            principal.customerId(),
            key,
            command
        );

        return ResponseEntity
            .status(result.httpStatus())
            .header(
                "Idempotent-Replayed",
                Boolean.toString(result.replayed())
            )
            .body(result.responseBody());
    }
}
```

`IdempotencyKey.parse` nên giới hạn độ dài và tập ký tự. Khóa được coi là dữ liệu
không tin cậy; không đưa nguyên văn vào log hoặc tên metric.

## 4. Tính dấu vân tay ổn định

Ví dụ sau chỉ băm các trường có biểu diễn chuẩn rõ ràng:

```java
@Component
public class CheckoutFingerprint {

    public String calculate(
        UUID customerId,
        CheckoutCommand command
    ) {
        String canonical = String.join(
            "\n",
            "schema=v1",
            "customerId=" + customerId,
            "cartId=" + command.cartId(),
            "cartVersion=" + command.cartVersion(),
            "quoteId=" + command.quoteId(),
            "shippingAddressId=" + command.shippingAddressId()
        );

        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] bytes = digest.digest(
                canonical.getBytes(StandardCharsets.UTF_8)
            );
            return HexFormat.of().formatHex(bytes);
        } catch (NoSuchAlgorithmException impossible) {
            throw new IllegalStateException("SHA-256 is unavailable", impossible);
        }
    }
}
```

Khi thêm trường có ảnh hưởng tới ý nghĩa checkout, phải tăng phiên bản
`schema=v1` và có quy tắc tương thích với bản ghi cũ. Nếu băm JSON, cần một bộ
chuẩn hóa được kiểm thử về thứ tự thuộc tính, `null`, số thập phân và Unicode.

## 5. Kho dữ liệu dùng để chiếm quyền

JDBC phù hợp cho câu lệnh cần biết chính xác có dòng nào được chèn:

```java
@Repository
public class CheckoutRequestStore {

    private final JdbcTemplate jdbc;
    private final ObjectMapper objectMapper;

    public CheckoutRequestStore(
        JdbcTemplate jdbc,
        ObjectMapper objectMapper
    ) {
        this.jdbc = jdbc;
        this.objectMapper = objectMapper;
    }

    public Optional<UUID> tryClaim(
        UUID customerId,
        String key,
        String fingerprint,
        Instant now
    ) {
        UUID requestId = UUID.randomUUID();

        List<UUID> ids = jdbc.query(
            """
            INSERT INTO checkout_request (
                request_id,
                customer_id,
                idempotency_key,
                request_fingerprint,
                status,
                created_at
            )
            VALUES (?, ?, ?, ?, 'PROCESSING', ?)
            ON CONFLICT (customer_id, idempotency_key) DO NOTHING
            RETURNING request_id
            """,
            (rs, rowNumber) -> rs.getObject("request_id", UUID.class),
            requestId,
            customerId,
            key,
            fingerprint,
            Timestamp.from(now)
        );

        return ids.stream().findFirst();
    }

    public StoredCheckout find(
        UUID customerId,
        String key
    ) {
        return jdbc.queryForObject(
            """
            SELECT request_id,
                   request_fingerprint,
                   status,
                   response_status,
                   response_body
            FROM checkout_request
            WHERE customer_id = ?
              AND idempotency_key = ?
            """,
            (rs, rowNumber) -> new StoredCheckout(
                rs.getObject("request_id", UUID.class),
                rs.getString("request_fingerprint"),
                CheckoutRequestStatus.valueOf(rs.getString("status")),
                rs.getObject("response_status", Integer.class),
                readJson(rs.getString("response_body"))
            ),
            customerId,
            key
        );
    }

    public void complete(
        UUID requestId,
        CheckoutOutcome outcome,
        int responseStatus,
        JsonNode responseBody,
        Instant now
    ) {
        int changed = jdbc.update(
            """
            UPDATE checkout_request
            SET status = 'COMPLETED',
                outcome = ?,
                response_status = ?,
                response_body = ?::jsonb,
                completed_at = ?
            WHERE request_id = ?
              AND status = 'PROCESSING'
            """,
            outcome.name(),
            responseStatus,
            responseBody.toString(),
            Timestamp.from(now),
            requestId
        );

        if (changed != 1) {
            throw new IllegalStateException(
                "Checkout request was not completed exactly once"
            );
        }
    }

    private JsonNode readJson(String json) throws SQLException {
        if (json == null) {
            return null;
        }
        try {
            return objectMapper.readTree(json);
        } catch (JsonProcessingException error) {
            throw new SQLException("Invalid stored checkout response", error);
        }
    }
}
```

`find` chạy sau một câu `INSERT ... DO NOTHING` trong giao dịch
`READ COMMITTED`. Nếu bên kia vừa chốt, câu `SELECT` mới lấy ảnh chụp mới và đọc
được bản ghi đó. Trường hợp không tìm thấy phải được xem là bất thường kỹ thuật,
không được tự tạo một kết quả giả.

## 6. Ghi đơn và outbox

```java
@Repository
public class OrderWriter {

    private final JdbcTemplate jdbc;

    public OrderWriter(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public UUID insert(
        UUID requestId,
        UUID customerId,
        PreparedCheckout checkout,
        Instant now
    ) {
        UUID orderId = UUID.randomUUID();

        jdbc.update(
            """
            INSERT INTO purchase_order (
                order_id,
                checkout_request_id,
                customer_id,
                cart_id,
                total_amount,
                currency,
                status,
                created_at
            )
            VALUES (?, ?, ?, ?, ?, ?, 'PENDING_PAYMENT', ?)
            """,
            orderId,
            requestId,
            customerId,
            checkout.cartId(),
            checkout.totalAmount(),
            checkout.currency(),
            Timestamp.from(now)
        );

        return orderId;
    }
}
```

```java
@Repository
public class PaymentOutbox {

    private final JdbcTemplate jdbc;
    private final ObjectMapper objectMapper;

    public PaymentOutbox(JdbcTemplate jdbc, ObjectMapper objectMapper) {
        this.jdbc = jdbc;
        this.objectMapper = objectMapper;
    }

    public void append(
        UUID orderId,
        PreparedCheckout checkout,
        Instant now
    ) {
        ObjectNode payload = objectMapper.createObjectNode()
            .put("orderId", orderId.toString())
            .put("amount", checkout.totalAmount().toPlainString())
            .put("currency", checkout.currency());

        jdbc.update(
            """
            INSERT INTO outbox_event (
                event_id,
                aggregate_type,
                aggregate_id,
                event_type,
                payload,
                created_at
            )
            VALUES (?, 'ORDER', ?, 'PAYMENT_REQUESTED', ?::jsonb, ?)
            """,
            UUID.randomUUID(),
            orderId,
            payload.toString(),
            Timestamp.from(now)
        );
    }
}
```

Mã lũy đẳng gửi sang dịch vụ thanh toán phải ổn định, chẳng hạn `orderId`. Không
tạo mã mới theo từng lần bộ phát outbox gửi lại.

## 7. Giao dịch checkout

```java
@Service
public class CheckoutTransaction {

    private final CheckoutRequestStore requests;
    private final CheckoutPolicy checkoutPolicy;
    private final OrderWriter orders;
    private final PaymentOutbox paymentOutbox;
    private final ObjectMapper objectMapper;
    private final Clock clock;

    public CheckoutTransaction(
        CheckoutRequestStore requests,
        CheckoutPolicy checkoutPolicy,
        OrderWriter orders,
        PaymentOutbox paymentOutbox,
        ObjectMapper objectMapper,
        Clock clock
    ) {
        this.requests = requests;
        this.checkoutPolicy = checkoutPolicy;
        this.orders = orders;
        this.paymentOutbox = paymentOutbox;
        this.objectMapper = objectMapper;
        this.clock = clock;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public CheckoutResult execute(
        UUID customerId,
        IdempotencyKey key,
        String fingerprint,
        CheckoutCommand command
    ) {
        Instant now = clock.instant();
        Optional<UUID> claimed = requests.tryClaim(
            customerId,
            key.value(),
            fingerprint,
            now
        );

        if (claimed.isEmpty()) {
            return replayExisting(customerId, key, fingerprint);
        }

        UUID requestId = claimed.orElseThrow();
        PreparedCheckout prepared = checkoutPolicy.prepare(
            customerId,
            command
        );

        UUID orderId = orders.insert(
            requestId,
            customerId,
            prepared,
            now
        );
        paymentOutbox.append(orderId, prepared, now);

        ObjectNode response = objectMapper.createObjectNode()
            .put("orderId", orderId.toString())
            .put("status", "PENDING_PAYMENT");

        requests.complete(
            requestId,
            CheckoutOutcome.SUCCEEDED,
            201,
            response,
            clock.instant()
        );

        return new CheckoutResult(201, response, false);
    }

    private CheckoutResult replayExisting(
        UUID customerId,
        IdempotencyKey key,
        String fingerprint
    ) {
        StoredCheckout existing = requests.find(
            customerId,
            key.value()
        );

        if (!existing.requestFingerprint().equals(fingerprint)) {
            throw new IdempotencyKeyReusedException();
        }

        if (existing.status() == CheckoutRequestStatus.PROCESSING) {
            throw new CheckoutStillProcessingException();
        }

        return new CheckoutResult(
            existing.responseStatus(),
            existing.responseBody(),
            true
        );
    }
}
```

`CheckoutPolicy.prepare` chỉ đọc hoặc ghi dữ liệu trong PostgreSQL của cùng ranh
giới nghiệp vụ; nó không gọi dịch vụ từ xa. Nếu bước này giữ tồn kho, nó phải
dùng giải pháp có điều kiện của ECOM-001 trong chính giao dịch này.

Một kết quả từ chối cuối cùng có thể được lưu bằng cách gọi `complete` với
`CheckoutOutcome.REJECTED`, mã HTTP phù hợp và nội dung lỗi ổn định. Lỗi kỹ thuật
thì phải ném ra ngoài để Spring hoàn tác toàn bộ giao dịch.

## 8. Lớp ứng dụng và Spring proxy

Việc tính dấu vân tay không cần giữ giao dịch:

```java
@Service
public class CheckoutApplicationService {

    private final CheckoutFingerprint fingerprints;
    private final CheckoutTransaction transaction;

    public CheckoutApplicationService(
        CheckoutFingerprint fingerprints,
        CheckoutTransaction transaction
    ) {
        this.fingerprints = fingerprints;
        this.transaction = transaction;
    }

    public CheckoutResult checkout(
        UUID customerId,
        IdempotencyKey key,
        CheckoutCommand command
    ) {
        String fingerprint = fingerprints.calculate(customerId, command);
        return transaction.execute(
            customerId,
            key,
            fingerprint,
            command
        );
    }
}
```

`CheckoutTransaction` là bean riêng để lời gọi đi qua Spring proxy. Đặt hai
phương thức trong cùng lớp rồi gọi nội bộ có thể bỏ qua proxy và làm chú thích
`@Transactional` không có tác dụng như mong đợi.

## 9. Vì sao quy tắc bất biến được bảo vệ

### Chỉ một yêu cầu thắng

Chỉ mục duy nhất phân xử mọi kết nối tới PostgreSQL. `tryClaim` trả một
`request_id` cho tối đa một giao dịch.

### Chỉ bên thắng tạo đơn

Nhánh tạo đơn chỉ chạy khi `claimed` có giá trị. Ràng buộc
`uk_order_checkout_request` là lớp phòng thủ cuối cùng nếu mã ứng dụng gọi nhầm
hai lần.

### Dữ liệu cùng chốt hoặc cùng hoàn tác

Quyền xử lý, đơn, outbox và phản hồi nằm trong cùng giao dịch. Bất kỳ lỗi không
được xử lý nào cũng làm Spring hoàn tác tất cả thay đổi.

### Bên thua có kết quả rõ ràng

Bên thua chờ hoặc nhận kết quả rỗng từ `DO NOTHING`, sau đó:

- cùng dấu vân tay và `COMPLETED`: phát lại phản hồi;
- khác dấu vân tay: từ chối;
- `PROCESSING` đã chốt: báo đang xử lý theo hợp đồng;
- không đọc được bản ghi: báo lỗi kỹ thuật, không tự tạo đơn.

## 10. Ánh xạ lỗi và chính sách thử lại

| Tín hiệu | Ý nghĩa | Xử lý |
| --- | --- | --- |
| `IdempotencyKeyReusedException` | Cùng khóa, khác nội dung | `409`; không thử lại với khóa đó |
| `CheckoutStillProcessingException` | Quy trình nhiều giao dịch chưa hoàn tất | `409` hoặc `425`; có thể gửi lại cùng khóa sau |
| Không có dòng từ `DO NOTHING` | Đã có bên thắng | Đọc và phát lại; không phải lỗi hệ thống |
| SQLSTATE `55P03` | Hết thời gian chờ khóa | Hoàn tác; thử lại có giới hạn bằng cùng khóa |
| SQLSTATE `40P01` | PostgreSQL chọn giao dịch này làm nạn nhân bế tắc | Giao dịch mới, tải lại dữ liệu và thử lại có giới hạn |
| SQLSTATE `40001` | Lỗi tuần tự hóa nếu dùng mức cô lập cao hơn | Thử lại toàn bộ giao dịch mới |
| Mất kết nối quanh `COMMIT` | Chưa biết kết quả | Tra cứu lại bằng cùng khóa |

Không ánh xạ mọi `DataIntegrityViolationException` thành “checkout trùng”. Vi
phạm `CHECK`, khóa ngoại hoặc ràng buộc outbox có nguyên nhân khác và phải được
phân loại theo SQLSTATE cùng tên ràng buộc.

## 11. Lưu phản hồi và thời hạn giữ khóa

Nội dung phát lại nên nhỏ, ổn định và không chứa bí mật. Có hai lựa chọn:

- lưu nguyên mã HTTP và nội dung phản hồi đã lọc;
- lưu mã tài nguyên cùng phiên bản phản hồi rồi dựng lại theo hợp đồng ổn định.

Lựa chọn thứ nhất cho kết quả phát lại chính xác hơn nhưng tốn chỗ và đòi hỏi
quy tắc bảo vệ dữ liệu. Lựa chọn thứ hai gọn hơn nhưng tài nguyên có thể đổi trạng
thái, khiến phản hồi không còn giống lần đầu.

Không xóa `checkout_request` trước thời hạn mà phía gọi còn được phép thử lại.
Nếu xóa quá sớm, một lần gửi lại muộn sẽ được coi là ý định mới. Việc dọn dẹp cần
dựa trên chính sách lưu trữ đã công bố, trạng thái cuối cùng và yêu cầu kiểm toán.

## 12. Các phương án thay thế

### Chỉ dùng `UNIQUE` trên đơn hàng và bắt `23505`

Phù hợp khi chỉ cần tối đa một đơn cho một mã nghiệp vụ và có thể đọc lại đơn.
Lần thử chèn nên chạy trong giao dịch riêng hoặc được ép `flush`; khối bắt lỗi và
đọc lại phải ở ngoài giao dịch đã bị PostgreSQL đánh dấu lỗi. Cách này khó lưu
phản hồi từ chối và trạng thái đang xử lý hơn bảng riêng.

### Chốt quyền xử lý ở giao dịch riêng

Giảm thời gian giữ tranh chấp trên chỉ mục nếu nghiệp vụ kéo dài, nhưng tạo ra
trạng thái `PROCESSING` bền vững sau sự cố. Chỉ chọn khi đã thiết kế quyền sở hữu,
thời hạn, cứu hộ và tính lũy đẳng cho từng tác dụng phụ.

### `SERIALIZABLE`

Có thể phát hiện chuỗi đọc rồi ghi không thể tuần tự hóa, nhưng yêu cầu thử lại
toàn bộ giao dịch và vẫn nên giữ ràng buộc duy nhất để diễn đạt bất biến trực
tiếp. Không cần nâng mức cô lập chỉ để thay thế một khóa duy nhất chính xác.

### Khóa bi quan trên giỏ hàng

Khóa một dòng giỏ hàng có thể tuần tự hóa checkout của cùng giỏ, nhưng không bảo
vệ các lần thử lại sau khi giỏ được sao chép, xóa hoặc đi qua đường ghi khác. Nó
cũng không cung cấp phản hồi đã lưu. Có thể dùng khóa này cho quy tắc giỏ hàng,
không thay cho hợp đồng lũy đẳng.

### Khóa JVM hoặc khóa phân tán

Khóa JVM không bao phủ nhiều máy chủ. Khóa phân tán thêm lỗi thuê khóa và chủ sở
hữu cũ, trong khi PostgreSQL vẫn cần ràng buộc để bảo vệ dữ liệu. Không chọn một
khóa ngoài chỉ mục duy nhất cho bất biến nằm trong cùng cơ sở dữ liệu.

## 13. So sánh định tính

| Cách | Tính đúng đắn | Hành vi bên thua | Tải và độ trễ | Vận hành nhiều máy chủ |
| --- | --- | --- | --- | --- |
| Bảng mã yêu cầu + `DO NOTHING` | Đầy đủ cho phát lại và đối chiếu nội dung | Chờ rồi đọc kết quả | Thêm một dòng và một lần đọc khi trùng | Tự nhiên qua PostgreSQL |
| `UNIQUE` trên đơn + bắt `23505` | Bảo vệ một đơn trên một khóa | Giao dịch thua lỗi, sau đó đọc ở giao dịch mới | Dùng luồng ngoại lệ khi trùng | An toàn nếu phân loại đúng ràng buộc |
| Chốt quyền ở giao dịch riêng | Có thể an toàn nếu có cơ chế cứu hộ đầy đủ | Có thể thấy `PROCESSING` | Giao dịch ngắn hơn nhưng nhiều trạng thái hơn | Phức tạp khi tiến trình sập |
| `SERIALIZABLE` | Phát hiện xung đột vị từ | Một bên có thể nhận `40001` và thử lại | Tăng số giao dịch thử lại khi tranh chấp | PostgreSQL điều phối |
| Khóa JVM | Không bảo vệ toàn hệ thống | Chỉ chờ trong một tiến trình | Nhẹ cục bộ | Không an toàn khi có nhiều máy chủ |

Không có con số thông lượng hoặc thời gian chờ phù hợp cho mọi hệ thống. Phải đo
tỷ lệ yêu cầu trùng, thời gian chờ chỉ mục, số lần hoàn tác và độ tuổi outbox trên
hạ tầng thật.

## 14. Danh sách kiểm tra triển khai

- [ ] Phía gọi giữ nguyên `Idempotency-Key` cho mọi lần thử của cùng ý định.
- [ ] Khóa được giới hạn theo khách hàng hoặc đơn vị thuê bao.
- [ ] Dấu vân tay do máy chủ tính từ dữ liệu đã chuẩn hóa và có phiên bản.
- [ ] Migration thật có `uk_checkout_request_customer_key`.
- [ ] Luồng chính dùng `INSERT ... ON CONFLICT DO NOTHING RETURNING`.
- [ ] Giao dịch dùng `READ COMMITTED` hoặc hành vi ở mức cô lập khác đã được kiểm
      thử rõ ràng.
- [ ] Đơn, outbox và phản hồi cùng chốt với quyền xử lý.
- [ ] Không gọi mạng trong giao dịch checkout.
- [ ] Phản hồi trùng, sai dấu vân tay và đang xử lý được phân biệt.
- [ ] Lỗi kỹ thuật mở giao dịch mới khi thử lại.
- [ ] Dịch vụ thanh toán nhận một mã lũy đẳng ổn định.
- [ ] Thời hạn lưu khóa dài hơn cửa sổ gửi lại đã cam kết.
- [ ] Log và metric không chứa khóa hoặc nội dung checkout thô.
- [ ] Có truy vấn đối soát số đơn và số outbox trên mỗi quyền xử lý.
