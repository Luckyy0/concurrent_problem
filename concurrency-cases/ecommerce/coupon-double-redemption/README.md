# ECOM-004 — Sử dụng coupon/voucher vượt giới hạn

## 1. Bài toán

Coupon `FLASH20` còn đúng một lượt dùng trên toàn hệ thống:

```text
global_limit   = 100
redeemed_count = 99
per_user_limit = 1
```

Khách hàng `C-17` chưa dùng mã này. Hai checkout độc lập của cùng khách hàng
được xử lý đồng thời trên hai máy chủ. Nếu cả hai giao dịch đều đọc bộ đếm, kiểm
tra trong Java rồi mới ghi, chúng có thể cùng thấy:

```text
99 < 100 và số lượt của C-17 = 0
```

Cả hai sau đó chấp nhận giảm giá. Hệ thống đã cho một khách hàng dùng mã hai lần
và tổng số lượt thật đạt `101`, dù bộ đếm cuối cùng có thể vẫn chỉ là `100` do
một lần ghi đè làm mất thay đổi của giao dịch kia.

Chuỗi thao tác gây lỗi là:

```text
đọc bộ đếm → kiểm tra giới hạn → quyết định chấp nhận → ghi bộ đếm và lịch sử
```

> **Nói ngắn gọn:** Giới hạn phải được kiểm tra tại thời điểm ghi, trong cùng
> giao dịch với bản ghi sử dụng mã; một giá trị đã đọc lên Java không còn đủ tin
> cậy khi giao dịch khác cũng đang thay đổi nó.

## 2. Tác nhân và dữ liệu dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Checkout A | Dùng `FLASH20` cho checkout `CO-A` của khách hàng `C-17` |
| Checkout B | Dùng cùng mã cho checkout `CO-B` của cùng khách hàng |
| `promotion` | Giữ thời hạn, giới hạn toàn cục và tổng lượt đã dùng |
| `promotion_user_usage` | Giữ số lượt của từng khách hàng |
| `promotion_redemption` | Lịch sử một lần mã được áp dụng cho một checkout |
| Máy chủ A, B | Chạy hai giao dịch và hai kết nối độc lập |
| PostgreSQL | Nguồn dữ liệu có thẩm quyền đối với các giới hạn |

Có hai điểm tranh chấp:

1. dòng `promotion` của `FLASH20`, dùng để bảo vệ giới hạn toàn cục;
2. dòng `(promotion_id, customer_id)` trong `promotion_user_usage`, dùng để bảo
   vệ giới hạn của một khách hàng.

Ngoài ra, khóa `(promotion_id, checkout_id)` có thể chưa tồn tại. Một ràng buộc
duy nhất sẽ ngăn cùng một checkout ghi hai lần sử dụng mã.

## 3. Quy tắc bất biến

Với mỗi chương trình khuyến mại `P`:

```text
0 <= promotion.redeemed_count <= promotion.global_limit
    nếu global_limit có giá trị

0 <= promotion_user_usage.used_count <= promotion.per_user_limit
    nếu per_user_limit có giá trị

Mỗi (promotion_id, checkout_id) có tối đa một promotion_redemption.

promotion.redeemed_count
    = số promotion_redemption ở trạng thái APPLIED của chương trình

promotion_user_usage.used_count
    = số promotion_redemption APPLIED của khách hàng trong chương trình
```

Hai công thức đối soát cuối liên quan nhiều dòng nên một ràng buộc `CHECK` trên
một bảng không thể tự bảo vệ toàn bộ. Ứng dụng phải cập nhật bộ đếm và lịch sử
trong cùng giao dịch, đồng thời chạy truy vấn đối soát định kỳ.

## 4. Hai loại giới hạn khác nhau

### Giới hạn toàn cục

Hai khách hàng khác nhau cùng tranh lượt cuối. Cả hai không trùng yêu cầu, nhưng
chỉ một bên được phép tăng `redeemed_count` từ `99` lên `100`.

### Giới hạn mỗi khách hàng

Một khách hàng có hai checkout khác nhau. Mỗi checkout là một ý định hợp lệ,
nhưng tổng số lượt của khách hàng không được vượt `per_user_limit`.

Khóa lũy đẳng chỉ nhận ra một lệnh bị gửi lại. Nó không thay thế giới hạn khi
hai lệnh khác nhau cùng tiêu thụ một hạn mức chung.

## 5. Ranh giới giao dịch

Một lần áp dụng mã thành công dùng một giao dịch ngắn:

```text
BEGIN
  1. chiếm quyền trên (promotion_id, checkout_id)
  2. tăng bộ đếm toàn cục nếu mã còn hiệu lực và còn lượt
  3. tăng bộ đếm của khách hàng nếu chưa đạt giới hạn
  4. chuyển promotion_redemption sang APPLIED
COMMIT
```

Nếu bước 2 hoặc 3 không thỏa điều kiện, giao dịch phải hoàn tác cả bản ghi chiếm
quyền và mọi bộ đếm đã thay đổi trước đó. Không được bắt ngoại lệ bên trong giao
dịch rồi trả kết quả bình thường, vì Spring có thể chốt trạng thái dở dang.

Thứ tự khóa trong giải pháp chính là:

```text
khóa một lần sử dụng mã → dòng promotion → dòng promotion_user_usage
```

Mọi đường ghi liên quan phải tuân theo cùng thứ tự. Nếu một chức năng quản trị
khóa dòng người dùng trước rồi mới khóa chương trình, nó có thể tạo vòng chờ và
gây bế tắc.

## 6. Cách bảo vệ được khuyến nghị

### Bước 1 — Một checkout chỉ dùng mã một lần

```sql
CONSTRAINT uk_redemption_promotion_checkout
    UNIQUE (promotion_id, checkout_id)
```

Ứng dụng chèn bản ghi bằng `INSERT ... ON CONFLICT DO NOTHING RETURNING`. Có dòng
trả về nghĩa là được phép tiếp tục; không có dòng nghĩa là checkout đó đã dùng
mã và cần đọc lại kết quả đã có.

### Bước 2 — Giới hạn toàn cục

```sql
UPDATE promotion
SET redeemed_count = redeemed_count + 1
WHERE promotion_id = :promotionId
  AND status = 'ACTIVE'
  AND starts_at <= CURRENT_TIMESTAMP
  AND ends_at > CURRENT_TIMESTAMP
  AND (global_limit IS NULL OR redeemed_count < global_limit)
RETURNING per_user_limit, redeemed_count;
```

PostgreSQL khóa dòng `promotion`. Bên đến sau chờ, rồi đánh giá lại mệnh đề
`WHERE` trên giá trị mới nhất. Khi lượt cuối đã được dùng, bên sau nhận `0` dòng.

### Bước 3 — Giới hạn mỗi khách hàng

```sql
INSERT INTO promotion_user_usage (
    promotion_id,
    customer_id,
    used_count,
    updated_at
)
VALUES (:promotionId, :customerId, 1, CURRENT_TIMESTAMP)
ON CONFLICT (promotion_id, customer_id) DO UPDATE
SET used_count = promotion_user_usage.used_count + 1,
    updated_at = EXCLUDED.updated_at
WHERE :perUserLimit IS NULL
   OR promotion_user_usage.used_count < :perUserLimit
RETURNING used_count;
```

Nếu hai giao dịch cùng tạo dòng người dùng chưa tồn tại, chỉ mục khóa chính phân
xử bên thắng. Nếu dòng đã tồn tại, PostgreSQL khóa dòng rồi kiểm tra lại điều
kiện trước khi tăng.

## 7. Hợp đồng kết quả

| Kết quả | Ý nghĩa | Có thử lại ngay không? |
| --- | --- | --- |
| `APPLIED` | Cả lịch sử và hai bộ đếm đã chốt | Không |
| `ALREADY_APPLIED` | Checkout này đã dùng mã; trả kết quả đã lưu | Không |
| `GLOBAL_LIMIT_REACHED` | Không còn lượt toàn cục | Không |
| `PER_USER_LIMIT_REACHED` | Khách hàng đã hết lượt | Không |
| `NOT_ACTIVE` | Mã chưa bắt đầu, đã hết hạn hoặc bị tắt | Không; chỉ thử lại nếu hợp đồng cho phép ở thời điểm khác |
| SQLSTATE `55P03` | Hết thời gian chờ khóa | Lỗi kỹ thuật; có thể thử lại có giới hạn |
| SQLSTATE `40P01` hoặc `40001` | Bế tắc hoặc lỗi tuần tự hóa | Giao dịch mới và thử lại có giới hạn |
| Mất phản hồi quanh `COMMIT` | Chưa biết kết quả đã chốt | Tra cứu bằng cùng checkout/khóa yêu cầu |

Kết quả `0` dòng ở bước toàn cục có thể do mã không tồn tại, sai trạng thái, sai
thời hạn hoặc hết lượt. Nếu API cần thông báo chính xác, ứng dụng đọc lại trạng
thái sau khi giao dịch thất bại; không kéo dài khóa chỉ để tạo một thông báo đẹp.

## 8. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh hoặc API | Ý nghĩa |
| --- | --- | --- |
| cập nhật có điều kiện | conditional `UPDATE` | Chỉ tăng bộ đếm khi điều kiện trong `WHERE` còn đúng |
| ràng buộc duy nhất | unique constraint | Cấm hai lịch sử cho cùng chương trình và checkout |
| thêm hoặc cập nhật nguyên tử | `INSERT ... ON CONFLICT DO UPDATE` | Tạo bộ đếm người dùng nếu chưa có, nếu có thì tăng có điều kiện |
| số dòng bị ảnh hưởng | affected-row count | Tín hiệu cho biết phép tăng có thật sự xảy ra hay không |
| kiểm tra lại điều kiện | predicate recheck | PostgreSQL đánh giá lại `WHERE` sau khi chờ giao dịch khác |
| giao dịch nhiều bất biến | multi-invariant transaction | Một giao dịch cùng bảo vệ giới hạn toàn cục, giới hạn người dùng và lịch sử |
| dòng nóng | hot row | Dòng bộ đếm toàn cục bị nhiều giao dịch cập nhật cùng lúc |

## 9. Chạy trên nhiều máy chủ

Khóa Java như `synchronized` chỉ bảo vệ một JVM. Hai checkout có thể đi qua hai
máy chủ khác nhau, hoặc một tác vụ quản trị có thể ghi trực tiếp vào PostgreSQL.

Ràng buộc và câu lệnh có điều kiện nằm ở cơ sở dữ liệu có thẩm quyền nên bảo vệ
mọi kết nối. Không cần thêm khóa phân tán chỉ để bảo vệ các dòng trong cùng một
PostgreSQL; khóa ngoài vẫn không thay thế các ràng buộc cuối cùng.

## 10. Hậu quả khi triển khai sai

- Tổng giá trị giảm giá vượt ngân sách chiến dịch.
- Một khách hàng dùng mã nhiều lần hơn chính sách cho phép.
- Bộ đếm hiển thị `100` nhưng bảng lịch sử có `101` lượt thật.
- Cùng một checkout nhận hai dòng giảm giá hoặc hai khoản điều chỉnh.
- Đối soát phải hủy ưu đãi sau khi đơn đã thanh toán.
- Thử lại lỗi kỹ thuật như một lệnh mới làm tăng thêm lượt dùng.
- Một dòng chương trình nóng tạo hàng đợi khóa và chiếm vùng kết nối.
- Bắt nhầm mọi lỗi dữ liệu thành “mã đã dùng” che giấu hỏng lược đồ.

## 11. Điều hướng tài liệu

- [Mã nguồn kiểm tra rồi mới tăng bộ đếm](broken-code.md)
- [Phân tích hai giới hạn và dòng thời gian](analysis.md)
- [Thiết kế Java và SQL an toàn](solutions.md)
- [Thực nghiệm đồng thời với PostgreSQL](experiments.md)
- [DB-006 — Ràng buộc duy nhất khi ghi đồng thời](../../postgresql/unique-constraint-concurrency/README.md)
- [LOCK-004 — Cập nhật có điều kiện](../../locking/conditional-atomic-update/README.md)
- [Cập nhật dữ liệu an toàn bằng SQL](../../concepts/atomic-database-operations.md)
- [Tính lũy đẳng và tính duy nhất](../../concepts/idempotency-and-uniqueness.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)
- [ECOM-003 — Tạo đơn hàng trùng](../duplicate-checkout-order/README.md)

## 12. Khi nên dùng cách này

Thiết kế bộ đếm phù hợp khi cần giới hạn chính xác và mọi dữ liệu liên quan nằm
trong cùng PostgreSQL. Giới hạn toàn cục tạo một dòng nóng; tải càng cao thì
thông lượng càng bị giới hạn bởi tốc độ cập nhật dòng đó. Tính đúng đắn vẫn giữ,
nhưng cần giới hạn thời gian chờ và đo mức tranh chấp.

Nếu chỉ cho phép mỗi khách hàng dùng đúng một lần, có thể dùng trực tiếp
`UNIQUE (promotion_id, customer_id)` trên lịch sử thay cho bộ đếm người dùng.
Khi cho phép nhiều hơn một lần, cần bộ đếm có điều kiện hoặc một mô hình cấp hạn
mức tương đương.

## 13. Phạm vi

Case này bảo vệ số lượt sử dụng coupon/voucher theo chương trình, khách hàng và
checkout. Chống tạo trùng toàn bộ đơn hàng thuộc ECOM-003. Trừ điểm khách hàng
thuộc ECOM-005. Tính giá, xếp chồng nhiều khuyến mại, hoàn tác ưu đãi sau hủy đơn
và cấp hạn mức giữa nhiều vùng dữ liệu là các bài toán riêng.
