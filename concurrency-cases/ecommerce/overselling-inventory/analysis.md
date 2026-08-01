# Phân tích lỗi bán vượt tồn kho

## 1. Trạng thái ban đầu

```text
Sản phẩm 77:
  on_hand_quantity  = 5
  available_quantity = 5
  reserved_quantity  = 0
  version             = 0

inventory_reservation: chưa có bản ghi
```

Yêu cầu A và B là hai lần mua hợp lệ, mỗi lần cần `4` sản phẩm. Chúng có kết
nối, giao dịch và ngữ cảnh Hibernate riêng. Hai yêu cầu có thể chạy trên hai máy
chủ ứng dụng khác nhau.

## 2. Kết quả mong đợi

Một trong hai yêu cầu được chấp nhận, yêu cầu còn lại bị từ chối vì không đủ
hàng:

```text
available_quantity = 1
reserved_quantity = 4
tổng lượng giữ hàng đã chốt = 4
```

Không quan trọng A hay B thắng. Điều bắt buộc là chỉ có một bên thắng.

## 3. Cách hai giao dịch cùng vượt qua bước kiểm tra

| Bước | Giao dịch A | Giao dịch B | PostgreSQL |
| --- | --- | --- | --- |
| 1 | `SELECT` sản phẩm 77 | | Trả ảnh chụp có số lượng `5` cho A |
| 2 | | `SELECT` sản phẩm 77 | Trả ảnh chụp có số lượng `5` cho B |
| 3 | Java kiểm tra `5 >= 4` | Java kiểm tra `5 >= 4` | Không biết về quyết định trong Java |
| 4 | Đổi thực thể thành `1/4` | Đổi thực thể thành `1/4` | Chưa có câu ghi nào nếu Hibernate chưa `flush()` |
| 5 | Chèn bản ghi giữ hàng A | Chèn bản ghi giữ hàng B | Hai khóa chính khác nhau nên đều hợp lệ |
| 6 | `UPDATE` tồn kho thành `1/4` | | A lấy khóa dòng |
| 7 | Chốt | `UPDATE` tồn kho thành `1/4` | B chờ khóa của A |
| 8 | | Chốt | B ghi đè cùng giá trị và cập nhật một dòng |

Khóa dòng đã tuần tự hóa hai câu ghi vật lý, nhưng không tuần tự hóa quyết định
nghiệp vụ được đưa ra trước đó.

## 4. Kết quả thực tế

| Dữ liệu | Mong đợi | Kết quả lỗi |
| --- | --- | --- |
| Yêu cầu được chấp nhận | `1` | `2` |
| `available_quantity` | `1` | `1` |
| `reserved_quantity` | `4` | `4` |
| Tổng lượng trong bảng giữ hàng | `4` | `8` |
| Ràng buộc trên dòng tồn kho | Đúng | Vẫn đúng |
| Đối soát hai bảng | Khớp | Lệch `4` |

Điểm nguy hiểm là dòng tồn kho không âm. Nếu chỉ theo dõi điều kiện
`available_quantity >= 0`, hệ thống có thể không phát hiện lỗi cho đến lúc giao
hàng hoặc chạy đối soát.

## 5. Nguyên nhân gốc

Chuỗi thao tác sau không nguyên tử:

```text
SELECT → kiểm tra trong Java → thay đổi thực thể → UPDATE
```

Giá trị được dùng để quyết định có thể đã cũ trước khi câu `UPDATE` được thực
thi. Câu ghi không mang theo điều kiện “số lượng vẫn còn ít nhất là 4” và cũng
không mang theo điều kiện “phiên bản vẫn là 0”. Vì vậy PostgreSQL không có căn
cứ để từ chối lần ghi đến sau.

## 6. Trách nhiệm của từng tầng

| Tầng | Điều xảy ra | Điều tầng đó không tự làm |
| --- | --- | --- |
| Java | Kiểm tra số lượng trên đối tượng trong bộ nhớ | Không biết đối tượng đã cũ nếu không tải lại |
| Spring | Mở và đóng giao dịch cho phương thức | Không tự khóa dòng chỉ vì có `@Transactional` |
| Hibernate | Theo dõi thay đổi và ghi khi `flush()` | Không phát hiện xung đột nếu thiếu `@Version` |
| PostgreSQL | Khóa dòng khi thực hiện `UPDATE` | Không biết điều kiện nghiệp vụ chỉ tồn tại trong Java |

Lỗi không phải do PostgreSQL cho phép hai câu `UPDATE` sửa cùng dòng trong cùng
một thời điểm. PostgreSQL vẫn khóa đúng. Vấn đề là câu ghi thứ hai hợp lệ về mặt
SQL dù quyết định của nó đã lỗi thời.

## 7. Ảnh chụp dữ liệu ở `READ COMMITTED`

Mỗi câu lệnh trong `READ COMMITTED` nhìn thấy dữ liệu đã chốt trước khi câu lệnh
bắt đầu. Do chưa có bên nào ghi khi hai câu `SELECT` chạy, cả A và B đều được
phép đọc số lượng `5`.

Đây không phải đọc dữ liệu chưa chốt. Hai bên đọc cùng một phiên bản đã chốt rồi
sau đó cùng đưa ra quyết định. Vì vậy nâng mức cô lập chỉ để “ngăn đọc bẩn” không
đúng với nguyên nhân của case này.

## 8. Khóa dòng trong cách làm lỗi

Khi A chạy `UPDATE`, PostgreSQL lấy khóa dòng của sản phẩm 77 và giữ khóa đến khi
A chốt hoặc hoàn tác. B chạy `UPDATE` sau đó phải chờ.

Sau khi A chốt, câu ghi của B tiếp tục. Vì câu `WHERE` chỉ có
`product_id = 77`, dòng vẫn thỏa điều kiện. B ghi lại `available_quantity = 1`
và PostgreSQL báo một dòng bị ảnh hưởng.

Khóa dòng không thể sửa một câu lệnh thiếu điều kiện bảo vệ.

## 9. Dòng thời gian khi dùng cập nhật có điều kiện

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE product_id = :productId
  AND available_quantity >= :quantity
RETURNING available_quantity, reserved_quantity;
```

| Bước | Giao dịch A | Giao dịch B | PostgreSQL |
| --- | --- | --- | --- |
| 1 | Chạy `UPDATE` với số lượng `4` | | Điều kiện `5 >= 4` đúng; A lấy khóa dòng |
| 2 | Giữ khóa, chưa chốt | Chạy cùng câu `UPDATE` | B chờ |
| 3 | Chốt | Đang chờ | Giải phóng khóa của A |
| 4 | | Tiếp tục | Kiểm tra lại `WHERE` trên giá trị `1` |
| 5 | | Nhận kết quả rỗng | Điều kiện `1 >= 4` sai |
| 6 | | Ghi kết quả từ chối và chốt | Không thay đổi tồn kho lần hai |

Phép trừ dùng giá trị hiện tại trong PostgreSQL, không dùng con số đã tính từ
một lần đọc trước. Điều kiện và thay đổi nằm trong cùng một câu lệnh nên không
có khoảng hở cho quyết định lỗi thời.

## 10. Nếu giao dịch giữ khóa hoàn tác

Giả sử A đã chạy câu cập nhật nhưng gặp lỗi trước khi chốt:

1. B vẫn chờ vì A còn giữ khóa.
2. A hoàn tác; thay đổi `5 → 1` biến mất.
3. B được đánh thức và xử lý trên giá trị ban đầu `5`.
4. Điều kiện `5 >= 4` đúng nên B giữ hàng thành công.

Kết quả cuối vẫn có đúng một lần giữ `4` sản phẩm. Việc A từng chạy câu SQL
không tạo ảnh hưởng bền vững vì giao dịch chưa chốt.

## 11. Lỗi ở bước ghi bản giữ hàng

Câu cập nhật tồn kho và câu chèn `inventory_reservation` phải dùng cùng một giao
dịch. Nếu câu chèn thất bại do ràng buộc hoặc lỗi kết nối:

```text
UPDATE tồn kho thành công
→ `INSERT` bản ghi giữ hàng thất bại
→ giao dịch hoàn tác
→ tồn kho trở về giá trị trước UPDATE
```

Nếu hai thao tác nằm ở hai giao dịch khác nhau, hệ thống có thể trừ hàng nhưng
không có bản ghi giải thích số lượng đã được dành cho đơn nào.

## 12. Phân biệt từ chối nghiệp vụ và lỗi kỹ thuật

Không có dòng nào được cập nhật là một kết quả bình thường khi không đủ hàng.
Nó khác với những trường hợp sau:

| Tình huống | Cách xử lý |
| --- | --- |
| Không có dòng thỏa `WHERE` | Trả kết quả không đủ hàng hoặc không khả dụng |
| SQLSTATE `55P03` | Hết thời gian chờ khóa; hoàn tác và báo hệ thống bận |
| SQLSTATE `40P01` | Bế tắc; giao dịch bị chọn làm bên phải hoàn tác |
| SQLSTATE `40001` | Lỗi tuần tự hóa; có thể thử lại toàn bộ giao dịch |
| Mất kết nối | Kết quả chốt có thể chưa rõ; không được tự suy diễn là thất bại |

Thử lại lỗi kỹ thuật phải có số lần tối đa, khoảng lùi và một giao dịch mới.
Không thử lại kết quả hết hàng vì dữ liệu không tự trở nên đủ chỉ nhờ lặp câu
lệnh.

## 13. Trường hợp phản hồi bị mất sau khi đã chốt

Nếu PostgreSQL đã chốt nhưng phản hồi HTTP bị mất, phía gọi có thể gửi lại yêu
cầu và trừ hàng lần nữa. Cập nhật có điều kiện chỉ bảo vệ số lượng không âm; nó
không biết hai yêu cầu có cùng ý định đặt hàng hay không.

Đó là bài toán chống xử lý trùng, được tách sang `ECOM-003`. Trong hệ thống thật,
cơ chế bảo vệ tồn kho của case này phải được kết hợp với mã yêu cầu bền vững.

## 14. Hibernate và cập nhật SQL trực tiếp

Câu SQL trực tiếp không đồng bộ thực thể đã được quản lý trong ngữ cảnh lưu trữ
(`persistence context`). Nếu một `InventoryItem` đã được tải trước đó, đối tượng
Java vẫn giữ số lượng cũ sau khi JDBC cập nhật cơ sở dữ liệu.

Thiết kế an toàn nhất là không tải thực thể tồn kho trong cùng giao dịch sử dụng
câu cập nhật trực tiếp. Nếu bắt buộc phải dùng, cần `flush()` thay đổi đang chờ
trước câu SQL và `clear()` hoặc `refresh()` sau đó theo một quy ước rõ ràng.

## 15. Chạy trên nhiều máy chủ

`synchronized`, `ReentrantLock` hoặc một bản đồ khóa theo `productId` chỉ bảo vệ
các luồng trong JVM sở hữu khóa. Chúng không tạo quan hệ loại trừ giữa máy chủ A
và máy chủ B sau bộ cân bằng tải.

Câu cập nhật có điều kiện hoạt động trên cả hai máy chủ vì mọi yêu cầu cuối cùng
đều tranh chấp tại cùng dòng PostgreSQL. Cơ sở dữ liệu là ranh giới có thẩm quyền
cho bất biến tồn kho.

## 16. Giới hạn của giải pháp một dòng

Cập nhật có điều kiện phù hợp khi số lượng cần bảo vệ nằm trên một dòng sản phẩm.
Nó chưa đủ cho các quy tắc như:

- phải giữ hàng đồng thời ở nhiều kho và chỉ thành công nếu tất cả cùng đủ;
- tổng số lượng của nhiều biến thể không vượt một giới hạn chung;
- chọn kho tối ưu dựa trên nhiều dòng rồi mới trừ;
- giữ một giỏ hàng gồm nhiều sản phẩm theo nguyên tắc tất cả hoặc không có gì.

Các trường hợp đó cần thứ tự khóa ổn định, một dòng tổng hợp có thẩm quyền hoặc
mức cô lập mạnh hơn, tùy bất biến cụ thể.

## 17. Tác động của sản phẩm nóng

Câu lệnh có điều kiện bảo đảm tính đúng đắn nhưng không làm biến mất tranh chấp.
Mọi lần mua cùng một sản phẩm vẫn xếp hàng tại một dòng. Khi lượng truy cập tăng:

- thời gian chờ khóa tăng;
- kết nối bị giữ lâu hơn;
- tỷ lệ hết thời gian chờ có thể tăng;
- thử lại thiếu giới hạn có thể khuếch đại tải.

Điều tiết đầu vào và kiến trúc cho đợt bán cao điểm thuộc phạm vi `ECOM-002`.

## 18. Dữ liệu cần theo dõi

- Số lần cập nhật thành công và số lần không cập nhật dòng nào theo sản phẩm.
- Thời gian chờ khóa và thời lượng giao dịch.
- SQLSTATE `55P03`, `40P01`, `40001`.
- Tổng `quantity` của các bản ghi giữ hàng so với `reserved_quantity`.
- Số giao dịch bị hoàn tác sau khi đã cập nhật tồn kho.
- Các sản phẩm tạo hàng đợi khóa dài hoặc làm tăng số kết nối chờ.

Kiểm thử chứng minh các dòng thời gian trên được mô tả trong
[experiments.md](experiments.md).
