# Phân tích mất cập nhật và ngữ nghĩa thao tác giỏ hàng

## 1. Trạng thái ban đầu

Giỏ `CART-42` đang hoạt động và có phiên bản `7`:

| Sản phẩm | Số lượng |
| --- | ---: |
| `SKU-A` | 1 |

Hai tab đã tải cùng ảnh chụp này. Tab A muốn thêm `SKU-B`; tab B muốn xóa
`SKU-A`.

Nếu diễn giải hai ý định theo một thứ tự hợp lệ, kết quả cuối phải là:

```text
[SKU-A] --thêm SKU-B--> [SKU-A, SKU-B] --xóa SKU-A--> [SKU-B]
```

Đảo thứ tự vẫn cho `[SKU-B]`. Vì vậy `[]` và `[SKU-A, SKU-B]` đều chứng minh có
một ý định bị mất.

## 2. Khoảng hở đọc–sửa–ghi

```text
thời điểm    giao dịch A                    giao dịch B
---------    ----------------------------   ----------------------------
t1           BEGIN                          BEGIN
t2           đọc giỏ phiên bản 7            đọc giỏ phiên bản 7
t3           tạo [A, B] trong Java           tạo [] trong Java
t4           ghi thay thế                    chưa ghi
t5           COMMIT                          ghi thay thế
t6                                          COMMIT
```

Mỗi giao dịch tự nhất quán với ảnh chụp nó đã đọc, nhưng không giao dịch nào
xác nhận ảnh chụp còn mới trước khi ghi. Cơ sở dữ liệu chỉ tuần tự hóa các câu
lệnh thực sự đụng cùng dòng; nó không hiểu rằng danh sách được tính từ một quyết
định cũ trong Java.

## 3. Kết quả mong đợi và kết quả thực tế

| Khía cạnh | Mong đợi | Khi thay toàn bộ không có phiên bản |
| --- | --- | --- |
| Thay đổi của A | `SKU-B` còn trong giỏ | Có thể bị B xóa mất |
| Thay đổi của B | `SKU-A` không còn | Có thể bị A làm xuất hiện lại |
| Phản hồi | Một bên thành công, một bên biết có xung đột | Cả hai có thể nhận thành công |
| Trạng thái | Tương ứng một thứ tự hợp lệ của hai ý định | Tương ứng ảnh chụp của bên ghi sau |
| Khả năng xử lý | Giao diện có thể tải lại và hỏi người dùng | Xung đột bị che giấu |

Đây là lỗi nghiệp vụ ngay cả khi các ràng buộc khóa ngoại, duy nhất và số lượng
đều hợp lệ. Ràng buộc cấu trúc không thể xác định thay đổi nào người dùng muốn
giữ.

## 4. Vì sao `READ COMMITTED` không đủ

Ở mức cô lập mặc định của PostgreSQL, mỗi câu lệnh nhìn thấy dữ liệu đã chốt
trước khi câu lệnh bắt đầu. Khi một `UPDATE` phải chờ giao dịch khác, PostgreSQL
có thể đánh giá lại điều kiện `WHERE` trên phiên bản dòng mới nhất.

Đây là cơ chế điều khiển đồng thời đa phiên bản (MVCC). Nó giúp thao tác đọc ít
chặn thao tác ghi, nhưng không thể tự biết một quyết định trong Java đang dựa
trên ảnh chụp cũ.

Điều đó chỉ giúp nếu điều kiện mang theo giả định nghiệp vụ:

```sql
WHERE cart_id = :cartId
  AND version = :expectedVersion
```

Nếu câu lệnh chỉ có `WHERE cart_id = :cartId`, bên chờ vẫn được phép ghi sau khi
khóa được nhả. Khóa bảo vệ tính toàn vẹn vật lý của dòng, không tự biết giá trị
được tính từ phiên bản `7`.

## 5. Đặt lớp bảo vệ ở đâu

| Lớp | Có thể làm gì | Không đủ nếu đứng một mình |
| --- | --- | --- |
| Giao diện | Gửi `ETag`, hiển thị xung đột, cho người dùng chọn | Có thể bị bỏ qua hoặc gửi yêu cầu song song |
| Java/Spring | Kiểm tra đầu vào, diễn giải lệnh, ánh xạ lỗi | Hai JVM không dùng chung bộ nhớ |
| JPA/Hibernate | Tạo câu cập nhật có điều kiện từ `@Version` | Đường SQL trực tiếp có thể bỏ qua phiên bản |
| PostgreSQL | Quyết định nguyên tử bằng phiên bản và giao dịch | Không tự suy ra ý định nếu API chỉ gửi ảnh chụp |

Bất biến được chốt ở PostgreSQL. Hợp đồng API phải cung cấp phiên bản dự kiến,
còn Java phải bảo đảm mọi đường ghi đều sử dụng cùng cơ chế.

## 6. Phiên bản phải thuộc toàn bộ giỏ

Giỏ hàng là một tập hợp gồm dòng gốc và nhiều dòng sản phẩm. Một phiên bản trên
từng `cart_item` không phát hiện hai thay đổi ở hai sản phẩm khác nhau, dù chúng
cùng được tạo từ một ảnh chụp toàn giỏ.

```text
shopping_cart.version = phiên bản của toàn bộ ảnh chụp

mọi INSERT / UPDATE / DELETE cart_item
    ⇒ phải kiểm tra phiên bản cũ
    ⇒ phải tạo phiên bản giỏ mới trong cùng giao dịch
```

Điều này có chủ ý làm hai thao tác trên hai sản phẩm khác nhau xung đột. Đổi lại,
phía gọi luôn có một mã phiên bản dễ hiểu và các quy tắc toàn giỏ, như giới hạn
số dòng, không bị chia nhỏ.

## 7. Cách kiểm tra rồi đổi tạo thứ tự

Hai giao dịch cùng chạy:

```sql
UPDATE shopping_cart
SET version = version + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE cart_id = :cartId
  AND version = 7
  AND status = 'ACTIVE'
RETURNING version;
```

Dòng thời gian:

```text
giao dịch A                         giao dịch B
--------------------------------   --------------------------------
UPDATE ... version = 7             UPDATE ... version = 7
khóa dòng, nhận version 8           chờ dòng shopping_cart
sửa cart_item                       ...
COMMIT                              được đánh thức
                                    kiểm tra lại version = 7
                                    không khớp, nhận 0 dòng
                                    ROLLBACK
```

Bên thua có thể đã chờ khóa trong thời gian ngắn, nhưng nó không được phép sửa
bảng con. Một dòng trả về là quyền tiếp tục; không dòng trả về là kết quả cần
phân loại.

## 8. Phân loại trường hợp không có dòng

Câu kiểm tra phiên bản gộp ba nguyên nhân: không tồn tại, sai trạng thái và sai
phiên bản. Ứng dụng có thể đọc lại sau khi câu lệnh trả `0`:

| Trạng thái đọc lại | Kết quả nghiệp vụ |
| --- | --- |
| Không có giỏ | `CART_NOT_FOUND` |
| `status != ACTIVE` | `CART_NOT_MUTABLE` |
| `version != expectedVersion` | `CART_VERSION_CONFLICT` |

Lần đọc để phân loại không cấp quyền ghi lại. Một giao dịch khác có thể tiếp tục
đổi giỏ sau đó; phản hồi chỉ cần cho biết yêu cầu hiện tại không được áp dụng và
phiên bản cung cấp là ảnh chụp để đồng bộ lại.

## 9. Khác biệt giữa các loại ý định

### 9.1. Tăng thêm số lượng

`AddItem(SKU-A, +2)` mô tả chênh lệch. Hai lệnh `+2` và `+3` có thể kết hợp thành
`+5`, miễn là kiểm tra lại giới hạn số lượng và tồn tại sản phẩm. Nếu tự thử lại,
mỗi lệnh cần mã duy nhất để mất phản hồi không làm cộng hai lần.

### 9.2. Đặt số lượng đích

`SetQuantity(SKU-A, 2)` không có tính giao hoán. Một tab đặt `2`, tab khác đặt
`5`; tự áp lại trên phiên bản mới chỉ khiến bên thử lại luôn thắng. Đây là quyết
định của người dùng, không phải lỗi tạm thời để thư viện thử lại mù.

### 9.3. Xóa sản phẩm

Xóa có vẻ lũy đẳng vì xóa hai lần vẫn là không có dòng. Nhưng nếu một tab xóa
sản phẩm cũ còn tab khác vừa thêm lại sản phẩm với cấu hình mới, tự xóa lại có
thể loại bỏ ý định mới. Phiên bản của toàn giỏ giữ lựa chọn này rõ ràng.

### 9.4. Thay toàn bộ giỏ

Ảnh chụp đầy đủ mang ít thông tin nhất về thay đổi. Không thể biết phần tử vắng
mặt là do người dùng xóa hay do tab chưa từng nhìn thấy. Vì vậy, yêu cầu này phải
khớp phiên bản và không được tự gộp.

## 10. JPA `@Version` và thời điểm phát hiện

Với trường `@Version` trên `ShoppingCart`, Hibernate sinh câu tương đương:

```sql
UPDATE shopping_cart
SET updated_at = ?,
    version = ?
WHERE cart_id = ?
  AND version = ?;
```

Nếu số dòng cập nhật là `0`, Hibernate ném ngoại lệ khóa lạc quan. Có ba điểm cần
chú ý:

1. Chỉ sửa bộ sưu tập con không phải lúc nào cũng làm gốc bị đánh dấu thay đổi
   theo cách ánh xạ và nhà cung cấp mong muốn. Nên thay đổi một trường của gốc,
   chẳng hạn `updatedAt`, và kiểm thử SQL thực tế.
2. Ngoại lệ thường xuất hiện ở `flush` hoặc khi chốt giao dịch, không nhất thiết
   tại `save()`.
3. Sau ngoại lệ, giao dịch và vùng quản lý thực thể phải bị bỏ. Nếu thử lại, phải
   mở giao dịch mới và tải lại toàn bộ dữ liệu.

Ngay cả khi Hibernate đã gửi câu lệnh bảng con trước khi phát hiện xung đột ở
dòng gốc, thao tác hoàn tác (`rollback`) phải xóa toàn bộ thay đổi. Cách kiểm tra
phiên bản bằng SQL ở đầu giao dịch phát hiện sớm hơn và làm đường ghi bảng con
rõ ràng hơn.

## 11. Chốt, hoàn tác và phản hồi

### Chốt thành công

Phản hồi chỉ được xem là thành công sau khi lớp quản lý giao dịch của Spring đã
chốt. Phiên bản mới trong phản hồi phải là phiên bản của trạng thái bền vững.

### Lỗi sau khi đã tăng phiên bản

Nếu chèn, sửa hoặc xóa `cart_item` vi phạm ràng buộc, toàn giao dịch phải hoàn
tác. `shopping_cart.version` cũng trở về giá trị cũ; không để lại phiên bản rỗng.

### Ngoại lệ tại `flush`

Gọi `flush()` giúp phát hiện xung đột trước khi dựng phản hồi bên trong dịch vụ.
Tuy nhiên, `flush` thành công chưa phải `COMMIT`; các lỗi muộn hơn vẫn phải làm
giao dịch thất bại.

### Hết thời gian chờ và bế tắc

`lock_timeout`, hủy câu lệnh hoặc nạn nhân bế tắc là lỗi kỹ thuật. Không ánh xạ
chúng thành `CART_VERSION_CONFLICT`, vì phía gọi cần chính sách thử lại khác.

## 12. Thử lại và khuếch đại tranh chấp

Khóa lạc quan phù hợp khi xung đột tương đối hiếm. Với một giỏ nóng, nhiều bên
cùng đọc một phiên bản sẽ tạo nhiều giao dịch thất bại; thử lại ngay lập tức làm
tăng tải và tiếp tục xung đột.

Chỉ tự thử lại khi lệnh có ngữ nghĩa an toàn, không có tác dụng ngoài giao dịch,
có mã chống gửi trùng và số lần thử bị giới hạn. Mỗi lần thử phải:

1. mở giao dịch mới;
2. tải phiên bản mới;
3. kiểm tra lại trạng thái và giới hạn;
4. áp lại chính ý định, không áp ảnh chụp cũ;
5. dùng khoảng chờ ngẫu nhiên ngắn nếu cần giảm va chạm.

## 13. Sự cố tiến trình và kết quả chốt chưa rõ

Nếu tiến trình dừng trước `COMMIT`, PostgreSQL hoàn tác cả phiên bản và sản phẩm.
Nếu máy chủ chốt thành công nhưng mất kết nối trước khi trả phản hồi, phía gọi
không biết thao tác đã được áp dụng hay chưa.

- Với `SetQuantity` hoặc `RemoveItem`, tải lại giỏ có thể cho biết trạng thái mới,
  nhưng không luôn chứng minh lệnh nào đã tạo ra nó.
- Với `AddItem(delta)`, gửi lại mù có thể cộng lần hai. Cần `commandId` cùng bảng
  kết quả lệnh nếu API hỗ trợ tự động phát lại an toàn.

Phiên bản xử lý xung đột giữa các ý định khác nhau; mã lệnh xử lý cùng một ý định
được gửi lại. Hai cơ chế không thay thế nhau.

## 14. Nhiều máy chủ và mọi đường ghi

Máy chủ A và B không cần liên lạc trực tiếp. Điều kiện trên
`shopping_cart.version` tạo cùng một thứ tự cho tất cả kết nối PostgreSQL.

Rủi ro lớn nhất là một đường ghi bỏ qua gốc:

```text
API cập nhật giỏ       ─┐
tác vụ dọn giỏ         ├─ phải dùng cùng cổng thay đổi và cùng phiên bản
công cụ hỗ trợ         ┤
SQL bảo trì trực tuyến ─┘
```

Nên giới hạn quyền cập nhật trực tiếp, tìm kiếm các truy vấn bulk vào
`cart_item`, ghi số liệu phiên bản trước/sau và có test tích hợp chứng minh mọi
thao tác làm phiên bản tăng.

## 15. Dữ liệu quan sát cần có

- số xung đột phiên bản theo loại lệnh;
- khoảng cách giữa phiên bản dự kiến và phiên bản hiện tại;
- tỷ lệ và số lần thử lại của lệnh tăng chênh lệch;
- thời gian chờ khóa, `lock_timeout` và bế tắc;
- số phản hồi mất kết nối phải tra cứu lại;
- số lần nội dung `cart_item` đổi mà phiên bản giỏ không đổi;
- độ trễ và kích thước giỏ tại lúc sửa.

Không dùng `cartId` làm nhãn số liệu có số lượng giá trị không giới hạn. Mã này
chỉ nên xuất hiện trong nhật ký có kiểm soát hoặc dấu vết truy vết.
