# Giải Pháp Chuẩn Mực — Cập Nhật Tích Hợp Bảo Vệ (Guarded Mutation)

## 1. Mục Tiêu Thiết Kế

Một kiến trúc xử lý tương tranh bền vững cần đáp ứng đồng thời 4 yêu cầu bảo vệ:

1. Thao tác Đánh giá điều kiện (Predicate validation) và Thay đổi dữ liệu (Mutation) phải được liên kết trong cùng một câu lệnh SQL (Nguyên tử - Atomic).
2. Quyết định nghiệp vụ (Thành công/Từ chối) hoàn toàn phụ thuộc vào số dòng dữ liệu bị ảnh hưởng (Affected rows) do câu lệnh SQL trả về.
3. Thiết lập mã định danh độc nhất (Command claim/Idempotency key) ngay từ khâu tiếp nhận yêu cầu, bảo vệ hệ thống khỏi các lệnh thử lại (Replay) của Client.
4. Quá trình kiểm tra Idempotency, cập nhật kho, ghi nhận lịch sử và tạo thông điệp ngoại vi (outbox event) phải đảm bảo nguyên tắc tất cả-hoặc-không-có-gì (All-or-nothing) thông qua một ranh giới transaction chung (Transaction boundary).

## 2. Cấu Trúc Bảng Dữ Liệu (Schema Proposal)

```sql
CREATE TABLE inventory_item (
    product_id BIGINT PRIMARY KEY,
    on_hand_quantity INTEGER NOT NULL,
    available_quantity INTEGER NOT NULL,
    reserved_quantity INTEGER NOT NULL,
    revision BIGINT NOT NULL DEFAULT 0,
    CHECK (on_hand_quantity >= 0),
    CHECK (available_quantity >= 0),
    CHECK (reserved_quantity >= 0),
    CHECK (available_quantity + reserved_quantity = on_hand_quantity)
);

CREATE TABLE inventory_reservation (
    command_id UUID PRIMARY KEY, -- Khóa chính đảm bảo tính lũy đẳng
    reservation_id UUID UNIQUE,
    order_id UUID NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    outcome VARCHAR(24) NOT NULL,
    request_fingerprint VARCHAR(64) NOT NULL,
    remaining_available INTEGER,
    remaining_reserved INTEGER,
    created_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    CHECK (outcome IN ('PROCESSING', 'RESERVED', 'OUT_OF_STOCK')),
    CHECK (
        outcome <> 'RESERVED'
        OR (
            reservation_id IS NOT NULL
            AND remaining_available IS NOT NULL
            AND remaining_reserved IS NOT NULL
        )
    )
);

CREATE TABLE outbox_event (
    event_id UUID PRIMARY KEY,
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(128) NOT NULL,
    event_type VARCHAR(128) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    published_at TIMESTAMPTZ
);
```

Trạng thái `PROCESSING` được thiết kế dành cho một transaction đang diễn ra. Ứng dụng sẽ không hoàn tất transaction nếu đơn vị nghiệp vụ (Command) vẫn dừng lại ở trạng thái này.

## 3. Câu Lệnh SQL Nguyên Tử Cốt Lõi

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity,
    revision = revision + 1
WHERE product_id = :productId
  AND available_quantity >= :quantity
RETURNING product_id,
          available_quantity,
          reserved_quantity,
          revision;
```

Dữ liệu đầu vào `:quantity` cần được bảo vệ bởi bộ lọc (Input validation) nhằm loại trừ giá trị âm. Nhờ sự tồn tại của kiểm tra điều kiện (Predicate), khi lệnh `RETURNING` báo cáo 0 dòng bị ảnh hưởng, lý do nghiệp vụ được xác định duy nhất là thiếu số lượng (Out of stock).

## 4. Các Lớp Giá Trị Truyền Tải (Value Objects)

```java
public record ReserveStockCommand(
        UUID commandId,
        UUID orderId,
        long productId,
        int quantity,
        String requestFingerprint
) {
    public ReserveStockCommand {
        Objects.requireNonNull(commandId);
        Objects.requireNonNull(orderId);
        Objects.requireNonNull(requestFingerprint);
        if (quantity <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }
    }
}

public record InventoryAfterReserve(
        long productId,
        int available,
        int reserved,
        long revision
) {}
```

Chữ ký truy vấn (`requestFingerprint`) phục vụ xác thực độ trùng khớp và cần được mã hóa thông tin cốt lõi thay vì phụ thuộc ngẫu nhiên từ Client.

## 5. Tầng Truy Cập Dữ Liệu (DAO) Xử Lý Tính Lũy Đẳng

```java
@Repository
public class ReservationCommandDao {
    private final NamedParameterJdbcTemplate jdbc;

    public ReservationCommandDao(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    // Đăng ký tiến trình xử lý ban đầu
    public boolean tryClaim(ReserveStockCommand command, Instant now) {
        int changed = jdbc.update("""
                INSERT INTO inventory_reservation (
                    command_id,
                    order_id,
                    product_id,
                    quantity,
                    outcome,
                    request_fingerprint,
                    created_at
                ) VALUES (
                    :commandId,
                    :orderId,
                    :productId,
                    :quantity,
                    'PROCESSING',
                    :fingerprint,
                    :createdAt
                )
                ON CONFLICT (command_id) DO NOTHING
                """, new MapSqlParameterSource()
                .addValue("commandId", command.commandId())
                .addValue("orderId", command.orderId())
                .addValue("productId", command.productId())
                .addValue("quantity", command.quantity())
                .addValue("fingerprint", command.requestFingerprint())
                .addValue("createdAt", Timestamp.from(now)));
        return changed == 1; // 1: Cấp thành công quyền; 0: Đã có thực thể chiếm hữu
    }

    public StoredReservation requireExisting(UUID commandId) {
        return jdbc.queryForObject("""
                SELECT command_id,
                       reservation_id,
                       order_id,
                       product_id,
                       quantity,
                       outcome,
                       request_fingerprint,
                       remaining_available,
                       remaining_reserved
                FROM inventory_reservation
                WHERE command_id = :commandId
                """, Map.of("commandId", commandId), STORED_ROW_MAPPER);
    }

    public void completeReserved(
            UUID commandId,
            UUID reservationId,
            InventoryAfterReserve stock,
            Instant completedAt
    ) {
        int changed = jdbc.update("""
                UPDATE inventory_reservation
                SET reservation_id = :reservationId,
                    outcome = 'RESERVED',
                    remaining_available = :available,
                    remaining_reserved = :reserved,
                    completed_at = :completedAt
                WHERE command_id = :commandId
                  AND outcome = 'PROCESSING'
                """, new MapSqlParameterSource()
                .addValue("commandId", commandId)
                .addValue("reservationId", reservationId)
                .addValue("available", stock.available())
                .addValue("reserved", stock.reserved())
                .addValue("completedAt", Timestamp.from(completedAt)));
        requireExactlyOne(changed);
    }

    public void completeOutOfStock(UUID commandId, Instant completedAt) {
        int changed = jdbc.update("""
                UPDATE inventory_reservation
                SET outcome = 'OUT_OF_STOCK',
                    completed_at = :completedAt
                WHERE command_id = :commandId
                  AND outcome = 'PROCESSING'
                """, Map.of(
                "commandId", commandId,
                "completedAt", Timestamp.from(completedAt)
        ));
        requireExactlyOne(changed);
    }

    private static void requireExactlyOne(int changed) {
        if (changed != 1) {
            throw new IllegalStateException(
                    "expected one command outcome row, got " + changed
            );
        }
    }
}
```

Tận dụng `ON CONFLICT (command_id) DO NOTHING` đóng vai trò là một cơ chế bảo vệ chống ghi đè dữ liệu. Với tình huống người dùng thực hiện tạo 2 yêu cầu giống hệt nhau đồng thời, yêu cầu đến sau (0 dòng ảnh hưởng) sẽ truy vấn lại báo cáo lịch sử và không kích hoạt quy trình thay đổi database thêm lần nữa.

## 6. Lớp DAO Cho Thao Tác Cập Nhật Kho Nguyên Tử (Mutation DAO)

```java
@Repository
public class InventoryMutationDao {
    private final NamedParameterJdbcTemplate jdbc;

    public InventoryMutationDao(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<InventoryAfterReserve> tryReserve(
            long productId,
            int quantity
    ) {
        List<InventoryAfterReserve> rows = jdbc.query("""
                UPDATE inventory_item
                SET available_quantity = available_quantity - :quantity,
                    reserved_quantity = reserved_quantity + :quantity,
                    revision = revision + 1
                WHERE product_id = :productId
                  AND available_quantity >= :quantity
                RETURNING product_id,
                          available_quantity,
                          reserved_quantity,
                          revision
                """, Map.of(
                "productId", productId,
                "quantity", quantity
        ), (rs, rowNum) -> new InventoryAfterReserve(
                rs.getLong("product_id"),
                rs.getInt("available_quantity"),
                rs.getInt("reserved_quantity"),
                rs.getLong("revision")
        ));

        if (rows.size() > 1) {
            throw new IllegalStateException(
                    "single-product mutation returned multiple rows"
            );
        }
        return rows.stream().findFirst();
    }
}
```

Phân lớp sử dụng JDBC này giúp giới hạn JPA không tải các phần tử Entity không mong muốn lên môi trường Persistence Context (RAM). Cơ chế `RETURNING` đồng bộ giá trị cập nhật ngay tại lúc SQL thành công.

## 7. Thiết Lập Giới Hạn Chờ Đợi Phục Vụ (Bounded Lock Wait)

```java
@ConfigurationProperties("app.inventory.database")
public record InventoryDatabaseTimeouts(
        Duration lockTimeout,
        Duration statementTimeout
) {
    public InventoryDatabaseTimeouts {
        if (!lockTimeout.isPositive()
                || statementTimeout.compareTo(lockTimeout) <= 0) {
            throw new IllegalArgumentException(
                    "statement timeout must exceed positive lock timeout"
        );
        }
    }

    String lockTimeoutValue() {
        return lockTimeout.toMillis() + "ms";
    }

    String statementTimeoutValue() {
        return statementTimeout.toMillis() + "ms";
    }
}

@Component
public class PostgreSqlStatementPolicy {
    private final NamedParameterJdbcTemplate jdbc;
    private final InventoryDatabaseTimeouts timeouts;

    public PostgreSqlStatementPolicy(
            NamedParameterJdbcTemplate jdbc,
            InventoryDatabaseTimeouts timeouts
    ) {
        this.jdbc = jdbc;
        this.timeouts = timeouts;
    }

    public void apply() {
        // Chủ động áp dụng thời gian chờ cho riêng cấu hình Transaction
        jdbc.queryForObject("""
                SELECT set_config('lock_timeout', :value, true)
                """, Map.of(
                "value", timeouts.lockTimeoutValue()
        ), String.class);
        jdbc.queryForObject("""
                SELECT set_config('statement_timeout', :value, true)
                """, Map.of(
                "value", timeouts.statementTimeoutValue()
        ), String.class);
    }
}
```
Thời gian tối đa thực hiện lệnh và độ trễ chờ đợi lock trên database cần được thiết lập ở phạm vi cụ thể của Connection để bảo vệ tài nguyên ứng dụng (Timeout guard).

## 8. Lõi Thực Thi Của Transaction (Transactional Worker)

```java
@Service
public class InventoryReservationTx {
    private final ReservationCommandDao commands;
    private final InventoryMutationDao inventory;
    private final OutboxRepository outbox;
    private final PostgreSqlStatementPolicy statementPolicy;
    private final EntityManager entityManager;
    private final Clock clock;

    public InventoryReservationTx(
            ReservationCommandDao commands,
            InventoryMutationDao inventory,
            OutboxRepository outbox,
            PostgreSqlStatementPolicy statementPolicy,
            EntityManager entityManager,
            Clock clock
    ) {
        this.commands = commands;
        this.inventory = inventory;
        this.outbox = outbox;
        this.statementPolicy = statementPolicy;
        this.entityManager = entityManager;
        this.clock = clock;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReservationResult reserve(ReserveStockCommand command) {
        Instant now = clock.instant();
        statementPolicy.apply(); // Gán thiết lập giới hạn thời gian (Timeout Policy)

        // Bước 1: Yêu cầu cấp đặc quyền (Idempotency Claiming)
        if (!commands.tryClaim(command, now)) {
            StoredReservation existing =
                    commands.requireExisting(command.commandId());
            existing.requireSameRequest(command);
            return existing.toResult(); // Hoàn trả theo dữ liệu cũ có sẵn
        }

        // Bước 2: Kích hoạt SQL nguyên tử
        Optional<InventoryAfterReserve> changed =
                inventory.tryReserve(
                        command.productId(),
                        command.quantity()
                );

        // Bước 3: Rẽ nhánh dựa trên Affected Rows
        if (changed.isEmpty()) { 
            commands.completeOutOfStock(command.commandId(), now); 
            return ReservationResult.outOfStock(command.commandId());
        }

        // Bước 4: Lấy số liệu phản hồi từ database
        InventoryAfterReserve stock = changed.orElseThrow();
        UUID reservationId = UUID.randomUUID();
        
        // Bước 5: Đóng kết quả bằng lịch sử tương ứng
        commands.completeReserved(
                command.commandId(),
                reservationId,
                stock,
                now
        );
        
        // Bước 6: Khởi tạo outbox event
        outbox.save(OutboxEvent.inventoryReserved(
                UUID.randomUUID(),
                reservationId,
                command,
                stock,
                now
        ));

        // Khởi động Flush nhằm xác nhận sự tương tác hoàn chỉnh thay vì báo lỗi ẩn sau
        entityManager.flush();
        return ReservationResult.reserved(
                command.commandId(),
                reservationId,
                stock.available(),
                stock.reserved()
        );
    }
}
```
Nhờ sử dụng DAO JDBC kết hợp với phương pháp Transaction hiện hữu, việc liên đới dữ liệu (Integrity) được bảo đảm. Khâu cuối cùng `flush` giúp truy vết lỗi tiềm năng.

## 9. Lớp Điều Phối Ngoại Trừ Khung Transaction (Non-transactional Coordinator)

```java
@Component
public class InventoryReservationCoordinator {
    private final InventoryReservationTx worker;
    private final SqlStateClassifier classifier;

    public InventoryReservationCoordinator(
            InventoryReservationTx worker,
            SqlStateClassifier classifier
    ) {
        this.worker = worker;
        this.classifier = classifier;
    }

    public ReservationResult reserve(ReserveStockCommand command) {
        // Cảnh báo: Ngăn ngừa Coordinator nằm vào khối @Transactional
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            throw new IllegalStateException(
                    "coordinator must run outside a transaction"
            );
        }

        try {
            return worker.reserve(command);
        } catch (RuntimeException failure) {
            // Nhận dạng nguyên nhân sự cố kỹ thuật
            return classifier.lockOrSerializationFailure(failure)
                    .map(kind -> ReservationResult.busy(
                            command.commandId(),
                            kind.name()
                    ))
                    .orElseThrow(() -> failure);
        }
    }
}

@Component
public class SqlStateClassifier {
    public Optional<DatabaseConflict> lockOrSerializationFailure(
            Throwable failure
    ) {
        for (Throwable current = failure;
             current != null;
             current = current.getCause()) {
            if (current instanceof SQLException sql) {
                return switch (sql.getSQLState()) {
                    case "55P03" -> Optional.of(DatabaseConflict.LOCK_TIMEOUT);
                    case "40P01" -> Optional.of(DatabaseConflict.DEADLOCK);
                    case "40001" ->
                            Optional.of(DatabaseConflict.SERIALIZATION_FAILURE);
                    default -> Optional.empty();
                };
            }
        }
        return Optional.empty();
    }
}
```
Bắt buộc thực thi điều phối bên ngoài lớp quản lý transaction. Qua đó, cơ chế xử lý không gặp nguy cơ báo hiệu (Exception state) bị chôn giấu trong Rollback ẩn. Nếu có nhu cầu thiết lập thêm tính năng lặp phản hồi (Retry backoff), thao tác cũng không bị tắc ngẽn connection.

## 10. Thay Thế Với Môi Trường Thuần JPA (Spring Data JPA Equivalent)

Dành cho những dự án có độ truy hồi tồn kho nội sinh:

```java
public interface InventoryItemRepository
        extends JpaRepository<InventoryItem, Long> {

    @Modifying
    @Query(value = """
            UPDATE inventory_item
            SET available_quantity = available_quantity - :quantity,
                reserved_quantity = reserved_quantity + :quantity,
                revision = revision + 1
            WHERE product_id = :productId
              AND available_quantity >= :quantity
            """, nativeQuery = true)
    int reserveIfEnough(
            @Param("productId") long productId,
            @Param("quantity") int quantity
    );
}
```

Service chịu trách nhiệm cần phải tiến hành phân tích số liệu số dòng trả về `changed == 1` trước các phân lớp sự kiện. Do không có `RETURNING`, hệ thống không dễ truy vấn ra số lượng cập nhật mới, yêu cầu việc lấy dữ liệu được tiến hành một cách thủ công với `clear` Entity.

## 11. Các Rủi Ro Hoàn Tác Kết Cấu (Rollback Dependency)

Nếu trong sự kiện tạo Outbox sinh lỗi bất thường, toàn bộ tiến trình sẽ:
```text
Cập nhật tồn kho trả về 1 dòng ảnh hưởng.
Đánh dấu Reservation thành công.
Xảy ra ngoại lệ.
Kích hoạt hủy bỏ tổng thể (Transaction Rollback).
Phục hồi trạng thái gốc.
Hủy thông báo đã được đưa vào database.
```

## 12. Tiến Trình Trả Lại Tồn Kho (Cancel/Release Procedure)

Sự kiện kết xuất hay trả kho nên là một phương thức riêng biệt, chuyển đổi trạng thái (State Transition) từ `RESERVED` về `CANCELLED`. Bắt buộc chỉ thực hiện quá trình cộng dồn đúng 1 lần (Idempotency enforced), tránh thao tác khôi phục kép (Double un-reserve).

## 13. Phân Tích Các Giải Pháp Tương Tranh (Alternative Perspectives)

| Kỹ Thuật (Strategy) | Số Round trips | Hệ Quả (When failed) | Phương Thức Trễ | Hoạt Động Cụm Đa Tầng (Multi-instance) |
| --- | --- | --- | --- | --- |
| Câu lệnh nguyên tử (`UPDATE`) | 1 Truy hồi | Trả số 0 dòng ảnh hưởng | Ít áp dụng Retry | Tương thích |
| Câu lệnh kèm `RETURNING` | 1 Truy hồi | Tập rỗng (Empty Set) | Ít áp dụng | Tương thích |
| Lock dữ liệu `FOR UPDATE` | 2 Truy hồi | Wait Lock Timeout | Có khả năng | Khá tương thích, dễ chèn lấp kết nối |
| Lock lạc quan (`@Version`) | 2 Truy hồi | Rớt do Version Check | Có khả năng Retry | Tương thích |

## 14. Tổng Kết Hướng Dẫn Vận Hành (Production Checklist)

- [ ] Lệnh kiểm tra nghiệp vụ và lệnh thay đổi dữ liệu đã được tổng hợp chung trong 1 mệnh đề.
- [ ] Tính năng thao tác số học sử dụng toán tử tham chiếu biến đổi tương đối (`field = field - delta`).
- [ ] Logic trả về đã phân hóa rõ kết quả thao tác của số `0` hoặc `1` (Affected Rows).
- [ ] Hợp nhất lỗi hệ thống với định vị mã ID.
- [ ] Schema có hệ thống Ràng buộc phòng thủ (Constraints Defensive checks).
- [ ] Đơn hàng có tham số Identifier để duy trì tính độc lập.
- [ ] Outbox event được cấu hình trong cùng 1 session database (Single transaction).
- [ ] Yêu cầu áp đặt các Timeout phù hợp lên luồng xử lý Transaction (Query Limits).
- [ ] Dùng công cụ Testcontainers giả lập database thật.
- [ ] Triển khai các phương thức kiểm tra Reconciliation (Đồng bộ kho kiểm kê hệ thống).
