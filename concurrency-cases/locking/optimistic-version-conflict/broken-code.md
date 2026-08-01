# Phản mẫu thiết kế: Cập nhật dữ liệu bỏ qua cơ chế version

## 1. Khai báo thực thể bỏ quên quản lý phiên bản (thiếu `@Version`)

Một khai báo lớp entity cơ bản thường xuất hiện trong các dự án:

```java
@Entity
@Table(name = "product_offer")
public class ProductOffer {

    @Id
    private Long offerId;

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private String title;

    protected ProductOffer() {
    }

    public void changePrice(BigDecimal newPrice) {
        if (newPrice == null || newPrice.signum() < 0) {
            throw new IllegalArgumentException("invalid price");
        }
        this.price = newPrice;
    }
}
```

Tương ứng với cấu trúc database:

```sql
create table product_offer (
    offer_id bigint primary key,
    price numeric(19, 2) not null,
    title varchar(200) not null
);
```

Sự thiếu vắng của cột `version` là lỗ hổng cốt lõi dẫn đến các sai phạm về xử lý tương tranh dữ liệu.

## 2. Kịch bản ghi đè ngầm qua vòng lặp đọc-sửa-đồng bộ

```java
@Service
public class BrokenOfferEditor {

    private final ProductOfferRepository offers;

    public BrokenOfferEditor(ProductOfferRepository offers) {
        this.offers = offers;
    }

    @Transactional
    public OfferView changePrice(long offerId, BigDecimal newPrice) {
        ProductOffer offer = offers.findById(offerId)
                .orElseThrow(() -> new OfferNotFoundException(offerId));

        offer.changePrice(newPrice);
        offers.flush();
        return OfferView.from(offer);
    }
}
```

Quá trình này chỉ định Hibernate thực thi một chuỗi câu lệnh UPDATE tiêu chuẩn:

```sql
update product_offer
set price = ?,
    title = ?
where offer_id = ?;
```

Mặc dù PostgreSQL đảm bảo quản lý row lock cho phép chỉ một transaction được thực thi tại một thời điểm, tuy nhiên câu lệnh SQL của transaction thứ hai sẽ được triển khai ngay lập tức sau khi transaction đầu hoàn tất. Lệnh truy vấn không có mệnh đề giới hạn bảo mật (chỉ sử dụng điều kiện `offer_id = ?`), do đó trạng thái thay đổi luôn báo cáo `1` dòng bị tác động. Database vật lý không có khả năng nhận biết ứng dụng Java đang tải lên một dữ liệu đã lỗi thời.

> **Ghi chú kỹ thuật:** Cơ chế row lock thuộc hạ tầng nguyên thủy chỉ đảm bảo quy trình nối tiếp transaction, nhưng hoàn toàn thiếu sót năng lực giám sát logic vòng đời dữ liệu. Khả năng bảo toàn lịch sử dữ liệu chỉ khả thi khi bổ sung mệnh đề chứa thông tin phiên bản kỳ vọng.

## 3. Dòng thời gian lỗi mất bản cập nhật

```text
Luồng A truy xuất: Giá=100
Luồng B truy xuất: Giá=100
Luồng A cập nhật thành 90, gửi lệnh SQL, và commit!
Luồng B cập nhật thành 80, gửi lệnh SQL tiếp nối, và commit!
Kết quả hệ thống: Database lưu trữ giá trị 80; Cả A và B đều nhận phản hồi cập nhật thành công!
```

Sự kiện mất dữ liệu cập nhật đã được xác lập: Toàn bộ quá trình và kết quả xử lý của Luồng A bị loại bỏ không dấu vết.

## 4. Ảo giác an toàn từ phương thức `save()`

```java
ProductOffer saved = offers.save(offer);
```

Phương thức `save()` trong môi trường Spring Data JPA không phải là cơ chế đảm bảo an toàn đa luồng. Nó đơn thuần là quy trình quyết định vòng đời "tạo mới nếu chưa có, cập nhật nếu đã tồn tại". `save()` không tự động tiêm mã chặn xung đột nếu lớp tương ứng không được thiết lập annotation `@Version`. Tính năng toàn vẹn cấu trúc thuộc về giao ước database, không phụ thuộc vào quy ước đặt tên của repository.

## 5. Sử dụng `@DynamicUpdate` sai mục đích

Khai báo `@DynamicUpdate` được Hibernate xây dựng nhằm giới hạn câu lệnh SQL UPDATE chỉ thay đổi những trường thực sự khác biệt so với bộ nhớ đệm. Cơ chế này giảm thiểu rủi ro khi 2 transaction độc lập hiệu chỉnh 2 trường riêng biệt trên cùng 1 cấu trúc. Tuy nhiên, nếu hai transaction cùng chỉ định vào trường `price`, `@DynamicUpdate` hoàn toàn vô giá trị trong việc bảo mật tính toàn vẹn. `@DynamicUpdate` thuộc nhóm tối ưu hiệu suất, không thuộc nhóm xử lý tương tranh database.

## 6. Lỗ hổng thiết kế API thiếu phiên bản kỳ vọng

```java
// Bỏ quên trường expectedVersion!
public record ChangePriceRequest(BigDecimal price) {
}

@Transactional
public OfferView changePrice(long id, ChangePriceRequest request) {
    ProductOffer current = offers.findById(id).orElseThrow();
    current.changePrice(request.price());
    return OfferView.from(current);
}
```

Ngay cả khi database và Hibernate đã thiết lập cơ chế khóa lạc quan (`@Version`), nếu dữ liệu yêu cầu (payload) của phía gọi không truyền lên thông tin phiên bản gốc, máy chủ backend sẽ tải lên đối tượng có phiên bản mới nhất và gán cấu trúc cập nhật lên đó. Quy trình này sẽ tạo nên câu lệnh SQL chứa tham số version đã được cập nhật bởi transaction kề trước, và kết quả dẫn tới vòng lặp ghi đè vẫn diễn ra trơn tru.

## 7. Phản mẫu ghi đè phiên bản thủ công

```java
offer.setVersion(offer.getVersion() + 1);
```

Bất cứ hành vi tương tác bằng mã lên trường `version` đều hủy hoại chu trình quản lý của trình cung cấp JPA. Nó dẫn tới hệ quả chênh lệch thông số giữa bộ nhớ đệm và database, làm vô hiệu hóa thuật toán phát sinh mệnh đề bảo mật trong câu SQL. Thuộc tính này phải được cấu hình ở chế độ hệ thống quản lý.

## 8. Khối catch không hợp lệ trong transaction hỏng

```java
@Transactional
public OfferView editWithRetry(EditOffer command) {
    try {
        return apply(command);
    } catch (ObjectOptimisticLockingFailureException conflict) {
        entityManager.clear();
        return apply(command); // Sử dụng transaction đã bị đánh dấu hủy
    }
}
```

Một khi ngoại lệ `OptimisticLockException` xuất hiện, cơ chế quản lý ngầm sẽ thiết lập cờ chỉ định rollback (`rollback-only`) lên ngữ cảnh transaction. Hành động gọi `clear()` chỉ có tác dụng cắt đứt cấu trúc tham chiếu trong bộ nhớ, nhưng hoàn toàn không thể tái khởi tạo lại transaction vật lý dưới tầng database. Việc yêu cầu thực thi trên một transaction đã chết sẽ sinh ra hàng loạt lỗi lan truyền.

## 9. Phản hồi ảo trước khi thực thi SQL

```java
@Transactional
public OfferView edit(...) {
    offer.changePrice(...);
    return OfferView.from(offer); // Trả về thành công nhưng transaction chưa commit
}
```

Việc cung cấp đối tượng trả về trong phạm vi phương thức không đại diện cho trạng thái đã commit. Thực tế, truy vấn SQL và ngoại lệ có thể chỉ được kích hoạt (flush) ở rìa phân lớp Spring proxy, sau khi luồng Java của phương thức đã chạy xong. Phản hồi nội bộ không định lượng được thành công cuối cùng của transaction.

## 10. Sai lầm về khóa cục bộ trong kiến trúc phân tán

```java
synchronized (this) {
    editor.changePrice(...);
}
```

Rào cản `synchronized` là một giải pháp khóa mức ứng dụng tại luồng cục bộ (JVM). Khi ứng dụng triển khai trên mô hình bộ cân bằng tải / kiến trúc nhiều instance, khóa JVM không thể kiểm soát sự cạnh tranh giữa các node độc lập tới cùng một database. Tình trạng ghi đè sẽ xuất hiện trở lại do sự thiếu vắng của rào cản SQL toàn cục.

## 11. Các bước dựng lại môi trường kiểm chứng tái hiện lỗi

1. Thiết lập database: Một bản ghi (offer) có giá `100` tồn tại.
2. Thiết lập môi trường: Tạo 2 luồng xử lý tương ứng với 2 ngữ cảnh transaction riêng biệt.
3. Đọc dữ liệu đồng thời: Điều khiển Luồng A và Luồng B thực hiện tải đối tượng.
4. Thực thi A: Luồng A thay đổi thông số thành `90`, kích hoạt `flush` và commit.
5. Thực thi B: Luồng B đổi giá thành `80`, kích hoạt `flush` và commit (tuần tự theo sau A).
6. Khảo sát: Cả 2 luồng đều xác nhận thành công. Trạng thái database cuối cùng chốt ở giá `80`. Thất bại bảo toàn dữ liệu được xác nhận.
7. (Lưu ý: Môi trường kiểm thử đòi hỏi gỡ bỏ annotation `@Transactional` bao quanh phương thức kiểm thử để duy trì ranh giới transaction ảo như trạng thái thực tế).

## 12. Danh mục các phương án sửa lỗi không hợp lệ

- Bọc thêm annotation `@Transactional` ở các lớp, hy vọng gia cố lỗi ngầm định.
- Chuyển thao tác `save()` thành `saveAndFlush()` mà không bổ sung chiến lược ngoại lệ tương ứng.
- Tự động thay đổi mức độ cách ly của database từ `READ COMMITTED` lên `REPEATABLE READ` mà thiếu kiến thức về việc database sẽ dội ngược lại các lớp lỗi.
- Đặt niềm tin bảo mật dữ liệu hoàn toàn vào tùy biến `@DynamicUpdate`.
- Hệ thống trả thông tin version cho phía gọi nhưng không thiết kế cơ chế bắt buộc phía gọi truyền xuống tham chiếu (xác thực không trạng thái).
- Bao quát mọi mã lỗi tương tranh để thử lại tự động bất chấp nghiệp vụ đòi hỏi phản hồi từ người dùng.
- Tự tạo logic version từ đồng hồ hệ thống (rủi ro lệch múi giờ phân tán).
- Sử dụng các kiến trúc Java Mutex, ReentrantLock ở phía ứng dụng để bảo vệ database.
- Bỏ qua công tác đối chiếu khẳng định bằng cách truy vấn ngược lại database sau khi quy trình kiểm chứng (assertion) báo cáo thành công.
