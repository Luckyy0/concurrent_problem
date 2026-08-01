# ECOM-006 — Mất cập nhật giỏ hàng khi thao tác đồng thời

## 1. Bài toán

Khách hàng mở cùng một giỏ hàng trên hai tab hoặc hai thiết bị. Cả hai nơi đều
đọc phiên bản `7`, khi giỏ đang có một sản phẩm `SKU-A`.

- Tab A thêm `SKU-B` và gửi toàn bộ giỏ `[SKU-A, SKU-B]`.
- Tab B xóa `SKU-A` và gửi toàn bộ giỏ `[]`.

Nếu máy chủ chỉ thay thế danh sách hiện tại bằng nội dung yêu cầu, hai giao dịch
đều có thể thành công. Lần ghi sau cùng xóa mất thay đổi của lần ghi trước. Đổi
thứ tự hoàn tất còn có thể làm `SKU-A` xuất hiện trở lại dù người dùng vừa xóa.

```text
giỏ ban đầu, phiên bản 7      [SKU-A]
ý định của tab A              thêm SKU-B
ý định của tab B              xóa SKU-A
kết quả tuần tự hợp lệ         [SKU-B]
kết quả do ghi đè              [] hoặc [SKU-A, SKU-B]
```

> **Nói ngắn gọn:** Máy chủ phải biết yêu cầu được tạo từ phiên bản nào của giỏ
> và không được âm thầm áp dụng một ảnh chụp đã cũ lên trạng thái mới hơn.

## 2. Tác nhân và điểm tranh chấp

| Thành phần | Vai trò |
| --- | --- |
| Tab A | Gửi thao tác thêm sản phẩm dựa trên phiên bản `7` |
| Tab B | Gửi thao tác xóa sản phẩm dựa trên phiên bản `7` |
| `shopping_cart` | Giữ trạng thái và phiên bản của toàn bộ giỏ |
| `cart_item` | Giữ số lượng theo từng sản phẩm |
| Máy chủ A, B | Có thể xử lý hai yêu cầu trên hai JVM khác nhau |
| PostgreSQL | Quyết định yêu cầu nào còn đúng phiên bản |

Điểm tranh chấp không chỉ là một dòng `cart_item`. Hai tab có thể sửa hai sản
phẩm khác nhau nhưng vẫn dựa trên cùng một ảnh chụp toàn giỏ. Vì vậy, phiên bản
phải thuộc về `shopping_cart`, là gốc của tập hợp dữ liệu này.

## 3. Quy tắc bất biến

```text
Mỗi thay đổi được chấp nhận phải xuất hiện trong trạng thái đã chốt.
Không yêu cầu nào được báo thành công nếu nó đã ghi đè một phiên bản mới hơn.
Mỗi sản phẩm xuất hiện tối đa một lần trong một giỏ.
quantity > 0 và không vượt giới hạn nghiệp vụ.
Phiên bản phản hồi phải nhận diện đúng trạng thái đã chốt.
Chỉ giỏ ACTIVE mới được sửa.
```

Hai thao tác cùng dựa trên phiên bản `7` không nhất thiết đều thất bại. Một bên
được phép tạo phiên bản `8`; bên còn lại phải nhận xung đột, tải lại giỏ và quyết
định theo trạng thái mới.

## 4. Phân biệt ảnh chụp và ý định

Một yêu cầu thay thế toàn bộ giỏ chỉ mô tả kết quả mà phía gọi nhìn thấy, không
nói rõ người dùng muốn thay đổi phần nào. Máy chủ không thể tự suy ra cách gộp
an toàn khi ảnh chụp đã cũ.

| Lệnh | Ý nghĩa | Có nên tự thử lại trên phiên bản mới? |
| --- | --- | --- |
| `AddItem(productId, delta)` | Tăng thêm một lượng cụ thể | Có thể, nếu có mã lệnh và kiểm tra lại giới hạn |
| `SetQuantity(productId, target)` | Đặt số lượng thành giá trị đích | Không nên; giá trị mới có thể ghi đè ý định khác |
| `RemoveItem(productId)` | Xóa sản phẩm đang thấy | Thường cần báo xung đột nếu sản phẩm đã đổi sau đó |
| `ReplaceCart(items)` | Thay toàn bộ ảnh chụp | Không; phải yêu cầu đúng phiên bản |

API nên nhận lệnh nhỏ, thể hiện đúng ý định. Tuy nhiên, lệnh nhỏ không loại bỏ
nhu cầu kiểm tra phiên bản khi người dùng cần biết trạng thái vừa xem còn hiệu
lực hay không.

## 5. Hợp đồng phiên bản

Khi đọc giỏ, máy chủ trả cả dữ liệu và phiên bản:

```http
ETag: "cart-7"

{
  "cartId": "...",
  "version": 7,
  "items": [ ... ]
}
```

Khi sửa, phía gọi gửi lại phiên bản đó bằng `If-Match` hoặc trường
`expectedVersion`. Không dùng cả hai cách với ý nghĩa khác nhau trong cùng API.

```http
PATCH /carts/{cartId}/items/{productId}
If-Match: "cart-7"
```

PostgreSQL chỉ cấp quyền sửa nếu phiên bản và trạng thái vẫn đúng:

```sql
UPDATE shopping_cart
SET version = version + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE cart_id = :cartId
  AND version = :expectedVersion
  AND status = 'ACTIVE'
RETURNING version;
```

- Trả một dòng: giao dịch đã giành quyền tạo phiên bản mới.
- Trả không dòng: giỏ không tồn tại, không còn hoạt động hoặc phiên bản đã cũ.
- Lỗi chờ khóa hay bế tắc: lỗi kỹ thuật, không phải xung đột phiên bản.

## 6. Ranh giới giao dịch

```text
BEGIN
  1. kiểm tra và tăng shopping_cart.version bằng câu UPDATE có điều kiện
  2. áp dụng thao tác lên cart_item
  3. đọc ảnh chụp kết quả và phiên bản mới
COMMIT
```

Ba bước phải nằm trong cùng một giao dịch. Nếu ghi `cart_item` thất bại, lần
tăng phiên bản cũng hoàn tác. Không trả phản hồi thành công trước khi giao dịch
đã chốt.

Giải pháp dùng JPA `@Version` cũng hợp lệ nếu mọi thay đổi của `cart_item` bắt
buộc đi qua gốc `ShoppingCart`, thực thể gốc thực sự bị đánh dấu thay đổi và
`flush()` được gọi trong giao dịch. Cập nhật trực tiếp bảng con sẽ bỏ qua lớp bảo
vệ này.

## 7. Kết quả bên thắng và bên thua

| Tình huống | Phản hồi | Thay đổi bền vững |
| --- | --- | --- |
| Phiên bản đúng | `200 OK` cùng phiên bản mới | Thao tác được áp dụng một lần |
| Phiên bản cũ | `409 Conflict` hoặc `412 Precondition Failed` | Không thay đổi giỏ |
| Giỏ không tồn tại | `404 Not Found` | Không thay đổi |
| Giỏ không còn `ACTIVE` | `409 CART_NOT_MUTABLE` | Không thay đổi |
| Hết thời gian chờ khóa | `503` hoặc lỗi kỹ thuật có thể thử lại | Toàn giao dịch hoàn tác |
| Mất phản hồi sau khi chốt | Kết quả chưa rõ với phía gọi | Tải lại giỏ hoặc tra bằng mã lệnh |

Phản hồi xung đột nên mang phiên bản hiện tại và có thể kèm ảnh chụp mới để giao
diện cho người dùng lựa chọn. Không âm thầm thử lại `ReplaceCart` hay
`SetQuantity`, vì lần thử lại sẽ đổi một xung đột thành ghi đè hợp lệ về mặt kỹ
thuật nhưng sai ý định.

## 8. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh hoặc API | Ý nghĩa |
| --- | --- | --- |
| mất cập nhật | lost update | Một thay đổi đã chốt bị lần ghi từ dữ liệu cũ ghi đè |
| gốc tập hợp | aggregate root | Thực thể đại diện và bảo vệ quy tắc của cả giỏ |
| phiên bản dự kiến | expected version | Phiên bản mà phía gọi đã dùng để tạo yêu cầu |
| kiểm tra rồi đổi | compare-and-set, CAS | Chỉ cập nhật nếu giá trị hiện tại vẫn đúng như dự kiến |
| ngữ nghĩa gộp | merge semantics | Quy tắc quyết định hai ý định có thể kết hợp hay không |
| xung đột lạc quan | optimistic conflict | Phát hiện dữ liệu đã đổi thay vì khóa từ lúc đọc |
| điều kiện HTTP | `If-Match`, `ETag` | Cách truyền và kiểm tra phiên bản qua HTTP |
| điều khiển đồng thời đa phiên bản | multi-version concurrency control, MVCC | Cơ chế cho mỗi câu lệnh đọc một ảnh chụp dữ liệu phù hợp |

## 9. Chạy trên nhiều máy chủ

`synchronized`, khóa theo `cartId` trong bộ nhớ hoặc khóa phiên HTTP chỉ có tác
dụng trong một tiến trình. Yêu cầu từ hai thiết bị có thể đến hai máy chủ khác
nhau, nên quyết định thắng–thua phải được thực hiện bằng phiên bản trong
PostgreSQL.

Phiên bản cũng phải được dùng bởi mọi đường ghi: API người dùng, tác vụ dọn giỏ,
dịch vụ hỗ trợ khách hàng và tiến trình nền. Chỉ cần một đường cập nhật trực tiếp
`cart_item`, hệ thống lại có thể báo phiên bản không đổi dù nội dung đã đổi.

## 10. Hậu quả khi triển khai sai

- Sản phẩm vừa thêm biến mất sau khi tab khác lưu giỏ.
- Sản phẩm đã xóa xuất hiện trở lại.
- Số lượng bị đặt về giá trị cũ mà không có cảnh báo.
- Giao diện nhận `200 OK` nhưng trạng thái sau đó không chứa thay đổi vừa gửi.
- Nhật ký chỉ cho thấy hai yêu cầu thành công, khó xác định ý định nào bị mất.
- Lần thử lại thao tác tăng số lượng có thể tăng hai lần nếu không có mã lệnh.
- Một cập nhật bảng con làm nội dung đổi nhưng `ETag` vẫn giữ nguyên.

## 11. Điều hướng tài liệu

- [Mã nguồn thay toàn bộ giỏ gây lỗi](broken-code.md)
- [Phân tích dòng thời gian và ngữ nghĩa lệnh](analysis.md)
- [Thiết kế Java và SQL an toàn](solutions.md)
- [Thực nghiệm đồng thời với PostgreSQL](experiments.md)
- [DB-001 — Mất cập nhật dưới MVCC](../../postgresql/lost-update-mvcc/README.md)
- [LOCK-001 — Xung đột phiên bản lạc quan](../../locking/optimistic-version-conflict/README.md)
- [Khóa lạc quan](../../concepts/optimistic-locking.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)
- [ECOM-003 — Tạo đơn hàng trùng khi checkout](../duplicate-checkout-order/README.md)

## 12. Khi nên dùng cách này

Phiên bản ở cấp giỏ phù hợp khi giỏ có kích thước giới hạn, xung đột không xảy
ra quá thường xuyên và sản phẩm trong giỏ cùng tạo nên một trạng thái người dùng
cần hiểu nhất quán. Cách này ưu tiên phát hiện xung đột rõ ràng hơn là âm thầm
gộp mọi thay đổi.

Nếu một giỏ nóng nhận rất nhiều thao tác cộng có tính giao hoán, có thể dùng câu
SQL cộng chênh lệch và thử lại có giới hạn. Phải giữ mã lệnh chống gửi trùng,
kiểm tra lại giới hạn số lượng và đo tỷ lệ xung đột. Khóa bi quan phù hợp hơn khi
xung đột dày và phép tính ngắn, nhưng làm tăng thời gian chờ và nguy cơ cạn vùng
kết nối.

## 13. Phạm vi

Case này chỉ xử lý giỏ hàng còn thay đổi được. Nó không giữ tồn kho, không khóa
giá và không bảo đảm sản phẩm vẫn mua được. Bước tạo đơn hàng và chốt checkout
thuộc [ECOM-003](../duplicate-checkout-order/README.md); hết hạn giữ tồn kho và
xác nhận thuộc case tiếp theo.
