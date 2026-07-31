# Phân tích chuyên sâu — Tại sao Kiểm tra (Predicate) và Ghi dữ liệu (Mutation) phải gộp làm một?

## Trạng thái ban đầu

Giả sử trong Database ta có:
```text
Sản phẩm số 77:
  Tổng trong kho (on_hand_quantity)   = 5
  Đang có sẵn (available_quantity) = 5
  Đã giữ chỗ (reserved_quantity)  = 0
  Phiên bản (revision)           = 10

Danh sách đơn đã giữ chỗ (inventory_reservation) = Rỗng
```

Lệnh A và Lệnh B là hai đơn hàng hoàn toàn hợp lệ, mỗi đơn đều muốn mua `4` chiếc. Chúng chạy trên hai máy chủ (application instances) khác nhau, với kết nối và giao dịch hoàn toàn độc lập.

## Kịch bản thảm họa: Đọc-Kiểm tra-Ghi (read–check–write)

| Bước | Giao dịch A (Máy chủ 1) | Giao dịch B (Máy chủ 2) | Dưới Database PostgreSQL |
| --- | --- | --- | --- |
| 1 | `SELECT` bình thường → Thấy `5` sẵn / `0` giữ | | Database lấy Ảnh chụp A |
| 2 | | `SELECT` bình thường → Thấy `5` sẵn / `0` giữ | Database lấy Ảnh chụp B |
| 3 | Code Java: `5 >= 4` → Cho qua | Code Java: `5 >= 4` → Cho qua | DB không hề biết ứng dụng đang làm gì |
| 4 | Code tự trừ: Còn `1` sẵn / `4` giữ | Code tự trừ: Còn `1` sẵn / `4` giữ | |
| 5 | Insert lịch sử đơn A | Insert lịch sử đơn B | Mã đơn khác nhau nên insert trót lọt |
| 6 | `UPDATE` thành `1` sẵn / `4` giữ | | A đang giữ khóa của dòng này |
| 7 | Chốt sổ (commit) | `UPDATE` đứng chờ A xong rồi ghi đè số `1/4` | Câu lệnh của B không có điều kiện `WHERE` gắt gao |
| 8 | | Chốt sổ (commit) | Báo cả 2 luồng đều sửa thành công (affected rows = 1) |

Đến cuối cùng, dữ liệu tồn kho nhìn thì vẫn hợp lệ (không bị âm), nhưng thực tế chúng ta đã nhận 2 đơn với tổng cộng `8` chiếc. Đây chính là thảm họa "ghi đè mất dữ liệu" (lost update) trên con số đếm, dẫn đến quyết định bán hàng sai lầm bét nhè.

## Bảng đối chiếu: Mong đợi vs Thực tế

| | Chuyện đáng lẽ phải xảy ra (Expected) | Chuyện tồi tệ đã xảy ra (Actual) |
| --- | --- | --- |
| Số đơn được duyệt | 1 đơn | 2 đơn |
| Tổng số lượng ghi trong lịch sử | 4 chiếc | 8 chiếc (Bán âm kho!) |
| Số lượng `reserved_quantity` | 4 chiếc | 4 chiếc |
| Số lượng `available_quantity` | 1 chiếc | 1 chiếc |
| Khớp sổ (Reconciliation) | Khớp hoàn toàn | Lệch mất `4` chiếc |
| Số phận của người đến sau | Báo Hết hàng (`OUT_OF_STOCK`) | Vẫn báo Thành công (`RESERVED`) |

## Cứu cánh: Câu lệnh Nguyên tử (Atomic statement)

Hãy dồn hết logic vào 1 câu SQL duy nhất:

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity,
    revision = revision + 1
WHERE product_id = :productId
  AND :quantity > 0
  AND available_quantity >= :quantity
RETURNING available_quantity, reserved_quantity, revision;
```

Phần `WHERE` chính là "bảo vệ nghiệp vụ"; phần `SET` là "ý định sửa đổi". Cả hai sẽ được PostgreSQL gói gọn thành một khối duy nhất (atomic) và chạy trên dữ liệu TƯƠI MỚI NHẤT.

## Kịch bản thành công: Cập nhật có điều kiện (conditional UPDATE)

| Bước | Giao dịch A | Giao dịch B | Dưới Database PostgreSQL |
| --- | --- | --- | --- |
| 1 | `UPDATE` mua `4` chiếc | | Kiểm tra `5 >= 4` → Đúng, A đổi thành `1` sẵn / `4` giữ |
| 2 | A đang chạy, chưa chốt | `UPDATE` mua `4` chiếc | B bị chặn lại, đứng chờ |
| 3 | Chốt sổ (commit) | Vẫn đang chờ | A nhả khóa ra |
| 4 | | B tự động đánh giá lại (recheck) | Dữ liệu lúc này chỉ còn `1` chiếc sẵn |
| 5 | | Trả về 0 dòng được sửa | Kiểm tra `1 >= 4` → Sai |
| 6 | | Lưu lịch sử `OUT_OF_STOCK`, chốt sổ | Không có dòng nào bị luồng B ghi đè |

> **Nói ngắn gọn:** Luồng B thất bại không phải do code Java rảnh rỗi "đọc lại", mà vì chính câu lệnh SQL đã khôn ngoan kiểm tra lại điều kiện trên chính kết quả mà luồng A vừa chốt xong.

## Mức cách ly `READ COMMITTED` và Đánh giá lại điều kiện (predicate recheck)

Khi dùng mức cách ly mặc định `READ COMMITTED`, PostgreSQL tìm dòng cần sửa bằng Ảnh chụp ban đầu. Nhưng nếu dòng đó đang bị luồng khác khóa, nó sẽ ngoan ngoãn đứng chờ cho đến khi luồng kia chạy xong (commit hoặc rollback).

### Nếu luồng trước chốt sổ (Holder commit)
PostgreSQL sẽ lấy dữ liệu mới cứng vừa được sửa, và ĐÁNH GIÁ LẠI cụm `WHERE`.
Nếu điều kiện vẫn đúng, luồng B được đi tiếp. Nếu điều kiện sai (như trong ví dụ trên), luồng B sẽ âm thầm bỏ qua và trả về số dòng bị ảnh hưởng là `0`.

### Nếu luồng trước bị hủy (Holder rollback)
Thì coi như luồng A chưa từng tồn tại. Luồng B sẽ lấy dữ liệu gốc (5 chiếc) để tính toán, điều kiện đúng, và B sẽ là người chiến thắng (sửa 1 dòng).

Tuy nhiên, thủ thuật đánh giá lại này chỉ chạy đúng với những điều kiện đơn giản trên một dòng cụ thể. Đừng dại dột dùng cho các cụm `WHERE` chứa JOIN hay truy vấn chéo nhiều bảng!

## Khóa dòng (Row lock) vẫn luôn rình rập

Code ứng dụng không gọi hàm "lock" không có nghĩa là Database không có khóa. Bản chất câu lệnh `UPDATE` sẽ giật lấy khóa dòng ngay khi nó tìm thấy dòng tương ứng. Kẻ đến sau có thể bị:
- Phải chờ rồi tự đánh giá lại.
- Chờ quá lâu vượt mức `lock_timeout` và văng lỗi `55P03`.
- Bị dính vào vòng luẩn quẩn (deadlock) và văng lỗi `40P01`.

Dù không khóa dòng thủ công bằng `SELECT ... FOR UPDATE`, thì lệnh `UPDATE` này vẫn giữ khóa cho tới tận khi giao dịch kết thúc (commit/rollback). Nhớ kỹ: Đừng chèn các hàm gọi API ra mạng (remote I/O) vào sau câu `UPDATE` nếu bạn không muốn cả hệ thống phải xếp hàng dài.

## Số dòng bị ảnh hưởng chính là một Giao thức (Protocol)

Con số `n` trả về từ SQL (hoặc từ Spring Data) không chỉ để in log cho vui, nó là kết quả của trận chiến đa luồng:

```text
n = 1 → Tuyệt vời! Đúng 1 dòng lọt qua khe cửa hẹp và được sửa thành công.
n = 0 → Đơn hàng rớt đài. Không có dòng nào được sửa (Hãy báo Hết hàng).
n > 1 → Chúc mừng, code của bạn có bug (sửa 1 sản phẩm mà ảnh hưởng nhiều dòng)!
```

`0` không phải là Exception văng ra màn hình. Lập trình viên phải tự viết lệnh `if (n == 0)` để xử lý luồng đi rẽ nhánh cho đúng trước khi tạo đơn.

## `RETURNING` giúp tránh việc Đọc-sau-khi-Ghi ngớ ngẩn

Câu lệnh `UPDATE ... RETURNING` vừa sửa vừa trả về ngay kết quả mới nhất:
```text
Trả về 1 dòng → Giữ chỗ thành công (kèm số lượng tồn kho còn lại).
Trả về 0 dòng → Chẳng có gì thay đổi.
```
Nếu bạn tách ra làm 2 câu lệnh: `UPDATE` xong rồi `SELECT` lên lại, bạn có thể vô tình đọc nhầm kết quả của một luồng khác vừa nhảy vào sửa. Hãy dùng `RETURNING` để lấy chính xác thành quả của người chiến thắng.

## Số 0 dòng không phải lúc nào cũng là `OUT_OF_STOCK` (Hết hàng)

Cùng là `0` nhưng có thể do:
- Sản phẩm bị xóa mất tiêu.
- Truyền số lượng mua bị âm.
- Đang bị sai thông tin khách hàng (tenant mismatch).
- Hoặc đơn giản là không đủ số lượng (Hết hàng).

Trong bài toán cụ thể này, vì chúng ta đã kiểm tra dữ liệu đầu vào kỹ, nên `0` chắc chắn là Hết hàng. Nhưng nếu hệ thống phức tạp, bạn phải thiết kế sao cho biết chính xác nguyên nhân (dùng Stored Procedure, hoặc chấp nhận gom chung thành lỗi `NOT_AVAILABLE`).

## Cẩn thận với Bộ đệm (Persistence context) của Hibernate

Nếu bạn gọi SQL thẳng xuống DB (Bulk DML) thì sẽ lách qua mặt Hibernate. Nếu bộ đệm JPA đang nhớ sản phẩm `77` có `5` chiếc:
- Đối tượng (object) trên RAM vẫn chứa con số cũ kỹ.
- Trả object này về cho API sẽ bị sai (stale).
- Đáng sợ nhất: Nếu sau đó bộ đệm lại lưu (flush) một thứ khác, nó có thể vô tình ghi đè lại con số cũ kỹ lên DB, phá nát kết quả của câu lệnh UPDATE nguyên tử vừa rồi.

**3 cách phòng thủ:**
1. Code sửa dữ liệu đừng bắt JPA load thực thể (entity) đó lên RAM.
2. Ép (flush) hết dữ liệu cũ xuống trước, gọi SQL Bulk DML, rồi dọn dẹp sạch sẽ bộ đệm (clear).
3. Nếu vẫn cần dùng tiếp, hãy ép JPA tải lại từ DB (refresh).
Chú ý: Đừng lạm dụng cờ `clearAutomatically=true` một cách máy móc, vì nó sẽ hất đổ mọi thay đổi chưa kịp lưu khác của bạn.

## Cấu trúc chuẩn của một Giao dịch (Transaction composition)

Đúng đắn không chỉ nằm ở câu SQL đếm số:
```text
BẮT ĐẦU GIAO DỊCH
  Đăng ký mã chống trùng (command ID)
  Sửa tồn kho có điều kiện (conditional UPDATE)
  Lưu lịch sử: THÀNH CÔNG hoặc HẾT HÀNG
  Lưu hộp thư đi (outbox) NẾU thành công
CHỐT SỔ GIAO DỊCH
```
Nếu bước Lưu hộp thư (outbox) bị lỗi, giao dịch phải HỦY (rollback) sạch sẽ để trả lại số tồn kho. Đừng bao giờ dại dột bọc `try-catch` lờ đi lỗi database để rồi giao dịch chạy lửng lơ. Luôn lưu Hộp thư (outbox) chung một giao dịch với lệnh sửa kho để đảm bảo "dữ liệu đi đôi với sự kiện".

## Trùng mã lệnh (Duplicate command)

Cập nhật nguyên tử chỉ chống bán lố hàng, chứ không ngăn được chuyện khách hàng bấm nút 2 lần và bị trừ hàng 2 lần (nếu kho vẫn đủ)!
Ta phải dùng khóa `command_id` để chống trùng lặp (idempotency):

```sql
INSERT INTO inventory_reservation (...)
VALUES (...)
ON CONFLICT (command_id) DO NOTHING
RETURNING command_id;
```

Gửi mã trùng thì Database sẽ chặn lại. Lệnh chống trùng lặp và lệnh sửa tồn kho phải đi đôi với nhau trong cùng một Giao dịch.

## Chốt sổ, Hoàn tác và Mớ bòng bong mất phản hồi

| Thời điểm đứt cáp / lỗi | Kết quả dưới Database | Cách ứng dụng dọn dẹp (Recovery) |
| --- | --- | --- |
| Trước khi UPDATE | Kho chưa suy suyển | Thử chạy lại (Fresh attempt) |
| Sau UPDATE, trước khi Chốt | Tự rollback mọi thứ | Chạy lại lệnh cũ (vì mã lệnh ổn định) |
| Chốt THÀNH CÔNG, nhưng rớt mạng chưa kịp báo về Client | Đã giữ chỗ chắc nịch | Gửi lại mã lệnh, Server trả về trạng thái cũ đã lưu |
| Trả về `0` dòng (Affected rows = 0) | Có thể vẫn ghi lịch sử bị Từ chối | Báo lại trạng thái `OUT_OF_STOCK` |
| Hết giờ chờ khóa / Kẹt xe (Deadlock) | Giao dịch tự hủy (Rollback) | Tự động thử lại lệnh mới nếu luật cho phép |

Không bao giờ được tự suy diễn: "Không nhận được phản hồi nghĩa là thất bại". Chỉ có Mã lệnh (Stable Command ID) mới là chân lý giúp phục hồi dữ liệu khi mạng chập chờn.

## Các Mức cách ly (Isolation) khác

Bài này viết dựa trên mức mặc định `READ COMMITTED`. 
Nếu nâng lên `REPEATABLE READ`, những kẻ đụng độ nhau thay vì trả về `0` dòng thì sẽ văng thẳng mặt lỗi sập tuần tự hóa `40001`. Khi đó, code xử lý lỗi và test của bạn cũng phải sửa lại cho phù hợp.
Tuyệt đối không tăng mức cách ly (Isolation) một cách mù quáng chỉ để thay cho một cái mệnh đề `WHERE` có thể viết rõ ràng trong câu SQL!

## Rào chắn (Constraint) chỉ là phòng thủ vòng ngoài

```sql
CHECK (available_quantity >= 0)
CHECK (reserved_quantity >= 0)
CHECK (available_quantity + reserved_quantity = on_hand_quantity)
```
Các rào chắn (Constraints) này giúp Database không nhận những con số âm lố bịch từ bất kỳ đâu gởi tới. Nhưng nó KHÔNG hề biết đối chiếu (cross-table) tổng số tồn kho với các dòng lịch sử bên bảng khác.
Do đó, bạn vẫn phải có các công việc Đối soát (Reconciliation job) định kỳ để chạy lệnh SUM() so sánh xem tổng các đơn trong kho có bị vênh với con số đếm hay không.

## Môi trường nhiều Máy chủ (Multi-instance)

Cho dù có 100 máy chủ (Scale-out) bắn SQL vào chung 1 Database, thì Database vẫn là "cảnh sát" kiểm duyệt mọi thứ bằng Khóa dòng và Kiểm tra điều kiện. Dữ liệu KHÔNG BAO GIỜ bị sai!
Tuy nhiên, nếu quá đông máy chủ cùng húc vào 1 sản phẩm hot, hệ thống sẽ bị xếp hàng dài cổ (waiters tăng), gây nghẽn Connection pool và hết hạn chờ (lock wait).

## Bắt bệnh theo từng Tầng (Root cause theo layer)

### Tầng Ứng dụng (Application)
Tách rời việc "Đọc lên kiểm tra" và "Ghi xuống" thành 2 bước rời rạc. Dùng con số "thiu" để tính toán logic.

### Tầng Spring
Lầm tưởng chữ `@Transactional` có phép thuật tự động gộp 2 câu lệnh SQL lại thành 1 cục (atomic). Quên kiểm tra kết quả trả về từ SQL.

### Tầng Hibernate/JPA
Bộ đệm (Cache) tự động sinh SQL ghi đè làm hỏng kết quả. Trộn lẫn code Bulk DML và code gọi thực thể (entity) mà không lo dọn dẹp.

### Tầng PostgreSQL
Hiểu nhầm mức `READ COMMITTED` tự động bảo vệ dữ liệu, mà không biết rằng chỉ có lệnh `UPDATE` kèm ĐIỀU KIỆN (Guarded UPDATE) mới kích hoạt cơ chế "đánh giá lại" (recheck) xịn sò của nó.

## Cần theo dõi gì trên hệ thống (Observability)?

Giám sát các chỉ số sinh tồn:
- Số lần bắn UPDATE thành công (1 dòng) và thất bại (0 dòng).
- Số lượng hàng tồn kho còn lại.
- Tốc độ tắc đường: thời gian chờ khóa (lock wait duration), các mã lỗi báo quá tải `55P03`, `40P01`, `40001`.
- Thời gian từ lúc UPDATE xong đến lúc Chốt sổ (nếu dài là có vấn đề).
- Tình trạng trùng mã lệnh.
- Cảnh báo lệch sổ đối soát.

## Khi nào cách này "chào thua"? (Scope boundary)

Kỹ thuật cập nhật nguyên tử có điều kiện này chỉ bá đạo khi dữ liệu kho và luật lệ nằm GỌN trong đúng một dòng dữ liệu (single row).
Nếu bạn bán các "combo" liên quan nhiều dòng sản phẩm, hoặc quy tắc dựa vào những dòng chưa từng tồn tại (new rows), 1 câu lệnh UPDATE này vô dụng. Khi đó bạn phải chuyển sang bài Khóa bi quan, Mức cách ly đặc biệt hoặc Thiết kế rào chắn toàn cục.
