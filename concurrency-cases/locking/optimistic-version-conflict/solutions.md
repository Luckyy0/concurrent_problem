# Kiến trúc khắc phục: Tích hợp `@Version` và cơ chế đối chiếu phiên bản kỳ vọng

## 1. Tái cấu trúc lược đồ database và entity

Tiến trình đầu tiên yêu cầu xây dựng tệp migration để cấu trúc lại database:

```sql
alter table product_offer add column version bigint;
-- Khởi tạo giá trị an toàn cho các bản ghi hiện hữu
update product_offer set version = 0 where version is null;
alter table product_offer alter column version set not null;
```

(Lưu ý: Trong giai đoạn chuyển giao, tất cả các tác vụ xử lý SQL thủ công không chứa logic gia tăng `version` sẽ phá vỡ tính nhất quán của hệ thống.)

Cập nhật lớp entity:

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

    @Version // Khai báo cột kiểm soát khóa lạc quan
    @Column(nullable = false)
    private long version;

    protected ProductOffer() {
    }

    // Chỉ cung cấp phương thức getter, NGHIÊM CẤM khai báo phương thức setter cho version!
    public long version() {
        return version;
    }

    public void changePrice(BigDecimal newPrice) {
        if (newPrice == null || newPrice.signum() < 0) {
            throw new IllegalArgumentException("Tham số giá không hợp lệ");
        }
        price = newPrice;
    }
}
```

Trách nhiệm cập nhật chỉ số `version` được giao phó hoàn toàn cho hệ thống quản trị bền vững (Hibernate/JPA).

## 2. Thiết lập giao thức ràng buộc với phía gọi (DTO phiên bản kỳ vọng)

```java
// Yêu cầu payload từ phía gọi đính kèm phiên bản đối chiếu
public record ChangeOfferPrice(
        long offerId,
        long expectedVersion,
        BigDecimal newPrice,
        UUID commandId
) {
}
```

Kiến trúc phòng thủ tại tầng dịch vụ:

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

        // Ngăn chặn rủi ro gián đoạn (trạng thái hiển thị tại phía gọi đã lỗi thời)
        if (offer.version() != command.expectedVersion()) {
            throw new StaleOfferEditException(
                    command.offerId(),
                    command.expectedVersion(), // Phiên bản phía gọi truyền tải
                    offer.version()            // Phiên bản thực tế tại database
            );
        }

        offer.changePrice(command.newPrice());
        offers.flush();
        // Phương thức UPDATE sẽ tự động chứa truy vấn WHERE version=?
        return OfferView.from(offer);
    }
}
```

> **Nguyên tắc cốt lõi:** So khớp `expectedVersion` để đối phó với hiện tượng độ trễ giao diện từ phía người dùng. Annotation `@Version` của Hibernate phụ trách giám sát sự tương tranh tính bằng mili-giây tại database. Cả hai lớp phòng thủ là bắt buộc.

## 3. Quản trị ngoại lệ cạnh tranh thất bại

```sql
-- Truy vấn sinh ra bởi hệ thống
update product_offer
set price = :price,
    title = :title,
    version = :version + 1
where offer_id = :id
  and version = :version;
```

Khi luồng chậm hơn thực thi, số bản ghi trả về bằng `0`, Hibernate kích hoạt ngoại lệ `OptimisticLockException` (được Spring bao bọc trong `ObjectOptimisticLockingFailureException`). Sự kiện này phải được truyền ngược lại ra ngoài ranh giới transaction để tiến hành giải phóng tài nguyên (rollback).

## 4. Ánh xạ ngoại lệ sang quy chuẩn API

```java
@RestControllerAdvice
public class OfferConflictAdvice {

    // Thu thập toàn bộ các trường hợp xung đột trạng thái
    @ExceptionHandler({
            StaleOfferEditException.class,
            ObjectOptimisticLockingFailureException.class
    })
    ResponseEntity<ProblemDetail> conflict(RuntimeException failure) {
        ProblemDetail problem = ProblemDetail.forStatus(
                HttpStatus.PRECONDITION_FAILED // Mã tiêu chuẩn 412
        );
        problem.setTitle("Phiên bản dữ liệu không hợp lệ. Bản ghi đã bị sửa đổi.");
        problem.setProperty("reloadRequired", true); // Tín hiệu định hướng phía gọi
        return ResponseEntity
                .status(HttpStatus.PRECONDITION_FAILED)
                .body(problem);
    }
}
```

Tuyệt đối tránh việc trả về cấu trúc đối tượng (DTO/entity) bị lỗi thời dưới dạng HTTP `200 Success`. Sử dụng tiêu chuẩn `412 Precondition Failed` kết hợp ETag (If-Match) hoặc mã `409 Conflict` tùy thuộc vào thiết kế API.

## 5. Ranh giới đồng bộ và xác nhận (flush và commit)

Triển khai `flush()` chủ động trong phạm vi phương thức giúp phân giải ngoại lệ sớm nhất. Tuy nhiên, hành động commit chỉ được điều phối tại vùng biên ngoài cùng của Spring proxy. Tiến trình bắt lỗi (try-catch) phải được bố trí tại các lớp cấp cao (controller/advice), không can thiệp sâu vào bên trong transaction đã phát sinh xung đột.

## 6. Hạn chế phụ thuộc vào `merge` trên thực thể tách rời

Việc truyền DTO trực tiếp thành dạng entity tách rời và gọi `merge()` gây ra các nhược điểm sau:

- Phát sinh rủi ro cập nhật dữ liệu ngoài ý muốn (mass assignment).
- Tương tác không kiểm soát tới các thực thể con phụ thuộc (phụ thuộc lan truyền).
- Bỏ qua hoặc làm chậm nhịp độ kiểm tra version.

Chiến lược chuẩn xác là ánh xạ thủ công từ DTO của phía gọi lên thực thể được quản lý vừa được tải từ database, đồng thời kiểm tra version thủ công. Thiết kế này bảo đảm tính an toàn tối đa.

## 7. Yêu cầu cấu trúc dành cho truy vấn SQL thuần / cập nhật hàng loạt

Khi áp dụng phương thức thao tác không qua ORM:

```sql
update product_offer
set price = :newPrice,
    version = version + 1 -- Bắt buộc tích hợp thuật toán gia tăng phiên bản
where offer_id = :id
  and version = :expectedVersion; -- Đính kèm mệnh đề giám sát
```

Hệ thống phải tự định nghĩa phương thức xác thực `affected rows == 0` và khởi chạy exception giả lập. Lập trình viên cần giới hạn kỹ thuật này trong hệ thống hỗn hợp nếu không có quy trình integration test nghiêm ngặt.

## 8. Nguyên tắc cấm tự động thử lại đối với thay đổi từ người dùng

Quy trình chuẩn cho chức năng chỉnh sửa từ người dùng:

```text
Transaction đến trước (chiến thắng) → Hoàn tất commit, trả về phiên bản hệ thống mới.
Transaction chậm (thua cuộc)  → Rollback toàn bộ, truyền thông báo yêu cầu phía gọi hợp nhất / tải lại trang.
```

Ngoại trừ các thao tác mang tính chất bù trừ không phụ thuộc trạng thái cũ (ví dụ: thống kê số lượng truy cập, thay đổi khoảng giá trị - chỉnh sửa tương đối), mọi thao tác cập nhật giá trị tuyệt đối không được cấp quyền chạy vòng lặp retry. (Tham khảo phân tích tự động thử lại tại `LOCK-002`).

## 9. Đánh giá các biện pháp thay thế

### Cập nhật so sánh và gán có điều kiện (CAS)

Dành cho hệ thống không áp dụng JPA:

```sql
update product_offer
set price = :price,
    version = version + 1
where offer_id = :id
  and version = :expectedVersion
returning version; -- Trích xuất phiên bản mới nhất
```

Hoạt động theo nguyên tắc đồng nhất với khóa lạc quan nhưng thực thi trên hạ tầng SQL trực tiếp.

### Khóa bi quan `FOR UPDATE`

Thực thi cấp phát khóa đồng bộ ngay tại thời điểm đọc (khóa đọc). Transaction tiếp cận sau phải đi vào hàng chờ giải phóng. Bắt buộc kết hợp với phiên bản kỳ vọng từ phía gọi. Giải pháp này gây nguy cơ thắt cổ chai và deadlock ở quy mô cao.

### Cập nhật tương đối bằng toán tử nguyên tử

Thay vì thiết lập cấu trúc thay thế "Giá = 90", cấu trúc mệnh đề "Giá = Giá cũ + 5%" (`price = price * 1.05`). Phương pháp đặc thù phù hợp cho logic tài chính nhưng hạn chế đối với quản trị dữ liệu đầu vào từ biểu mẫu.

### Mức độ cách ly `SERIALIZABLE`

Môi trường database quản lý toàn bộ xung đột và giải quyết các bài toán hiện tượng phantom/read skew. Đánh đổi bằng hiệu suất giảm sút nghiêm trọng và buộc hệ thống luôn trong trạng thái tiếp nhận ngoại lệ `40001` (lỗi tuần tự hóa) để tự tái chạy các tác vụ liên đới.

## 10. Ma trận xử lý hành vi lỗi

| Biến cố transaction | Xử lý tại database | Xử lý tại ứng dụng / API |
| --- | --- | --- |
| Đối chiếu phiên bản thành công | Thay đổi `1` bản ghi, version tự tăng | Phản hồi thành công đi kèm version cập nhật |
| Yêu cầu từ phía gọi lỗi thời | Truy vấn UPDATE bị từ chối | Ánh xạ HTTP `412/409`, yêu cầu đồng bộ dữ liệu |
| Tình trạng tương tranh | Sửa đổi `0` dòng, transaction rollback | Báo cáo xung đột lạc quan |
| Giới hạn thời gian (timeout) | Hủy ngang transaction | Ghi nhận lỗi hệ thống cục bộ |
| Đụng độ thao tác xóa | Sửa đổi `0` dòng | Ghi nhận cấu trúc lỗi dữ liệu không tìm thấy |
| Trục trặc máy chủ trước commit | PostgreSQL thu hồi thay đổi | Báo cáo giao dịch chưa hoàn tất |
| Trục trặc máy chủ sau commit | Dữ liệu tích hợp an toàn | Cần ID tính lũy đẳng (idempotency) hỗ trợ tra cứu tiến trình |

## 11. Bảng ma trận đánh đổi kiến trúc

| Kỹ thuật bảo mật | Tính toàn vẹn | Khả năng tranh chấp | Độ trễ (latency) | Khả năng mở rộng nhiều instance |
| --- | --- | --- | --- | --- |
| **`@Version` (Khóa lạc quan)** | Phát hiện ghi đè tối đa | **Không áp đặt khóa khi đọc** | Phát sinh lỗi tại pha commit | Hiệu quả cao |
| So khớp thuần SQL (CAS) | Tương đồng `@Version` | Tương đồng `@Version` | Phát sinh lỗi trong truy vấn con | Hiệu quả cao |
| `FOR UPDATE` (Khóa bi quan) | Hàng đợi đồng bộ cao | **Gây rủi ro quá tải / thắt cổ chai** | Dễ dàng sinh trễ luồng xử lý | Hiệu quả cao |
| Khóa cục bộ (`synchronized`) | Vô nghĩa | Khóa luồng tại một tiến trình đơn | N/A | **Thất bại toàn diện** |

## 12. Danh mục kiểm tra cho môi trường thực tế

- [ ] Thuộc tính version cấp phát cờ `NOT NULL`, loại bỏ hàm setter khỏi entity.
- [ ] Thiết kế kiến trúc API tiếp nhận giá trị phiên bản kỳ vọng (ETag).
- [ ] Phân tích log SQL xác minh khối lượng truy vấn chứa mệnh đề `AND version=?`.
- [ ] Tệp tin script/batch ngoại vi được tích hợp logic tự động cập nhật phiên bản.
- [ ] Khối bắt lỗi `try/catch` dành cho tranh chấp định vị ngoài ranh giới transaction ảo.
- [ ] Nghiêm cấm ứng dụng hệ thống thử lại tự động trên các biểu mẫu dữ liệu nhập của người dùng.
- [ ] Phản hồi API thành công (HTTP 2xx) chỉ khả thi nếu database hoàn thành pha commit.
- [ ] Không cấp phát các định dạng thông tin có tính duy nhất cao (high-cardinality) vào công cụ đo chỉ số metrics.
- [ ] Bộ bài kiểm thử integration được cấu hình thông qua container database vật lý thay thế H2.
