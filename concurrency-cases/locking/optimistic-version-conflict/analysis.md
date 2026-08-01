# Phân Tích Chuyên Sâu: Cơ Chế Cột Version và Xung Đột Khóa Lạc Quan

## 1. Bối cảnh khởi động

Xem xét đối tượng sản phẩm `42` hiện có giá trị là `100` và đang ở phiên bản `7`. Có hai luồng transaction độc lập (Luồng A và Luồng B) đang vận hành ở mức độ cách ly `READ COMMITTED` (chỉ đọc dữ liệu đã được commit).

## 2. Diễn biến dòng thời gian khi áp dụng `@Version`

| Bước | Tiến trình A | Tiến trình B |
| ---: | --- | --- |
| 1 | Truy xuất dữ liệu: `100 / v7` | |
| 2 | | Truy xuất dữ liệu: `100 / v7` |
| 3 | Cập nhật bộ nhớ đệm: Giá = `90` | Cập nhật bộ nhớ đệm: Giá = `80` |
| 4 | Thực thi SQL: `UPDATE ... version=7` → Trả về `1` bản ghi | |
| 5 | Commit transaction: Trạng thái database = `90 / v8` | |
| 6 | | Thực thi SQL: `UPDATE ... version=7` → Trả về `0` bản ghi |
| 7 | | Hệ thống từ chối, transaction bị rollback |

Kết quả cuối cùng trong database: Sản phẩm `42` có giá `90`, phiên bản `8`. Tiến trình B không nhận được thông báo thành công ảo cũng như không tạo ra trạng thái dữ liệu ghi cục bộ.

> **Ghi chú kỹ thuật:** Cơ chế phiên bản kỳ vọng thay đổi nguyên tắc "ai ghi sau kẻ đó thắng" thành quy trình "kiểm chứng nguyên tử": Một transaction chỉ được phép cập nhật dữ liệu nếu nó nắm giữ đúng phiên bản hiện tại nhất.

## 3. Giao ước SQL

Khi tồn tại thuộc tính `@Version`, ORM (Hibernate) sẽ tự động tạo chuỗi truy vấn cập nhật:

```sql
update product_offer
set price = :newPrice,
    title = :title,
    version = :nextVersion       -- Hệ thống tự động gia tăng 1 đơn vị
where offer_id = :offerId
  and version = :expectedVersion; -- Đảm bảo tính nhất quán với phiên bản truy xuất
```

Số dòng bị ảnh hưởng trả về từ database đóng vai trò cốt lõi:

- Kết quả `1`: Phép kiểm tra thành công, bản ghi đã được cập nhật và phiên bản được gia tăng.
- Kết quả `0`: Phép kiểm tra thất bại, bản ghi không còn tồn tại hoặc phiên bản cơ sở đã bị sửa đổi bởi một transaction khác. ORM sẽ ghi nhận trạng thái dữ liệu hiện hành là lỗi thời.

Tránh việc phân tích chuỗi thông báo lỗi. Phương thức chuẩn là bắt (catch) các ngoại lệ tiêu chuẩn như `OptimisticLockException` (theo chuẩn Jakarta Persistence) hoặc `ObjectOptimisticLockingFailureException` (theo Spring Framework).

## 4. Kiểm soát đa phiên bản (MVCC) và khóa dòng (row lock)

Khóa lạc quan (`@Version`) KHÔNG thiết lập cơ chế lock trong quá trình đọc (`SELECT`). Tại cùng thời điểm, cả A và B đều được cấp quyền truy xuất phiên bản `7`.
Tuy nhiên, tại pha thực thi cập nhật:

- Transaction A sẽ yêu cầu và nhận được row lock từ database, thực hiện tăng phiên bản lên `8` và chờ lệnh commit.
- Nếu transaction B tiếp cận database trong khi A chưa hoàn tất commit, B sẽ bị đưa vào hàng đợi chờ lock.
- Sau khi A hoàn thành và giải phóng lock, B tiến hành thao tác cập nhật nhưng không tìm thấy bản ghi có `version=7` để sửa đổi.
- Do đó, số dòng bị ảnh hưởng của B là `0` và database không bị ghi đè dữ liệu.

Kỹ thuật "lạc quan" đề cập tới việc dời pha kiểm tra xung đột đến khâu commit dữ liệu, thay vì chặn luồng từ lúc truy xuất dữ liệu. Cơ chế row lock của database vật lý vẫn tham gia vào khâu xử lý đồng thời.

## 5. Kiểm tra thay đổi (dirty checking) và đồng bộ (flush)

Các thao tác thay đổi thuộc tính trên đối tượng Java chỉ tác động tới bộ nhớ đệm (RAM). Lệnh SQL thực tế chỉ được phát sinh trong các tình huống:

- Transaction yêu cầu đồng bộ chủ động (`EntityManager.flush()` hoặc `JpaRepository.flush()`).
- Tự động đồng bộ (auto-flush) trước các truy vấn đọc liên quan để đảm bảo tính nhất quán.
- Tại thời điểm commit transaction.

Điểm thời gian ném (throw) ngoại lệ phụ thuộc vào thao tác `flush`. Việc không chủ động gọi `flush` sẽ đẩy thời điểm bắt lỗi về sát ranh giới commit của Spring proxy, làm tăng độ trễ kiểm soát và gây khó khăn cho việc xử lý vòng lặp lỗi.

## 6. Xử lý vòng đời transaction sau xung đột

Khi `OptimisticLockException` xảy ra, ngữ cảnh transaction sẽ bị chỉ định rollback (`rollback-only`). Toàn bộ thực thể trong bộ đệm của tiến trình thua cuộc mất đi tính hợp lệ.

**CÁC PHẢN MẪU:**

- Bỏ qua ngoại lệ và trả về trạng thái thành công ảo.
- Sử dụng phương thức `clear()` để tiếp tục chuỗi xử lý trên transaction đã mất hiệu lực.
- Cố gắng lưu lại thực thể lỗi thời vào các transaction khác mà không tải lại trạng thái.

Ngoại lệ phát sinh đồng nghĩa với việc kết thúc transaction. Kế hoạch tiếp theo bắt buộc phải dựa trên một ngữ cảnh transaction hoàn toàn mới (tham khảo chi tiết `LOCK-002`).

## 7. Hai khoảng thời gian dễ tổn thương

Kiến trúc chia thành hai cửa sổ rủi ro gây ra tình trạng dữ liệu lỗi thời:

1. **Rủi ro gián đoạn:** Xảy ra giữa lúc dữ liệu hiển thị trên ứng dụng phía gọi và thời điểm gửi yêu cầu thay đổi. Trong thời gian này, database có thể đã bị sửa đổi.
2. **Rủi ro transaction:** Xảy ra ngay trong quá trình xử lý đồng bộ của máy chủ backend, từ lúc đọc dữ liệu cho tới trước thời điểm `flush`.

Giải pháp kiểm soát toàn diện bao gồm:
- So khớp giá trị `expectedVersion` do phía gọi cung cấp với đối tượng đọc từ database, giải quyết triệt để rủi ro gián đoạn.
- Sử dụng mệnh đề truy vấn `version` tự sinh bởi Hibernate bảo vệ hệ thống trước rủi ro transaction.

Triển khai tiêu chuẩn RESTful API bằng cơ chế header:

```text
Truy xuất (GET) /offers/42 → ETag: "7"
Cập nhật (PUT) /offers/42 kèm theo header If-Match: "7"
```

Khi phát sinh lỗi, hệ thống phải mã hóa phản hồi thành `412 Precondition Failed` hoặc `409 Conflict`. Tránh việc tiết lộ mô hình thực thể trong phần thân phản hồi nhằm bảo đảm tính ẩn danh cấu trúc.

## 8. Hợp nhất thực thể tách rời - Cân nhắc kỹ lưỡng

Hàm `merge()` được cung cấp để hợp nhất lại thực thể tách rời (detached). Mặc dù Hibernate sẽ phân tích cột `version` trong quá trình hợp nhất, quá trình đối chiếu thực tế vẫn có thể bị trì hoãn tới giai đoạn commit. Hàm này trả về một thực thể được quản lý mới, trong khi đối tượng tham số vẫn tách rời.

Khuyến nghị thực hành chuẩn cho kiến trúc DTO: Áp dụng phương thức gán thuộc tính cụ thể từ đối tượng DTO của phía gọi vào thực thể được lấy ra từ database trong CÙNG phiên làm việc, kết hợp so khớp giá trị version. Điều này giúp ngăn chặn triệt để lỗi cập nhật dữ liệu ngoài ý muốn và gia tăng mức kiểm soát dữ liệu. Yêu cầu CỐT LÕI: Luôn sử dụng giá trị version kỳ vọng từ phía gọi làm thông số quyết định.

## 9. Ranh giới transaction của tập hợp

Trách nhiệm của thuộc tính version là bảo vệ trạng thái nhất quán của đối tượng gốc tập hợp.

- Phiên bản bảo vệ một bản ghi riêng lẻ sẽ không kiểm soát được các vi phạm write skew ở cấp độ toàn cục.
- Cấu hình mapping quyết định việc cập nhật các thực thể con có kích hoạt gia tăng phiên bản của thực thể cha hay không.
- Hibernate không tự động áp dụng `version` trên các lệnh thao tác hàng loạt bằng SQL thuần. Việc quản lý phiên bản trong trường hợp này phụ thuộc hoàn toàn vào lập trình viên.

Hệ thống đòi hỏi sự tương tác qua nhiều thực thể phức tạp cần thiết lập môi trường integration test toàn diện.

## 10. Chính sách không thử lại tự động

Hành vi tự động tải lại dữ liệu mới nhất và tiến hành lưu đè từ phía backend (tự động thử lại) sẽ vi phạm nguyên tắc toàn vẹn dữ liệu trực quan: Tiến trình B đang thực hiện yêu cầu sửa đổi trên nền tảng của `version=7`. Hệ thống sẽ bóp méo ý định nghiệp vụ khi tự động hợp nhất các dữ liệu này trên nền tảng `version=8`.

Các thông điệp cảnh báo xung đột phải cung cấp trạng thái dữ liệu mới nhất, yêu cầu luồng thao tác của người dùng phê duyệt phương án giải quyết. Chỉ thực hiện kỹ thuật thử lại (retry) với các thay đổi liên quan đến tính toán khoảng (chỉnh sửa tương đối - ví dụ: cộng/trừ), chứ không áp dụng vào các tác vụ cập nhật tuyệt đối.

## 11. Các trạng thái ngoại lệ

- Commit transaction thành công: Phiên bản cập nhật và dữ liệu đồng nhất lưu vào database.
- Xung đột lạc quan: Hệ thống loại bỏ mọi thay đổi, tự động thu hồi row lock.
- Giới hạn thời gian khóa (lock timeout): Yêu cầu xử lý và cấu hình hoàn toàn khác biệt so với lỗi xung đột phiên bản.
- Sự cố hệ thống trước commit: Mọi transaction đang mở được PostgreSQL rollback.
- Sự cố hệ thống sau commit nhưng chưa báo cáo: Phía gọi sẽ không nhận được phản hồi. Sử dụng ID đảm bảo tính lũy đẳng để khôi phục hoặc tra cứu thay vì tiếp tục truyền lại phiên bản cũ.
- Xóa đồng thời: Hệ thống báo lỗi cập nhật 0 dòng. Nghiệp vụ cần ánh xạ nó thành ngoại lệ không tìm thấy dữ liệu, thay vì xung đột phiên bản.

## 12. Triển khai trong kiến trúc nhiều instance

Cơ chế `version` tồn tại như một ràng buộc vật lý tại hệ thống database. Nhờ vậy, thiết kế này hoàn toàn vô nhiễm với các yếu tố mở rộng node ứng dụng. Các từ khóa khóa mức ngôn ngữ (ví dụ: `synchronized` trong Java) chỉ giới hạn ở phạm vi bộ nhớ của một máy chủ và không có giá trị trong kiến trúc triển khai nhiều instance.

Quy trình kiểm toán cần kiểm soát nghiêm ngặt đối với bất kỳ câu lệnh cập nhật SQL thuần nào (di chuyển database, các tác vụ chạy theo lô). Bất kỳ tương tác thay đổi nào không điều hướng thuộc tính `version = version + 1` đều làm mất tính liền mạch của hệ thống bảo vệ.

## 13. So sánh cơ chế xử lý tương tranh

| Cơ chế | Xử lý tiến trình thua cuộc | Môi trường áp dụng |
| --- | --- | --- |
| **`@Version` (Khóa lạc quan)** | Ngoại lệ rollback, hủy luồng xử lý | Quản trị giao diện, độ tranh chấp từ thấp đến trung bình |
| Điều kiện truy vấn SQL | Thường trả mã lỗi nghiệp vụ | Bổ sung, điều chỉnh định lượng |
| Khóa bi quan `FOR UPDATE` | Đóng băng tiến trình / Ngắt transaction | Hàng đợi hoặc xử lý tuần tự có tính cạnh tranh lớn |
| Mức cách ly `SERIALIZABLE` | Cấp cờ lỗi yêu cầu thử lại (`40001`) | Dữ liệu phụ thuộc đa hệ thống phức tạp |

## 14. Bản đồ trách nhiệm các tầng

| Tầng phân lớp | Nhiệm vụ yêu cầu |
| --- | --- |
| Tầng hiển thị / API | Phải quản trị `expectedVersion`, đảm bảo việc luân chuyển trạng thái không bị gián đoạn. |
| Spring Framework | Tổ chức ngữ cảnh transaction, chuyển đổi ngoại lệ từ lõi JDBC. |
| Hibernate / JPA | Áp dụng dirty checking, tự động tính toán thuộc tính `version` và kích hoạt lỗi theo số dòng bị ảnh hưởng. |
| Database | Thiết lập hệ thống MVCC, cấp quyền row lock và kiểm soát tính ACID. |

## 15. Tiêu chuẩn giám sát kỹ thuật

Yêu cầu vận hành trên môi trường thực tế (production):

- Thu thập các chỉ số về số lượng xung đột khóa lạc quan phân nhóm theo domain hoặc bảng.
- Tích hợp thông tin log có cấu trúc lưu trữ các trường dữ liệu phiên bản kỳ vọng và phiên bản hiện tại khi phát sinh ngoại lệ.
- Thống kê tỷ lệ phản hồi lỗi HTTP (ví dụ: `409 Conflict`) để đánh giá khả năng phản ứng lại từ phía gọi.
- Giám sát độ trễ transaction, số lượng câu lệnh SQL và vị trí phân bổ `flush`.
- Biến động dữ liệu bất thường (tăng đột biến lỗi khóa lạc quan) là cơ sở truy vết các sự cố bỏ qua phiên bản.
