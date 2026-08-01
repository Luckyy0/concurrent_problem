# ECOM-001 — Bán vượt số lượng tồn kho

## 1. Bài toán

Kho còn `5` sản phẩm. Hai khách hàng cùng mua `4` sản phẩm và hai yêu cầu được
xử lý trên hai máy chủ ứng dụng khác nhau.

Nếu mỗi yêu cầu đều đọc số lượng tồn kho, kiểm tra trong Java rồi ghi lại kết
quả, cả hai có thể cùng nhìn thấy số lượng `5` và cùng được chấp nhận. Khi đó hệ
thống đã bán `8` sản phẩm dù kho chỉ có `5`.

Đây là lỗi **bán vượt tồn kho** (`overselling`). Nguyên nhân không nằm ở phép
trừ, mà ở chuỗi thao tác không nguyên tử:

```text
đọc số lượng → quyết định còn hàng → tính số lượng mới → ghi đè
```

> **Nói ngắn gọn:** Quyết định “còn đủ hàng” phải được bảo vệ tại nơi lưu số
> lượng tồn kho có thẩm quyền, không được dựa vào một giá trị cũ trong Java.

## 2. Tác nhân và dữ liệu dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Khách hàng A | Mua `4` sản phẩm |
| Khách hàng B | Mua `4` sản phẩm |
| `inventory_item` | Giữ số lượng có sẵn và số lượng đã dành cho đơn hàng |
| Yêu cầu A, B | Hai yêu cầu độc lập, có giao dịch cơ sở dữ liệu riêng |
| PostgreSQL | Nguồn dữ liệu có thẩm quyền đối với tồn kho |

Điểm tranh chấp là dòng `inventory_item` của cùng một `product_id`. Khóa trong
JVM không đủ bảo vệ dòng này vì A và B có thể chạy trên hai máy chủ khác nhau.

## 3. Quy tắc bất biến

Trong phạm vi không có nhập thêm hàng hoặc hủy giữ hàng, hệ thống phải giữ đúng:

```text
available_quantity >= 0

available_quantity + reserved_quantity = on_hand_quantity

reserved_quantity = tổng `quantity` của các bản ghi giữ hàng đã chốt

Với tồn kho 5 và hai yêu cầu mua 4:
chỉ đúng một yêu cầu được chấp nhận.
```

Ràng buộc `CHECK` có thể bảo vệ hai công thức đầu trên một dòng. Nó không tự đối
chiếu được bộ đếm trong `inventory_item` với các bản ghi ở bảng
`inventory_reservation`.

## 4. Ranh giới giao dịch

Một lần giữ hàng thành công gồm hai thao tác ghi:

1. Trừ `available_quantity` và cộng `reserved_quantity`.
2. Tạo bản ghi `inventory_reservation` với trạng thái `RESERVED`.

Hai thao tác phải nằm trong cùng một giao dịch. Nếu việc tạo bản ghi giữ hàng
thất bại, thay đổi trên `inventory_item` cũng phải được hoàn tác.

Không gọi dịch vụ thanh toán, gửi thông điệp hoặc thực hiện tác vụ mạng trong
giao dịch đang giữ khóa dòng tồn kho. Những công việc đó làm tăng thời gian giữ
khóa và cần một thiết kế giao tiếp riêng.

## 5. Vì sao `@Transactional` chưa đủ

`@Transactional` bảo đảm các thao tác trong một giao dịch cùng chốt hoặc cùng
hoàn tác. Nó không ngăn giao dịch khác đọc cùng giá trị trước khi Hibernate ghi
dữ liệu xuống PostgreSQL.

Ở mức cô lập `READ COMMITTED`, hai câu `SELECT` thông thường có thể cùng đọc
`available_quantity = 5`. Nếu thực thể không có `@Version`, hai câu `UPDATE`
sau đó đều có thể ghi giá trị tuyệt đối `1`. Dòng tồn kho trông vẫn hợp lệ, nhưng
hai bản ghi giữ hàng đã ghi nhận tổng cộng `8` sản phẩm.

## 6. Cách xử lý được khuyến nghị

Khi điều kiện chỉ phụ thuộc vào dòng tồn kho, hãy gộp việc kiểm tra và trừ hàng
vào một câu lệnh SQL:

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE product_id = :productId
  AND available_quantity >= :quantity
RETURNING available_quantity, reserved_quantity;
```

Kết quả được hiểu như sau:

| Kết quả | Ý nghĩa nghiệp vụ |
| --- | --- |
| Trả về một dòng | Giữ hàng thành công |
| Không trả về dòng nào | Dòng không tồn tại hoặc không còn đủ hàng |
| Lỗi chờ khóa, bế tắc hoặc tuần tự hóa | Lỗi kỹ thuật; giao dịch phải hoàn tác |

Khi hai yêu cầu cập nhật cùng một sản phẩm, PostgreSQL khóa dòng cho yêu cầu đến
trước. Yêu cầu còn lại chờ, sau đó kiểm tra lại mệnh đề `WHERE` trên giá trị mới
nhất. Vì chỉ còn `1`, điều kiện `1 >= 4` sai và yêu cầu đó không trừ hàng.

## 7. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh hoặc API | Ý nghĩa |
| --- | --- | --- |
| bán vượt tồn kho | overselling | Hệ thống chấp nhận lượng bán lớn hơn lượng có thể cung cấp |
| mất cập nhật | lost update | Một lần ghi đè làm mất tác động nghiệp vụ của lần ghi khác |
| cập nhật có điều kiện | conditional `UPDATE` | Chỉ thay đổi dòng khi điều kiện trong `WHERE` còn đúng |
| số dòng bị ảnh hưởng | affected-row count | Số dòng thật sự được câu lệnh cập nhật |
| kiểm tra lại điều kiện | predicate recheck | PostgreSQL đánh giá lại `WHERE` sau khi chờ giao dịch khác |
| khóa lạc quan | optimistic locking | Phát hiện xung đột bằng cột phiên bản và `@Version` |
| khóa bi quan | pessimistic locking | Khóa dòng trước khi đọc và ra quyết định bằng `FOR UPDATE` |
| điểm tranh chấp nóng | hot row | Một dòng bị nhiều yêu cầu cập nhật đồng thời |

## 8. Hợp đồng kết quả

Ứng dụng không được coi việc câu lệnh SQL chạy xong là đồng nghĩa với giữ hàng
thành công. Mã nguồn phải xử lý rõ ba nhóm kết quả:

- thành công: một dòng được cập nhật và toàn bộ giao dịch đã chốt;
- từ chối nghiệp vụ: không có dòng nào thỏa điều kiện;
- lỗi kỹ thuật: chờ khóa quá lâu, bế tắc, mất kết nối hoặc giao dịch bị hoàn tác.

Nếu sản phẩm có thể bị xóa hoặc vô hiệu hóa trong lúc đặt hàng, kết quả không có
dòng không nên tự động được dịch thành “hết hàng”. Case này giả định dòng tồn kho
đã được tạo trước và không bị xóa cứng trong quá trình bán.

## 9. Hậu quả khi triển khai sai

- Nhận nhiều đơn hàng hơn lượng có thể giao.
- Bộ đếm tồn kho không khớp với tổng các bản ghi giữ hàng.
- Phải hủy đơn, hoàn tiền hoặc bù hàng thủ công.
- Dữ liệu vẫn vượt qua `CHECK`, khiến lỗi chỉ lộ ra khi đối soát.
- Lỗi chỉ xuất hiện dưới tải đồng thời hoặc khi chạy nhiều máy chủ.
- Việc thử lại không giới hạn có thể biến một sản phẩm nóng thành hàng đợi khóa
  và làm cạn vùng kết nối.

## 10. Điều hướng tài liệu

- [Mã nguồn gây lỗi](broken-code.md)
- [Phân tích dòng thời gian tranh chấp](analysis.md)
- [Các phương án xử lý](solutions.md)
- [Thực nghiệm với PostgreSQL](experiments.md)
- [Cập nhật dữ liệu an toàn bằng SQL](../../concepts/atomic-database-operations.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)
- [DB-001 — Mất cập nhật dưới MVCC](../../postgresql/lost-update-mvcc/README.md)
- [LOCK-001 — Xung đột phiên bản](../../locking/optimistic-version-conflict/README.md)
- [LOCK-003 — Khóa bi quan](../../locking/pessimistic-write-for-update/README.md)
- [LOCK-004 — Cập nhật có điều kiện](../../locking/conditional-atomic-update/README.md)

## 11. Khi nên dùng cách này

Cập nhật có điều kiện là lựa chọn mặc định tốt khi một dòng tồn kho là nguồn dữ
liệu có thẩm quyền và điều kiện có thể diễn đạt trong `WHERE`.

Dùng `@Version` khi nghiệp vụ cần sửa một thực thể phức tạp và xung đột không
thường xuyên. Dùng `FOR UPDATE` khi Java phải đọc thêm dữ liệu rồi mới quyết định.
Với sản phẩm có lượng truy cập rất cao, tính đúng đắn của câu lệnh trên vẫn giữ
nguyên nhưng thông lượng có thể giảm; vấn đề đó thuộc `ECOM-002`.

## 12. Phạm vi

Case này chỉ bảo vệ việc trừ tồn kho có thẩm quyền. Nó không giải quyết yêu cầu
đặt hàng bị gửi trùng, thanh toán lặp lại, hủy giữ hàng, chia kho theo nhiều địa
điểm hoặc điều phối tồn kho giữa nhiều dịch vụ. Yêu cầu tạo đơn bị gửi trùng được
phân tích riêng trong `ECOM-003`.
