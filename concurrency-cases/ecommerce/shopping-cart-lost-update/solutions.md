# Thiết kế giỏ hàng an toàn trước thao tác đồng thời

## 1. Mục tiêu thiết kế

Giải pháp chính phải bảo đảm:

1. Mọi yêu cầu sửa giỏ mang theo phiên bản mà phía gọi đã đọc.
2. Cùng một phiên bản chỉ một giao dịch được quyền tạo phiên bản kế tiếp.
3. Thay đổi dòng gốc và các dòng sản phẩm cùng chốt hoặc cùng hoàn tác.
4. Không đường ghi nào sửa `cart_item` mà bỏ qua phiên bản toàn giỏ.
5. Xung đột do dữ liệu cũ được tách khỏi lỗi khóa, bế tắc và lỗi hạ tầng.
6. Không tự thử lại những lệnh có thể làm đổi ý định người dùng.
7. Phản hồi thành công chứa đúng trạng thái và phiên bản đã chốt.

## 2. Lược đồ dữ liệu

```sql
CREATE TABLE shopping_cart (
    cart_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    status VARCHAR(20) NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT ck_shopping_cart_status
        CHECK (status IN ('ACTIVE', 'CHECKED_OUT', 'ABANDONED')),
    CONSTRAINT ck_shopping_cart_version
        CHECK (version >= 0)
);

CREATE UNIQUE INDEX uk_customer_active_cart
    ON shopping_cart (customer_id)
    WHERE status = 'ACTIVE';

CREATE TABLE cart_item (
    cart_id UUID NOT NULL,
    product_id UUID NOT NULL,
    quantity INTEGER NOT NULL,
    added_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,

    CONSTRAINT pk_cart_item
        PRIMARY KEY (cart_id, product_id),
    CONSTRAINT fk_cart_item_cart
        FOREIGN KEY (cart_id)
        REFERENCES shopping_cart(cart_id)
        ON DELETE CASCADE,
    CONSTRAINT ck_cart_item_quantity
        CHECK (quantity > 0 AND quantity <= 999)
);
```

Giới hạn `999` chỉ minh họa một ràng buộc phải có giá trị do nghiệp vụ quy định;
không sao chép con số này nếu hệ thống thật có giới hạn khác. Khóa chính ghép bảo
đảm mỗi sản phẩm chỉ có một dòng trong giỏ.

Giỏ không phải nơi giữ tồn kho. Số lượng hợp lệ trong bảng này không bảo đảm kho
còn hàng khi checkout.

## 3. Hợp đồng yêu cầu

```java
public sealed interface CartMutation
    permits AddItem, SetQuantity, RemoveItem {

    UUID cartId();
    long expectedVersion();
}

public record AddItem(
    UUID cartId,
    UUID productId,
    int quantityDelta,
    long expectedVersion,
    UUID commandId
) implements CartMutation {
    public AddItem {
        Objects.requireNonNull(cartId);
        Objects.requireNonNull(productId);
        Objects.requireNonNull(commandId);
        if (quantityDelta <= 0 || expectedVersion < 0) {
            throw new IllegalArgumentException("invalid add-item request");
        }
    }
}

public record SetQuantity(
    UUID cartId,
    UUID productId,
    int targetQuantity,
    long expectedVersion
) implements CartMutation {
    public SetQuantity {
        Objects.requireNonNull(cartId);
        Objects.requireNonNull(productId);
        if (targetQuantity <= 0 || expectedVersion < 0) {
            throw new IllegalArgumentException("invalid quantity request");
        }
    }
}

public record RemoveItem(
    UUID cartId,
    UUID productId,
    long expectedVersion
) implements CartMutation {
    public RemoveItem {
        Objects.requireNonNull(cartId);
        Objects.requireNonNull(productId);
        if (expectedVersion < 0) {
            throw new IllegalArgumentException("invalid expected version");
        }
    }
}
```

`commandId` đặc biệt quan trọng nếu `AddItem` được tự thử lại hoặc gửi lại sau
khi mất phản hồi, vì cùng một chênh lệch không được cộng hai lần. Để ngắn gọn,
phần triển khai chính bên dưới tập trung vào kiểm tra phiên bản; hệ thống thật
nên lưu mã lệnh và kết quả theo nguyên tắc của
[tính lũy đẳng](../../concepts/idempotency-and-uniqueness.md).

## 4. Giải pháp chính: giành phiên bản bằng SQL

### 4.1. Kho dữ liệu của dòng gốc

```java
@Repository
public class CartVersionStore {

    private final NamedParameterJdbcTemplate jdbc;

    public CartVersionStore(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public OptionalLong claimNextVersion(UUID cartId, long expectedVersion) {
        List<Long> versions = jdbc.query(
            """
            UPDATE shopping_cart
               SET version = version + 1,
                   updated_at = CURRENT_TIMESTAMP
             WHERE cart_id = :cartId
               AND version = :expectedVersion
               AND status = 'ACTIVE'
            RETURNING version
            """,
            Map.of(
                "cartId", cartId,
                "expectedVersion", expectedVersion
            ),
            (rs, rowNum) -> rs.getLong("version")
        );

        return versions.isEmpty()
            ? OptionalLong.empty()
            : OptionalLong.of(versions.getFirst());
    }

    public Optional<CartHead> findHead(UUID cartId) {
        return jdbc.query(
            """
            SELECT cart_id, status, version
              FROM shopping_cart
             WHERE cart_id = :cartId
            """,
            Map.of("cartId", cartId),
            (rs, rowNum) -> new CartHead(
                rs.getObject("cart_id", UUID.class),
                CartStatus.valueOf(rs.getString("status")),
                rs.getLong("version")
            )
        ).stream().findFirst();
    }
}
```

`RETURNING` vừa cho biết giao dịch có thắng hay không, vừa trả phiên bản mới.
Không tách thành `SELECT version` rồi `UPDATE`, vì khoảng hở giữa hai câu lại mở
ra một cuộc đua.

### 4.2. Kho dữ liệu sản phẩm

```java
@Repository
public class CartItemStore {

    private final NamedParameterJdbcTemplate jdbc;

    public void add(UUID cartId, UUID productId, int delta) {
        int changed = jdbc.update(
            """
            INSERT INTO cart_item (
                cart_id, product_id, quantity, added_at, updated_at
            )
            VALUES (
                :cartId, :productId, :delta,
                CURRENT_TIMESTAMP, CURRENT_TIMESTAMP
            )
            ON CONFLICT (cart_id, product_id)
            DO UPDATE
               SET quantity = cart_item.quantity + EXCLUDED.quantity,
                   updated_at = CURRENT_TIMESTAMP
             WHERE cart_item.quantity + EXCLUDED.quantity <= 999
            """,
            Map.of(
                "cartId", cartId,
                "productId", productId,
                "delta", delta
            )
        );

        if (changed != 1) {
            throw new CartQuantityLimitException(productId);
        }
    }

    public void setQuantity(UUID cartId, UUID productId, int target) {
        int changed = jdbc.update(
            """
            UPDATE cart_item
               SET quantity = :target,
                   updated_at = CURRENT_TIMESTAMP
             WHERE cart_id = :cartId
               AND product_id = :productId
            """,
            Map.of(
                "cartId", cartId,
                "productId", productId,
                "target", target
            )
        );

        if (changed != 1) {
            throw new CartItemNotFoundException(productId);
        }
    }

    public void remove(UUID cartId, UUID productId) {
        int changed = jdbc.update(
            """
            DELETE FROM cart_item
             WHERE cart_id = :cartId
               AND product_id = :productId
            """,
            Map.of("cartId", cartId, "productId", productId)
        );

        if (changed != 1) {
            throw new CartItemNotFoundException(productId);
        }
    }
}
```

Các phương thức này không được gọi trực tiếp từ bộ điều khiển HTTP hay tác vụ
nền. Chúng chỉ là bước sau khi giao dịch đã giành phiên bản dòng gốc.

### 4.3. Một lần thực hiện giao dịch

```java
@Service
public class CartMutationAttempt {

    private final CartVersionStore versions;
    private final CartItemStore items;
    private final CartViewStore views;

    @Transactional
    public CartView add(AddItem command) {
        long nextVersion = claim(command.cartId(), command.expectedVersion());
        items.add(command.cartId(), command.productId(), command.quantityDelta());
        return views.load(command.cartId(), nextVersion);
    }

    @Transactional
    public CartView setQuantity(SetQuantity command) {
        long nextVersion = claim(command.cartId(), command.expectedVersion());
        items.setQuantity(
            command.cartId(),
            command.productId(),
            command.targetQuantity()
        );
        return views.load(command.cartId(), nextVersion);
    }

    @Transactional
    public CartView remove(RemoveItem command) {
        long nextVersion = claim(command.cartId(), command.expectedVersion());
        items.remove(command.cartId(), command.productId());
        return views.load(command.cartId(), nextVersion);
    }

    private long claim(UUID cartId, long expectedVersion) {
        OptionalLong next = versions.claimNextVersion(cartId, expectedVersion);
        if (next.isPresent()) {
            return next.getAsLong();
        }

        CartHead current = versions.findHead(cartId)
            .orElseThrow(CartNotFoundException::new);

        if (current.status() != CartStatus.ACTIVE) {
            throw new CartNotMutableException(current.status());
        }

        throw new StaleCartVersionException(
            expectedVersion,
            current.version()
        );
    }
}
```

Nếu sửa sản phẩm hoặc tải ảnh chụp kết quả thất bại, ngoại lệ thoát khỏi phương
thức và Spring hoàn tác cả lần tăng phiên bản. Không bắt ngoại lệ rồi trả một kết
quả bình thường bên trong giao dịch.

`CartViewStore.load` phải đọc từ cùng kết nối giao dịch. Phản hồi chỉ đi ra khỏi
lớp quản lý giao dịch sau khi `COMMIT` hoàn tất.

## 5. Ánh xạ xung đột ở ngoài giao dịch

```java
@RestControllerAdvice
public class CartErrorHandler {

    @ExceptionHandler(StaleCartVersionException.class)
    ResponseEntity<CartConflictBody> stale(StaleCartVersionException error) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(
            new CartConflictBody(
                "CART_VERSION_CONFLICT",
                error.expectedVersion(),
                error.currentVersion()
            )
        );
    }

    @ExceptionHandler(CartNotMutableException.class)
    ResponseEntity<ProblemDetail> notMutable(CartNotMutableException error) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        problem.setTitle("CART_NOT_MUTABLE");
        problem.setDetail("Cart is no longer active");
        return ResponseEntity.status(HttpStatus.CONFLICT).body(problem);
    }
}
```

Nếu API dùng `If-Match` đúng theo điều kiện HTTP, `412 Precondition Failed` là
lựa chọn tự nhiên. Nếu phiên bản nằm trong lệnh nghiệp vụ, `409 Conflict` thường
dễ dùng. Chọn một quy ước nhất quán và ghi rõ trong hợp đồng API.

Lỗi `CannotAcquireLockException`, `DeadlockLoserDataAccessException` hoặc lỗi
kết nối không được ánh xạ thành phiên bản cũ. Chúng là lỗi kỹ thuật và chỉ được
thử lại theo chính sách riêng.

## 6. Vì sao giải pháp chính hoạt động

1. Câu `UPDATE` trên `shopping_cart` vừa kiểm tra vừa tăng phiên bản trong một
   thao tác nguyên tử.
2. Khóa dòng PostgreSQL tạo thứ tự giữa các giao dịch dùng cùng phiên bản.
3. Sau khi bên thắng chốt, điều kiện của bên chờ được kiểm tra lại và không còn
   đúng.
4. Bên thua dừng trước khi sửa `cart_item`.
5. Dòng gốc và dòng con dùng cùng giao dịch, nên lỗi giữa chừng không để lại
   phiên bản hay nội dung nửa vời.
6. Cơ chế nằm trong cơ sở dữ liệu nên có hiệu lực với nhiều máy chủ.

## 7. Thay toàn bộ giỏ một cách an toàn

Nếu cần duy trì API thay toàn bộ, vẫn phải giành phiên bản trước:

```java
@Transactional
public CartView replace(ReplaceCart command) {
    long nextVersion = claim(command.cartId(), command.expectedVersion());
    validateNoDuplicateProducts(command.items());

    itemStore.deleteAll(command.cartId());
    itemStore.insertAll(command.cartId(), command.items());

    return viewStore.load(command.cartId(), nextVersion);
}
```

`DELETE` và `INSERT` hàng loạt nằm trong cùng giao dịch nên trạng thái rỗng tạm
thời không lộ ra ngoài sau khi chốt. Dù vậy, API lệnh nhỏ vẫn tốt hơn vì thể hiện
ý định và giảm lượng ghi.

Không tự thử lại `replace`. Khi phiên bản không khớp, trả ảnh chụp mới để phía
gọi gộp có chủ đích hoặc yêu cầu người dùng xác nhận.

## 8. Biến thể JPA với `@Version`

### 8.1. Gốc tập hợp

```java
@Entity
@Table(name = "shopping_cart")
public class ShoppingCart {

    @Id
    @Column(name = "cart_id", nullable = false)
    private UUID id;

    @Version
    @Column(name = "version", nullable = false)
    private long version;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private CartStatus status;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @OneToMany(
        mappedBy = "cart",
        cascade = CascadeType.ALL,
        orphanRemoval = true
    )
    private final List<CartItem> items = new ArrayList<>();

    public void setQuantity(UUID productId, int target, Clock clock) {
        ensureActive();
        CartItem item = findItem(productId);
        item.setQuantity(target);
        updatedAt = clock.instant(); // buộc gốc tham gia lần ghi phiên bản
    }

    public long version() {
        return version;
    }
}
```

Không cung cấp setter cho `version` và không sao chép giá trị này từ JSON vào
thực thể. Trường `updatedAt` thay đổi giúp câu cập nhật gốc được sinh rõ ràng;
vẫn phải kiểm tra SQL thực tế của phiên bản Hibernate đang dùng.

### 8.2. Dịch vụ giao dịch

```java
@Service
public class JpaCartMutationAttempt {

    private final ShoppingCartRepository carts;
    private final EntityManager entityManager;
    private final Clock clock;

    @Transactional
    public CartView setQuantity(SetQuantity command) {
        ShoppingCart cart = carts.findWithItemsById(command.cartId())
            .orElseThrow(CartNotFoundException::new);

        if (cart.version() != command.expectedVersion()) {
            throw new StaleCartVersionException(
                command.expectedVersion(),
                cart.version()
            );
        }

        cart.setQuantity(command.productId(), command.targetQuantity(), clock);
        entityManager.flush();

        return CartView.from(cart);
    }
}
```

Kiểm tra sớm trong Java cho phản hồi dễ hiểu khi yêu cầu đã cũ trước lúc tải.
Nó không thay thế `@Version`: hai giao dịch vẫn có thể cùng vượt qua kiểm tra,
và chỉ điều kiện trong câu `UPDATE` mới phân xử cuộc đua.

`flush()` làm ngoại lệ xuất hiện trong phương thức, nhưng bộ chặn giao dịch của
Spring (`transaction interceptor`) vẫn là nơi chốt. Bộ điều khiển HTTP hoặc một
dịch vụ phía ngoài giao dịch bắt `ObjectOptimisticLockingFailureException` và ánh
xạ thành xung đột. Không dùng lại thực thể hay vùng quản lý thực thể sau ngoại
lệ.

## 9. Thử lại có chọn lọc cho lệnh tăng chênh lệch

Chỉ lệnh có ngữ nghĩa cộng thêm mới là ứng viên để thử lại tự động:

```java
@Service
public class AddItemCoordinator {

    private final CartMutationAttempt attempt;
    private final CartQuery cartQuery;

    public CartView addWithBoundedRetry(AddItem original) {
        AddItem current = original;

        for (int number = 1; number <= 3; number++) {
            try {
                return attempt.add(current); // proxy khác, giao dịch mới mỗi lần
            } catch (StaleCartVersionException conflict) {
                if (number == 3) {
                    throw conflict;
                }
                long latest = cartQuery.requireActive(original.cartId()).version();
                current = new AddItem(
                    original.cartId(),
                    original.productId(),
                    original.quantityDelta(),
                    latest,
                    original.commandId()
                );
            }
        }

        throw new IllegalStateException("unreachable");
    }
}
```

Con số lần thử phải là cấu hình và được đo, không phải lời hứa rằng ba lần luôn
đủ. Trước khi dùng mẫu này, cần thêm bảng chiếm `commandId`; nếu không, sự cố mất
phản hồi sau `COMMIT` vẫn có thể cộng lại chênh lệch.

Không áp bộ phối hợp này cho `SetQuantity`, `RemoveItem` hoặc `ReplaceCart` nếu
nghiệp vụ chưa định nghĩa rõ cách gộp.

## 10. Các lựa chọn khác

### 10.1. Khóa bi quan

```sql
SELECT cart_id, version
FROM shopping_cart
WHERE cart_id = :cartId
  AND status = 'ACTIVE'
FOR UPDATE;
```

Sau khi giữ khóa, tải và sửa sản phẩm trong giao dịch ngắn. Cách này giảm số
giao dịch thất bại khi xung đột dày, nhưng bên thua phải chờ, dễ tăng độ trễ và
chiếm kết nối. Vẫn cần giới hạn thời gian chờ và thứ tự khóa nhất quán.

### 10.2. Câu SQL cộng chênh lệch

`INSERT ... ON CONFLICT DO UPDATE SET quantity = quantity + delta` giữ được các
phép cộng cùng sản phẩm. Nó không giải quyết `SetQuantity`, `RemoveItem`, giới
hạn toàn giỏ hay phiên bản `ETag`. Nếu dùng, phải xác định rõ đây là một đường
lệnh riêng và vẫn cập nhật phiên bản giỏ trong cùng giao dịch.

### 10.3. Mức cô lập `SERIALIZABLE`

PostgreSQL có thể hủy một giao dịch khi phát hiện lịch sử không tuần tự hóa được.
Ứng dụng vẫn phải thử lại cả giao dịch trong giao dịch mới và vẫn cần định nghĩa
ý nghĩa của ảnh chụp phía khách. Đây không phải cách thay thế hợp đồng phiên bản
cho xung đột người dùng nhìn thấy.

### 10.4. Gộp tại phía khách

Khi nhận xung đột, giao diện có thể so sánh ảnh chụp gốc, thay đổi cục bộ và ảnh
chụp mới. Chỉ tự gộp phần không mâu thuẫn; phần cùng sản phẩm cần người dùng chọn.
Máy chủ vẫn kiểm tra phiên bản thêm lần nữa khi gửi kết quả gộp.

## 11. So sánh định tính

| Cách làm | Điểm mạnh | Đổi lại |
| --- | --- | --- |
| SQL kiểm tra phiên bản ở đầu | Phát hiện sớm, rõ ràng với nhiều bảng con | Cần đường ghi tập trung và SQL riêng |
| JPA `@Version` ở gốc | Tự nhiên với mô hình thực thể | Phải bảo đảm gốc luôn bị đổi và không có cập nhật hàng loạt bỏ qua |
| Khóa bi quan | Ít giao dịch thua khi tranh chấp dày | Chờ khóa, giữ kết nối, cần giới hạn thời gian chờ |
| Cộng chênh lệch nguyên tử | Tốt cho lệnh tăng có thể kết hợp | Không phù hợp với đặt đích hoặc thay toàn bộ |
| `SERIALIZABLE` | Phát hiện nhiều lịch sử nguy hiểm | Có thể hủy giao dịch; vẫn cần chính sách thử lại và hợp đồng API |

Không chọn giải pháp từ một con số hiệu năng giả định. Đo tỷ lệ xung đột, thời
gian giữ giao dịch, số lần thử lại và kích thước giỏ trên tải thực tế.

## 12. Danh sách kiểm tra triển khai

- [ ] `shopping_cart.version` không âm, không nhận trực tiếp từ JSON vào entity.
- [ ] Mọi lệnh mang `expectedVersion` hoặc `If-Match`.
- [ ] Mọi thay đổi sản phẩm kiểm tra và tăng phiên bản gốc trong cùng giao dịch.
- [ ] Không kho dữ liệu hoặc bộ điều khiển nào cập nhật hàng loạt `cart_item` độc lập.
- [ ] Bên thua không có thay đổi bền vững.
- [ ] Xung đột phiên bản tách khỏi hết thời gian chờ, bế tắc và lỗi kết nối.
- [ ] Không thử lại mù lệnh đặt đích hoặc thay toàn bộ.
- [ ] Lệnh tăng được gửi lại có `commandId` và kết quả phát lại.
- [ ] Test dùng PostgreSQL thật và buộc hai giao dịch cùng đọc một phiên bản.
- [ ] Số liệu theo loại lệnh, không dùng `cartId` làm nhãn không giới hạn.
