# 1001 Bể Phốt Code Lỗi: Thay đổi dữ liệu mà quên "Dò Version"

## 1. Class Entity Trắng Tay (không có `@Version`)

Nhìn cái Class này xem, trông có vẻ rất chuẩn mực:

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

Dưới DB cũng chỉ là một cái bảng bình thường:

```sql
create table product_offer (
    offer_id bigint primary key,
    price numeric(19, 2) not null,
    title varchar(200) not null
);
```

Thiếu một cột `version` là khởi nguồn cho mọi tội lỗi!

## 2. Viết Service Tải -> Sửa -> Xả (read → mutate → flush)

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

Bạn gọi hàm này, Hibernate sẽ bắn ra một câu SQL cập nhật tuyệt đối như sau:

```sql
update product_offer
set price = ?,
    title = ?
where offer_id = ?;
```

Mặc dù PostgreSQL có dùng khóa dòng (row lock) để xếp hàng hai ông nội A và B, nhưng sau khi A chốt sổ xong xuôi, thằng B nối gót chạy lệnh UPDATE. Khốn nỗi câu SQL của B chỉ có điều kiện khóa chính (`where offer_id = ?`), nên nó phang thẳng số của nó vào. DB báo số dòng thay đổi là `1` (thành công rực rỡ!). Database thì hiền khô, nó đâu biết lệnh này được suy luận ra từ một đống dữ liệu đã ôi thiu!

> **Nói ngắn gọn:** Khóa dòng chỉ cản 2 người cùng sửa chung 1 dòng ở cùng 1 mili-giây. Chứ nó mù lòa trong việc phát hiện logic dữ liệu bị cũ, trừ khi bạn chêm cái `expected version` vào mệnh đề `WHERE`.

## 3. Dòng thời gian dắt nhau xuống vực (Timeline lỗi)

```text
A bốc lên giá=100
B bốc lên giá=100
A sửa thành 90, xả SQL, chốt sổ!
B sửa thành 80, xả SQL theo ngay sau A, chốt sổ!
Giá cuối cùng dưới DB = 80; Cả A và B đều cười hề hề vì được báo Thành Công!
```

Đây chính là thảm họa Mất Dữ Liệu (lost update): Công sức của A bị B xóa sổ gọn hơ.

## 4. Ảo tưởng hàm `save()` thần thánh

```java
ProductOffer saved = offers.save(offer);
```

Bạn nghĩ gọi `save()` là an toàn? Ồ không! Hàm `save()` chỉ giỏi trò "đã có thì sửa, chưa có thì tạo mới" (persist/merge). Chứ nếu bạn không khai báo `@Version` trong Class, nó sẽ không tự rảnh háng đi chèn thêm điều kiện dò Version cho bạn đâu. Xung đột là do Bản Hợp Đồng SQL quyết định, chứ không liên quan gì đến mấy cái tên hàm Repository kêu cho rổn rảng.

## 5. Dùng `@DynamicUpdate` là Đủ? Còn lâu!

`@DynamicUpdate` là trò mèo giúp Hibernate chỉ ra lệnh UPDATE những cột nào thực sự thay đổi. Trò này giúp đỡ ghi đè lung tung khi 2 người sửa 2 cột KHÁC NHAU. Nhưng nếu 2 người CÙNG SỬA cột `price` thì nó cũng bó tay chịu chết. Đây chỉ là mẹo tối ưu hiệu năng SQL, tuyệt đối không phải cơ chế bắt lỗi Xung đột!

## 6. Lỗ hổng to đùng: API quên mất Client đang cầm Version nào

```java
// Thiếu mất expectedVersion ở đây nè!
public record ChangePriceRequest(BigDecimal price) {
}

@Transactional
public OfferView changePrice(long id, ChangePriceRequest request) {
    ProductOffer current = offers.findById(id).orElseThrow();
    current.changePrice(request.price());
    return OfferView.from(current);
}
```

Kể cả khi bạn đã cài `@Version` dưới Database. Nếu trên cái Request không gửi kèm `version`, Backend sẽ ngây thơ chui xuống DB bốc cái bản MỚI NHẤT lên. Lệnh update của bạn sẽ được Hibernate điền đúng cái Version mới nhất này. Lại THÀNH CÔNG RỰC RỠ, và B lại tiếp tục xóa sạch công sức của A.

## 7. Cầm đèn chạy trước ô tô: Tự Tăng Version bằng tay

```java
offer.setVersion(offer.getVersion() + 1);
```

Cái `version` sinh ra là để thằng Quản Gia (Hibernate / JPA) lo. Nếu bạn táy máy thò tay vào tự cộng thêm 1, nó sẽ loạn cào cào và phá nát cái mệnh đề so sánh lúc sinh câu SQL. Bỏ tay ra ngay!

## 8. Bắt Lỗi và Cố Đấm Ăn Xôi (Catch conflict rồi chạy tiếp)

```java
@Transactional
public OfferView editWithRetry(EditOffer command) {
    try {
        return apply(command);
    } catch (ObjectOptimisticLockingFailureException conflict) {
        entityManager.clear();
        return apply(command); // Vẫn bám víu vào cái Giao Dịch đã CHẾT
    }
}
```

Một khi `OptimisticLockException` văng ra thì Giao Dịch hiện tại đã bị cắm cờ CHẾT (rollback-only). Bạn có gọi `clear()` thì nó chỉ gỡ liên kết mấy cái Object trên RAM ra thôi, chứ cái Giao Dịch vật lý dưới DB không hề được tạo mới. Cố tình code tiếp trong này là tự đào mố chôn mình. (Đọc bài `LOCK-002` để biết cách Retry chuẩn).

## 9. Cú sốc phút chót (Conflict ở một nơi xa)

```java
@Transactional
public OfferView edit(...) {
    offer.changePrice(...);
    return OfferView.from(offer); // Thấy return không có nghĩa là đã an toàn!
}
```

Bạn cứ nghĩ gọi API xong trả về `OfferView` là ngon lành rồi? Chưa đâu. SQL chỉ thực sự xả xuống và Commit ở Mép Ngoài Cùng của cái Proxy. Tức là hàm Java của bạn chạy xong, đi ra ngoài rìa rồi mới văng Lỗi. Đừng bao giờ đinh ninh rằng giá trị Return có nghĩa là DB đã chốt sổ nhé!

## 10. Chặn cổng Làng mà quên chặn cổng Tỉnh (Local lock sai)

```java
synchronized (this) {
    editor.changePrice(...);
}
```

Bạn tự mãn vì đã rào `synchronized` nên không luồng nào chen vào được? Chúc mừng bạn, nó chạy tốt trên máy của bạn (1 App). Nhưng ra Production, bạn chạy 5 cái Container chia tải (load balancer). 5 máy tính chả liên quan gì đến nhau, chúng cùng thọc tay vào gọi DB, và dữ liệu lại bị ghi đè te tua vì DB thiếu cái hàng rào SQL Version.

## 11. Các Bước Để Dựng Lại Hiện Trường (Tái hiện lỗi)

1. Gieo mầm: Cho một offer giá `100` yên vị dưới DB.
2. Dựng sân khấu: Mở 2 luồng Giao dịch Vật lý riêng biệt (persistence contexts).
3. Cho luồng A và B cùng bốc cái offer đó lên.
4. Lệnh cho luồng A đổi giá thành `90`, xả lệnh và chốt sổ.
5. Lệnh cho luồng B đổi giá thành `80`, xả lệnh và chốt sổ (ngay sau A).
6. Kiểm chứng: Cả hai không ai văng lỗi, DB chốt giá `80`. Bùm!
7. (Nên dùng PostgreSQL thật qua Testcontainers, nhớ tháo cái `@Transactional` ở Test Class ra để mô phỏng đúng).

## 12. Danh sách Những pha Chữa cháy Đi vào Lòng Đất

- Chỉ nhét thêm `@Transactional` vào và cầu nguyện.
- Thay vì gọi `save()`, gọi hẳn `saveAndFlush()` cho ngầu (và vô dụng).
- Nghe lời xúi giục đổi Mức cách ly từ `READ COMMITTED` lên `REPEATABLE READ` mà không biết cách hứng lỗi từ DB dội về.
- Chỉ dùng `@DynamicUpdate`.
- Khôn ngoan tới mức trả về Version cho Client, nhưng quên bắt Client phải nhét cái Version đó vào lại lúc bấm Lưu.
- Bao hết mọi lỗi Conflict lại rồi Tự Động thử lại dù người dùng chưa cho phép.
- Lấy Đồng hồ của hệ thống (Timestamp) ra chế thành Version (Máy chủ đồng hồ bị lệch là vỡ mồm).
- Dùng Mutex/Lock của Java để đỡ đạn cho hệ thống nhiều Server.
- Cài assert bắt Lỗi ngon lành nhưng quên mất kiểm tra xem Dữ liệu cuối cùng dưới DB có bị rác hay không.
