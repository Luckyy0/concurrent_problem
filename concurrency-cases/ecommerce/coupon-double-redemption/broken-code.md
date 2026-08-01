# Mã nguồn cho phép dùng mã vượt giới hạn

## 1. Lược đồ thiếu lớp bảo vệ

```sql
CREATE TABLE promotion (
    promotion_id UUID PRIMARY KEY,
    code VARCHAR(40) NOT NULL,
    status VARCHAR(20) NOT NULL,
    starts_at TIMESTAMPTZ NOT NULL,
    ends_at TIMESTAMPTZ NOT NULL,
    global_limit BIGINT,
    per_user_limit INTEGER,
    redeemed_count BIGINT NOT NULL DEFAULT 0
);

CREATE TABLE promotion_redemption (
    redemption_id UUID PRIMARY KEY,
    promotion_id UUID NOT NULL REFERENCES promotion(promotion_id),
    customer_id UUID NOT NULL,
    checkout_id UUID NOT NULL,
    discount_amount NUMERIC(19, 2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
);
```

Lược đồ này có ba khoảng hở:

- không cấm hai dòng cùng `(promotion_id, checkout_id)`;
- không có lớp bảo vệ cho giới hạn mỗi khách hàng;
- không buộc `redeemed_count` nằm trong `global_limit`.

## 2. Thực thể chương trình khuyến mại

```java
@Entity
@Table(name = "promotion")
public class Promotion {

    @Id
    @Column(name = "promotion_id", nullable = false)
    private UUID id;

    @Column(name = "code", nullable = false)
    private String code;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private PromotionStatus status;

    @Column(name = "starts_at", nullable = false)
    private Instant startsAt;

    @Column(name = "ends_at", nullable = false)
    private Instant endsAt;

    @Column(name = "global_limit")
    private Long globalLimit;

    @Column(name = "per_user_limit")
    private Integer perUserLimit;

    @Column(name = "redeemed_count", nullable = false)
    private long redeemedCount;

    protected Promotion() {
    }

    public boolean isActiveAt(Instant now) {
        return status == PromotionStatus.ACTIVE
            && !now.isBefore(startsAt)
            && now.isBefore(endsAt);
    }

    public boolean hasGlobalCapacity() {
        return globalLimit == null || redeemedCount < globalLimit;
    }

    public void recordRedemption() {
        redeemedCount++;
    }
}
```

Thực thể không có `@Version`. Hai giao dịch có thể tải cùng giá trị
`redeemedCount = 99`, cùng tăng thành `100` rồi cùng ghi giá trị tuyệt đối `100`.

## 3. Kho dữ liệu

```java
public interface PromotionRepository
    extends JpaRepository<Promotion, UUID> {
}
```

```java
public interface PromotionRedemptionRepository
    extends JpaRepository<PromotionRedemption, UUID> {

    boolean existsByPromotionIdAndCheckoutId(
        UUID promotionId,
        UUID checkoutId
    );

    long countByPromotionIdAndCustomerId(
        UUID promotionId,
        UUID customerId
    );
}
```

Hai truy vấn kiểm tra chỉ đọc ảnh chụp hiện tại. Chúng không giữ một lượt dùng
cho giao dịch đang chạy.

## 4. Dịch vụ bị lỗi

```java
@Service
public class BrokenPromotionService {

    private final PromotionRepository promotions;
    private final PromotionRedemptionRepository redemptions;
    private final DiscountCalculator discounts;
    private final Clock clock;

    public BrokenPromotionService(
        PromotionRepository promotions,
        PromotionRedemptionRepository redemptions,
        DiscountCalculator discounts,
        Clock clock
    ) {
        this.promotions = promotions;
        this.redemptions = redemptions;
        this.discounts = discounts;
        this.clock = clock;
    }

    @Transactional
    public RedemptionResult apply(RedeemPromotionCommand command) {
        if (redemptions.existsByPromotionIdAndCheckoutId(
            command.promotionId(),
            command.checkoutId()
        )) {
            throw new PromotionAlreadyAppliedException();
        }

        Promotion promotion = promotions.findById(command.promotionId())
            .orElseThrow(PromotionNotFoundException::new);

        Instant now = clock.instant();
        if (!promotion.isActiveAt(now)) {
            throw new PromotionNotActiveException();
        }
        if (!promotion.hasGlobalCapacity()) {
            throw new GlobalLimitReachedException();
        }

        long customerUses = redemptions.countByPromotionIdAndCustomerId(
            command.promotionId(),
            command.customerId()
        );
        if (
            promotion.getPerUserLimit() != null
                && customerUses >= promotion.getPerUserLimit()
        ) {
            throw new PerUserLimitReachedException();
        }

        BigDecimal discount = discounts.calculate(command, promotion);
        promotion.recordRedemption();

        PromotionRedemption redemption = PromotionRedemption.applied(
            UUID.randomUUID(),
            command.promotionId(),
            command.customerId(),
            command.checkoutId(),
            discount,
            now
        );
        redemptions.save(redemption);

        return new RedemptionResult(redemption.id(), discount);
    }
}
```

Đây là một cách viết dễ gặp trong ứng dụng thật: giao dịch rõ ràng, kiểm tra thời
hạn, kiểm tra hai giới hạn và lưu lịch sử. Lỗi không nằm ở việc thiếu
`@Transactional`; lỗi nằm ở các quyết định dựa trên dữ liệu có thể cũ trước khi
giao dịch ghi.

## 5. Điều kiện tái hiện

Trạng thái ban đầu:

```text
FLASH20.redeemed_count = 99
FLASH20.global_limit   = 100
FLASH20.per_user_limit = 1
Số lượt của C-17       = 0
```

Hai lệnh dùng hai `checkout_id` khác nhau. Chúng không phải bản sao của cùng một
yêu cầu; ECOM-003 không thể gộp chúng thành một lệnh.

Để tái hiện ổn định, đặt rào chắn sau khi cả hai giao dịch đã đọc
`redeemed_count` và `customerUses`, nhưng trước `recordRedemption()`.

## 6. Dòng thời gian gây lỗi

| Bước | Giao dịch A — `CO-A` | Giao dịch B — `CO-B` |
| --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` |
| 2 | Không thấy lịch sử cho `CO-A` | Không thấy lịch sử cho `CO-B` |
| 3 | Đọc bộ đếm toàn cục `99` | Đọc bộ đếm toàn cục `99` |
| 4 | Đếm lượt `C-17 = 0` | Đếm lượt `C-17 = 0` |
| 5 | Kết luận cả hai giới hạn còn chỗ | Kết luận cả hai giới hạn còn chỗ |
| 6 | Tăng đối tượng Java thành `100` | Tăng đối tượng Java thành `100` |
| 7 | Chèn lịch sử `R-A` | Chèn lịch sử `R-B` |
| 8 | Ghi `redeemed_count = 100`; `COMMIT` | Ghi `redeemed_count = 100`; `COMMIT` |

Trạng thái cuối:

```text
promotion.redeemed_count = 100
promotion_redemption     = 2 dòng mới
số lượt thật của C-17    = 2
```

Bộ đếm nhìn có vẻ hợp lệ nhưng không còn khớp với lịch sử. Đây vừa là lỗi mất
cập nhật trên bộ đếm, vừa là lỗi kiểm tra giới hạn bằng hai câu lệnh tách rời.

## 7. Vì sao `count()` không phải là khóa

`countByPromotionIdAndCustomerId()` chỉ đếm các dòng đã chốt nhìn thấy trong ảnh
chụp của câu lệnh. Lịch sử chưa chốt của giao dịch khác không xuất hiện như một
dòng bình thường, và kết quả `0` không khóa “lượt đầu tiên” của khách hàng.

`SELECT ... FOR UPDATE` trên bảng lịch sử cũng không khóa được một dòng chưa tồn
tại. Cần một khóa duy nhất hoặc một dòng bộ đếm người dùng có khóa chính rõ ràng.

## 8. Các cách sửa chưa đủ

### Chỉ thêm `@Transactional`

Giao dịch bảo đảm mỗi lần gọi cùng chốt hoặc cùng hoàn tác. Nó không làm hai lần
gọi nối đuôi nhau và không biến chuỗi đọc–kiểm tra–ghi thành một thao tác nguyên
tử.

### Chỉ dùng `synchronized`

Khóa này không bao phủ máy chủ khác, tác vụ nền hoặc câu SQL chạy ngoài JVM.

### Chỉ thêm `@Version`

`@Version` có thể phát hiện hai giao dịch cùng sửa dòng `promotion`, nhưng bên
thua phải tải lại và thử toàn bộ giao dịch. Dưới tranh chấp cao, nhiều lần thử
lại cùng đổ vào một dòng nóng. Nó cũng không tự bảo vệ dòng bộ đếm người dùng
chưa tồn tại hoặc tạo hợp đồng phát lại lịch sử cũ.

### Chỉ đặt `CHECK (redeemed_count <= global_limit)`

Ràng buộc này ngăn một giá trị bộ đếm vượt giới hạn. Trong dòng thời gian trên,
cả hai giao dịch đều ghi `100`, nên `CHECK` vẫn đúng dù có hai lịch sử mới.

### Chỉ thêm `UNIQUE (promotion_id, checkout_id)`

Ràng buộc ngăn cùng checkout dùng mã hai lần. Hai checkout khác nhau của cùng
khách hàng vẫn có thể vượt giới hạn mỗi người, và hai khách hàng khác nhau vẫn có
thể tranh lượt toàn cục cuối cùng.

### Dùng `SELECT ... FOR UPDATE` sau khi đã đọc

Khóa phải được lấy trước khi quyết định. Đọc không khóa, tính toán rồi mới khóa
không sửa được quyết định đã dựa trên dữ liệu cũ. Nếu dùng khóa bi quan, mọi
đường ghi phải lấy các dòng theo cùng thứ tự.

### Cập nhật tương đối nhưng thiếu điều kiện

```sql
UPDATE promotion
SET redeemed_count = redeemed_count + 1
WHERE promotion_id = :promotionId;
```

Câu lệnh tránh mất cập nhật nhưng vẫn cho bộ đếm tăng từ `100` lên `101`. Điều
kiện giới hạn phải nằm trong chính mệnh đề `WHERE`.

### Bắt lỗi rồi tiếp tục trong cùng giao dịch

Sau lỗi ràng buộc `23505`, PostgreSQL đánh dấu giao dịch hiện tại là lỗi cho tới
khi hoàn tác. Không được bắt ngoại lệ và tiếp tục tăng bộ đếm trong giao dịch đó.
Với xung đột dự kiến, `ON CONFLICT DO NOTHING RETURNING` cho tín hiệu sạch hơn.

### Hoàn tác bằng một lệnh bù ở giao dịch khác

Nếu đã tăng bộ đếm toàn cục rồi phát hiện khách hàng hết lượt, đừng chốt và sau
đó giảm bù. Tiến trình có thể sập giữa hai giao dịch. Hai bộ đếm và lịch sử phải
cùng nằm trong một giao dịch ban đầu.

## 9. Dấu hiệu trên hệ thống thật

- `redeemed_count` không bằng số lịch sử `APPLIED`.
- Một khách hàng có nhiều lượt hơn `per_user_limit`.
- Cùng một checkout có nhiều dòng giảm giá.
- Vượt ngân sách thường xuất hiện ở cuối chiến dịch hoặc trong đợt tải cao.
- Nhiều giao dịch cập nhật cùng một `promotion_id` và chờ khóa lâu.
- Lỗi bế tắc xuất hiện giữa chức năng đổi mã và chức năng quản trị hạn mức.
- Phản hồi báo thất bại nhưng lịch sử đã chốt do mất kết nối sau `COMMIT`.

Không dùng `promotion_id`, `customer_id` hoặc mã coupon thô làm nhãn metric có
số lượng giá trị không giới hạn. Log nên dùng mã tương quan đã được kiểm soát.
