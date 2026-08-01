# Mã nguồn thay toàn bộ giỏ gây mất cập nhật

## 1. Mô hình dữ liệu không có phiên bản

```java
@Entity
@Table(name = "shopping_cart")
public class ShoppingCart {

    @Id
    @Column(name = "cart_id", nullable = false)
    private UUID id;

    @Column(name = "customer_id", nullable = false)
    private UUID customerId;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private CartStatus status;

    @OneToMany(
        mappedBy = "cart",
        cascade = CascadeType.ALL,
        orphanRemoval = true
    )
    private final List<CartItem> items = new ArrayList<>();

    public void replaceItems(List<RequestedItem> requestedItems) {
        items.clear();
        requestedItems.forEach(request -> items.add(
            CartItem.create(this, request.productId(), request.quantity())
        ));
    }
}
```

```java
@Entity
@Table(
    name = "cart_item",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_cart_item_product",
        columnNames = {"cart_id", "product_id"}
    )
)
public class CartItem {

    @Id
    @GeneratedValue
    @Column(name = "cart_item_id", nullable = false)
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "cart_id", nullable = false)
    private ShoppingCart cart;

    @Column(name = "product_id", nullable = false)
    private UUID productId;

    @Column(name = "quantity", nullable = false)
    private int quantity;
}
```

Ràng buộc duy nhất ngăn hai dòng cùng sản phẩm trong một giỏ, nhưng không phát
hiện ảnh chụp cũ. `ShoppingCart` không có `@Version`, nên Hibernate không thêm
điều kiện phiên bản khi ghi.

## 2. API nhận toàn bộ ảnh chụp

```java
public record ReplaceCartRequest(List<CartItemRequest> items) {
}

public record CartItemRequest(UUID productId, int quantity) {
}
```

Yêu cầu không có `expectedVersion` và cũng không dùng `If-Match`. Máy chủ không
biết danh sách được tạo từ trạng thái nào.

```java
@RestController
@RequestMapping("/carts")
public class CartController {

    private final BrokenCartService cartService;

    @PutMapping("/{cartId}")
    CartResponse replace(
        @PathVariable UUID cartId,
        @RequestBody ReplaceCartRequest request
    ) {
        return cartService.replace(cartId, request);
    }
}
```

`PUT` tự nó không bảo vệ đồng thời. Không có điều kiện phiên bản, hai yêu cầu hợp
lệ về cú pháp vẫn có thể ghi đè nhau.

## 3. Đọc rồi thay toàn bộ trong Java

```java
@Service
public class BrokenCartService {

    private final ShoppingCartRepository carts;

    @Transactional
    public CartResponse replace(
        UUID cartId,
        ReplaceCartRequest request
    ) {
        ShoppingCart cart = carts.findById(cartId)
            .orElseThrow(CartNotFoundException::new);

        if (cart.getStatus() != CartStatus.ACTIVE) {
            throw new CartNotMutableException();
        }

        List<RequestedItem> replacement = request.items().stream()
            .map(item -> new RequestedItem(
                item.productId(),
                item.quantity()
            ))
            .toList();

        cart.replaceItems(replacement);
        return CartResponse.from(cart);
    }
}
```

Mỗi giao dịch có vùng quản lý thực thể riêng. Nếu cả hai cùng đọc giỏ trước khi
bên nào chốt, cả hai đều sửa bản sao của mình. `READ COMMITTED` không tự phát
hiện rằng quyết định trong Java dựa trên dữ liệu cũ.

Việc trả `CartResponse` còn diễn ra trước lúc phương thức `@Transactional` kết
thúc. Hibernate có thể chỉ `flush` khi giao dịch chuẩn bị chốt; ngoại lệ ghi dữ
liệu vì thế xuất hiện muộn hơn chỗ mã nguồn trông có vẻ đã thành công.

## 4. Dòng thời gian gây lỗi

Giỏ ban đầu có `SKU-A` số lượng `1`.

```text
Tab A / giao dịch A                 Tab B / giao dịch B
-------------------------------    -------------------------------
đọc [A:1]                          đọc [A:1]
thay bằng [A:1, B:1]               thay bằng []
flush INSERT/DELETE                 chưa ghi
COMMIT                              flush DELETE
                                    COMMIT
```

Tùy thứ tự câu lệnh và cách ánh xạ bộ sưu tập, kết quả có thể là danh sách của
bên ghi sau, hỗn hợp không mong muốn hoặc lỗi ràng buộc. Không kết quả nào cho
phép máy chủ tuyên bố an toàn rằng cả hai ý định đã được giữ.

## 5. Khóa Java chỉ che lỗi trong một JVM

```java
private final ConcurrentMap<UUID, ReentrantLock> locks =
    new ConcurrentHashMap<>();

public CartResponse replaceWithLocalLock(
    UUID cartId,
    ReplaceCartRequest request
) {
    Lock lock = locks.computeIfAbsent(cartId, ignored -> new ReentrantLock());
    lock.lock();
    try {
        return replace(cartId, request);
    } finally {
        lock.unlock();
    }
}
```

Cách này có thêm ba vấn đề:

1. Máy chủ A và B có hai bản đồ khóa khác nhau.
2. Gọi `replace()` trong cùng thực thể có thể bỏ qua proxy Spring, nên
   `@Transactional` không hoạt động như người viết tưởng.
3. Khóa có thể tích tụ theo `cartId` và không giải quyết yêu cầu phát lại sau
   khi mất phản hồi.

## 6. `save()` không phải phép kiểm tra phiên bản

```java
ShoppingCart saved = carts.save(cart);
```

`save()` chỉ yêu cầu JPA lưu trạng thái. Nếu thực thể không có `@Version`, nó
không biến thao tác thành kiểm tra rồi đổi. Với thực thể đã được tải trong cùng
giao dịch, lệnh này thường còn không cần thiết vì Hibernate tự dò thay đổi.

## 7. Chỉ đặt `@Version` trên từng sản phẩm vẫn sai ranh giới

```java
@Version
private long version; // trên CartItem
```

Cách này có thể phát hiện hai giao dịch cùng sửa một dòng `cart_item`, nhưng
không bảo vệ các tình huống:

- một tab thêm `SKU-B`, tab kia xóa `SKU-A`;
- một tab xóa sản phẩm, tab kia thay toàn bộ giỏ;
- giỏ đang rỗng nên chưa có dòng con để mang phiên bản;
- quy tắc giới hạn tổng số sản phẩm của cả giỏ;
- phản hồi cần một phiên bản duy nhất cho toàn bộ ảnh chụp.

Phiên bản đúng phải nằm trên `shopping_cart` và mọi đường sửa bảng con phải làm
thay đổi phiên bản đó.

## 8. Cập nhật trực tiếp bảng con bỏ qua gốc giỏ

```java
@Modifying
@Query("""
    update CartItem i
       set i.quantity = :quantity
     where i.cart.id = :cartId
       and i.productId = :productId
    """)
int setQuantity(UUID cartId, UUID productId, int quantity);
```

Câu cập nhật hàng loạt không tải `ShoppingCart`, không kiểm tra
`ShoppingCart.version` và không đồng bộ vùng quản lý thực thể đang có. Nội dung
giỏ đổi trong khi `ETag` vẫn giữ nguyên; yêu cầu khác dùng phiên bản cũ sẽ được
coi nhầm là mới.

## 9. Thử lại mù làm sai ý định

```java
@Retryable(ObjectOptimisticLockingFailureException.class)
public CartResponse replace(UUID cartId, ReplaceCartRequest request) {
    // tải giỏ mới nhưng vẫn áp lại toàn bộ ảnh chụp cũ
}
```

Giả sử tab B xóa `SKU-A` sau khi tab A đã thêm `SKU-B`. Tự tải phiên bản mới rồi
áp lại danh sách rỗng không giải quyết xung đột; nó chỉ biến lần ghi đè thành một
giao dịch hợp lệ. Lệnh thay toàn bộ và đặt số lượng phải trả xung đột để phía gọi
quyết định.

## 10. Dấu hiệu thường gặp

- API trả thành công nhưng sản phẩm vừa thêm biến mất sau lần tải lại.
- Lỗi chỉ xuất hiện khi người dùng có nhiều tab hoặc ứng dụng di động cùng mở.
- Nhật ký có hai yêu cầu `200` gần nhau cho cùng `cartId`.
- `shopping_cart.updated_at` hoặc phiên bản không đổi dù `cart_item` đã đổi.
- Test tuần tự luôn qua, còn test đồng thời không có điểm hẹn nên lúc qua lúc
  thất bại.
