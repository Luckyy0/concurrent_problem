# ECOM-007 — Hết hạn khoản giữ tồn kho đồng thời với xác nhận

## 1. Bài toán

Checkout đã giữ `1` sản phẩm cho khách hàng đến `10:00:00`. Đúng lúc thời hạn
kết thúc, hai tác nhân cùng xử lý một bản ghi `inventory_reservation`:

- yêu cầu checkout xác nhận khoản giữ để tạo đơn;
- tác vụ nền đánh dấu khoản giữ đã hết hạn và trả sản phẩm về kho khả dụng.

Nếu cả hai cùng đọc trạng thái `RESERVED` rồi tự quyết định trong Java, cả hai
có thể thực hiện tác dụng phụ của mình. Kết quả là đơn đã được tạo và lệnh thanh
toán đã phát sinh, nhưng sản phẩm cũng được trả lại để khách khác mua.

```text
kho trước khi giữ        available = 0, reserved = 1, on_hand = 1
xác nhận thành công      reserved giảm 1, on_hand giảm 1
hết hạn thành công       reserved giảm 1, available tăng 1
kết quả sai              đơn tồn tại và available lại bằng 1
```

> **Nói ngắn gọn:** Trạng thái khoản giữ phải quyết định ai được phép tác động
> tới tồn kho. Xác nhận và hết hạn không được cùng thắng.

## 2. Tác nhân và dữ liệu dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Yêu cầu checkout | Xác nhận khoản giữ và tạo đơn |
| Tác vụ hết hạn | Tìm khoản giữ quá hạn và hoàn lại tồn kho |
| Máy chủ A, B | Có thể xử lý hai nhánh trên hai JVM khác nhau |
| `inventory_reservation` | Giữ trạng thái và thời điểm hết hạn của toàn bộ khoản giữ |
| `inventory_reservation_line` | Giữ sản phẩm và số lượng đã dành |
| `inventory_item` | Giữ các bộ đếm `on_hand`, `available`, `reserved` |
| `checkout_request` | Lưu quyền xử lý và phản hồi để chống checkout trùng |
| `purchase_order`, `outbox_event` | Đơn hàng và lệnh thanh toán chỉ được tạo khi xác nhận thắng |
| PostgreSQL | Nguồn dữ liệu có thẩm quyền đối với trạng thái, thời gian và tồn kho |

Điểm tranh chấp chính là dòng `inventory_reservation` của cùng một
`reservation_id`. Các dòng `inventory_item` là điểm tranh chấp tiếp theo khi
bên thắng chuyển số lượng sang trạng thái mới.

## 3. Máy trạng thái và quy tắc bất biến

```text
                         xác nhận trước hạn
                 ┌────────────────────────────> CONFIRMED
                 │
RESERVED ────────┤
                 │       hết hạn
                 ├────────────────────────────> EXPIRED
                 │
                 └────────────────────────────> RELEASED
                         hủy chủ động
```

Mỗi khoản giữ chỉ đi qua đúng một nhánh kết thúc. Không có chuyển trạng thái từ
`EXPIRED` hoặc `RELEASED` sang `CONFIRMED`.

```text
available_quantity >= 0
reserved_quantity >= 0
on_hand_quantity >= 0

available_quantity + reserved_quantity = on_hand_quantity

RESERVED  → số lượng còn nằm trong reserved_quantity
CONFIRMED → số lượng rời reserved_quantity và on_hand_quantity
EXPIRED   → số lượng rời reserved_quantity và trở lại available_quantity

Mỗi reservation_id có tối đa một purchase_order.
Khoản giữ EXPIRED không có lệnh thanh toán mới.
Cùng checkout request chỉ gây tác động một lần.
```

## 4. Ý nghĩa của thời hạn

`expires_at` là thời điểm quyền xác nhận kết thúc, không phải lời hứa rằng tác vụ
nền sẽ chạy chính xác vào thời điểm đó. Một dòng vẫn mang trạng thái `RESERVED`
sau hạn đã là khoản giữ hết hiệu lực về mặt nghiệp vụ.

Vì vậy, nhánh xác nhận phải tự kiểm tra cả trạng thái và thời hạn. Nó không được
giả định rằng `RESERVED` đồng nghĩa còn hạn chỉ vì tác vụ nền đang chậm.

Đồng hồ có thẩm quyền là PostgreSQL:

- không nhận thời điểm quyết định từ trình duyệt;
- không dùng `Instant.now()` trên từng máy chủ để phân xử;
- dùng `clock_timestamp()` ngay trong câu chuyển trạng thái.

Một yêu cầu gửi trước hạn nhưng chưa giành được trạng thái không được bảo đảm sẽ
thắng. Điểm xác định kết quả là câu cập nhật có điều kiện tại PostgreSQL.

## 5. Hai phép chuyển trạng thái có điều kiện

Xác nhận chỉ được phép trước hạn:

```sql
UPDATE inventory_reservation
SET status = 'CONFIRMED',
    confirmed_order_id = :orderId,
    confirmation_request_id = :checkoutRequestId,
    confirmed_at = clock_timestamp(),
    updated_at = clock_timestamp()
WHERE reservation_id = :reservationId
  AND status = 'RESERVED'
  AND expires_at > clock_timestamp()
RETURNING reservation_id;
```

Hết hạn chỉ được phép từ thời điểm hết hạn trở đi:

```sql
UPDATE inventory_reservation
SET status = 'EXPIRED',
    expired_at = clock_timestamp(),
    updated_at = clock_timestamp()
WHERE reservation_id = :reservationId
  AND status = 'RESERVED'
  AND expires_at <= clock_timestamp()
RETURNING reservation_id;
```

Hai câu lệnh cập nhật cùng một dòng. PostgreSQL khóa dòng cho bên đến trước. Bên
còn lại chờ hoặc bị tác vụ `SKIP LOCKED` bỏ qua; sau khi khóa được nhả, điều kiện
`status = 'RESERVED'` không còn đúng nên bên đó nhận `0` dòng.

Trong triển khai hoàn chỉnh, nên lấy một thời điểm duy nhất cho từng câu lệnh để
ghi dấu và kiểm tra cùng một mốc. [solutions.md](solutions.md) trình bày biến thể
CTE dùng `clock_timestamp()` đúng một lần.

## 6. Ranh giới giao dịch

### Khi xác nhận thắng

```text
BEGIN
  1. chiếm hoặc phát lại checkout_request theo ECOM-003
  2. chuyển reservation: RESERVED → CONFIRMED bằng điều kiện trạng thái + thời hạn
  3. chuyển từng dòng tồn kho: reserved giảm, on_hand giảm
  4. tạo purchase_order với reservation_id duy nhất
  5. ghi outbox_event khởi tạo thanh toán
  6. lưu phản hồi checkout
COMMIT
```

### Khi hết hạn thắng

```text
BEGIN
  1. chọn một khoản giữ đến hạn và khóa bằng FOR UPDATE SKIP LOCKED
  2. chuyển reservation: RESERVED → EXPIRED
  3. chuyển từng dòng tồn kho: reserved giảm, available tăng
COMMIT
```

Nếu bất kỳ bước cập nhật bộ đếm, tạo đơn hoặc ghi outbox nào thất bại, thay đổi
trạng thái cũng phải hoàn tác. Không gọi cổng thanh toán qua mạng trong lúc đang
giữ khóa khoản giữ và khóa tồn kho.

## 7. Kết quả của bên thắng và bên thua

| Tình huống | Kết quả | Tác động bền vững |
| --- | --- | --- |
| Xác nhận thắng trước hạn | `CONFIRMED` | Tiêu thụ lượng đã giữ, tạo một đơn và một lệnh thanh toán logic |
| Hết hạn thắng | `EXPIRED` | Trả lượng đã giữ về kho khả dụng đúng một lần |
| Xác nhận đến sau `EXPIRED` | `RESERVATION_EXPIRED` | Không tạo đơn, không phát lệnh thanh toán |
| Tác vụ hết hạn thấy `CONFIRMED` | Bỏ qua | Không hoàn kho |
| Cùng yêu cầu xác nhận gửi lại | Phát lại kết quả checkout | Không tiêu thụ tồn kho lần hai |
| Hết thời gian chờ khóa | Lỗi kỹ thuật | Giao dịch hoàn tác; không giả làm kết quả hết hạn |
| Mất phản hồi sau `COMMIT` | Kết quả chốt chưa rõ | Gửi lại cùng khóa lũy đẳng để tra và phát lại |

## 8. Tính lũy đẳng không thay thế chuyển trạng thái

Khóa lũy đẳng của ECOM-003 nhận ra cùng một ý định checkout được gửi lại. Điều
kiện trạng thái trong ECOM-007 phân xử hai ý định khác nhau: xác nhận và hết hạn.

Chỉ dùng một trong hai cơ chế vẫn sai:

- có khóa lũy đẳng nhưng không có chuyển trạng thái có điều kiện: tác vụ hết hạn
  vẫn có thể hoàn kho sau khi checkout được chấp nhận;
- có chuyển trạng thái an toàn nhưng không có khóa lũy đẳng: mất phản hồi có thể
  làm phía gọi tạo một checkout request mới và một đơn mới.

`confirmation_request_id` và ràng buộc duy nhất trên `purchase_order.reservation_id`
là lớp phòng thủ bổ sung. Phản hồi đầy đủ vẫn được lưu tại `checkout_request`.

## 9. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh hoặc API | Ý nghĩa |
| --- | --- | --- |
| khoản giữ tồn kho | inventory reservation / hold | Số lượng tạm thời không còn bán cho yêu cầu khác |
| thời hạn giữ | lease time | Khoảng thời gian quyền xác nhận còn hiệu lực |
| chuyển trạng thái có điều kiện | guarded transition | Chỉ đổi trạng thái nếu trạng thái và thời gian hiện tại còn phù hợp |
| trạng thái kết thúc | terminal state | Trạng thái không được chuyển tiếp sang nhánh cạnh tranh khác |
| đồng hồ có thẩm quyền | authoritative clock | Nguồn thời gian duy nhất dùng để phân xử hết hạn |
| bỏ qua dòng đang khóa | `SKIP LOCKED` | Cho tác vụ khác xử lý dòng chưa bị tác vụ khác giữ |
| khóa lũy đẳng | idempotency key | Mã ổn định để phát lại cùng một ý định mà không làm lại tác dụng phụ |
| hộp thư đi bền vững | transactional outbox | Bảng lưu lệnh gửi ra ngoài trong cùng giao dịch với dữ liệu nghiệp vụ |
| điểm phân xử | linearization point | Thao tác nguyên tử tạo ra thứ tự kết quả mà mọi máy chủ phải chấp nhận |
| kết quả chốt chưa rõ | ambiguous commit outcome | Phía gọi mất phản hồi và không biết giao dịch đã chốt hay chưa |

## 10. Chạy trên nhiều máy chủ

`synchronized`, `ReentrantLock` hoặc bộ lập lịch chỉ chạy một luồng trên một JVM
không bảo vệ được khi checkout và tác vụ nằm ở các máy chủ khác nhau. Chúng cũng
mất tác dụng sau khi tiến trình khởi động lại.

Điều kiện trên `inventory_reservation.status`, thời hạn do PostgreSQL quyết định
và giao dịch cập nhật bộ đếm có hiệu lực với mọi kết nối. Nhiều tác vụ hết hạn
có thể chạy song song bằng `SKIP LOCKED` mà không cần bầu một máy chủ duy nhất.

## 11. Hậu quả khi triển khai sai

- Đơn đã tạo nhưng tồn kho bị trả lại và bán cho khách khác.
- Khách đã thanh toán nhưng không còn hàng để giao.
- Một khoản giữ bị hoàn kho hai lần làm `available_quantity` lớn hơn thực tế.
- Khoản giữ đã hết hạn vẫn được xác nhận vì tác vụ chạy chậm.
- Bộ đếm tồn kho không khớp với tổng các khoản giữ còn hoạt động.
- Hai tác vụ giữ khóa quá lâu, chiếm vùng kết nối và làm chậm checkout.
- Mất phản hồi khiến lần thử lại tạo thêm đơn hoặc lệnh thanh toán.
- Nhân viên phải hủy đơn, hoàn tiền và điều chỉnh tồn kho thủ công.

## 12. Điều hướng tài liệu

- [Mã nguồn đọc–kiểm tra–ghi gây lỗi](broken-code.md)
- [Phân tích dòng thời gian và hành vi khóa](analysis.md)
- [Thiết kế Java và SQL an toàn](solutions.md)
- [Thực nghiệm đồng thời với PostgreSQL](experiments.md)
- [ECOM-001 — Bán vượt tồn kho](../overselling-inventory/README.md)
- [ECOM-003 — Tạo đơn checkout trùng](../duplicate-checkout-order/README.md)
- [Tính lũy đẳng và tính duy nhất](../../concepts/idempotency-and-uniqueness.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## 13. Khi nên dùng cách này

Dùng chuyển trạng thái có điều kiện khi một dòng khoản giữ là nguồn quyết định
có thẩm quyền và hai nhánh cạnh tranh có thể diễn đạt trong `WHERE`. Đây là cơ
chế nhỏ nhất bảo vệ đúng bất biến trong một PostgreSQL.

Dùng `FOR UPDATE` khi quyết định cần đọc thêm nhiều dữ liệu trước khi đổi trạng
thái, nhưng giữ giao dịch ngắn. Dùng `@Version` khi xung đột hiếm và thời gian
vẫn được kiểm tra bằng nguồn có thẩm quyền. Với tác vụ theo lô, `SKIP LOCKED` giúp
phân việc; nó không thay thế điều kiện trạng thái.

## 14. Phạm vi

Case này xử lý vòng đời khoản giữ tồn kho trong cùng cơ sở dữ liệu: xác nhận,
hết hạn, bộ đếm tồn kho và kết quả checkout liên quan. Việc trừ kho lúc tạo khoản
giữ thuộc ECOM-001; chống tạo checkout trùng thuộc ECOM-003.

Giữ ghế có quy tắc chọn ghế và vòng đời riêng thuộc `BOOK-004`. Xác nhận thanh
toán qua nhiều dịch vụ, bù trừ sau khi cổng thanh toán đã thu tiền và điều phối
kho ở nhiều cơ sở dữ liệu cần một luồng công việc phân tán khác.
