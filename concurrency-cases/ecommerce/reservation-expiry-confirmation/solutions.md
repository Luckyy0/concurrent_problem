# Thiết kế vòng đời khoản giữ tồn kho an toàn

## 1. Mục tiêu thiết kế

Giải pháp chính phải bảo đảm:

1. Xác nhận và hết hạn không thể cùng thắng một khoản giữ.
2. Thời hạn được đánh giá bằng đồng hồ PostgreSQL tại câu chuyển trạng thái.
3. Trạng thái khoản giữ và bộ đếm tồn kho cùng chốt hoặc cùng hoàn tác.
4. Chỉ xác nhận thắng mới được tạo đơn và lệnh thanh toán trong outbox.
5. Cùng yêu cầu checkout được phát lại mà không tiêu thụ tồn kho lần hai.
6. Nhiều tác vụ hết hạn có thể chia việc mà không hoàn kho trùng.
7. Mọi dòng tồn kho của một khoản giữ được khóa theo thứ tự ổn định.
8. Xung đột nghiệp vụ được tách khỏi hết thời gian chờ, bế tắc và lỗi kết nối.

## 2. Lược đồ dữ liệu

```sql
CREATE TABLE inventory_item (
    product_id BIGINT PRIMARY KEY,
    on_hand_quantity INTEGER NOT NULL,
    available_quantity INTEGER NOT NULL,
    reserved_quantity INTEGER NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT ck_inventory_non_negative CHECK (
        on_hand_quantity >= 0
        AND available_quantity >= 0
        AND reserved_quantity >= 0
    ),
    CONSTRAINT ck_inventory_conservation CHECK (
        available_quantity + reserved_quantity = on_hand_quantity
    ),
    CONSTRAINT ck_inventory_version CHECK (version >= 0)
);

CREATE TABLE inventory_reservation (
    reservation_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    cart_id UUID NOT NULL,
    status VARCHAR(16) NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    confirmation_request_id UUID,
    confirmed_order_id UUID,
    confirmed_at TIMESTAMPTZ,
    expired_at TIMESTAMPTZ,
    released_at TIMESTAMPTZ,
    version BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT ck_reservation_status CHECK (
        status IN ('RESERVED', 'CONFIRMED', 'EXPIRED', 'RELEASED')
    ),
    CONSTRAINT ck_reservation_expiry CHECK (expires_at > created_at),
    CONSTRAINT ck_reservation_version CHECK (version >= 0),
    CONSTRAINT ck_reservation_state_fields CHECK (
        (
            status = 'RESERVED'
            AND confirmation_request_id IS NULL
            AND confirmed_order_id IS NULL
            AND confirmed_at IS NULL
            AND expired_at IS NULL
            AND released_at IS NULL
        )
        OR
        (
            status = 'CONFIRMED'
            AND confirmation_request_id IS NOT NULL
            AND confirmed_order_id IS NOT NULL
            AND confirmed_at IS NOT NULL
            AND expired_at IS NULL
            AND released_at IS NULL
        )
        OR
        (
            status = 'EXPIRED'
            AND confirmation_request_id IS NULL
            AND confirmed_order_id IS NULL
            AND confirmed_at IS NULL
            AND expired_at IS NOT NULL
            AND released_at IS NULL
        )
        OR
        (
            status = 'RELEASED'
            AND confirmation_request_id IS NULL
            AND confirmed_order_id IS NULL
            AND confirmed_at IS NULL
            AND expired_at IS NULL
            AND released_at IS NOT NULL
        )
    ),
    CONSTRAINT uk_reservation_confirmed_order
        UNIQUE (confirmed_order_id)
);

CREATE TABLE inventory_reservation_line (
    reservation_id UUID NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL,

    CONSTRAINT pk_reservation_line
        PRIMARY KEY (reservation_id, product_id),
    CONSTRAINT fk_reservation_line_reservation
        FOREIGN KEY (reservation_id)
        REFERENCES inventory_reservation(reservation_id)
        ON DELETE CASCADE,
    CONSTRAINT fk_reservation_line_inventory
        FOREIGN KEY (product_id)
        REFERENCES inventory_item(product_id),
    CONSTRAINT ck_reservation_line_quantity CHECK (quantity > 0)
);

CREATE INDEX ix_reservation_due
    ON inventory_reservation (expires_at, reservation_id)
    WHERE status = 'RESERVED';
```

`purchase_order` cần ràng buộc `UNIQUE (reservation_id)`. Bảng
`checkout_request` và ràng buộc khóa lũy đẳng dùng thiết kế của ECOM-003; không
lặp lại toàn bộ migration tại đây.

`version` phục vụ quan sát và các đường quản trị, nhưng giải pháp chính phân xử
bằng trạng thái cùng thời hạn trong SQL. Không ánh xạ cùng cột này thành
`@Version` nếu cũng cập nhật nó bằng JDBC trên một thực thể đang được Hibernate
quản lý trong cùng giao dịch.

## 3. Chụp thời điểm quyết định đúng một lần

Gọi `clock_timestamp()` nhiều lần trong một câu lệnh có thể tạo các giá trị hơi
khác nhau. Biểu thức bảng chung (`WITH`, còn gọi là CTE) sau chụp một mốc rồi dùng lại cho cả điều kiện và cột dấu thời
gian:

```sql
WITH decision AS MATERIALIZED (
    SELECT clock_timestamp() AS decided_at
)
UPDATE inventory_reservation AS reservation
SET status = 'CONFIRMED',
    confirmation_request_id = :checkoutRequestId,
    confirmed_order_id = :orderId,
    confirmed_at = decision.decided_at,
    version = reservation.version + 1,
    updated_at = decision.decided_at
FROM decision
WHERE reservation.reservation_id = :reservationId
  AND reservation.customer_id = :customerId
  AND reservation.status = 'RESERVED'
  AND reservation.expires_at > decision.decided_at
RETURNING reservation.reservation_id,
          reservation.confirmed_order_id,
          reservation.confirmed_at,
          reservation.version;
```

`MATERIALIZED` cùng hàm dễ biến đổi `clock_timestamp()` làm ý định “một mốc cho
một câu lệnh” rõ ràng. Không nhận `decided_at` từ request hoặc đồng hồ JVM.

## 4. Kho chuyển trạng thái

```java
public record ConfirmedReservation(
    UUID reservationId,
    UUID orderId,
    Instant confirmedAt,
    long version
) {
}

public record ExpiredReservation(
    UUID reservationId,
    Instant expiredAt,
    long version
) {
}
```

```java
@Repository
public class ReservationTransitionStore {

    private final NamedParameterJdbcTemplate jdbc;

    public ReservationTransitionStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<ConfirmedReservation> tryConfirm(
        UUID reservationId,
        UUID customerId,
        UUID checkoutRequestId,
        UUID orderId
    ) {
        List<ConfirmedReservation> rows = jdbc.query(
            """
            WITH decision AS MATERIALIZED (
                SELECT clock_timestamp() AS decided_at
            )
            UPDATE inventory_reservation AS reservation
               SET status = 'CONFIRMED',
                   confirmation_request_id = :checkoutRequestId,
                   confirmed_order_id = :orderId,
                   confirmed_at = decision.decided_at,
                   version = reservation.version + 1,
                   updated_at = decision.decided_at
              FROM decision
             WHERE reservation.reservation_id = :reservationId
               AND reservation.customer_id = :customerId
               AND reservation.status = 'RESERVED'
               AND reservation.expires_at > decision.decided_at
            RETURNING reservation.reservation_id,
                      reservation.confirmed_order_id,
                      reservation.confirmed_at,
                      reservation.version
            """,
            Map.of(
                "reservationId", reservationId,
                "customerId", customerId,
                "checkoutRequestId", checkoutRequestId,
                "orderId", orderId
            ),
            (rs, rowNumber) -> new ConfirmedReservation(
                rs.getObject("reservation_id", UUID.class),
                rs.getObject("confirmed_order_id", UUID.class),
                rs.getTimestamp("confirmed_at").toInstant(),
                rs.getLong("version")
            )
        );

        return onlyRow(rows);
    }

    public Optional<ExpiredReservation> tryExpire(UUID reservationId) {
        List<ExpiredReservation> rows = jdbc.query(
            """
            WITH decision AS MATERIALIZED (
                SELECT clock_timestamp() AS decided_at
            )
            UPDATE inventory_reservation AS reservation
               SET status = 'EXPIRED',
                   expired_at = decision.decided_at,
                   version = reservation.version + 1,
                   updated_at = decision.decided_at
              FROM decision
             WHERE reservation.reservation_id = :reservationId
               AND reservation.status = 'RESERVED'
               AND reservation.expires_at <= decision.decided_at
            RETURNING reservation.reservation_id,
                      reservation.expired_at,
                      reservation.version
            """,
            Map.of("reservationId", reservationId),
            (rs, rowNumber) -> new ExpiredReservation(
                rs.getObject("reservation_id", UUID.class),
                rs.getTimestamp("expired_at").toInstant(),
                rs.getLong("version")
            )
        );

        return onlyRow(rows);
    }

    private static <T> Optional<T> onlyRow(List<T> rows) {
        if (rows.size() > 1) {
            throw new IllegalStateException("single transition returned many rows");
        }
        return rows.stream().findFirst();
    }
}
```

Một giá trị trả về là quyền tiếp tục. Kết quả rỗng không phải tín hiệu cho phép
ứng dụng tự ghi trạng thái bằng một cách khác.

## 5. Khóa dòng tồn kho theo thứ tự ổn định

```java
@Repository
public class ReservationInventoryStore {

    private final NamedParameterJdbcTemplate jdbc;

    public ReservationInventoryStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public void lockLinesInProductOrder(UUID reservationId) {
        List<Long> expected = lineProductIds(reservationId);

        List<Long> locked = jdbc.query(
            """
            SELECT item.product_id
              FROM inventory_item AS item
              JOIN inventory_reservation_line AS line
                ON line.product_id = item.product_id
             WHERE line.reservation_id = :reservationId
             ORDER BY item.product_id
             FOR UPDATE OF item
            """,
            Map.of("reservationId", reservationId),
            (rs, rowNumber) -> rs.getLong("product_id")
        );

        if (!locked.equals(expected) || locked.isEmpty()) {
            throw new InventoryReservationInvariantException(
                "reservation lines and inventory rows do not match"
            );
        }
    }

    public void consumeHeld(UUID reservationId) {
        int expected = lineCount(reservationId);
        List<Long> changed = jdbc.query(
            """
            UPDATE inventory_item AS item
               SET on_hand_quantity = item.on_hand_quantity - line.quantity,
                   reserved_quantity = item.reserved_quantity - line.quantity,
                   version = item.version + 1,
                   updated_at = clock_timestamp()
              FROM inventory_reservation_line AS line
             WHERE line.reservation_id = :reservationId
               AND item.product_id = line.product_id
               AND item.on_hand_quantity >= line.quantity
               AND item.reserved_quantity >= line.quantity
            RETURNING item.product_id
            """,
            Map.of("reservationId", reservationId),
            (rs, rowNumber) -> rs.getLong("product_id")
        );

        requireAllLines(expected, changed.size());
    }

    public void releaseHeld(UUID reservationId) {
        int expected = lineCount(reservationId);
        List<Long> changed = jdbc.query(
            """
            UPDATE inventory_item AS item
               SET available_quantity = item.available_quantity + line.quantity,
                   reserved_quantity = item.reserved_quantity - line.quantity,
                   version = item.version + 1,
                   updated_at = clock_timestamp()
              FROM inventory_reservation_line AS line
             WHERE line.reservation_id = :reservationId
               AND item.product_id = line.product_id
               AND item.reserved_quantity >= line.quantity
            RETURNING item.product_id
            """,
            Map.of("reservationId", reservationId),
            (rs, rowNumber) -> rs.getLong("product_id")
        );

        requireAllLines(expected, changed.size());
    }

    private List<Long> lineProductIds(UUID reservationId) {
        return jdbc.queryForList(
            """
            SELECT product_id
              FROM inventory_reservation_line
             WHERE reservation_id = :reservationId
             ORDER BY product_id
            """,
            Map.of("reservationId", reservationId),
            Long.class
        );
    }

    private int lineCount(UUID reservationId) {
        Integer count = jdbc.queryForObject(
            """
            SELECT count(*)
              FROM inventory_reservation_line
             WHERE reservation_id = :reservationId
            """,
            Map.of("reservationId", reservationId),
            Integer.class
        );
        return Objects.requireNonNull(count);
    }

    private static void requireAllLines(int expected, int actual) {
        if (expected == 0 || expected != actual) {
            throw new InventoryReservationInvariantException(
                "expected " + expected + " inventory rows, changed " + actual
            );
        }
    }
}
```

`lockLinesInProductOrder` phải được gọi trước `consumeHeld` hoặc `releaseHeld`.
Việc kiểm tra số dòng bảo vệ trường hợp dữ liệu bị lệch; ngoại lệ phải thoát khỏi
giao dịch để hoàn tác trạng thái khoản giữ.

Với số dòng lớn, có thể gộp việc đếm và cập nhật trong thủ tục SQL. Dù chọn cách
nào, không coi cập nhật một phần là thành công.

## 6. Hợp đồng lệnh xác nhận

```java
public record ConfirmReservationCommand(
    UUID customerId,
    String idempotencyKey,
    String requestFingerprint,
    UUID reservationId
) {
    public ConfirmReservationCommand {
        Objects.requireNonNull(customerId);
        Objects.requireNonNull(idempotencyKey);
        Objects.requireNonNull(requestFingerprint);
        Objects.requireNonNull(reservationId);
        if (idempotencyKey.isBlank()) {
            throw new IllegalArgumentException("idempotency key is blank");
        }
    }
}

public enum ReservationConfirmationOutcome {
    CONFIRMED,
    RESERVATION_EXPIRED,
    RESERVATION_RELEASED,
    ALREADY_CONFIRMED
}

public record ReservationConfirmationResult(
    ReservationConfirmationOutcome outcome,
    UUID orderId,
    boolean replayed
) {
}
```

`requestFingerprint` được máy chủ tính từ dữ liệu chuẩn hóa, gồm khách hàng,
`reservationId`, phiên bản báo giá và dữ liệu checkout liên quan. Không nhận dấu
vân tay do phía gọi tự khai báo.

## 7. Giao dịch xác nhận tích hợp tính lũy đẳng

```java
@Service
public class ReservationCheckoutTransaction {

    private final CheckoutRequestStore checkoutRequests;
    private final ReservationTransitionStore transitions;
    private final ReservationStateStore reservationStates;
    private final ReservationInventoryStore inventory;
    private final PurchaseOrderWriter orders;
    private final PaymentOutbox outbox;

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReservationConfirmationResult confirm(
        ConfirmReservationCommand command
    ) {
        CheckoutClaim claim = checkoutRequests.tryClaim(
            command.customerId(),
            command.idempotencyKey(),
            command.requestFingerprint()
        );

        if (claim instanceof CheckoutClaim.Replay replay) {
            return replay.result();
        }

        UUID checkoutRequestId = ((CheckoutClaim.New) claim).requestId();
        UUID orderId = UUID.randomUUID();

        Optional<ConfirmedReservation> confirmed = transitions.tryConfirm(
            command.reservationId(),
            command.customerId(),
            checkoutRequestId,
            orderId
        );

        if (confirmed.isEmpty()) {
            ReservationConfirmationResult rejected = classifyLoss(
                command,
                checkoutRequestId
            );
            checkoutRequests.complete(checkoutRequestId, rejected);
            return rejected;
        }

        inventory.lockLinesInProductOrder(command.reservationId());
        inventory.consumeHeld(command.reservationId());

        orders.insertPendingPayment(
            orderId,
            command.customerId(),
            command.reservationId(),
            checkoutRequestId
        );
        outbox.appendStartPayment(orderId, checkoutRequestId);

        ReservationConfirmationResult result =
            new ReservationConfirmationResult(
                ReservationConfirmationOutcome.CONFIRMED,
                orderId,
                false
            );
        checkoutRequests.complete(checkoutRequestId, result);
        return result;
    }

    private ReservationConfirmationResult classifyLoss(
        ConfirmReservationCommand command,
        UUID checkoutRequestId
    ) {
        ReservationSnapshot current = reservationStates.requireCurrent(
            command.reservationId(),
            command.customerId()
        );

        return switch (current.status()) {
            case EXPIRED -> expired();
            case RELEASED -> released();
            case CONFIRMED -> current.confirmationRequestId()
                .filter(checkoutRequestId::equals)
                .map(ignored -> alreadyConfirmed(current.orderId()))
                .orElseThrow(ReservationAlreadyConfirmedException::new);
            case RESERVED -> {
                if (current.expiredAccordingToDatabase()) {
                    yield expired();
                }
                throw new ReservationTransitionConflictException();
            }
        };
    }
}
```

`CheckoutRequestStore` dùng `INSERT ... ON CONFLICT DO NOTHING RETURNING` và lưu
phản hồi như ECOM-003. Quyền xử lý, chuyển trạng thái, bộ đếm, đơn, outbox và kết
quả cùng nằm trong giao dịch này.

Nếu khoản giữ vẫn `RESERVED` nhưng đã quá hạn, checkout được chốt với kết quả từ
chối ổn định. Tác vụ sẽ hoàn kho sau; checkout không tự xác nhận chỉ vì tác vụ
chưa chạy.

`ReservationStateStore.requireCurrent` tính cờ hết hạn bằng
`expires_at <= clock_timestamp()` trong PostgreSQL, không bằng đồng hồ Java.

## 8. Vì sao giao dịch xác nhận hoạt động

1. Chỉ một khóa checkout được phép chạy nhánh mới; bản gửi lại phát lại phản hồi.
2. Câu `tryConfirm` chỉ trả dòng nếu khoản giữ còn `RESERVED` và còn hạn.
3. Chính câu đó khóa dòng khoản giữ và chuyển sang trạng thái kết thúc.
4. Chỉ sau khi nhận dòng, mã nguồn mới tiêu thụ bộ đếm và tạo đơn.
5. Ràng buộc duy nhất trên `purchase_order.reservation_id` chặn đơn thứ hai.
6. Lỗi ở bất kỳ bước sau làm hoàn tác cả trạng thái `CONFIRMED`.
7. Outbox tách lời gọi thanh toán khỏi giao dịch đang giữ khóa.

## 9. Chọn công việc hết hạn bằng `SKIP LOCKED`

```java
@Repository
public class ReservationExpiryQueue {

    private final NamedParameterJdbcTemplate jdbc;

    public Optional<UUID> lockNextDue() {
        List<UUID> rows = jdbc.query(
            """
            SELECT reservation_id
              FROM inventory_reservation
             WHERE status = 'RESERVED'
               AND expires_at <= clock_timestamp()
             ORDER BY expires_at, reservation_id
             LIMIT 1
             FOR UPDATE SKIP LOCKED
            """,
            Map.of(),
            (rs, rowNumber) -> rs.getObject("reservation_id", UUID.class)
        );

        return rows.stream().findFirst();
    }
}
```

Khóa chỉ có ý nghĩa khi phương thức được gọi bên trong giao dịch hết hạn và giao
dịch đó giữ nguyên kết nối đến lúc hoàn kho xong.

## 10. Một giao dịch hết hạn một khoản giữ

```java
public enum ExpiryAttempt {
    EXPIRED,
    NO_DUE_RESERVATION
}
```

```java
@Service
public class ReservationExpiryTransaction {

    private final ReservationExpiryQueue queue;
    private final ReservationTransitionStore transitions;
    private final ReservationInventoryStore inventory;

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ExpiryAttempt expireNext() {
        Optional<UUID> candidate = queue.lockNextDue();
        if (candidate.isEmpty()) {
            return ExpiryAttempt.NO_DUE_RESERVATION;
        }

        UUID reservationId = candidate.orElseThrow();
        ExpiredReservation expired = transitions.tryExpire(reservationId)
            .orElseThrow(() -> new InventoryReservationInvariantException(
                "locked due reservation did not expire"
            ));

        inventory.lockLinesInProductOrder(expired.reservationId());
        inventory.releaseHeld(expired.reservationId());
        return ExpiryAttempt.EXPIRED;
    }
}
```

Sau `lockNextDue`, checkout khác muốn cập nhật cùng khoản giữ phải chờ. Nếu cập
nhật bộ đếm thất bại, ngoại lệ làm hoàn tác cả `EXPIRED` và khóa được nhả.

## 11. Bộ điều phối tác vụ

```java
@Component
public class ReservationExpiryWorker {

    private final ReservationExpiryTransaction transaction;
    private final int maxItemsPerRun;

    @Scheduled(fixedDelayString = "${reservation.expiry-delay}")
    public void run() {
        for (int processed = 0; processed < maxItemsPerRun; processed++) {
            if (transaction.expireNext()
                == ExpiryAttempt.NO_DUE_RESERVATION) {
                return;
            }
        }
    }
}
```

`ReservationExpiryWorker` và `ReservationExpiryTransaction` là hai bean khác
nhau để mỗi lời gọi đi qua Spring proxy và mở một giao dịch mới. Không đặt
`expireNext()` cùng lớp rồi gọi nội bộ.

Giới hạn số phần tử mỗi lượt tạo cơ hội cho công việc khác dùng vùng kết nối.
Nhiều tác vụ hoặc nhiều máy chủ có thể chạy cùng mã; `SKIP LOCKED` chia các dòng
đang có, còn điều kiện trạng thái bảo vệ tính đúng đắn cuối cùng.

## 12. Giới hạn thời gian chờ khóa

`SKIP LOCKED` tránh chờ ở dòng khoản giữ, nhưng cập nhật `inventory_item` vẫn có
thể chờ giao dịch khác. Có thể đặt giới hạn cục bộ cho giao dịch:

```sql
SET LOCAL lock_timeout = '750ms';
SET LOCAL statement_timeout = '3s';
```

Các giá trị phải đến từ đo đạc và mục tiêu vận hành của hệ thống, không sao chép
con số minh họa một cách máy móc. Hết thời gian chờ phải hoàn tác rồi thử lại ở
giao dịch mới; không đánh dấu khoản giữ là đã hết hạn nếu bộ đếm chưa hoàn tất.

## 13. Hành vi của bên thua

| Cuộc đua | Bên thắng | Bên thua |
| --- | --- | --- |
| Xác nhận trước hạn với tác vụ | Xác nhận đổi sang `CONFIRMED` | Tác vụ bỏ qua dòng khóa hoặc không còn thấy `RESERVED` |
| Tác vụ hết hạn với xác nhận | Tác vụ đổi sang `EXPIRED` và hoàn kho | Xác nhận nhận `0` dòng, đọc lại và trả `RESERVATION_EXPIRED` |
| Hai tác vụ | Một tác vụ khóa và xử lý dòng | Tác vụ kia `SKIP LOCKED`, lấy dòng khác hoặc kết thúc lượt |
| Hai bản cùng checkout key | Một giao dịch chiếm `checkout_request` | Bên kia chờ rồi phát lại phản hồi đã lưu |
| Hai checkout key khác nhau | Một bên có thể xác nhận | Bên kia thấy `CONFIRMED`; không tạo đơn thứ hai |

Kết quả `0` dòng do trạng thái kết thúc là quyết định nghiệp vụ, không phải lỗi
cần thử lại vô hạn.

## 14. Hoàn tác, sự cố tiến trình và kết quả chưa rõ

- Lỗi trước `COMMIT`: PostgreSQL hoàn tác trạng thái, bộ đếm, đơn, outbox và kết
  quả checkout.
- Tác vụ dừng khi đang giữ khóa: kết nối đóng, giao dịch hoàn tác; tác vụ khác có
  thể lấy lại dòng.
- Checkout chốt nhưng mất phản hồi: gửi lại cùng khóa lũy đẳng để phát lại cùng
  `orderId`.
- Tác vụ chốt rồi mất phản hồi nội bộ: lần chạy sau không còn thấy `RESERVED`,
  nên không hoàn kho lần hai.
- Bế tắc hoặc `lock_timeout`: mở giao dịch mới, tải lại và thử có giới hạn.

Không bắt ngoại lệ PostgreSQL rồi tiếp tục dùng cùng giao dịch. Giao dịch có thể
đã bị đánh dấu chỉ cho phép hoàn tác (`rollback-only`) và các thực thể đang quản lý không còn đáng tin.

## 15. Phương án thay thế: khóa bi quan

Checkout và tác vụ đều có thể tải dòng bằng:

```sql
SELECT reservation_id, status, expires_at
FROM inventory_reservation
WHERE reservation_id = :reservationId
FOR UPDATE;
```

Sau khi giữ khóa, đọc `clock_timestamp()` từ PostgreSQL, quyết định nhánh rồi cập
nhật trạng thái cùng bộ đếm. Cách này phù hợp khi quyết định cần nhiều thuộc tính
khó diễn đạt trong một `WHERE`.

Đổi lại, bên thua chờ từ lúc đọc; giao dịch dài làm tăng độ trễ và chiếm kết nối.
Mọi đường phải khóa theo cùng thứ tự và không gọi dịch vụ từ xa khi đang giữ khóa.

## 16. Phương án thay thế: JPA `@Version`

```java
@Version
@Column(name = "version", nullable = false)
private long version;
```

Hai giao dịch cùng đọc một phiên bản; bên chốt sau nhận
`ObjectOptimisticLockingFailureException` khi câu:

```sql
UPDATE inventory_reservation
SET status = ?, version = ?
WHERE reservation_id = ?
  AND version = ?;
```

ảnh hưởng `0` dòng. Bên thua phải hoàn tác và tải lại trong giao dịch mới.

`@Version` không tự giải quyết thời gian. Nhánh xác nhận vẫn phải dùng thời gian
PostgreSQL, và tác vụ theo lô vẫn cần cách chia việc. Khi một lượng lớn khoản giữ
cùng hết hạn, khóa lạc quan có thể tạo nhiều xung đột và lần thử lại hơn giải
pháp `SKIP LOCKED`.

## 17. Các lựa chọn không thay thế bất biến

### `SERIALIZABLE`

Mức cô lập cao hơn có thể hủy một lịch thực thi nguy hiểm, nhưng ứng dụng vẫn
phải có máy trạng thái, cập nhật bộ đếm đúng một lần và thử lại toàn giao dịch.
Nó không định nghĩa phản hồi lũy đẳng hay thời hạn.

### Một bộ lập lịch duy nhất

Chỉ chạy một tác vụ làm giảm cạnh tranh giữa tác vụ, nhưng không ngăn checkout ở
máy chủ khác. Nó còn tạo điểm lỗi nếu bị xem là lớp bảo vệ duy nhất.

### Khóa phân tán

Khi trạng thái và bộ đếm cùng ở PostgreSQL, thêm khóa ngoài làm tăng số thành
phần lỗi mà không mạnh hơn câu cập nhật có điều kiện. Nếu khóa hết hạn khi tiến
trình tạm dừng, chủ cũ còn có thể tiếp tục; PostgreSQL vẫn phải từ chối chủ cũ.

### Hàng đợi

Hàng đợi hữu ích khi cần điều tiết tải hoặc xử lý bất đồng bộ. Gửi sự kiện xác
nhận và hết hạn vào hàng đợi không tự bảo đảm thứ tự giữa partition, lần gửi lại
và tác dụng phụ trong cơ sở dữ liệu. Consumer vẫn cần chuyển trạng thái có điều
kiện và tính lũy đẳng.

## 18. So sánh định tính

| Cách làm | Điểm mạnh | Đổi lại |
| --- | --- | --- |
| Cập nhật trạng thái có điều kiện | Phát hiện xung đột đúng tại nguồn dữ liệu; ít bước | Cần SQL riêng và xử lý rõ kết quả `0/1` |
| `FOR UPDATE SKIP LOCKED` cho tác vụ | Chia việc giữa nhiều tác vụ, tránh chờ dòng đang xử lý | Không bảo đảm công bằng; cần theo dõi dòng quá hạn bị kẹt |
| Khóa bi quan cho mọi nhánh | Dễ đọc nhiều dữ liệu rồi quyết định | Chờ khóa lâu hơn, tăng nguy cơ cạn kết nối và bế tắc |
| JPA `@Version` | Tích hợp tự nhiên với thực thể | Xung đột phát hiện muộn; thời gian và tác vụ vẫn cần thiết kế riêng |
| `SERIALIZABLE` | Phát hiện thêm nhiều lịch sử không an toàn | Tăng lỗi tuần tự hóa và yêu cầu thử lại toàn giao dịch |
| Khóa phân tán | Có thể phối hợp tài nguyên ngoài PostgreSQL | Thêm TTL, fencing và lỗi mạng; không thay thế điều kiện DB |

Không chọn kích thước lô, số tác vụ hoặc thời gian chờ từ số liệu giả định. Đo
tuổi khoản giữ quá hạn, thời gian giao dịch, thời gian chờ khóa, tỷ lệ bỏ qua và
số lần thử lại trên tải thực tế.

## 19. Danh sách kiểm tra triển khai

- [ ] `RESERVED` chỉ đi tới một trạng thái kết thúc.
- [ ] Xác nhận kiểm tra `status` và `expires_at` trong cùng câu ghi.
- [ ] Hết hạn kiểm tra `status` và thời hạn trong cùng giao dịch hoàn kho.
- [ ] Thời gian phân xử đến từ PostgreSQL, không từ request hoặc JVM.
- [ ] Kết quả `0` dòng dừng mọi tác dụng phụ của nhánh thua.
- [ ] Các dòng tồn kho được khóa theo `product_id` ổn định.
- [ ] Số dòng tồn kho cập nhật phải bằng số dòng khoản giữ.
- [ ] Trạng thái, bộ đếm, đơn, outbox và kết quả cùng hoàn tác khi cần.
- [ ] `purchase_order.reservation_id` có ràng buộc duy nhất.
- [ ] Checkout dùng khóa lũy đẳng và phát lại phản hồi.
- [ ] Tác vụ dùng giao dịch ngắn, `SKIP LOCKED` và giới hạn số phần tử mỗi lượt.
- [ ] Không có lời gọi mạng khi đang giữ khóa.
- [ ] Timeout và bế tắc không bị đổi thành lỗi hết hạn nghiệp vụ.
- [ ] Test tích hợp dùng PostgreSQL thật và kiểm tra toàn bộ bất biến.
