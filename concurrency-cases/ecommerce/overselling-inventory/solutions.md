# Các phương án ngăn bán vượt tồn kho

## 1. Mục tiêu thiết kế

Giải pháp phải bảo đảm các yêu cầu sau:

1. Không chấp nhận lần giữ hàng làm `available_quantity` xuống dưới `0`.
2. Quyết định đủ hàng và thao tác trừ hàng được bảo vệ tại PostgreSQL.
3. Bản ghi giữ hàng và thay đổi bộ đếm cùng chốt hoặc cùng hoàn tác.
4. Hoạt động đúng khi ứng dụng chạy nhiều máy chủ.
5. Phân biệt hết hàng với lỗi chờ khóa, bế tắc và lỗi kết nối.

Với bất biến nằm trên một dòng sản phẩm, cập nhật có điều kiện là phương án được
khuyến nghị vì nó bảo vệ đúng chỗ và chỉ cần một câu lệnh ghi.

## 2. Ràng buộc phòng thủ trong cơ sở dữ liệu

```sql
CREATE TABLE inventory_item (
    product_id BIGINT PRIMARY KEY,
    on_hand_quantity INTEGER NOT NULL,
    available_quantity INTEGER NOT NULL,
    reserved_quantity INTEGER NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    CONSTRAINT ck_inventory_non_negative CHECK (
        on_hand_quantity >= 0
        AND available_quantity >= 0
        AND reserved_quantity >= 0
    ),
    CONSTRAINT ck_inventory_conservation CHECK (
        available_quantity + reserved_quantity = on_hand_quantity
    )
);

CREATE TABLE inventory_reservation (
    reservation_id UUID PRIMARY KEY,
    order_id UUID NOT NULL,
    product_id BIGINT NOT NULL REFERENCES inventory_item(product_id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    status VARCHAR(24) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    CONSTRAINT ck_reservation_status CHECK (
        status IN ('RESERVED', 'RELEASED')
    )
);

CREATE INDEX ix_reservation_product_status
    ON inventory_reservation(product_id, status);
```

Các ràng buộc là lớp bảo vệ cuối cùng đối với trạng thái của từng dòng. Tiến
trình đối soát vẫn cần kiểm tra bộ đếm với tổng các bản ghi giữ hàng vì `CHECK` không
được dùng để đọc bảng khác.

## 3. Phương án khuyến nghị: cập nhật có điều kiện

### Câu lệnh cốt lõi

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity,
    version = version + 1
WHERE product_id = :productId
  AND available_quantity >= :quantity
RETURNING product_id,
          available_quantity,
          reserved_quantity,
          version;
```

Điều kiện `available_quantity >= :quantity` được kiểm tra tại thời điểm ghi.
Phép trừ dùng chính giá trị hiện tại của dòng. Một dòng trả về nghĩa là giữ hàng
thành công; kết quả rỗng nghĩa là không có dòng nào đủ điều kiện.

### Dữ liệu đầu vào và đầu ra

```java
public record ReserveStock(
        UUID orderId,
        long productId,
        int quantity
) {
    public ReserveStock {
        Objects.requireNonNull(orderId, "orderId");
        if (quantity <= 0) {
            throw new IllegalArgumentException(
                    "quantity must be positive"
            );
        }
    }
}

public record StockAfterReserve(
        long productId,
        int availableQuantity,
        int reservedQuantity,
        long version
) {
}

public enum ReserveOutcome {
    RESERVED,
    NOT_AVAILABLE
}

public record ReserveResult(
        UUID orderId,
        UUID reservationId,
        ReserveOutcome outcome,
        Integer remainingAvailable
) {
    public static ReserveResult reserved(
            UUID orderId,
            UUID reservationId,
            int remainingAvailable
    ) {
        return new ReserveResult(
                orderId,
                reservationId,
                ReserveOutcome.RESERVED,
                remainingAvailable
        );
    }

    public static ReserveResult notAvailable(UUID orderId) {
        return new ReserveResult(
                orderId,
                null,
                ReserveOutcome.NOT_AVAILABLE,
                null
        );
    }
}
```

Tên `NOT_AVAILABLE` được dùng thay vì mặc định là `OUT_OF_STOCK`, vì không có
dòng trả về cũng có thể do sản phẩm không tồn tại. Nếu hệ thống bảo đảm dòng tồn
kho luôn tồn tại và không bị xóa cứng, lớp API có thể ánh xạ kết quả này thành
“hết hàng”.

### Lớp cập nhật tồn kho

`UPDATE ... RETURNING` phù hợp với JDBC vì có thể lấy kết quả mới ngay trong
cùng câu lệnh:

```java
@Repository
public class InventoryStockDao {

    private final NamedParameterJdbcTemplate jdbc;

    public InventoryStockDao(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<StockAfterReserve> tryReserve(
            long productId,
            int quantity
    ) {
        List<StockAfterReserve> rows = jdbc.query("""
                UPDATE inventory_item
                SET available_quantity = available_quantity - :quantity,
                    reserved_quantity = reserved_quantity + :quantity,
                    version = version + 1
                WHERE product_id = :productId
                  AND available_quantity >= :quantity
                RETURNING product_id,
                          available_quantity,
                          reserved_quantity,
                          version
                """, Map.of(
                "productId", productId,
                "quantity", quantity
        ), (rs, rowNumber) -> new StockAfterReserve(
                rs.getLong("product_id"),
                rs.getInt("available_quantity"),
                rs.getInt("reserved_quantity"),
                rs.getLong("version")
        ));

        if (rows.size() > 1) {
            throw new IllegalStateException(
                    "one product update returned multiple rows"
            );
        }
        return rows.stream().findFirst();
    }
}
```

Đầu vào đã từ chối số lượng không dương trước khi chạy SQL. Vì `product_id` là
khóa chính, kết quả hợp lệ chỉ có thể chứa `0` hoặc `1` dòng.

### Lớp ghi bản giữ hàng

```java
@Repository
public class InventoryReservationDao {

    private final NamedParameterJdbcTemplate jdbc;

    public InventoryReservationDao(
            NamedParameterJdbcTemplate jdbc
    ) {
        this.jdbc = jdbc;
    }

    public void insertReserved(
            UUID reservationId,
            ReserveStock command,
            Instant createdAt
    ) {
        int changed = jdbc.update("""
                INSERT INTO inventory_reservation (
                    reservation_id,
                    order_id,
                    product_id,
                    quantity,
                    status,
                    created_at
                ) VALUES (
                    :reservationId,
                    :orderId,
                    :productId,
                    :quantity,
                    'RESERVED',
                    :createdAt
                )
                """, new MapSqlParameterSource()
                .addValue("reservationId", reservationId)
                .addValue("orderId", command.orderId())
                .addValue("productId", command.productId())
                .addValue("quantity", command.quantity())
                .addValue("createdAt", Timestamp.from(createdAt)));

        if (changed != 1) {
            throw new IllegalStateException(
                    "expected one reservation row, got " + changed
            );
        }
    }
}
```

### Ranh giới giao dịch

```java
@Service
public class InventoryReservationTx {

    private final InventoryStockDao stock;
    private final InventoryReservationDao reservations;
    private final Clock clock;

    public InventoryReservationTx(
            InventoryStockDao stock,
            InventoryReservationDao reservations,
            Clock clock
    ) {
        this.stock = stock;
        this.reservations = reservations;
        this.clock = clock;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReserveResult reserve(ReserveStock command) {
        Optional<StockAfterReserve> changed = stock.tryReserve(
                command.productId(),
                command.quantity()
        );

        if (changed.isEmpty()) {
            return ReserveResult.notAvailable(command.orderId());
        }

        StockAfterReserve current = changed.orElseThrow();
        UUID reservationId = UUID.randomUUID();

        reservations.insertReserved(
                reservationId,
                command,
                clock.instant()
        );

        return ReserveResult.reserved(
                command.orderId(),
                reservationId,
                current.availableQuantity()
        );
    }
}
```

Spring chỉ trao kết quả cho phía gọi sau khi phương thức đi qua lớp đại diện giao
dịch và việc chốt không phát sinh lỗi. Nếu `insertReserved()` thất bại, ngoại lệ
làm toàn bộ giao dịch hoàn tác, kể cả câu cập nhật tồn kho đã trả về một dòng.

## 4. Giới hạn thời gian chờ khóa

Sản phẩm có nhiều người mua có thể tạo hàng đợi. Có thể đặt giới hạn trong phạm
vi giao dịch:

```java
@Component
public class InventoryStatementPolicy {

    private final NamedParameterJdbcTemplate jdbc;

    public InventoryStatementPolicy(
            NamedParameterJdbcTemplate jdbc
    ) {
        this.jdbc = jdbc;
    }

    public void apply(Duration lockTimeout) {
        jdbc.queryForObject("""
                SELECT set_config('lock_timeout', :value, true)
                """, Map.of(
                "value", lockTimeout.toMillis() + "ms"
        ), String.class);
    }
}
```

Tham số thứ ba của `set_config` là `true`, nên cấu hình chỉ tồn tại trong giao
dịch hiện tại. Hết thời gian chờ khóa tạo lỗi kỹ thuật `55P03`; không được đổi lỗi
này thành kết quả hết hàng.

## 5. Phiên bản chỉ cần số dòng bị ảnh hưởng

Nếu không cần số lượng còn lại, Spring Data JPA có thể trả về số dòng cập nhật:

```java
public interface InventoryMutationRepository
        extends JpaRepository<InventoryItem, Long> {

    @Modifying(flushAutomatically = true, clearAutomatically = true)
    @Query(value = """
            UPDATE inventory_item
            SET available_quantity = available_quantity - :quantity,
                reserved_quantity = reserved_quantity + :quantity,
                version = version + 1
            WHERE product_id = :productId
              AND available_quantity >= :quantity
            """, nativeQuery = true)
    int reserveIfEnough(
            @Param("productId") long productId,
            @Param("quantity") int quantity
    );
}
```

Mã gọi phải kiểm tra `changed == 1` trước khi tạo bản ghi giữ hàng. Việc tự động
`clear()` có thể loại bỏ các thay đổi chưa ghi của thực thể khác; chỉ dùng khi
ranh giới giao dịch và ngữ cảnh Hibernate đã được kiểm soát rõ.

## 6. Phương án thay thế: khóa lạc quan với `@Version`

Khóa lạc quan phù hợp khi cần thao tác trên thực thể và xung đột hiếm:

```java
@Version
@Column(name = "version", nullable = false)
private long version;

public void reserve(int quantity) {
    if (!hasEnough(quantity)) {
        throw new IllegalStateException("insufficient stock");
    }
    availableQuantity -= quantity;
    reservedQuantity += quantity;
    // Không tự tăng version; Hibernate quản lý thuộc tính @Version.
}
```

Khi thuộc tính đã có `@Version`, mã nghiệp vụ không được tự tăng nó. Hibernate
tự đặt phiên bản kế tiếp và dùng phiên bản cũ trong điều kiện của câu ghi.

Hibernate phát sinh câu lệnh có điều kiện phiên bản:

```sql
UPDATE inventory_item
SET available_quantity = ?,
    reserved_quantity = ?,
    version = ?
WHERE product_id = ?
  AND version = ?;
```

Hai giao dịch cùng đọc `version = 0`. Bên chốt trước đổi phiên bản thành `1`.
Câu ghi của bên còn lại ảnh hưởng `0` dòng và Hibernate báo xung đột lạc quan.

```java
@Service
public class VersionedInventoryTx {

    private final InventoryItemRepository items;
    private final InventoryReservationRepository reservations;
    private final Clock clock;

    // Constructor injection omitted.

    @Transactional
    public ReserveResult reserve(ReserveStock command) {
        InventoryItem item = items.findById(command.productId())
                .orElseThrow(ProductNotFoundException::new);

        if (!item.hasEnough(command.quantity())) {
            return ReserveResult.notAvailable(command.orderId());
        }

        item.reserve(command.quantity());
        InventoryReservation reservation =
                InventoryReservation.reserved(command.orderId(), command.productId(), command.quantity(), clock.instant());
        reservations.save(reservation);
        items.flush(); // Đưa điểm phát hiện xung đột vào trong phương thức.

        return ReserveResult.reserved(
                command.orderId(),
                reservation.getReservationId(),
                item.getAvailableQuantity()
        );
    }
}
```

Nếu thử lại, mỗi lần phải mở giao dịch mới và tải lại tồn kho. Không đặt vòng lặp
thử lại bên trong chính phương thức `@Transactional`. Với sản phẩm nóng, nhiều
bên cùng đọc rồi thất bại có thể tạo bão thử lại; khi đó cập nhật có điều kiện
thường có chi phí thấp hơn.

## 7. Phương án thay thế: `FOR UPDATE`

Khóa bi quan phù hợp khi quyết định cần nhiều thuộc tính không thể diễn đạt gọn
trong một câu `UPDATE`:

```java
public interface InventoryItemRepository
        extends JpaRepository<InventoryItem, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
            select item
            from InventoryItem item
            where item.productId = :productId
            """)
    Optional<InventoryItem> findForUpdate(
            @Param("productId") long productId
    );
}
```

```java
@Transactional
public ReserveResult reserve(ReserveStock command) {
    InventoryItem item = items.findForUpdate(command.productId())
            .orElseThrow(ProductNotFoundException::new);

    if (!item.hasEnough(command.quantity())) {
        return ReserveResult.notAvailable(command.orderId());
    }

    item.reserve(command.quantity());
    InventoryReservation reservation =
            InventoryReservation.reserved(command.orderId(), command.productId(), command.quantity(), clock.instant());
    reservations.save(reservation);
    return ReserveResult.reserved(
            command.orderId(),
            reservation.getReservationId(),
            item.getAvailableQuantity()
    );
}
```

Giao dịch đến sau chờ ngay ở bước đọc. Khóa được giữ đến khi chốt hoặc hoàn tác,
vì vậy phần xử lý sau `findForUpdate()` phải ngắn và không chứa lời gọi mạng.

## 8. So sánh các phương án

| Phương án | Nơi phát hiện xung đột | Bên thua | Điểm phù hợp | Hạn chế chính |
| --- | --- | --- | --- | --- |
| `UPDATE` có điều kiện | Mệnh đề `WHERE` tại PostgreSQL | Chờ rồi nhận `0` dòng hoặc lỗi chờ khóa | Điều kiện nằm trên một dòng | Khó biểu diễn quyết định phức tạp |
| `@Version` | Số dòng của câu ghi theo phiên bản | Nhận xung đột, có thể thử lại | Xung đột hiếm, cần mô hình thực thể | Tốn công đã làm trước khi phát hiện; dễ khuếch đại thử lại |
| `FOR UPDATE` | Ngay khi đọc dòng | Chờ, hết thời gian hoặc tiếp tục với dữ liệu mới | Java cần đọc rồi quyết định | Giữ khóa lâu hơn và thêm lượt truy vấn |
| `SERIALIZABLE` | PostgreSQL phát hiện chu trình phụ thuộc | Một giao dịch nhận `40001` | Bất biến trải nhiều dòng, khó khóa rõ | Bắt buộc có vòng thử lại toàn giao dịch |

Không cần khóa phân tán khi một PostgreSQL duy nhất là nơi có thẩm quyền đối với
dòng tồn kho. Thêm một khóa bên ngoài chỉ tạo thêm trạng thái phải điều phối.

## 9. Hoàn tác và thử lại

- Kết quả không đủ hàng là quyết định nghiệp vụ, không tự động thử lại.
- Lỗi `55P03`, `40P01`, `40001` có thể thử lại nếu toàn bộ yêu cầu có thể phát
  lại an toàn.
- Mỗi lần thử dùng giao dịch mới, có giới hạn số lần và khoảng lùi.
- Mất kết nối sau khi gửi `COMMIT` tạo kết quả chưa rõ; cơ chế chống xử lý trùng
  thuộc `ECOM-003`, không được giả định giao dịch đã thất bại.

## 10. Trả hàng đã giữ

Không được chỉ cộng ngược số lượng theo một yêu cầu HTTP bất kỳ. Việc trả hàng
cần chuyển một bản ghi giữ hàng từ `RESERVED` sang `RELEASED` đúng một lần, rồi mới
cộng lại bộ đếm trong cùng giao dịch. Nếu hai yêu cầu hủy cùng tranh chấp, điều
kiện trạng thái phải bảo đảm chỉ một bên thực hiện phép cộng.

Chi tiết vòng đời giữ và hết hạn không thuộc phạm vi ECOM-001.

## 11. Khi lượng truy cập rất cao

Mọi phương án đúng vẫn phải tuần tự hóa thay đổi tại dòng sản phẩm. Không tăng
vùng kết nối hoặc thử lại vô hạn để che hàng đợi khóa. Cần đo thời gian chờ,
giới hạn công việc nhận vào và chuyển sang thiết kế của `ECOM-002` khi một sản
phẩm trở thành điểm nóng kéo dài.

## 12. Danh sách kiểm tra trước khi triển khai

- [ ] Điều kiện đủ hàng nằm trong cơ chế ghi có thẩm quyền.
- [ ] Số lượng được trừ tương đối từ giá trị hiện tại.
- [ ] Mã nguồn xử lý rõ kết quả `0` và `1` dòng.
- [ ] Bản ghi giữ hàng dùng cùng giao dịch với thay đổi tồn kho.
- [ ] Lỗi ở bước sau làm hoàn tác toàn bộ giao dịch.
- [ ] Không có lời gọi mạng khi đang giữ khóa dòng.
- [ ] Lỗi chờ khóa không bị đổi thành lỗi hết hàng.
- [ ] Thử lại mở giao dịch mới và có giới hạn.
- [ ] Không trộn thực thể cũ với SQL trực tiếp mà thiếu `flush`/`clear`.
- [ ] Có đối soát giữa bộ đếm và các bản ghi giữ hàng.
- [ ] Kiểm thử dùng PostgreSQL thật qua Testcontainers.
