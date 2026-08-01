# Mã nguồn tạo đơn trùng

## 1. Lược đồ không bảo vệ khóa checkout

Lược đồ sau có khóa chính ngẫu nhiên nhưng không có ràng buộc trên ý định
checkout:

```sql
CREATE TABLE purchase_order (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    checkout_key VARCHAR(80) NOT NULL,
    cart_id UUID NOT NULL,
    total_amount NUMERIC(19, 2) NOT NULL,
    currency CHAR(3) NOT NULL,
    status VARCHAR(30) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
);
```

Hai dòng có `order_id` khác nhau nhưng cùng `(customer_id, checkout_key)` đều
hợp lệ đối với PostgreSQL. Khóa chính chỉ bảo vệ danh tính kỹ thuật, không bảo vệ
khóa nghiệp vụ.

## 2. Thực thể và kho dữ liệu

```java
@Entity
@Table(name = "purchase_order")
public class PurchaseOrder {

    @Id
    @Column(name = "order_id", nullable = false)
    private UUID id;

    @Column(name = "customer_id", nullable = false)
    private UUID customerId;

    @Column(name = "checkout_key", nullable = false, length = 80)
    private String checkoutKey;

    @Column(name = "cart_id", nullable = false)
    private UUID cartId;

    @Column(name = "total_amount", nullable = false, precision = 19, scale = 2)
    private BigDecimal totalAmount;

    @Column(name = "currency", nullable = false, length = 3)
    private String currency;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OrderStatus status;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;

    protected PurchaseOrder() {
    }

    public static PurchaseOrder pending(
        UUID customerId,
        String checkoutKey,
        CheckoutCommand command,
        Instant now
    ) {
        PurchaseOrder order = new PurchaseOrder();
        order.id = UUID.randomUUID();
        order.customerId = customerId;
        order.checkoutKey = checkoutKey;
        order.cartId = command.cartId();
        order.totalAmount = command.totalAmount();
        order.currency = command.currency();
        order.status = OrderStatus.PENDING_PAYMENT;
        order.createdAt = now;
        return order;
    }

    public UUID id() {
        return id;
    }
}
```

```java
public interface PurchaseOrderRepository
    extends JpaRepository<PurchaseOrder, UUID> {

    boolean existsByCustomerIdAndCheckoutKey(
        UUID customerId,
        String checkoutKey
    );
}
```

Phương thức `existsBy...` trông hợp lý khi đọc mã tuần tự. Nó không đặt quyền sở
hữu lên một khóa chưa tồn tại.

## 3. Dịch vụ bị lỗi

```java
@Service
public class BrokenCheckoutService {

    private final PurchaseOrderRepository orders;
    private final PaymentGatewayClient paymentGateway;
    private final Clock clock;

    public BrokenCheckoutService(
        PurchaseOrderRepository orders,
        PaymentGatewayClient paymentGateway,
        Clock clock
    ) {
        this.orders = orders;
        this.paymentGateway = paymentGateway;
        this.clock = clock;
    }

    @Transactional
    public CheckoutResponse checkout(
        UUID customerId,
        String idempotencyKey,
        CheckoutCommand command
    ) {
        if (orders.existsByCustomerIdAndCheckoutKey(
            customerId,
            idempotencyKey
        )) {
            throw new DuplicateCheckoutException(idempotencyKey);
        }

        PurchaseOrder order = orders.save(
            PurchaseOrder.pending(
                customerId,
                idempotencyKey,
                command,
                clock.instant()
            )
        );

        PaymentAttempt payment = paymentGateway.createPayment(
            order.id(),
            command.totalAmount(),
            command.currency()
        );

        return new CheckoutResponse(order.id(), payment.id());
    }
}
```

Đây không phải mã cố tình ngớ ngẩn. Nó dùng giao dịch, kiểm tra trùng và lưu mã
checkout vào đơn. Tuy nhiên, nó có hai lỗi độc lập:

1. `exists → insert` là hai câu lệnh, nên hai giao dịch có thể cùng vượt qua bước
   kiểm tra.
2. Lời gọi thanh toán từ xa diễn ra trước khi giao dịch cơ sở dữ liệu chốt.

`orders.save()` còn có thể chỉ đưa thực thể vào vùng nhớ của Hibernate. Câu
`INSERT` thật sự có thể xuất hiện khi `flush()` hoặc `COMMIT`, sau khi lời gọi
thanh toán đã chạy.

## 4. Điều kiện tái hiện

Lỗi xuất hiện khi có đủ các điều kiện sau:

- hai lần gọi dùng cùng `customerId` và `idempotencyKey`;
- mỗi lần gọi có một giao dịch và kết nối riêng;
- cả hai câu `SELECT exists` hoàn tất trước câu `INSERT` đầu tiên;
- lược đồ không có `UNIQUE (customer_id, checkout_key)` hoặc bảng chiếm quyền
  tương đương;
- hai yêu cầu có thể chạy trên hai luồng hoặc hai máy chủ ứng dụng.

## 5. Dòng thời gian gây lỗi

Trạng thái đầu:

```text
Không có đơn nào cho (C-17, CHECKOUT-9001).
```

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` |
| 2 | `exists(...)` trả `false` | |
| 3 | | `exists(...)` trả `false` |
| 4 | Tạo đơn `O-A` trong vùng nhớ Hibernate | |
| 5 | | Tạo đơn `O-B` trong vùng nhớ Hibernate |
| 6 | Gọi cổng thanh toán cho `O-A` | |
| 7 | | Gọi cổng thanh toán cho `O-B` |
| 8 | `INSERT O-A`; `COMMIT` | |
| 9 | | `INSERT O-B`; `COMMIT` |

Kết quả:

```text
purchase_order: O-A, O-B
payment attempt: P-A, P-B
```

Mỗi giao dịch riêng lẻ đều chốt hợp lệ. Lỗi nằm ở quy tắc nghiệp vụ bao trùm hai
giao dịch, không phải ở tính nguyên tử bên trong một giao dịch.

## 6. `@Transactional` không khóa khoảng trống

Ở mức `READ COMMITTED`, mỗi câu `SELECT` chỉ thấy dữ liệu đã chốt trước khi câu
lệnh đó bắt đầu. A chưa chèn nên B không thấy gì; B chưa chèn nên A cũng không
thấy gì.

`@Transactional` không biến `SELECT exists` và `INSERT` thành một câu lệnh
nguyên tử. `SELECT ... FOR UPDATE` cũng không giải quyết trực tiếp vì không có
dòng nào để khóa.

## 7. Các cách sửa chưa đủ

### Thêm `synchronized`

```java
public synchronized CheckoutResponse checkout(...) {
    // cùng phần xử lý cũ
}
```

Cách này chỉ tuần tự hóa luồng trong một đối tượng của một JVM. Máy chủ B không
nhìn thấy khóa của máy chủ A; khởi động lại tiến trình cũng làm mất khóa.

### Chỉ thêm ràng buộc duy nhất rồi trả lỗi

Ràng buộc duy nhất sẽ ngăn đơn thứ hai và là lớp bảo vệ bắt buộc. Nhưng nếu ứng
dụng chỉ trả `500` hoặc `409` cho mọi xung đột, nó chưa thực hiện đầy đủ hợp đồng
lũy đẳng: lần gửi lại cùng nội dung phải nhận kết quả của đơn đã tạo.

Nếu bắt `DataIntegrityViolationException` bên trong cùng giao dịch sau khi
PostgreSQL báo `23505`, giao dịch đã ở trạng thái lỗi. Không được tiếp tục
`SELECT` trong giao dịch đó như thể không có chuyện gì xảy ra.

### Tạo khóa mới cho mỗi lần thử lại

Khóa lũy đẳng đại diện cho ý định nghiệp vụ, không đại diện cho lần truyền HTTP.
Tạo khóa mới sau lỗi mạng làm máy chủ hiểu lần thử lại là một checkout mới.

### Chỉ so sánh khóa, không so sánh nội dung

Nếu cùng khóa được dùng cho giỏ hàng hoặc số tiền khác, phát lại đơn cũ một cách
im lặng sẽ che giấu lỗi phía gọi và có thể trả nhầm kết quả. Phải lưu dấu vân tay
do máy chủ tính và từ chối trường hợp không khớp.

### Gọi thanh toán trong giao dịch

Giao dịch cơ sở dữ liệu không thể hoàn tác một lời gọi HTTP đã thành công. Việc
giữ kết nối trong lúc chờ mạng còn kéo dài thời gian giao dịch và làm tăng nguy
cơ hết vùng kết nối.

### Dùng `ON CONFLICT DO UPDATE` giả để lấy ID

Cập nhật một cột thành chính giá trị cũ chỉ để ép `RETURNING` có thể tạo phiên
bản dòng mới, kích hoạt trigger và làm sai `updated_at`. Luồng này chỉ cần
`DO NOTHING`, sau đó đọc bản ghi đã có bằng một câu lệnh phù hợp.

## 8. Dấu hiệu trên hệ thống thật

- Nhiều `purchase_order` có cùng khách hàng và mã checkout.
- Hai đơn được tạo cách nhau vài mili giây trên các máy chủ khác nhau.
- Nhiều lệnh thanh toán có cùng giỏ hàng nhưng khác `order_id`.
- Tỷ lệ đơn trùng tăng cùng lỗi mạng, thao tác nhấn lại hoặc lần thử lại từ cổng
  API.
- Lỗi xuất hiện muộn tại `flush` hoặc `COMMIT`, không đúng vị trí `save()`.
- Khách hàng báo lỗi dù một đơn thực tế đã được tạo thành công.

Mã yêu cầu và dấu vân tay nên được băm hoặc rút gọn khi ghi log. Không ghi toàn
bộ nội dung giỏ hàng, địa chỉ hoặc dữ liệu thanh toán vào log chẩn đoán.
