# ECOM-003 — Tạo đơn hàng trùng khi checkout đồng thời

## 1. Bài toán

Khách hàng nhấn nút checkout hai lần vì giao diện phản hồi chậm. Cùng lúc đó,
cổng API tự gửi lại yêu cầu do lần gọi đầu hết thời gian chờ. Hai yêu cầu mang
cùng một ý định mua hàng có thể được chuyển tới hai máy chủ ứng dụng khác nhau.

Nếu mỗi máy chủ chỉ kiểm tra “đơn này đã tồn tại chưa?” rồi mới tạo đơn, cả hai
có thể cùng nhận câu trả lời “chưa”. Kết quả là một lần checkout tạo hai đơn và
hai lệnh khởi tạo thanh toán.

Chuỗi thao tác gây lỗi là:

```text
kiểm tra khóa yêu cầu → thấy chưa có → tạo đơn → khởi tạo thanh toán
```

Đây là lỗi **xử lý trùng một lệnh checkout** (`duplicate checkout command`).
Nó khác với lỗi bán vượt tồn kho: một yêu cầu có thể chỉ chạy một lần nhưng vẫn
trừ kho sai; ngược lại, tồn kho có thể được trừ an toàn nhưng cùng một yêu cầu
vẫn tạo hai đơn.

> **Nói ngắn gọn:** Mã yêu cầu phải được chiếm quyền xử lý bằng một thao tác ghi
> nguyên tử trước khi tạo đơn, không được bảo vệ bằng bước đọc kiểm tra.

## 2. Tác nhân và dữ liệu dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Khách hàng | Gửi lại cùng một ý định checkout do nhấn hai lần hoặc không nhận được phản hồi |
| Cổng API | Có thể thử lại yêu cầu sau lỗi mạng hoặc hết thời gian chờ |
| Máy chủ A, B | Xử lý hai bản sao của yêu cầu trên hai giao dịch độc lập |
| `checkout_request` | Ghi nhận quyền xử lý và kết quả bền vững của một mã yêu cầu |
| `purchase_order` | Đơn hàng được tạo bởi yêu cầu đã thắng quyền xử lý |
| `outbox_event` | Lệnh bền vững để khởi tạo thanh toán sau khi giao dịch chốt |
| PostgreSQL | Nguồn dữ liệu có thẩm quyền để phân xử yêu cầu trùng |

Điểm tranh chấp là khóa nghiệp vụ `(customer_id, idempotency_key)` chưa tồn tại.
Không có dòng sẵn để `SELECT ... FOR UPDATE`; chính chỉ mục duy nhất của
PostgreSQL sẽ phân xử máy chủ nào được tạo khóa này.

## 3. Quy tắc bất biến

Hệ thống phải giữ đúng các quy tắc sau:

```text
Với mỗi (customer_id, idempotency_key):
  có tối đa một checkout_request;
  có tối đa một purchase_order;
  có tối đa một lệnh thanh toán logic trong outbox.

Cùng khóa + cùng dấu vân tay yêu cầu:
  mọi lần gọi nhận cùng kết quả đã lưu.

Cùng khóa + khác dấu vân tay yêu cầu:
  yêu cầu sau bị từ chối và không tạo thêm dữ liệu nghiệp vụ.
```

“Một lệnh thanh toán logic” không đồng nghĩa thông điệp chỉ được giao đúng một
lần. Bộ phát outbox vẫn có thể gửi lại; phía xử lý thanh toán phải áp dụng tính
lũy đẳng của `BANK-005`.

## 4. Khóa yêu cầu và dấu vân tay

Phía gọi tạo một `Idempotency-Key` mới cho một ý định checkout và giữ nguyên khóa
đó khi thử lại. Máy chủ luôn ghép khóa với phạm vi khách hàng hoặc đơn vị thuê
bao; không dùng khóa do phía gọi gửi như một khóa duy nhất trên toàn hệ thống.

Máy chủ tự tính **dấu vân tay yêu cầu** (`request fingerprint`) từ dữ liệu đã
chuẩn hóa, chẳng hạn:

```text
customer_id | cart_id | cart_version | quote_id | shipping_address_id
```

Không băm trực tiếp chuỗi JSON thô vì thứ tự thuộc tính và cách biểu diễn số có
thể thay đổi dù ý nghĩa không đổi. Cũng không tin một giá trị băm do phía gọi tự
khai báo. Phiên bản của thuật toán chuẩn hóa phải được quản lý khi hợp đồng thay
đổi.

## 5. Ranh giới giao dịch

Các lỗi cú pháp, xác thực và định dạng khóa được kiểm tra trước khi mở giao dịch.
Một lần checkout thành công sau đó dùng một giao dịch ngắn:

```text
BEGIN
  1. INSERT checkout_request ... ON CONFLICT DO NOTHING RETURNING request_id
  2. nếu thắng: kiểm tra dữ liệu nghiệp vụ cần thiết
  3. nếu thắng: tạo purchase_order và các dòng đơn hàng
  4. nếu thắng: ghi một outbox_event khởi tạo thanh toán
  5. nếu thắng: lưu mã phản hồi và nội dung phản hồi vào checkout_request
COMMIT
```

Nếu bước 2–5 thất bại bằng lỗi kỹ thuật, toàn bộ giao dịch hoàn tác, bao gồm cả
quyền xử lý vừa ghi. Một lần gọi lại cùng khóa có thể thử lại từ đầu.

Không gọi cổng thanh toán qua HTTP trong giao dịch này. Nếu lời gọi mạng thành
công nhưng giao dịch cơ sở dữ liệu hoàn tác, hệ thống có thể đã thu tiền mà
không có đơn. Nếu giao dịch chốt trước nhưng tiến trình sập trước lời gọi mạng,
đơn lại không được thanh toán. Outbox tách hai ranh giới và cho phép gửi lại an
toàn theo mã lệnh ổn định.

## 6. Cách phân xử được khuyến nghị

Bảng lưu mã yêu cầu có ràng buộc duy nhất:

```sql
CONSTRAINT uk_checkout_request_customer_key
    UNIQUE (customer_id, idempotency_key)
```

Ứng dụng xin quyền xử lý bằng một câu lệnh:

```sql
INSERT INTO checkout_request (
    request_id,
    customer_id,
    idempotency_key,
    request_fingerprint,
    status,
    created_at
)
VALUES (
    :requestId,
    :customerId,
    :idempotencyKey,
    :fingerprint,
    'PROCESSING',
    clock_timestamp()
)
ON CONFLICT (customer_id, idempotency_key) DO NOTHING
RETURNING request_id;
```

| Kết quả | Cách xử lý |
| --- | --- |
| Có `request_id` trả về | Yêu cầu này thắng quyền xử lý và được phép tạo đơn |
| Không có dòng trả về | Đọc bản ghi đã có, đối chiếu dấu vân tay và trạng thái |
| Hết thời gian chờ hoặc mất kết nối | Kết quả kỹ thuật chưa rõ; phía gọi phải thử lại bằng đúng khóa cũ |

Ở mức cô lập mặc định `READ COMMITTED`, câu `INSERT` của bên đến sau có thể chờ
giao dịch đang giữ cùng khóa. Nếu bên trước chốt, câu lệnh trả về rỗng; câu
`SELECT` kế tiếp dùng ảnh chụp mới và thấy kết quả đã chốt. Nếu bên trước hoàn
tác, bên đang chờ có thể trở thành bên thắng.

## 7. Hợp đồng phản hồi

| Tình huống | Phản hồi | Có tạo thêm đơn không? |
| --- | --- | --- |
| Khóa mới | Chạy checkout và lưu phản hồi | Có đúng một đơn nếu nghiệp vụ chấp nhận |
| Cùng khóa, cùng dấu vân tay, đã hoàn tất | Trả lại mã HTTP và nội dung đã lưu | Không |
| Cùng khóa, khác dấu vân tay | `409 IDEMPOTENCY_KEY_REUSED` | Không |
| Bản ghi đã chốt ở trạng thái đang xử lý | `409 REQUEST_IN_PROGRESS` hoặc `425` theo hợp đồng đã công bố | Không |
| Lỗi kỹ thuật trước `COMMIT` | Lỗi có thể thử lại bằng cùng khóa | Không còn thay đổi bền vững |
| Mất phản hồi sau `COMMIT` | Lần thử lại nhận phản hồi đã lưu | Không |

Thiết kế chính của case này không chốt riêng trạng thái `PROCESSING`: quyền xử
lý, đơn hàng, outbox và kết quả cùng nằm trong một giao dịch. Vì vậy, yêu cầu
đồng thời thường chờ rồi thấy `COMPLETED`, thay vì thấy trạng thái đang xử lý.
Trạng thái `PROCESSING` đã chốt chỉ xuất hiện nếu hệ thống chọn quy trình nhiều
giao dịch hoặc còn dữ liệu từ phiên bản cũ; khi đó phải có chính sách cứu hộ rõ
ràng, không được xóa khóa mù quáng.

## 8. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh hoặc API | Ý nghĩa |
| --- | --- | --- |
| khóa lũy đẳng | idempotency key | Mã ổn định để nhận ra các lần gửi lại của cùng một ý định |
| khóa nghiệp vụ duy nhất | unique business key | Bộ cột mà PostgreSQL cấm xuất hiện hai lần |
| dấu vân tay yêu cầu | request fingerprint | Mã băm do máy chủ tính để phát hiện cùng khóa nhưng khác nội dung |
| chiếm quyền nguyên tử | atomic claim | Một thao tác ghi duy nhất quyết định yêu cầu nào được xử lý |
| phát lại phản hồi | response replay | Trả kết quả đã lưu mà không chạy lại nghiệp vụ |
| kết quả chốt chưa rõ | ambiguous commit outcome | Phía gọi mất phản hồi và không biết giao dịch đã chốt hay chưa |
| hộp thư đi bền vững | transactional outbox | Bảng lưu lệnh gửi ra ngoài trong cùng giao dịch với dữ liệu nghiệp vụ |

## 9. Chạy trên nhiều máy chủ

`synchronized`, `ReentrantLock` hoặc một `ConcurrentHashMap` chỉ phối hợp các
luồng trong một JVM. Hai yêu cầu có thể đi vào hai máy chủ khác nhau, hoặc một
máy chủ có thể khởi động lại và mất toàn bộ trạng thái trong bộ nhớ.

Ràng buộc duy nhất nằm tại PostgreSQL nên mọi máy chủ cùng chịu một luật phân
xử. Khóa trong JVM có thể giảm số lần tranh chấp cục bộ, nhưng không được xem là
lớp bảo vệ quy tắc bất biến.

## 10. Hậu quả khi triển khai sai

- Khách hàng nhìn thấy nhiều đơn cho một lần mua.
- Tồn kho được giữ nhiều lần dù từng phép trừ kho riêng lẻ vẫn an toàn.
- Nhiều lệnh thanh toán được tạo, dẫn đến thu tiền trùng nếu hệ thống sau không
  có cơ chế chống lặp.
- Phiếu giảm giá, điểm thưởng hoặc hạn mức mua bị tiêu thụ nhiều lần.
- Nhân viên phải xác định đơn chuẩn để hủy và hoàn tiền thủ công.
- Mất phản hồi quanh `COMMIT` biến lần thử lại hợp lý thành một đơn mới.
- Ghi log toàn bộ khóa hoặc nội dung checkout có thể làm lộ dữ liệu nhạy cảm.

## 11. Điều hướng tài liệu

- [Mã nguồn gây tạo đơn trùng](broken-code.md)
- [Phân tích ảnh chụp MVCC và các tình huống lỗi](analysis.md)
- [Thiết kế Java và SQL an toàn](solutions.md)
- [Thực nghiệm đồng thời với PostgreSQL](experiments.md)
- [DB-006 — Ràng buộc duy nhất khi ghi đồng thời](../../postgresql/unique-constraint-concurrency/README.md)
- [BANK-005 — Khởi tạo thanh toán lũy đẳng](../../banking/idempotent-payment-creation/README.md)
- [Tính lũy đẳng và tính duy nhất](../../concepts/idempotency-and-uniqueness.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)
- [ECOM-001 — Bán vượt tồn kho](../overselling-inventory/README.md)

## 12. Khi nên dùng cách này

Dùng bảng mã yêu cầu riêng khi API cần phát lại kết quả, phát hiện việc dùng lại
khóa với nội dung khác và theo dõi vòng đời của yêu cầu. Nếu chỉ cần bảo đảm một
khóa nghiệp vụ tạo tối đa một dòng, ràng buộc duy nhất trên bảng đơn hàng có thể
đủ; nhưng nó không tự định nghĩa phản hồi cho lần gửi lại.

Giữ toàn bộ phần ghi nội bộ trong một giao dịch khi các bảng cùng nằm trong một
cơ sở dữ liệu và công việc đủ ngắn. Nếu quy trình phải gọi nhiều dịch vụ hoặc kéo
dài, cần một luồng công việc bền vững với trạng thái, quyền sở hữu có thời hạn và
cơ chế cứu hộ; đó là một thiết kế khác, không chỉ là thêm `@Transactional`.

## 13. Phạm vi

Case này bảo vệ một lệnh checkout khỏi tạo nhiều đơn và nhiều lệnh thanh toán
logic. Việc trừ tồn kho an toàn vẫn thuộc `ECOM-001`; ECOM-003 phải gọi cơ chế đó
nếu checkout đồng thời giữ hàng. Việc bảo đảm phía thanh toán không thu tiền hai
lần thuộc `BANK-005`. Trình tự chuyển trạng thái đơn, hoàn tiền và quy trình
phân tán nhiều dịch vụ không nằm trong phạm vi case này.
