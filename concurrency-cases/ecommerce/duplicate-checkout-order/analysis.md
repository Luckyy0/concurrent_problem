# Phân tích tranh chấp khi tạo đơn hàng

## 1. Trạng thái ban đầu

Hai yêu cầu A và B mang cùng dữ liệu:

```text
customer_id      = C-17
idempotency_key  = CHECKOUT-9001
fingerprint      = sha256(v1 | C-17 | CART-8 | version-12 | QUOTE-5 | ADDR-3)
```

Trước khi chạy:

```text
checkout_request = 0 dòng
purchase_order   = 0 dòng
outbox_event     = 0 dòng khởi tạo thanh toán
```

A và B dùng hai kết nối, hai giao dịch và có thể chạy trên hai máy chủ ứng dụng.

## 2. Kết quả mong đợi và kết quả thực tế

| Nội dung | Mong đợi | Thiết kế bị lỗi |
| --- | --- | --- |
| Số đơn | `1` | `2` |
| Lệnh thanh toán logic | `1` | `2` |
| Phản hồi A, B | Cùng một `order_id` | Hai `order_id` khác nhau hoặc một bên nhận lỗi |
| Dùng lại khóa với nội dung khác | Bị từ chối | Có thể trả nhầm đơn hoặc tạo đơn mới |
| Mất phản hồi sau chốt | Tra cứu lại kết quả cũ | Thử lại tạo thêm đơn |

## 3. Dòng thời gian của `check → insert`

Giả sử chưa có ràng buộc duy nhất theo khóa checkout:

| Bước | Giao dịch A | Giao dịch B | PostgreSQL |
| --- | --- | --- | --- |
| 1 | `BEGIN` | | Chưa có khóa nghiệp vụ |
| 2 | `SELECT exists` | | Ảnh chụp của câu lệnh không thấy dòng nào |
| 3 | | `BEGIN`; `SELECT exists` | Ảnh chụp khác cũng không thấy dòng nào |
| 4 | Chuẩn bị đơn `O-A` | | Chưa có khóa trên khoảng trống |
| 5 | | Chuẩn bị đơn `O-B` | Chưa có xung đột vật lý |
| 6 | `INSERT O-A` | | Khóa chính ngẫu nhiên không trùng |
| 7 | | `INSERT O-B` | Khóa chính ngẫu nhiên khác nên vẫn hợp lệ |
| 8 | `COMMIT` | `COMMIT` | Hai dòng cùng bền vững |

Nguyên nhân chính xác là chuỗi `SELECT → quyết định → INSERT` không nguyên tử.
Việc hai yêu cầu “chạy cùng lúc” chỉ là điều kiện làm lộ khoảng hở đó.

## 4. Vì sao ảnh chụp MVCC không bảo vệ khóa chưa tồn tại

PostgreSQL dùng cơ chế nhiều phiên bản dữ liệu (MVCC). Ở `READ COMMITTED`, mỗi
câu lệnh đọc có một ảnh chụp các dòng đã chốt trước khi câu lệnh bắt đầu.

Một câu `SELECT` thông thường không tạo lời hứa rằng kết quả “không có dòng” sẽ
vẫn đúng tới cuối giao dịch. Nó cũng không khóa được mọi khóa nghiệp vụ có thể
được chèn trong tương lai. Vì vậy cả A và B đều có thể quan sát cùng một khoảng
trống hợp lệ tại thời điểm đọc.

Tăng mức cô lập có thể thay đổi loại xung đột, nhưng ràng buộc duy nhất vẫn là
cách diễn đạt trực tiếp nhất cho quy tắc “tối đa một dòng trên một khóa”.

## 5. Chỉ mục duy nhất phân xử như thế nào

Khi có `UNIQUE (customer_id, idempotency_key)`, hai lệnh `INSERT` không còn độc
lập. Chúng cùng nhắm tới một giá trị trong chỉ mục duy nhất.

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | `INSERT ... ON CONFLICT DO NOTHING` trả `request_id` | |
| 2 | Tạo đơn, ghi outbox, lưu phản hồi | `INSERT` cùng khóa bắt đầu và chờ A |
| 3a | `COMMIT` | Câu `INSERT` hoàn tất nhưng không chèn dòng |
| 4a | | `SELECT` kế tiếp thấy bản ghi đã chốt của A |

PostgreSQL có thể phải chờ kết quả của giao dịch đang giữ giá trị chỉ mục trước
khi quyết định xung đột. `DO NOTHING` tránh biến xung đột dự kiến thành lỗi
`23505`; nó không có nghĩa câu lệnh luôn hoàn tất ngay.

Ở `READ COMMITTED`, câu `INSERT` và câu `SELECT` kế tiếp dùng hai ảnh chụp câu
lệnh khác nhau. Đây là lý do bên thua có thể đọc kết quả mà bên thắng vừa chốt.
Nếu đổi sang `REPEATABLE READ`, không được giữ nguyên giả định này mà không kiểm
thử; giao dịch có thể không thấy dòng theo ảnh chụp cũ hoặc nhận lỗi tuần tự hóa.

## 6. Khi bên thắng hoàn tác

Nhánh khác của cùng dòng thời gian:

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | Chèn quyền xử lý | |
| 2 | Gặp lỗi khi tạo đơn hoặc ghi outbox | Chờ trên giá trị chỉ mục |
| 3 | `ROLLBACK` | Được đánh thức |
| 4 | | Chèn quyền xử lý thành công |
| 5 | | Tạo đơn, ghi outbox, lưu phản hồi; `COMMIT` |

Vì quyền xử lý và dữ liệu nghiệp vụ cùng một giao dịch, hoàn tác không để lại
một dòng `PROCESSING` bền vững. B trở thành bên thắng hợp lệ, không phải chiếm
cướp một khóa còn chủ sở hữu.

## 7. Mất phản hồi quanh thời điểm chốt

Phía gọi có thể nhận lỗi mạng trong ba thời điểm khác nhau:

### Trước khi PostgreSQL nhận giao dịch

Không có dữ liệu. Gửi lại cùng khóa sẽ chạy như yêu cầu mới.

### Giao dịch đã hoàn tác

Quyền xử lý, đơn và outbox đều biến mất. Gửi lại cùng khóa có thể thử lại.

### Giao dịch đã chốt nhưng phản hồi HTTP bị mất

Đây là **kết quả chốt chưa rõ** (`ambiguous commit outcome`). Phía gọi không được
tạo khóa mới hoặc đoán giao dịch thất bại. Lần gửi lại bằng khóa cũ sẽ không tạo
đơn; nó đọc và phát lại phản hồi đã lưu.

> **Nói ngắn gọn:** Khi không biết `COMMIT` đã thành công hay chưa, hãy hỏi lại
> bằng cùng khóa, không thực hiện một ý định mới.

## 8. Cùng khóa nhưng khác nội dung

Giả sử A dùng `CHECKOUT-9001` cho `CART-8`, còn B dùng cùng khóa cho `CART-9`.
Ràng buộc duy nhất chỉ cho biết khóa đã tồn tại; nó không biết hai nội dung có
cùng ý nghĩa hay không.

Bên thua phải đọc `request_fingerprint` ban đầu:

```text
fingerprint giống nhau → phát lại kết quả
fingerprint khác nhau  → từ chối IDEMPOTENCY_KEY_REUSED
```

Không được cập nhật dấu vân tay cũ bằng giá trị mới. Làm vậy sẽ thay đổi lịch sử
và có thể khiến lần gọi ban đầu nhận phản hồi của một checkout khác.

## 9. Trạng thái đang xử lý

Trong thiết kế một giao dịch, dòng `PROCESSING` chưa chốt không nhìn thấy được
như một dòng bình thường. Bên trùng thường chờ trên chỉ mục; sau khi A chốt, B
thấy `COMPLETED`.

Nếu hệ thống chốt quyền xử lý trong giao dịch thứ nhất rồi thực hiện nghiệp vụ ở
giao dịch khác, `PROCESSING` trở thành trạng thái bền vững có thể quan sát. Cách
đó cần thêm:

- mã chủ sở hữu và thời hạn thuê;
- quy tắc xác định thao tác nào có thể tiếp tục;
- cơ chế cứu hộ sau khi tiến trình sập;
- hàng đợi hoặc bộ điều phối tránh hai chủ cùng chạy;
- chính sách cho tác dụng phụ có kết quả chưa rõ.

Chỉ đổi trạng thái cũ sang `FAILED` theo một bộ hẹn giờ là chưa đủ. Tiến trình cũ
có thể còn chạy và tiếp tục ghi sau khi bị coi là hết hạn; tài nguyên đích phải
có cách từ chối chủ sở hữu cũ.

## 10. Hibernate và thời điểm phát hiện xung đột

`JpaRepository.save()` không bắt buộc gửi `INSERT` ngay. Hibernate có thể trì
hoãn câu SQL tới `flush()` hoặc `COMMIT`. Vì vậy:

- lỗi duy nhất có thể xuất hiện sau khi phương thức dịch vụ gần kết thúc;
- lời gọi mạng đặt sau `save()` vẫn có thể chạy trước khi quyền xử lý được xác
  nhận ở PostgreSQL;
- bắt lỗi tại dòng `save()` không bảo đảm bắt được xung đột;
- sau SQLSTATE `23505`, giao dịch hiện tại không còn dùng được để đọc kết quả.

Giải pháp chính dùng JDBC với `ON CONFLICT DO NOTHING RETURNING`, nên kết quả
thắng/thua xuất hiện ngay tại câu lệnh chiếm quyền mà không làm hỏng giao dịch.

## 11. Kết quả bị từ chối bởi nghiệp vụ

Cần tách hai nhóm lỗi:

| Nhóm | Ví dụ | Cách lưu |
| --- | --- | --- |
| Lỗi đầu vào trước giao dịch | Khóa sai định dạng, không xác thực | Không chiếm quyền; trả lỗi trực tiếp |
| Kết quả nghiệp vụ cuối cùng | Giỏ đã hết hạn, báo giá không còn hợp lệ | Có thể lưu và phát lại như một kết quả hoàn tất |
| Lỗi kỹ thuật tạm thời | Bế tắc, hết thời gian chờ khóa, mất kết nối trước chốt | Hoàn tác; cho phép thử lại cùng khóa |

Nếu lưu kết quả từ chối, bản ghi `checkout_request` phải chứa mã HTTP và nội dung
ổn định. Nếu chọn không lưu, hợp đồng phải nói rõ lần thử lại có thể đánh giá lại
nghiệp vụ và cho kết quả khác.

## 12. Outbox và ranh giới thanh toán

Giao dịch checkout chỉ ghi một lệnh thanh toán logic vào `outbox_event`. Ràng
buộc duy nhất theo `(aggregate_type, aggregate_id, event_type)` ngăn hai dòng
outbox cho cùng đơn và loại lệnh.

Bộ phát outbox thường làm:

```text
đọc dòng chưa phát → gửi thông điệp → đánh dấu đã phát
```

Nếu tiến trình sập sau khi gửi nhưng trước khi đánh dấu, thông điệp sẽ được gửi
lại. Vì thế dịch vụ thanh toán vẫn phải nhận một mã lũy đẳng ổn định, thường là
`order_id` hoặc `event_id`. Outbox bảo đảm không mất ý định gửi; nó không tự tạo
giao hàng đúng một lần qua mạng.

## 13. Quan hệ với an toàn tồn kho

Ràng buộc khóa checkout trả lời câu hỏi:

```text
Ý định này đã được xử lý chưa?
```

Câu cập nhật tồn kho có điều kiện của ECOM-001 trả lời câu hỏi:

```text
Tại thời điểm ghi, còn đủ hàng để giữ không?
```

Hệ thống thật cần cả hai nếu checkout giữ hàng. Chỉ có khóa lũy đẳng không ngăn
hai khách hàng khác nhau cùng mua vượt tồn kho. Chỉ có cập nhật tồn kho an toàn
không ngăn một khách hàng gửi lại cùng ý định và giữ hàng hai lần bằng hai đơn.

## 14. Thử lại, hết thời gian chờ và bế tắc

- Lần thử lại phải giữ nguyên khóa và dấu vân tay.
- Mỗi lần thử kỹ thuật mở một giao dịch mới; không tái sử dụng giao dịch đã hoàn
  tác.
- SQLSTATE `55P03`, `40P01` hoặc lỗi kết nối không được ánh xạ thành “đơn đã tồn
  tại”.
- Thử lại phải có giới hạn, khoảng lùi và nằm trong thời hạn phản hồi tổng.
- Nếu hết thời gian chờ khi đang tranh chấp chỉ mục, không biết bên kia sẽ chốt
  hay hoàn tác; lần gọi sau phải tra cứu bằng cùng khóa.
- Không thử lại bên trong giao dịch đang giữ thêm khóa của giỏ hàng hoặc tồn kho;
  điều đó kéo dài tranh chấp và tăng nguy cơ bế tắc.

## 15. Chạy trên nhiều máy chủ

Hai máy chủ không chia sẻ khóa Java, bộ nhớ đệm hoặc trạng thái `in-flight`.
Chúng chỉ cùng nhìn thấy PostgreSQL. Vì vậy lớp bảo vệ có thẩm quyền phải nằm ở
ràng buộc duy nhất và giao dịch cơ sở dữ liệu.

Một khóa phân tán bên ngoài không cần thiết cho bất biến nằm hoàn toàn trong một
cơ sở dữ liệu. Nó thêm thời hạn thuê, lỗi mạng và trường hợp chủ cũ tiếp tục chạy
mà vẫn phải giữ ràng buộc cuối cùng tại PostgreSQL.

## 16. Nguyên nhân gốc theo tầng

| Tầng | Vấn đề |
| --- | --- |
| API | Không buộc lần thử lại dùng cùng khóa hoặc không định nghĩa phát lại phản hồi |
| Ứng dụng | Dùng `exists → insert`, không đối chiếu dấu vân tay |
| Spring | Hiểu nhầm `@Transactional` là khóa loại trừ giữa các yêu cầu |
| Hibernate | `save()` trì hoãn SQL, làm xung đột xuất hiện muộn |
| PostgreSQL | Lược đồ thiếu ràng buộc duy nhất trên khóa nghiệp vụ |
| Tích hợp | Gọi thanh toán trong giao dịch hoặc không dùng mã lũy đẳng ở phía nhận |

## 17. Dữ liệu cần theo dõi

Nên đo riêng các sự kiện:

```text
checkout.claim.created
checkout.claim.duplicate
checkout.response.replayed
checkout.fingerprint.mismatch
checkout.claim.wait_duration
checkout.transaction.rollback
checkout.commit_outcome_unknown
checkout.outbox.pending_age
checkout.outbox.redelivery
```

Theo dõi số đơn trên mỗi khóa nghiệp vụ bằng truy vấn đối soát. Chỉ số trùng tăng
có thể là hành vi thử lại bình thường; số đơn trên một khóa lớn hơn `1` mới là vi
phạm bất biến. Không gắn khóa thô làm nhãn metric vì sẽ tạo số lượng chuỗi thời
gian quá lớn và có thể lộ dữ liệu.

## 18. Phạm vi phân tích

Phân tích này giả định bảng mã yêu cầu, đơn hàng và outbox nằm trong cùng một
PostgreSQL. Khi các dữ liệu nằm ở nhiều dịch vụ, không có một giao dịch cục bộ
bao trùm tất cả; cần saga, inbox/outbox và quy tắc bù trừ riêng.

An toàn số lượng tồn kho thuộc ECOM-001. Tính lũy đẳng của thao tác thu tiền tại
dịch vụ thanh toán thuộc BANK-005.
