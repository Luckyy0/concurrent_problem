# ECOM-002 — Tranh chấp tồn kho nóng trong đợt bán cao điểm

## 1. Bài toán

Một sản phẩm trong chương trình bán giới hạn nhận lượng yêu cầu lớn trong thời
gian ngắn. Tất cả yêu cầu đều cập nhật cùng một dòng `inventory_item`.

Case ECOM-001 đã dùng câu `UPDATE` có điều kiện nên tồn kho không bị âm và không
bán vượt số lượng. Tuy nhiên, tính đúng đắn của dữ liệu chưa bảo đảm hệ thống có
thể phục vụ ổn định:

```text
nhiều yêu cầu cùng vào
→ nhiều giao dịch cùng lấy kết nối
→ cùng chờ khóa một dòng sản phẩm
→ vùng kết nối bị chiếm hết
→ yêu cầu khác cũng không lấy được kết nối
→ hết thời gian phản hồi và phát sinh thêm lần thử lại
```

Đây là vấn đề **tranh chấp trên sản phẩm nóng** (`hot-stock contention`). Dữ liệu
có thể vẫn đúng trong khi khả năng phục vụ của toàn hệ thống suy giảm.

> **Nói ngắn gọn:** Cơ sở dữ liệu bảo vệ số lượng hàng; lớp điều tiết đầu vào bảo
> vệ khả năng xử lý của cơ sở dữ liệu.

## 2. Tác nhân và tài nguyên dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Người mua | Gửi yêu cầu giữ cùng một sản phẩm |
| Các máy chủ ứng dụng | Nhận yêu cầu song song sau bộ cân bằng tải |
| Vùng kết nối | Giới hạn số giao dịch có thể cùng truy cập PostgreSQL |
| Dòng `inventory_item` | Điểm tranh chấp duy nhất của sản phẩm nóng |
| PostgreSQL | Nguồn dữ liệu có thẩm quyền đối với tồn kho |
| Yêu cầu khác | Cùng dùng vùng kết nối nhưng không liên quan đợt bán |

Mỗi yêu cầu đang chờ khóa dòng vẫn có thể giữ một kết nối. Vì vậy hàng đợi khóa
của một sản phẩm có thể chiếm tài nguyên vốn dành cho toàn bộ ứng dụng.

## 3. Quy tắc bất biến và mục tiêu vận hành

### Quy tắc bất biến

```text
available_quantity >= 0

available_quantity + reserved_quantity = on_hand_quantity

tổng lượng giữ hàng đã chốt không vượt quá on_hand_quantity
```

### Mục tiêu vận hành

- Số giao dịch cùng đi vào đường xử lý sản phẩm nóng phải có giới hạn.
- Yêu cầu không được chờ vô hạn để lấy kết nối hoặc khóa dòng.
- Kết quả hết hàng không được thử lại.
- Lỗi quá tải phải được trả rõ là `BUSY` hoặc `TOO_MANY_REQUESTS`, không giả thành
  hết hàng.
- Lưu lượng của đợt bán không được chiếm toàn bộ tài nguyên của chức năng khác.
- Mọi cơ chế từ chối sớm chỉ bảo vệ tải; câu SQL có điều kiện vẫn bảo vệ tồn kho.

## 4. Vì sao câu SQL đúng vẫn có thể gây nghẽn

Câu lệnh sau bảo vệ dữ liệu đúng cách:

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE product_id = :productId
  AND available_quantity >= :quantity;
```

Khi nhiều giao dịch cùng chạy, PostgreSQL chỉ cho một bên sửa dòng tại một thời
điểm. Các bên còn lại chờ khóa và lần lượt kiểm tra lại mệnh đề `WHERE`.

Cơ chế này không tạo bão xung đột như `@Version`, nhưng nó vẫn tạo hàng đợi. Nếu
số yêu cầu đến nhanh hơn tốc độ dòng nóng có thể được cập nhật, hàng đợi sẽ tăng.
Mở thêm kết nối chỉ cho phép nhiều giao dịch chờ hơn; nó không làm một dòng được
ghi song song.

## 5. Các hàng đợi cần phân biệt

Một yêu cầu có thể chờ ở nhiều tầng:

```text
bộ cân bằng tải
→ luồng xử lý HTTP
→ cổng điều tiết sản phẩm nóng
→ vùng kết nối
→ khóa dòng PostgreSQL
→ phản hồi cho phía gọi
```

Nếu không có cổng điều tiết, vùng kết nối và PostgreSQL trở thành nơi hấp thụ
toàn bộ đợt tăng tải. Khi đó việc bảo vệ diễn ra quá muộn và ảnh hưởng sang các
yêu cầu không liên quan.

## 6. Hướng xử lý được khuyến nghị

Thiết kế đồng bộ nên có các lớp sau:

1. Giới hạn số yêu cầu sản phẩm nóng được phép vào giao dịch trên mỗi máy chủ.
2. Từ chối nhanh yêu cầu vượt giới hạn bằng một kết quả có thể nhận biết.
3. Giữ giao dịch ngắn và tiếp tục dùng `UPDATE` có điều kiện.
4. Đặt giới hạn riêng cho thời gian lấy kết nối, chờ khóa và chạy câu lệnh.
5. Không tự động thử lại kết quả hết hàng hoặc lỗi quá tải trong cùng đường xử lý.
6. Dành ngân sách kết nối cho các chức năng không thuộc đợt bán.

Nếu nghiệp vụ không chấp nhận từ chối nhanh và cho phép trả kết quả sau, có thể
chuyển sang hàng đợi bền vững được phân vùng theo `product_id`. Khi đó API tiếp
nhận yêu cầu, trả mã đã nhận và khách hàng tra cứu kết quả sau. Cơ chế này thay
đổi hợp đồng API và kéo theo yêu cầu chống xử lý trùng, phục hồi và theo dõi tồn
đọng.

## 7. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh | Ý nghĩa |
| --- | --- | --- |
| sản phẩm nóng | hot key / hot row | Một sản phẩm nhận nhiều thao tác ghi cùng lúc |
| hàng đợi khóa | lock queue | Các giao dịch đang chờ quyền sửa cùng một dòng |
| bão thử lại | retry storm | Lần thử lại tạo thêm tải và tiếp tục thất bại |
| điều tiết đầu vào | admission control | Giới hạn số yêu cầu được phép đi vào phần tài nguyên khan hiếm |
| tạo áp lực ngược | backpressure | Buộc phía gửi giảm tốc độ hoặc chờ ngoài cơ sở dữ liệu |
| từ chối tải | load shedding | Từ chối sớm một phần yêu cầu để bảo vệ hệ thống |
| vùng cô lập tài nguyên | bulkhead | Tách ngân sách để một nhóm tải không chiếm hết tài nguyên chung |
| ngân sách chờ | timeout budget | Giới hạn thời gian cho từng bước trong thời hạn phản hồi tổng |

## 8. Hợp đồng kết quả

| Kết quả | Ý nghĩa | Có tự động thử lại không? |
| --- | --- | --- |
| `RESERVED` | Giao dịch đã giữ hàng và chốt thành công | Không |
| `OUT_OF_STOCK` | Câu cập nhật không tìm thấy dòng đủ hàng | Không |
| `BUSY` | Cổng điều tiết không cấp quyền xử lý | Không thử ngay; phía gọi tuân theo chính sách đã công bố |
| SQLSTATE `55P03` | Hết thời gian chờ khóa | Chỉ thử lại có giới hạn nếu còn thời hạn tổng |
| Lỗi lấy kết nối | Không vào được giao dịch | Từ chối sớm; không lặp tức thì |
| Kết quả chốt chưa rõ | Mất phản hồi quanh thời điểm `COMMIT` | Tra cứu bằng mã yêu cầu; thuộc ECOM-003 |

`BUSY` và `OUT_OF_STOCK` không cùng ý nghĩa. Một sản phẩm bận vẫn có thể còn hàng;
một sản phẩm hết hàng không trở nên còn hàng nhờ thử lại ngay.

## 9. Chạy trên nhiều máy chủ

`Semaphore` trong Java chỉ giới hạn tải của một máy chủ. Nếu có nhiều máy chủ,
tổng số yêu cầu được phép vào cơ sở dữ liệu là tổng giới hạn của tất cả máy chủ.
Khi tăng số bản sao ứng dụng, phải đánh giá lại ngân sách kết nối và mức tải tổng.

Cổng cục bộ vẫn hữu ích để bảo vệ vùng kết nối của từng máy, nhưng không được mô
tả như một khóa phân tán hay lớp bảo vệ tính đúng đắn. Tồn kho vẫn được bảo vệ tại
PostgreSQL bằng câu cập nhật có điều kiện.

## 10. Hậu quả khi triển khai sai

- Vùng kết nối bị lấp đầy bởi các giao dịch chỉ đang chờ một dòng.
- API không liên quan cũng chậm hoặc hết thời gian lấy kết nối.
- Lần thử lại từ ứng dụng, cổng API và khách hàng cùng khuếch đại tải.
- Yêu cầu đã quá thời hạn phản hồi vẫn tiếp tục làm việc trong cơ sở dữ liệu.
- Tăng kích thước vùng kết nối làm PostgreSQL nhận thêm bên chờ nhưng không tăng
  tốc độ ghi của dòng nóng.
- Bộ nhớ ứng dụng tăng do hàng đợi không giới hạn.
- Một cờ “hết hàng” lưu tạm bị dùng như nguồn sự thật và từ chối sai sau khi nhập
  thêm hàng.

## 11. Điều hướng tài liệu

- [Mã nguồn gây quá tải](broken-code.md)
- [Phân tích các hàng đợi và dòng thời gian](analysis.md)
- [Thiết kế điều tiết và xử lý](solutions.md)
- [Thực nghiệm với PostgreSQL](experiments.md)
- [ECOM-001 — Bán vượt tồn kho](../overselling-inventory/README.md)
- [LOCK-005 — Lựa chọn chiến lược khi tranh chấp cao](../../locking/high-contention-strategy-selection/README.md)
- [Cập nhật dữ liệu an toàn bằng SQL](../../concepts/atomic-database-operations.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## 12. Khi nên dùng từng hướng

Điều tiết đồng bộ và từ chối nhanh phù hợp khi API cần trả kết quả ngay, hệ thống
chấp nhận báo bận và lượng tồn kho được xử lý bằng giao dịch ngắn.

Hàng đợi bền vững phù hợp khi cần hấp thụ đợt tăng tải, có thể trả kết quả sau và
đội vận hành chấp nhận độ phức tạp của xử lý bất đồng bộ. Hàng đợi không làm tăng
số sản phẩm có thể bán; nó chỉ chuyển nơi chờ ra khỏi vùng kết nối PostgreSQL.

## 13. Phạm vi

Case này phân tích kiến trúc khi một sản phẩm đúng về dữ liệu nhưng quá nóng về
tải. Không đưa ra ngưỡng thông lượng, kích thước vùng kết nối hoặc số giấy phép
chung cho mọi hệ thống. Các giá trị phải được đo trên hạ tầng thật.

Chống tạo đơn trùng thuộc `ECOM-003`. Chi tiết giao hàng qua hệ thống thông điệp,
thứ tự phân vùng và xử lý lại thuộc nhóm `MSG-*`. Việc chia tồn kho giữa nhiều
kho hoặc nhiều dịch vụ không nằm trong phạm vi case này.
