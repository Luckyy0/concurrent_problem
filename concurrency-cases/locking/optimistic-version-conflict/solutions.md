# Giải pháp Chuẩn mực: Dùng `@Version` và Hợp Đồng Ký Kết (Expected-version)

## 1. Nâng cấp Class Entity (Có cột Version)

Đầu tiên, phải chạy kịch bản (migration) sửa Database:

```sql
alter table product_offer add column version bigint;
-- Nhét số 0 vào mấy dòng cũ cho khỏi lỗi
update product_offer set version = 0 where version is null;
alter table product_offer alter column version set not null;
```

(Nhớ là khi đang chuyển giao Code Cũ và Code Mới, ông Code Cũ nào viết SQL mà không tự tăng `version` là vỡ mồm cả lũ đấy nhé).

Tiếp theo, cập nhật Class Entity:

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

    @Version // Bùa hộ mệnh đây rồi!
    @Column(nullable = false)
    private long version;

    protected ProductOffer() {
    }

    // Chỉ viết hàm LẤY (get), TUYỆT ĐỐI KHÔNG VIẾT HÀM GÁN (set) cho version!
    public long version() {
        return version;
    }

    public void changePrice(BigDecimal newPrice) {
        if (newPrice == null || newPrice.signum() < 0) {
            throw new IllegalArgumentException("Giá bị âm kìa ba");
        }
        price = newPrice;
    }
}
```

Từ giờ, việc tăng `version` hãy để Hibernate (Quản gia) tự lo.

## 2. Ép Client phải mang "Dữ liệu Kỳ vọng" (Expected Version) lên

```java
// Bắt buộc Client phải truyền lên cái version mà nó đang nhìn thấy
public record ChangeOfferPrice(
        long offerId,
        long expectedVersion,
        BigDecimal newPrice,
        UUID commandId
) {
}
```

Và đây là cách Service phòng thủ:

```java
@Service
public class OfferEditor {

    private final ProductOfferRepository offers;

    public OfferEditor(ProductOfferRepository offers) {
        this.offers = offers;
    }

    @Transactional
    public OfferView changePrice(ChangeOfferPrice command) {
        ProductOffer offer = offers.findById(command.offerId())
                .orElseThrow(() ->
                        new OfferNotFoundException(command.offerId())
                );

        // Chặn cửa sổ thứ 1 (Form của Client đã ôi thiu)
        if (offer.version() != command.expectedVersion()) {
            throw new StaleOfferEditException(
                    command.offerId(),
                    command.expectedVersion(), // Client nghĩ là...
                    offer.version()            // Nhưng DB lại đang là...
            );
        }

        offer.changePrice(command.newPrice());
        offers.flush();
        // Lệnh UPDATE sẽ có thêm cái đuôi WHERE version=? chặn Cửa sổ thứ 2
        return OfferView.from(offer);
    }
}
```

> **Nói ngắn gọn:** Bạn check `expectedVersion` để đỡ đạn cho việc "Client treo máy đi cafe". Còn `@Version` của Hibernate giúp đỡ đạn cho trường hợp "Hai Request lao vào DB cùng 1 mili-giây". Thiếu một trong hai là nát bét.

## 3. Quăng Exception cho Kẻ Thua Cuộc

```sql
-- Dưới gầm DB nó chạy câu này
update product_offer
set price = :price,
    title = :title,
    version = :version + 1
where offer_id = :id
  and version = :version;
```

Ai chạy chậm, trả về `0` dòng -> Hibernate lập tức quăng `OptimisticLockException` (Spring thường bọc lại thành `ObjectOptimisticLockingFailureException`). Cái Exception này bắt buộc phải văng ra khỏi ranh giới hàm (thoát transaction) để Giao Dịch được dọn dẹp sạch sẽ (rollback).

## 4. Báo cho Client bằng mã Lỗi chuẩn (API mapping)

```java
@RestControllerAdvice
public class OfferConflictAdvice {

    // Bắt một lượt cả 2 loại Lỗi Đụng Độ
    @ExceptionHandler({
            StaleOfferEditException.class,
            ObjectOptimisticLockingFailureException.class
    })
    ResponseEntity<ProblemDetail> conflict(RuntimeException failure) {
        ProblemDetail problem = ProblemDetail.forStatus(
                HttpStatus.PRECONDITION_FAILED // Trả mã 412
        );
        problem.setTitle("Có người nhanh tay sửa mất rồi!");
        problem.setProperty("reloadRequired", true); // Báo Client tải lại màn hình
        return ResponseEntity
                .status(HttpStatus.PRECONDITION_FAILED)
                .body(problem);
    }
}
```

Đừng bao giờ nhổ toẹt cái Object ôi thiu trả về như thể là thao tác đã thành công nhé! Nếu Client xài Header `If-Match` thì trả mã `412 Precondition Failed` là đẹp nhất. Mã `409 Conflict` cũng hợp lý tùy Gu của công ty.

## 5. Ranh giới Xả (Flush) và Chốt (Commit)

Gọi `flush()` ngay trong hàm giúp bạn tóm được Exception sớm. Nhưng nhớ là hàm đó chạy xong, chui ra đến cửa cái Proxy thì DB mới thực sự gọi lệnh Chốt (Commit). Việc `catch` lỗi Đụng Độ bắt buộc phải nằm ở vòng ngoài cùng (Controller hoặc Advice), chứ ĐỪNG thọc tay `catch` bên trong Transaction và cố chấp chạy tiếp.

## 6. Tránh xài `merge` với DTO ngoài đường (Detached merge)

Thay vì bốc nguyên cái cục DTO của Client ép thành Entity rồi ném vào hàm `merge()`:

- Rất dễ dính đòn "Cập nhật lố" (mass assignment - Client gửi bậy id/role cũng bị sửa).
- Vô tình đụng tới những Object con lằng nhằng (cascade).
- Lỗi khó bắt hơn vì Hibernate delay việc kiểm tra.

Hãy Mapping bằng tay (DTO -> Managed Entity đang nằm trên RAM) và so sánh Version rành rọt. Viết code dài hơn 3 dòng nhưng đổi lại Giấc ngủ ngon.

## 7. Pháp Sư chơi hệ SQL thuần (Bulk/native writers)

Bắt buộc phải xài SQL chay à? Cứ việc, nhưng phải tuân thủ Luật Chơi:

```sql
update product_offer
set price = :newPrice,
    version = version + 1 -- Phải tự cộng!
where offer_id = :id
  and version = :expectedVersion; -- Phải tự Check!
```

Chạy xong nhớ tự bắt `affected rows == 0` rồi dội Exception. Khuyến cáo không nên chạy mấy lệnh kiểu này song song với các luồng xài JPA thông thường nếu không Test kỹ.

## 8. CẤM tự động Thử Lại (Auto-retry) Hành vi của User

Quy trình vàng cho Màn hình nhập liệu (User edit):

```text
Kẻ đến trước (Winner) → Chốt sổ thành công (commit), trả về version mới.
Kẻ đến sau (Loser)  → Dọn dẹp phế tích (rollback), báo lỗi bắt User tự F5 tải lại form.
```

Trừ khi bạn cập nhật mấy cái thứ như "Đếm View", "Cộng Trừ Điểm Tích Lũy" (thứ mà kết quả không phụ thuộc vào trạng thái cũ), thì mới được phép thiết kế vòng lặp Retry tự động. (Muốn code Retry an toàn, đọc thêm `LOCK-002`).

## 9. Những Binh Khí Khác (Phương án thay thế)

### Cập nhật chèn điều kiện (Conditional compare-and-set)

Nếu không xài Entity Manager của JPA:

```sql
update product_offer
set price = :price,
    version = version + 1
where offer_id = :id
  and version = :expectedVersion
returning version; -- Lấy luôn version mới nhét vào log
```

Nó hoàn toàn tương đương Khóa Lạc Quan, chỉ khác là bạn phải tự gõ SQL.

### Khóa Bi Quan `FOR UPDATE`

Nghĩa là "Khóa ngay từ lúc Đọc". Kẻ đến sau sẽ bị Block treo máy đợi kẻ trước làm xong. Dùng trò này bạn vẫn phải bắt Client truyền cái `expected version` lên để biết nó có bấm form cũ hay không. Dùng cẩn thận coi chừng quá giờ (timeout) và Khóa chéo (deadlock).

### Cập nhật "Tương đối" (Atomic domain SQL)

Thay vì SET thẳng giá = 90, bạn cập nhật kiểu "Tăng giá cũ thêm 5%" (`price = price * 1.05`). Cách này rất đỉnh cao cho một số Logic đặc thù, nhưng không xài được cho các màn hình Admin Form nhập liệu.

### Chế độ `SERIALIZABLE`

Bật chế độ này thì DB lo hết rủi ro. Tuy nhiên, hiệu năng sẽ bị bóp nghẹt và code của bạn lúc nào cũng phải chực chờ DB dội lỗi `40001` (Serialization Failure) để... chạy lại toàn bộ luồng từ đầu.

## 10. Bảng Sinh Tử (Failure behavior)

| Tình Huống | Ở dưới Database | Trên API / Ứng Dụng |
| --- | --- | --- |
| Version khớp ngon lành | Chỉnh `1` dòng, version tăng | Gửi Success + kèm Version Mới Nhất |
| Client mang data thiu | SQL UPDATE bị chặn không chạy | Trả mã `412/409`, xúi Client F5 đi |
| Đua nhau sát nút (Race) | Chỉnh `0` dòng, Rollback ráo | Báo Lỗi Xung đột Lạc Quan |
| Quá giờ chờ (Timeout) | Rollback Giao Dịch | Báo Lỗi Kỹ Thuật (Đừng xúi Client F5) |
| Thằng Sửa đụng Thằng Xóa | Chỉnh `0` dòng | Báo lỗi Không Tìm Thấy (Not Found) / Conflict |
| Máy chủ sập trước lúc chốt | DB dọn dẹp (Rollback) | Lệnh chưa vào, có thể thử lại |
| Sập sau chốt, chưa kịp báo | Lệnh đã vô DB | Phải có Mã Request ID để tra cứu xem xong chưa |

## 11. Bảng Cân Nhắc Lợi Hại (Trade-off)

| Binh Khí | Sự Đúng Đắn | Mức Kẹt Xe (Contention) | Độ Trễ (Latency) | Khả năng chạy Đa Server |
| --- | --- | --- | --- | --- |
| **`@Version` (Lạc Quan)** | Phát hiện Ghi Đè cực tốt | **Đọc thoải mái không Lock** | Nổ lỗi vào phút chót | Ngon lành |
| SQL Có Điều Kiện (CAS) | Tương đương @Version | Nhẹ nhàng y chang | Nổ lỗi ở từng câu lệnh | Ngon lành |
| Khóa Bi Quan `FOR UPDATE` | Ép người ta xếp hàng | **Gây kẹt xe, dội bom DB** | Phải đứng chờ (Wait) | Ngon lành |
| Khóa Nhốt `synchronized` | Chỉ lừa trẻ con | Ai gọi trúng máy mình thì đợi | Không bảo vệ được DB | **Tạch** |

## 12. Bùa Chú Trước Khi Lên Môi Trường Thật (Checklist Production)

- [ ] Cột Version dưới DB đã gắn cờ `NOT NULL`, Class Entity dùng đúng `@Version`, và KHÔNG có hàm gán (setter).
- [ ] API đã bắt Client nộp thuế (truyền lên cái Expected Version / ETag).
- [ ] Soi thử câu SQL chạy ra thấy có cái đuôi kiểm tra và cộng Version.
- [ ] Các thanh niên viết SQL chay (batch/native) đã bị gõ đầu phải nhét logic tăng Version vào code của chúng nó.
- [ ] Block try/catch Xung Đột nằm LỚP NGOÀI CÙNG, chứ không dính dáng gì đến cái Giao Dịch đã sập hầm.
- [ ] Không có cái tính năng ngu xuẩn nào tự động Retry bắt Code thay mặt User gõ phím.
- [ ] Gửi Response Thành Công chỉ khi Database gật đầu cái rụp (Commit).
- [ ] Mấy cái Đồ Thị (Metrics) log lỗi không được chứa High-Cardinality (ví dụ: cấm log mã sản phẩm vào Label của Prometheus làm nổ RAM).
- [ ] Code Test chạy bằng PostgreSQL thật (Testcontainers) và moi được Dữ liệu cuối cùng ra đối chiếu.
