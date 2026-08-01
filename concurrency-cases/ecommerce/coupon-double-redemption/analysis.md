# Phân tích tranh chấp trên giới hạn coupon/voucher

## 1. Trạng thái ban đầu

```text
promotion P-20:
  status          = ACTIVE
  redeemed_count  = 99
  global_limit    = 100
  per_user_limit  = 1

promotion_user_usage(P-20, C-17): chưa có dòng
promotion_redemption của C-17:    chưa có dòng
```

Giao dịch A áp dụng mã cho checkout `CO-A`. Giao dịch B áp dụng mã cho checkout
`CO-B`. Đây là hai lệnh khác nhau của cùng khách hàng, không phải một yêu cầu bị
gửi lại.

## 2. Kết quả mong đợi và thực tế

| Nội dung | Mong đợi | Thiết kế bị lỗi |
| --- | --- | --- |
| Lượt mới toàn cục | `1` | `2` lịch sử nhưng bộ đếm có thể chỉ tăng `1` |
| Lượt mới của `C-17` | `1` | `2` |
| Giao dịch thành công | Một bên | Cả hai bên |
| Quan hệ bộ đếm–lịch sử | Khớp nhau | Có thể lệch nhau |

## 3. Ba bất biến phải phối hợp

Một lần sử dụng mã chỉ hợp lệ khi đồng thời thỏa:

1. checkout này chưa dùng chương trình;
2. chương trình còn lượt toàn cục;
3. khách hàng còn lượt cá nhân.

Không có một khóa đơn lẻ tự động bảo vệ cả ba điều kiện. Giải pháp cần dùng cơ
chế phù hợp cho từng loại dữ liệu rồi gộp các thay đổi vào cùng giao dịch:

| Bất biến | Cơ chế có thẩm quyền |
| --- | --- |
| Một lần trên mỗi checkout | `UNIQUE (promotion_id, checkout_id)` |
| Tổng lượt toàn cục | `UPDATE` có điều kiện trên dòng `promotion` |
| Tổng lượt mỗi khách hàng | Thêm hoặc tăng có điều kiện trên `promotion_user_usage` |
| Bộ đếm khớp lịch sử | Cùng ranh giới `COMMIT`/`ROLLBACK` và đối soát |

## 4. Dòng thời gian bị lỗi với cùng khách hàng

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | Đếm lượt của `C-17`, nhận `0` | |
| 2 | | Đếm lượt của `C-17`, nhận `0` |
| 3 | Đọc `redeemed_count = 99` | |
| 4 | | Đọc `redeemed_count = 99` |
| 5 | Quyết định chấp nhận | Quyết định chấp nhận |
| 6 | Chèn lịch sử cho `CO-A` | Chèn lịch sử cho `CO-B` |
| 7 | Ghi bộ đếm tuyệt đối `100` | Ghi bộ đếm tuyệt đối `100` |
| 8 | `COMMIT` | `COMMIT` |

Mỗi câu đọc đều đúng tại thời điểm nó chạy. Lỗi nằm ở giả định rằng kết quả đọc
vẫn còn đúng khi câu ghi diễn ra.

## 5. Dòng thời gian bị lỗi với hai khách hàng

Để tách riêng giới hạn toàn cục, cho A thuộc `C-17`, B thuộc `C-18`; cả hai đều
còn lượt cá nhân:

```text
A đọc 99 < 100 → chấp nhận
B đọc 99 < 100 → chấp nhận
A và B tạo hai lịch sử
```

Ngay cả khi bảng lịch sử có `UNIQUE (promotion_id, customer_id)` cho chính sách
mỗi người một lần, hai khách hàng khác nhau không xung đột trên ràng buộc đó.
Giới hạn toàn cục vẫn cần một phép tăng có điều kiện.

## 6. Ảnh chụp MVCC và khoảng trống

Ở `READ COMMITTED`, mỗi câu `SELECT` nhìn thấy các dòng đã chốt trước khi câu đó
bắt đầu. Nó không thấy thay đổi chưa chốt của giao dịch khác như dữ liệu bình
thường.

Kết quả `count = 0` không tạo một dòng để khóa. Vì vậy hai giao dịch có thể cùng
kết luận khách hàng chưa dùng mã. Đây là dạng kiểm tra một tập hợp rồi chèn thêm
dòng; chỉ khóa hàng hiện có là chưa đủ.

Ràng buộc duy nhất giải quyết trường hợp giới hạn đúng một lần. Với giới hạn lớn
hơn một, một dòng bộ đếm theo khách hàng biến điều kiện trên nhiều lịch sử thành
điều kiện trên một dòng có thể cập nhật nguyên tử.

## 7. Giới hạn toàn cục với `UPDATE` có điều kiện

```sql
UPDATE promotion
SET redeemed_count = redeemed_count + 1
WHERE promotion_id = :promotionId
  AND redeemed_count < global_limit
RETURNING redeemed_count;
```

Dòng thời gian tại lượt cuối:

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | Khóa dòng `promotion` | |
| 2 | Kiểm tra `99 < 100`, tăng thành `100` | Chờ khóa cùng dòng |
| 3 | `COMMIT` | Được đánh thức |
| 4 | | PostgreSQL kiểm tra lại `100 < 100` |
| 5 | | Điều kiện sai, nhận `0` dòng |

Câu lệnh không dựa vào giá trị `99` đã tải lên Java. PostgreSQL tính trên phiên
bản dòng mới nhất sau khi B lấy được khóa.

## 8. Khi bên giữ khóa hoàn tác

Nếu A tăng từ `99` lên `100` rồi gặp lỗi trước `COMMIT`:

```text
A ROLLBACK
→ giá trị bền vững trở lại 99
→ B được đánh thức
→ B kiểm tra 99 < 100
→ B có thể tăng thành 100 và chốt
```

Không có lượt nào bị mất chỉ vì một giao dịch thất bại.

## 9. Bộ đếm người dùng chưa tồn tại

Câu `UPDATE` thông thường không thể khóa hoặc tăng một dòng chưa tồn tại. Phép
thêm hoặc cập nhật sau xử lý cả lần đầu và các lần sau:

```sql
INSERT INTO promotion_user_usage (..., used_count)
VALUES (..., 1)
ON CONFLICT (promotion_id, customer_id) DO UPDATE
SET used_count = promotion_user_usage.used_count + 1
WHERE promotion_user_usage.used_count < :perUserLimit
RETURNING used_count;
```

Hai giao dịch cùng chèn lần đầu sẽ tranh chấp trên khóa chính. Bên thua chờ kết
quả của bên thắng. Nếu bên thắng chốt với `used_count = 1` và giới hạn là `1`,
điều kiện cập nhật của bên sau sai và không có dòng trả về. Nếu bên thắng hoàn
tác, bên đang chờ có thể chèn dòng mới.

## 10. Vì sao phải hoàn tác khi bước sau thất bại

Giải pháp chính tăng bộ đếm toàn cục trước rồi mới tăng bộ đếm người dùng. Giả sử
chương trình còn nhiều lượt toàn cục nhưng `C-17` đã hết lượt:

```text
1. tăng redeemed_count thành công
2. tăng used_count trả về 0 dòng
3. ném PerUserLimitReachedException ra khỏi phương thức @Transactional
4. Spring ROLLBACK
5. redeemed_count trở lại giá trị cũ; lịch sử PROCESSING cũng biến mất
```

Nếu mã bắt ngoại lệ ở bước 2 và trả về `PER_USER_LIMIT_REACHED` ngay bên trong
giao dịch, bước 1 có thể bị chốt. Kết quả là chương trình mất một lượt dù không
có lần sử dụng hợp lệ.

## 11. Thứ tự khóa và bế tắc

Luồng áp dụng mã chính lấy khóa theo thứ tự:

```text
khóa duy nhất của lịch sử → promotion → promotion_user_usage
```

Hai checkout khác nhau không tranh khóa lịch sử, nhưng cùng chương trình sẽ xếp
hàng ở dòng `promotion`. Sau đó chúng mới chạm dòng người dùng.

Một chức năng quản trị làm ngược lại có thể tạo vòng chờ:

```text
Tx A giữ promotion, chờ user_usage
Tx B giữ user_usage, chờ promotion
```

PostgreSQL phát hiện bế tắc và hủy một giao dịch bằng SQLSTATE `40P01`. Cách xử
lý đúng là thống nhất thứ tự khóa; thử lại chỉ là lớp phục hồi, không phải cách
che một thứ tự khóa không nhất quán.

## 12. Cùng checkout được gửi lại

Hai lần gọi cùng `(promotion_id, checkout_id)` tranh ràng buộc duy nhất:

- bên thắng chốt: bên thua đọc lịch sử `APPLIED` và trả lại kết quả;
- bên thắng hoàn tác: bên đang chờ có thể trở thành bên thắng;
- cùng khóa nhưng khác dấu vân tay: từ chối vì dùng lại khóa cho nội dung khác.

Đây là lớp chống lặp của một lần sử dụng mã. Toàn bộ hợp đồng checkout lũy đẳng,
bao gồm lưu phản hồi đơn hàng, thuộc ECOM-003.

## 13. Thời hạn và đồng hồ

Điều kiện hoạt động nên được kiểm tra trong câu cập nhật có thẩm quyền:

```sql
starts_at <= CURRENT_TIMESTAMP
AND ends_at > CURRENT_TIMESTAMP
```

Như vậy hai máy chủ có đồng hồ lệch nhẹ không tự đưa ra hai quyết định khác nhau.
`CURRENT_TIMESTAMP` ổn định trong một giao dịch PostgreSQL. Cần quy định rõ biên
kết thúc là đóng hay mở; ví dụ trên cho phép dùng tại `starts_at` nhưng không cho
dùng đúng tại `ends_at`.

Việc một mã hết hạn trong lúc giao dịch đang chờ khóa được quyết định khi câu
lệnh thật sự đánh giá mệnh đề `WHERE`, không phải theo thời điểm yêu cầu vào máy
chủ.

## 14. Phân loại kết quả `0` dòng

Câu cập nhật toàn cục có thể trả `0` vì:

- không có `promotion_id`;
- trạng thái không phải `ACTIVE`;
- chưa tới thời gian bắt đầu;
- đã hết hạn;
- đã đạt giới hạn toàn cục.

Nếu mọi lý do đều được công bố chung là `PROMOTION_UNAVAILABLE`, không cần thêm
câu đọc. Nếu API phải trả lý do cụ thể, hãy hoàn tác giao dịch áp dụng mã rồi đọc
trạng thái bằng một giao dịch ngắn mới. Kết quả có thể thay đổi ngay sau khi đọc,
nhưng đó chỉ là thông báo chẩn đoán; quyết định không áp dụng mã đã được bảo vệ
bởi câu cập nhật.

## 15. Hibernate và SQL cập nhật trực tiếp

Câu `UPDATE`/upsert trực tiếp bỏ qua ngữ cảnh lưu trữ của Hibernate. Nếu một
thực thể `Promotion` đã được tải trong cùng giao dịch, nó vẫn giữ bộ đếm cũ và có
thể ghi đè khi `flush()`.

Cách an toàn là không quản lý thực thể `Promotion` trong giao dịch sử dụng JDBC
cho bộ đếm. Nếu bắt buộc trộn hai cách, phải `flush()` thay đổi đang chờ trước
câu SQL trực tiếp và `clear()` ngữ cảnh sau đó; thiết kế này khó kiểm soát hơn.

## 16. Chốt, hoàn tác và mất phản hồi

| Tình huống | Trạng thái bền vững | Hành vi lần sau |
| --- | --- | --- |
| Chốt thành công | Hai bộ đếm và lịch sử cùng tồn tại | Đọc lại `APPLIED` |
| Lỗi trước chốt | Không còn thay đổi của lần thử | Có thể thử lại bằng cùng khóa |
| Hết thời gian chờ khóa | Giao dịch thất bại hoặc chưa chạy phép tăng | Giao dịch mới, thử lại có giới hạn |
| Mất phản hồi sau chốt | Kết quả có thể đã tồn tại | Tra cứu lịch sử; không tạo checkout mới |
| Bế tắc | Một giao dịch bị PostgreSQL hoàn tác | Thử lại toàn bộ với thứ tự khóa đúng |

## 17. Tranh chấp trên dòng toàn cục

Mọi lần dùng cùng chương trình đều tăng một dòng `promotion`, nên đây là dòng
nóng. Câu lệnh có điều kiện giữ dữ liệu đúng nhưng không làm dòng đó có thể ghi
song song.

Tăng số kết nối hoặc số máy chủ chỉ tạo thêm bên chờ. Với chiến dịch cực lớn,
có thể phải cấp trước các khối hạn mức cho phân vùng hoặc dùng một hàng đợi theo
chương trình. Những cách đó đổi mô hình lỗi và cần cơ chế thu hồi hạn mức chưa
dùng; không được cộng các bộ đếm gần đúng rồi vẫn tuyên bố giới hạn toàn cục là
chính xác.

## 18. Chạy trên nhiều máy chủ

Mọi máy chủ cùng gửi câu SQL tới PostgreSQL chính. Khóa dòng, chỉ mục duy nhất và
việc kiểm tra lại điều kiện không phụ thuộc tiến trình Java nào thắng lịch lập
lịch.

Khóa JVM có thể giảm tranh chấp trong một máy nhưng không bảo vệ toàn hệ thống.
Khóa phân tán không loại bỏ nhu cầu về ràng buộc và giao dịch ở cơ sở dữ liệu có
thẩm quyền.

## 19. Nguyên nhân gốc theo tầng

| Tầng | Vấn đề |
| --- | --- |
| Nghiệp vụ | Không biểu diễn riêng giới hạn toàn cục, giới hạn người dùng và một lần trên checkout |
| Ứng dụng | Dùng `read → check → write` và tin kết quả `count()` còn đúng |
| Spring | Hiểu nhầm `@Transactional` là khóa loại trừ giữa yêu cầu |
| Hibernate | Ghi bộ đếm tuyệt đối từ thực thể cũ và có thể trì hoãn `UPDATE` tới `flush()` |
| PostgreSQL | Lược đồ thiếu ràng buộc duy nhất và câu ghi thiếu điều kiện giới hạn |
| Vận hành | Các đường ghi lấy khóa khác thứ tự hoặc thử lại không giới hạn |

## 20. Dữ liệu cần theo dõi

```text
promotion.redemption.applied
promotion.redemption.replayed
promotion.redemption.global_limit_rejected
promotion.redemption.user_limit_rejected
promotion.redemption.not_active
promotion.redemption.lock_wait_duration
promotion.redemption.deadlock
promotion.redemption.rollback
promotion.counter.reconciliation_mismatch
```

Theo dõi độ lệch giữa bộ đếm và lịch sử là bắt buộc. Chỉ nhìn
`redeemed_count <= global_limit` không phát hiện được trường hợp hai lịch sử cùng
ghi đè thành một lần tăng.

## 21. Phạm vi phân tích

Case này giả định các bảng khuyến mại nằm trong cùng PostgreSQL. Khi bộ đếm toàn
cục và lịch sử nằm ở hai dịch vụ, không có một giao dịch cục bộ bao trùm; cần mô
hình cấp hạn mức hoặc quy trình bù trừ riêng.

Chống tạo đơn trùng thuộc ECOM-003. Trừ điểm khách hàng thuộc ECOM-005. Việc hoàn
lại lượt dùng sau hủy đơn cần một chuyển trạng thái có điều kiện và lịch sử kiểm
toán riêng, không chỉ giảm bộ đếm bằng một yêu cầu HTTP bất kỳ.
