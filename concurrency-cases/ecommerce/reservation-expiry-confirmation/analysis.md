# Phân tích cuộc đua giữa hết hạn và xác nhận khoản giữ

## 1. Trạng thái ban đầu

Khoản giữ `R-42` chứa hai dòng:

| Sản phẩm | Số lượng | Kho khả dụng | Đang giữ | Tồn thực tế |
| --- | ---: | ---: | ---: | ---: |
| `SKU-A` | 1 | 0 | 1 | 1 |
| `SKU-B` | 2 | 3 | 2 | 5 |

```text
reservation.status = RESERVED
expires_at          = 10:00:00
purchase_order      = chưa có
outbox payment      = chưa có
```

Checkout muốn xác nhận `R-42`; tác vụ muốn hết hạn chính `R-42`. Cả hai thao tác
đều hợp lệ ở những thời điểm khác nhau, nhưng chỉ một thao tác được phép giành
trạng thái kết thúc.

## 2. Chuỗi thao tác không nguyên tử

Bản lỗi thực hiện:

```text
đọc reservation
→ kiểm tra status và expires_at trong Java
→ quyết định xác nhận hoặc hết hạn
→ sửa inventory / tạo order
→ ghi status
```

Điều kiện được kiểm tra trên một ảnh chụp, còn tác dụng phụ diễn ra sau đó. Không
có phép ghi nào chứng minh giả định `status = RESERVED` vẫn đúng tại thời điểm
chuyển trạng thái.

## 3. Dòng thời gian lỗi với hai tác nhân

```text
thời điểm  checkout / giao dịch C          tác vụ / giao dịch E
---------  -----------------------------   -----------------------------
t1         BEGIN                           BEGIN
t2         đọc R-42 = RESERVED             đọc R-42 = RESERVED
t3         cho rằng còn hạn                 cho rằng đã tới hạn
t4         tạo order + outbox               hoàn SKU-A, SKU-B về available
t5         ghi status = CONFIRMED           ghi status = EXPIRED
t6         COMMIT                           COMMIT
```

Nếu C ghi trạng thái sau cùng, dòng khoản giữ là `CONFIRMED` nhưng tồn kho đã
được hoàn. Nếu E ghi sau cùng, dòng là `EXPIRED` nhưng đơn và lệnh thanh toán đã
được tạo. Trạng thái cuối không thể xóa tác dụng phụ đã chốt ở nhánh còn lại.

## 4. Kết quả mong đợi và kết quả thực tế

| Trường hợp | Kết quả đúng | Bản lỗi có thể tạo ra |
| --- | --- | --- |
| Xác nhận giành trạng thái trước hạn | `CONFIRMED`, tiêu thụ lượng giữ, một đơn | Đơn tồn tại nhưng lượng giữ được hoàn |
| Hết hạn giành trạng thái | `EXPIRED`, hoàn lượng giữ, không có đơn | Đơn/lệnh thanh toán vẫn được tạo |
| Hai tác vụ hết hạn | Chỉ một tác vụ hoàn kho | Cả hai cùng cộng `available` |
| Xác nhận gửi lại | Phát lại cùng kết quả | Tạo mã đơn hoặc lệnh thanh toán mới |
| Lỗi giữa các bước | Toàn bộ hoàn tác | Trạng thái và bộ đếm lệch nhau |

## 5. Lớp gây lỗi chính xác

| Lớp | Vai trò | Vấn đề nếu đứng một mình |
| --- | --- | --- |
| JVM | Chạy kiểm tra thời gian và trạng thái | Hai JVM có bộ nhớ và đồng hồ riêng |
| Spring | Bao các thao tác trong một giao dịch | Không tự ngăn giao dịch khác cùng đọc `RESERVED` |
| Hibernate | Dò thay đổi và ghi lúc `flush` | Không có `@Version` thì câu ghi chỉ tìm theo khóa chính |
| PostgreSQL | Giữ trạng thái và bộ đếm có thẩm quyền | Chỉ phân xử đúng nếu điều kiện nghiệp vụ nằm trong câu ghi |
| Bộ lập lịch tác vụ | Quyết định lúc quét ứng viên | Chạy chậm không được phép kéo dài thời hạn giữ |

Nguyên nhân gốc là chuỗi `read → check time/state → perform side effects → write`
không nguyên tử. Đây không chỉ là “hai request chạy cùng lúc”.

## 6. Ảnh chụp MVCC dưới `READ COMMITTED`

Ở mức cô lập mặc định, mỗi câu lệnh nhìn một ảnh chụp các dữ liệu đã chốt trước
khi câu lệnh bắt đầu. Hai câu `SELECT` thông thường có thể cùng thấy
`status = 'RESERVED'`.

Điều khiển đồng thời đa phiên bản (MVCC) giúp thao tác đọc không phải luôn chặn
thao tác ghi. Nó không giữ giả định đã đọc cho một câu `UPDATE` chạy sau đó. Nếu
câu cập nhật chỉ có:

```sql
WHERE reservation_id = :reservationId
```

bên ghi sau vẫn được phép cập nhật dòng sau khi chờ khóa. PostgreSQL không biết
quyết định trong Java dựa trên trạng thái cũ.

## 7. Chuyển trạng thái có điều kiện là điểm phân xử

Hai nhánh phải đưa toàn bộ điều kiện vào câu ghi:

```text
CONFIRM: status = RESERVED AND expires_at > database_time
EXPIRE:  status = RESERVED AND expires_at <= database_time
```

Hai miền thời gian dùng `>` và `<=`, nên không có khoảng trống tại đúng
`expires_at`. Trạng thái chung bảo đảm chỉ một câu lệnh có thể đổi dòng thành
trạng thái kết thúc.

Kết quả `1` dòng không chỉ là chi tiết kỹ thuật. Nó là quyền thực hiện tác dụng
phụ của nhánh đó. Kết quả `0` dòng bắt buộc dừng trước khi sửa tồn kho, tạo đơn
hoặc ghi outbox.

## 8. Hành vi khóa khi hai câu cùng cập nhật

### Xác nhận giữ khóa trước

```text
C: UPDATE ... RESERVED → CONFIRMED  ── giữ khóa dòng reservation
E: UPDATE ... RESERVED → EXPIRED    ── chờ hoặc bị SKIP LOCKED bỏ qua
C: cập nhật bộ đếm tồn kho, tạo order, COMMIT
E: kiểm tra lại status, nhận 0 dòng
```

### Hết hạn giữ khóa trước

```text
E: UPDATE ... RESERVED → EXPIRED    ── giữ khóa dòng reservation
C: UPDATE ... RESERVED → CONFIRMED  ── chờ
E: hoàn bộ đếm tồn kho, COMMIT
C: kiểm tra lại status, nhận 0 dòng và trả RESERVATION_EXPIRED
```

PostgreSQL khóa dòng vì cả hai đều thực hiện `UPDATE`. Sau khi bên giữ khóa chốt,
bên chờ đánh giá lại điều kiện trên phiên bản dòng mới và thấy `status` không còn
`RESERVED`.

Nếu bên giữ khóa hoàn tác, thay đổi trạng thái biến mất. Bên chờ được đánh thức
và có thể tiếp tục nếu điều kiện trạng thái cùng thời hạn của chính câu lệnh vẫn
đúng.

## 9. Đồng hồ nào quyết định

### Không dùng thời gian từ phía gọi

Trình duyệt có thể sai giờ hoặc cố tình gửi thời điểm cũ. `expires_at` và thời
điểm quyết định phải được so sánh ở hệ thống có thẩm quyền.

### Không dùng đồng hồ riêng của từng JVM

Đồng bộ NTP làm giảm sai lệch nhưng không tạo một phép phân xử nguyên tử. Một
tiến trình còn có thể bị tạm dừng sau khi gọi `Instant.now()` rồi tiếp tục với
một thời điểm đã cũ.

### Dùng thời gian trong câu SQL

`clock_timestamp()` được đánh giá tại PostgreSQL trong lúc câu lệnh chạy. Nên
chụp nó đúng một lần trong biểu thức `WITH` và dùng cùng giá trị cho điều kiện cùng cột dấu
thời gian.

`CURRENT_TIMESTAMP` là thời điểm bắt đầu giao dịch. Nếu giao dịch đã mở từ lâu,
nó có thể làm một khoản giữ quá hạn trông như vẫn còn hạn. Vì vậy, case này dùng
`clock_timestamp()` tại câu chuyển trạng thái và giữ giao dịch ngắn.

## 10. Thời điểm tuyến tính hóa nghiệp vụ

Thời điểm tuyến tính hóa là lần cập nhật có điều kiện thành công trên dòng khoản
giữ. Nó tạo một thứ tự duy nhất mà mọi máy chủ phải chấp nhận.

Một request bắt đầu trước hạn nhưng bị tạm dừng không tự sở hữu khoản giữ. Nếu
tác vụ đã chuyển dòng sang `EXPIRED`, yêu cầu đó thua. Ngược lại, nếu xác nhận đã
đổi dòng sang `CONFIRMED` trước khi tác vụ giành khóa, việc tác vụ chạy sau hạn
không được hoàn kho.

Giao dịch xác nhận có thể chốt sau `expires_at` nếu câu chuyển trạng thái đã thắng
trước hạn. Chính sách này tránh việc thời gian trôi qua trong vài câu ghi nội bộ
làm đảo ngược một quyền đã được cấp. Không thực hiện I/O từ xa sau khi giành
trạng thái và trước `COMMIT`.

## 11. Bộ đếm tồn kho theo từng trạng thái

Khi tạo khoản giữ theo ECOM-001:

```text
available -= quantity
reserved  += quantity
on_hand    không đổi
```

Khi xác nhận:

```text
reserved -= quantity
on_hand  -= quantity
available không đổi
```

Khi hết hạn hoặc hủy:

```text
reserved  -= quantity
available += quantity
on_hand    không đổi
```

Câu cập nhật nên có điều kiện `reserved_quantity >= quantity`. Nếu điều kiện
không đúng hoặc thiếu một dòng sản phẩm, đó là lỗi bất biến cần hoàn tác và cảnh
báo; không được bỏ qua rồi vẫn chốt trạng thái kết thúc.

## 12. Nhiều sản phẩm và thứ tự khóa

Một khoản giữ có thể chứa nhiều sản phẩm. Sau khi giành trạng thái khoản giữ,
giao dịch phải khóa các dòng `inventory_item` theo `product_id` tăng dần trước
khi cập nhật.

Nếu một đường khóa `SKU-A → SKU-B` còn đường khác khóa `SKU-B → SKU-A`, hai giao
dịch có thể tạo vòng chờ và PostgreSQL phải chọn một nạn nhân bế tắc. Thứ tự ổn
định làm giảm nguy cơ này nhưng không loại bỏ nhu cầu xử lý SQLSTATE `40P01`.

Không dựa vào thứ tự ngẫu nhiên của `HashMap`, danh sách từ API hoặc kế hoạch
thực thi không có `ORDER BY` để quyết định thứ tự khóa.

## 13. Tác vụ theo lô và `SKIP LOCKED`

Nhiều tác vụ có thể lấy việc bằng:

```sql
SELECT reservation_id
FROM inventory_reservation
WHERE status = 'RESERVED'
  AND expires_at <= clock_timestamp()
ORDER BY expires_at, reservation_id
LIMIT :batchSize
FOR UPDATE SKIP LOCKED;
```

- `FOR UPDATE` giữ dòng đến `COMMIT` hoặc `ROLLBACK`.
- `SKIP LOCKED` tránh để một tác vụ đứng chờ dòng tác vụ khác đang xử lý.
- `ORDER BY` làm thứ tự lấy việc ổn định.
- `LIMIT` giữ giao dịch không ôm quá nhiều dòng.

Không tách việc chọn ứng viên và hoàn kho thành hai giao dịch mà bỏ điều kiện
trạng thái. Danh sách ứng viên có thể cũ ngay sau khi khóa được nhả.

`SKIP LOCKED` không bảo đảm công bằng tuyệt đối. Một dòng bị tranh chấp có thể bị
bỏ qua nhiều lượt, nên cần quét lặp, đo tuổi khoản giữ quá hạn và cảnh báo dòng
bị kẹt.

## 14. Tính lũy đẳng của nhánh xác nhận

Hai vấn đề khác nhau phải được bảo vệ:

1. Xác nhận và hết hạn tranh chấp cùng trạng thái — giải bằng chuyển trạng thái
   có điều kiện.
2. Cùng checkout request được gửi lại — giải bằng khóa lũy đẳng của ECOM-003.

Nếu xác nhận thắng rồi phản hồi bị mất, lần gửi lại cùng khóa phải đọc
`checkout_request` đã hoàn tất và phát lại cùng `orderId`. Nó không chạy lại câu
tiêu thụ tồn kho.

`purchase_order.reservation_id` duy nhất là lớp phòng thủ để một lỗi mã nguồn
không tạo hai đơn cho cùng khoản giữ. Nó không tự lưu mã HTTP, nội dung phản hồi
hay phát hiện cùng khóa nhưng khác dấu vân tay.

## 15. Chốt và hoàn tác

### Xác nhận thắng rồi tạo đơn thất bại

Lỗi khóa ngoại, ràng buộc duy nhất hoặc ghi outbox làm giao dịch hoàn tác. Trạng
thái trở lại `RESERVED`; thay đổi `reserved/on_hand` cũng biến mất. Tác vụ đang
chờ có thể thử giành trạng thái sau đó.

### Hết hạn thắng rồi cập nhật tồn kho thất bại

Trạng thái `EXPIRED` phải hoàn tác cùng bộ đếm. Khoản giữ vẫn `RESERVED` nhưng đã
quá hạn; tác vụ khác sẽ xử lý lại. Không chốt riêng trạng thái rồi “sửa kho sau”.

### Tác vụ dừng giữa giao dịch

Kết nối bị đóng làm PostgreSQL hoàn tác và nhả khóa. Dòng lại có thể được tác vụ
khác chọn.

### Tiến trình dừng sau `COMMIT`

Trạng thái và bộ đếm đã bền vững. Tác vụ chạy lại nhận `0` dòng hoặc không còn
thấy ứng viên. Checkout gửi lại dùng cùng khóa lũy đẳng để phát lại kết quả.

## 16. Hết thời gian chờ, bế tắc và thử lại

| Tín hiệu | Ý nghĩa | Cách xử lý |
| --- | --- | --- |
| `0` dòng, trạng thái `EXPIRED` | Thua nghiệp vụ | Trả `RESERVATION_EXPIRED`, không thử xác nhận lại |
| `0` dòng, trạng thái `CONFIRMED` cùng request | Yêu cầu đã hoàn tất | Phát lại kết quả |
| SQLSTATE `55P03` | Hết thời gian chờ khóa | Giao dịch mới, thử lại có giới hạn |
| SQLSTATE `40P01` | Nạn nhân bế tắc | Giao dịch mới, khóa lại theo thứ tự và thử có giới hạn |
| SQLSTATE `40001` | Lỗi tuần tự hóa nếu dùng mức cao hơn | Thử lại toàn giao dịch mới |
| Mất kết nối quanh `COMMIT` | Kết quả chưa rõ | Tra bằng khóa lũy đẳng; không tạo ý định mới |

Tác vụ hết hạn có thể thử lại vì điều kiện `status = 'RESERVED'` làm thao tác an
toàn trước gửi lại. Checkout phải giữ nguyên khóa lũy đẳng; một khóa mới được xem
là ý định mới và có thể tạo kết quả khác.

Không thử lại trong giao dịch đã nhận lỗi PostgreSQL. Giao dịch đó phải kết thúc,
vùng quản lý thực thể bị bỏ và lần thử sau tải lại trạng thái.

## 17. Nhiều máy chủ

Checkout ở máy chủ A, tác vụ ở máy chủ B và tác vụ khác ở máy chủ C đều cập nhật
cùng dòng PostgreSQL. Khóa dòng và điều kiện trạng thái tạo một thứ tự chung mà
không cần khóa trong bộ nhớ hay một bộ lập lịch duy nhất.

Mọi đường thay đổi vòng đời phải dùng cùng quy tắc: checkout, hết hạn, hủy chủ
động, công cụ hỗ trợ và tác vụ sửa lỗi. Chỉ một câu SQL quản trị bỏ qua trạng thái
cũng có thể hoàn kho một khoản giữ đã xác nhận.

## 18. Dữ liệu quan sát cần có

- số lần xác nhận thắng, hết hạn thắng và xác nhận đến sau hạn;
- số dòng tác vụ bỏ qua vì đang khóa;
- tuổi lớn nhất của khoản giữ `RESERVED` đã quá hạn;
- thời gian chờ khóa, số `55P03`, `40P01` và số lần thử lại;
- số lần bất biến bộ đếm không khớp với tổng khoản giữ;
- số đơn tham chiếu khoản giữ không ở `CONFIRMED`;
- số lần phát lại checkout và dùng lại khóa với nội dung khác;
- độ dài lô cùng thời gian giao dịch tác vụ.

Không dùng `reservationId` làm nhãn số liệu có số lượng giá trị không giới hạn.
Mã này chỉ nên nằm trong nhật ký có kiểm soát hoặc dấu vết truy vết.
