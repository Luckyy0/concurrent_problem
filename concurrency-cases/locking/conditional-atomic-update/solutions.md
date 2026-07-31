# Giải pháp Chuẩn mực — Vừa cập nhật vừa tự vệ (Guarded Mutation)

## Mục tiêu thiết kế

Giải pháp chuẩn phải đồng thời thỏa mãn 4 lớp bảo vệ:

1. Việc "Kiểm tra kho" và "Trừ kho" phải dính chặt vào nhau trong duy nhất 1 câu SQL.
2. Số lượng dòng trả về từ lệnh SQL sẽ là căn cứ duy nhất để quyết định là "Thành công" hay "Hết hàng".
3. Phải lưu nháp một mã (command claim) ngay từ đầu để chống việc bị gửi đúp (replay) dẫn đến trừ hàng 2 lần.
4. Lệnh chống trùng, lệnh trừ kho, lưu lịch sử, và lưu hộp thư ra ngoài (outbox) phải CÙNG SỐNG hoặc CÙNG CHẾT trong 1 Giao dịch (Transaction).

## Cấu trúc Bảng (Schema) đề xuất

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
    command_id UUID PRIMARY KEY, -- Mã đơn lệnh duy nhất
    reservation_id UUID UNIQUE,
    order_id UUID NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    outcome VARCHAR(24) NOT NULL, -- Kết quả
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

Trạng thái `PROCESSING` (Đang xử lý) chỉ là một trạng thái tạm thời trong lúc Giao dịch đang chạy. Ứng dụng không bao giờ được phép Chốt sổ (commit) nửa vời khi cái đơn vẫn còn nằm ở chữ `PROCESSING`.

## Trái tim của Giải pháp: Câu SQL Nguyên tử

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

Dữ liệu truyền vào `:quantity` đã được code Java chặn lại nếu là số âm. Nhờ khóa ngoại và vòng đời thiết kế chuẩn, ta chắc chắn mã `:productId` phải tồn tại. Do đó, nếu hàm `RETURNING` trả về rỗng (0 dòng) thì lý do duy nhất chỉ có thể là `OUT_OF_STOCK` (Hết hàng).

## Đối tượng chuyên chở dữ liệu (Value objects)

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
            throw new IllegalArgumentException("quantity must be positive"); // Chặn số lượng âm
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

Cái "Dấu vân tay" (Fingerprint) dùng để chống trùng mã được băm (hash) từ chính những thông tin quan trọng của Đơn hàng, chứ không được lấy 1 chuỗi băm vớ vẩn do frontend gửi lên.

## Tầng giao tiếp (DAO) để Đăng ký và Chống trùng lặp

```java
@Repository
public class ReservationCommandDao {
    private final NamedParameterJdbcTemplate jdbc;

    public ReservationCommandDao(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    // Cố gắng đăng ký quyền xử lý cái mã lệnh này
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
        return changed == 1; // 1 = Thành công chưa có ai đăng ký; 0 = Đã có người gửi mã này rồi
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

Dùng tuyệt chiêu `ON CONFLICT DO NOTHING` trên Khóa chính (Primary key) để làm chốt chặn phân xử đụng độ hoàn hảo. 
Nếu có ai đó bấm đúp đơn hàng vào cùng lúc, 1 luồng sẽ bị PostgreSQL cho đứng chờ. Sau khi luồng đi trước chốt sổ (commit), luồng đứng sau sẽ được đánh giá lại và query ra cái trạng thái chốt sổ từ DB chứ không bị chạy lại toàn bộ quy trình. 

## Tầng giao tiếp (DAO) để cập nhật Kho kèm hàm `RETURNING`

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

Lớp DAO này KHÔNG HỀ tải đối tượng `InventoryItem` lên bằng JPA, thế nên bộ đệm trên RAM hoàn toàn trong sạch, không có dữ liệu thiu. Lệnh `RETURNING` tự động ném ra con số mới nhất sau khi sửa xong.

## Giới hạn thời gian chờ khóa (Bounded lock wait)

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
        // Chủ động áp thời gian chờ cho riêng kết nối này!
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

Nhớ rằng cái Cài đặt này chỉ áp dụng ranh giới giao dịch hiện tại (transaction-local). Các cấu hình này cực kỳ quan trọng, ví dụ thiết lập `750ms/1500ms`. Và đừng bao giờ tin tưởng truyền con số timeout này trực tiếp từ Frontend lên!

## Trái tim của Giao dịch (Transactional worker)

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

    // Bắt đầu một Giao dịch, nếu chết là cả đám chết chung
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReservationResult reserve(ReserveStockCommand command) {
        Instant now = clock.instant();
        statementPolicy.apply(); // Gắn luật chờ Timeout

        // Bước 1: Thử giành quyền xử lý mã đơn này
        if (!commands.tryClaim(command, now)) {
            // Có người xử lý rồi, đọc kết quả của người ta rồi trả về thôi
            StoredReservation existing =
                    commands.requireExisting(command.commandId());
            existing.requireSameRequest(command);
            return existing.toResult();
        }

        // Bước 2: Bắn SQL Nguyên tử cập nhật và giành hàng
        Optional<InventoryAfterReserve> changed =
                inventory.tryReserve(
                        command.productId(),
                        command.quantity()
                );

        // Bước 3: Rẽ nhánh nghiệp vụ dựa trên số dòng bị sửa (affected rows)
        if (changed.isEmpty()) { // Trả về 0 dòng
            commands.completeOutOfStock(command.commandId(), now); // Sửa trạng thái lại thành Hết hàng
            return ReservationResult.outOfStock(command.commandId());
        }

        // Bước 4: Lấy số liệu trả về
        InventoryAfterReserve stock = changed.orElseThrow();
        UUID reservationId = UUID.randomUUID();
        
        // Bước 5: Cập nhật lại lịch sử thành THÀNH CÔNG
        commands.completeReserved(
                command.commandId(),
                reservationId,
                stock,
                now
        );
        
        // Bước 6: Thả tin nhắn vào Hộp thư
        outbox.save(OutboxEvent.inventoryReserved(
                UUID.randomUUID(),
                reservationId,
                command,
                stock,
                now
        ));

        // Cưỡng ép xả đống lệnh này xuống DB để phát hiện lỗi khóa ngoại (nếu có) trước khi return
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

Hãy nhớ rằng JDBC Dao (SQL thuần) và JPA (Lưu Outbox) phải dùng CHUNG một Kết nối (DataSource) và một Quản lý Giao dịch (Spring transaction manager) thì chúng mới cùng sống cùng chết được. Lệnh `flush()` ở cuối đảm bảo rác rưởi lọt ra được báo lỗi sớm thay vì ngâm giấm đến cuối Giao dịch.

> **Nói ngắn gọn:** Kết quả của câu SQL sẽ là người cầm cân nảy mực phân xử thắng/bại. Còn cục Giao dịch (Transaction) sẽ quyết định sinh mạng của cục dữ liệu tồn kho, kết quả lịch sử và cục tin nhắn Hộp thư ra ngoài cùng nhau!

## Điều phối Lỗi bên ngoài Giao dịch (Map technical failure ngoài transaction)

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
        // Cảnh sát: Hàm này KHÔNG được phép có @Transactional
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            throw new IllegalStateException(
                    "coordinator must run outside a transaction"
            );
        }

        try {
            return worker.reserve(command);
        } catch (RuntimeException failure) {
            // Bước phân loại mã lỗi
            return classifier.lockOrSerializationFailure(failure)
                    .map(kind -> ReservationResult.busy(
                            command.commandId(),
                            kind.name()
                    ))
                    .orElseThrow(() -> failure); // Bị lỗi khác thì quăng luôn
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

Nơi bắt các lỗi hệ thống (catch block) PHẢI đứng bên ngoài phạm vi của `@Transactional` để đảm bảo Giao dịch đã được hủy (rollback) sạch sẽ trước khi biến nó thành kết quả báo về là "Hệ thống bận" (Busy).

Đừng có hở một chút bị Exception rồi map nó thành chữ "HẾT HÀNG", cũng không được lơ đi những lỗi Database nguy hiểm. Nếu cần tính năng "Tự động thử lại" (Retry), hãy dùng 1 transaction hoàn toàn mới mẻ, giữ nguyên ID của đơn và cấu hình thuật toán Backoff (đợi ngẫu nhiên) hợp lý. Đừng bao giờ chơi vòng lặp "Retry nã đại bác" liên tục.

## Cách dùng Spring Data JPA thuần túy (Nếu không muốn xài JdbcTemplate)

Nếu API của bạn không cần trả về con số tồn kho mới nhất, bạn có thể rút gọn lại bằng cái này:

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

Service CẦN BẮT BUỘC phải soi xem kết quả `changed == 1` hay không. Và nhắc lại lần nữa: Đừng có gọi hàm tải `InventoryItem` lên RAM trước, trừ phi bạn biết dùng các cờ dọn dẹp như `clear/refresh`. 
Có thể lạm dụng `@Modifying(flushAutomatically = true, clearAutomatically = true)` nhưng phải cẩn thận vì nó sẽ quét sạch cả các entity chả liên quan ra khỏi bộ nhớ. Đừng dùng nó như kiểu "nhắm mắt đưa chân".

## Chuỗi Hoàn tác (Rollback composition)

Nếu bước chèn Hộp thư (outbox) bị lỗi văng Exception:

```text
Lệnh UPDATE có điều kiện trả về ảnh hưởng 1 dòng
→ Lịch sử được đóng dấu là THÀNH CÔNG (RESERVED)
→ Bỗng nhiên lỗi rào chắn ở Hộp thư
→ CẢ GIAO DỊCH LẬT KÈO, HỦY BỎ (ROLLBACK)
→ Trả số tồn kho về nguyên vẹn như cũ
→ Mã đơn đăng ký/Lịch sử cũng bay màu sạch sẽ không kèn không trống
```

Khách hàng thấy rớt mạng, gửi lại Y CHANG cái mã đơn đó, code vẫn cho đi vào và đăng ký quyền như một người mới toanh.

## Trả lại hàng / Hủy đơn (Cancel/release stock)

Đổi ý hủy đơn là một hành động (state transition) hoàn toàn độc lập. Mã đơn phải chứng minh nó từng là của mình, cập nhật trạng thái từ `RESERVED` thành `CANCELLED` CHỈ 1 LẦN DUY NHẤT và cộng trả lại tồn kho. Đừng có ngây thơ viết riêng 1 câu UPDATE tự cộng hàng lên mù quáng, rủi mạng lag nó gửi 2 lần hủy đơn là bạn cộng hàng lố cho 1 đơn cũ.

## Các Giải pháp Kế vị (Alternatives)

### Cách 1: Khóa Bi quan (PESSIMISTIC_WRITE)
Nói dễ hiểu là dùng `SELECT ... FOR UPDATE`. Rất hợp nếu logic quyết định bán hàng phức tạp dính tới 2-3 bảng, nhưng nhược điểm là bạn tốn 1 cuốc đi ra đi vô DB để Đọc, và phải cắn răng ôm Khóa khóa luôn thằng khác trong lúc Code Java của bạn đang suy nghĩ.

### Cách 2: Khóa Lạc quan (`@Version`)
Thích hợp với việc sửa bự nguyên 1 màn hình và hiếm khi có 2 thằng đụng nhau. Chứ với kiểu đếm tồn kho, cách Cập nhật Có điều kiện (Atomic UPDATE) vừa gọn nhẹ lại chả phải lo cảnh 1 luồng hụt chân.

### Cách 3: Ỉ Lại Rào chắn (Constraint-only)
Rào `CHECK available >= 0` ngăn kho âm là cực tốt, nhưng nó câm như hến, chả tự biết móc nối với bảng lịch sử, nên phải dùng kèm như 1 lớp phòng thủ sâu (defense in depth).

### Cách 4: Mức cách ly tối thượng (`SERIALIZABLE`)
Bảo vệ bao la mọi lỗi, nhưng đổi lại code sẽ ném lỗi `40001` dồn dập vào mặt bạn bắt bạn phải code vòng lặp Tự thử lại (Retry). Trò chẻ dao mổ trâu giết gà, không cần thiết khi 1 cục `WHERE` đã giải quyết quá xinh đẹp.

### Cách 5: Đưa vào Hàng Đợi (Queue/serialized processing)
Giống như bắt mọi người xếp hàng mua trà sữa, tránh húc nhau sứt đầu mẻ trán. Nhưng cực kỳ rườm rà về thiết kế, hãy dùng nó vì quy mô hệ thống quá khủng, chứ đừng lấy cớ "Do em viết SQL kém" để xài.

## So sánh thực tiễn (Định tính)

| Chiến thuật | Số cuốc (Round trips) | Người thua | Tự thử lại (Retry) | Tuổi thọ khóa | Môi trường nhiều Máy chủ |
| --- | --- | --- | --- | --- | --- |
| Câu lệnh Có Điều Kiện (`UPDATE`) | 1 cuốc (Mutation) | Lấy về `0` dòng | Ít khi cần | Rất ngắn (Từ lúc phát SQL đến cuối Tx) | Dư sức |
| Kèm `RETURNING` | 1 cuốc | Về rỗng | Ít khi | Rất ngắn | Dư sức |
| `FOR UPDATE` + Ghi | 2 cuốc | Đứng ngó / Hết giờ | Không bắt buộc | Dài (Ngâm từ lúc bắt đầu đọc) | Dư sức |
| Gắn `@Version` | 2 cuốc | Rớt cái bẹp lúc Flush | Có thể | Rất ngắn | Dư sức |
| Khóa Java (`synchronized`) | Tùy duyên | Ngồi chờ | Không | Vô dụng với đa máy chủ | Nát bét |

## Lời khuyên cuối cùng (Khuyến nghị)

Với kiểu chỉ có 1 dòng dữ liệu tồn kho rõ ràng, cứ lôi Câu Lệnh Nguyên Tử `UPDATE ... RETURNING` làm võ công phòng thân. Nhớ rào kỹ logic trước SQL, quy định rõ số "0" trả về mang ý nghĩa gì, lưu Hộp thư chung Giao dịch, và nhớ CẤM JPA hốt cái Entity đó lên RAM.

Khi sản phẩm quá hot (hot-row), không phải cứ đi tháo tung trần thời gian timeout lên là ngon. Hãy đo lường và tính đường rút lui (Tham khảo thêm bài `LOCK-005`).

## Checklist trước khi đem lên Production

- [ ] Cụm `WHERE` đã chứa sạch bách logic kiểm tra tồn kho?
- [ ] Cụm `SET` đang trừ trực tiếp bằng `(cột + tham số)` chứ không phải lấy 1 số chết áp vào?
- [ ] Số dòng bị sửa đổi chỉ có thể là 0 hoặc 1?
- [ ] Số dòng trả về (affected/returned row) đã được lưu vào biến và xài `if - else` đàng hoàng?
- [ ] Con số "0 dòng" đã được định nghĩa và kiểm chứng đàng hoàng?
- [ ] Đã ốp Constraint chống số Âm trong Schema chưa?
- [ ] Kiểm tra xem có lệnh `findById` dư thừa nào tải Entity lên không?
- [ ] Mã đơn hàng đã dùng Unique ID chống trùng và vân tay (fingerprint) đã ổn định?
- [ ] Lịch sử / Hộp thư / Ghi kho đã nằm trọn vẹn trong CÙNG 1 Giao dịch `@Transactional`?
- [ ] Thời gian Timeout (Khóa và Lệnh) có nhỏ hơn tổng thời gian Deadline của HTTP Request?
- [ ] Lỗi hết giờ / kẹt xe `55P03`, `40P01` đã phân loại chưa hay nhắm mắt phán "HẾT HÀNG"?
- [ ] Đã viết Test bằng PostgreSQL THẬT (chống chỉ định H2/Mocks)?
- [ ] Đã hẹn giờ Job (Cronjob) chạy quét lệch đối soát định kỳ?
