# Phân tích tranh chấp giữa các yêu cầu hoàn tiền

## 1. Trạng thái ban đầu

Giả sử giao dịch `CH-1` có dữ liệu:

```text
captured_amount          = 1.000.000
allocated_refund_amount  = 0
completed_refund_amount  = 0
currency                 = VND
```

Hai lệnh độc lập đến gần như cùng lúc:

```text
RF-A: hoàn 700.000, khóa lũy đẳng KEY-A
RF-B: hoàn 600.000, khóa lũy đẳng KEY-B
```

Mỗi lệnh riêng lẻ đều hợp lệ. Chỉ tổng của chúng vượt số tiền đã thu.

## 2. Chuỗi thao tác không nguyên tử

Phiên bản lỗi thường thực hiện:

```text
SELECT số tiền đã hoàn
→ tính số tiền còn lại trong Java
→ kiểm tra đủ hạn mức
→ INSERT refund
→ UPDATE payment_charge bằng giá trị tuyệt đối
→ gọi nhà cung cấp
```

Quyết định “còn đủ tiền” và hành động “dành phần tiền đó” nằm ở hai câu lệnh
khác nhau. Không có thao tác nào làm điểm phân xử chung cho mọi kết nối.

## 3. Dòng thời gian của hai yêu cầu khác nhau

| Thời điểm | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| t1 | Đọc `allocated = 0` | |
| t2 | | Đọc `allocated = 0` |
| t3 | Thấy còn `1.000.000`, chấp nhận `700.000` | |
| t4 | | Thấy còn `1.000.000`, chấp nhận `600.000` |
| t5 | Tạo refund A | Tạo refund B |
| t6 | Ghi bản sao thành `700.000` | |
| t7 | | Ghi bản sao thành `600.000` |
| t8 | Chốt | Chốt |

Kết quả thực tế có thể là:

```text
tổng refund được tạo       = 1.300.000
giá trị trên payment_charge = 600.000 hoặc 700.000
```

Đây là hai lỗi cùng xuất hiện: vượt hạn mức và mất cập nhật trên bảng tổng hợp.

## 4. Dòng thời gian của cùng một yêu cầu gửi lại

Hai lần gọi đều mang `KEY-A` và cùng dấu vân tay:

| Thời điểm | Lần gọi A | Lần gửi lại B |
| --- | --- | --- |
| t1 | Kiểm tra khóa, chưa có | |
| t2 | | Kiểm tra khóa, chưa có |
| t3 | Dành `700.000` | |
| t4 | | Dành thêm `700.000` hoặc chờ khóa dòng charge |
| t5 | Tạo refund R1 | Tạo refund R2 hoặc bị lỗi muộn |

Điều kiện hạn mức có thể ngăn lần thứ hai nếu tiền còn lại không đủ, nhưng như
vậy vẫn chưa phải tính lũy đẳng: lần gửi lại phải nhận đúng kết quả R1, không
phải một lỗi hạn mức mới. Ngược lại, khóa lũy đẳng không ngăn hai khóa khác nhau
cùng vượt hạn mức.

## 5. Kết quả mong đợi và kết quả thực tế

| Quy tắc | Kết quả mong đợi | Phiên bản lỗi |
| --- | --- | --- |
| Không vượt số tiền đã thu | `allocated <= captured` | Có thể tạo tổng refund lớn hơn captured |
| Không mất cập nhật | Bảng tổng hợp bằng sổ lịch sử | Lần ghi sau đè lần ghi trước |
| Gửi lại cùng ý định | Phát lại cùng kết quả | Có thể tạo refund mới hoặc trả lỗi khác |
| Cùng khóa khác nội dung | Từ chối rõ ràng | Có thể nhầm là cùng yêu cầu |
| Một quyết định, một outbox | Cùng chốt hoặc cùng hoàn tác | Có thể thiếu một phía |
| Lỗi kỹ thuật | Hoàn tác và báo lỗi kỹ thuật | Có thể bị đổi thành `LIMIT_EXCEEDED` |

## 6. Lớp gây lỗi chính xác

Lỗi không nằm riêng ở Java, JPA hay mức cô lập. Lỗi nằm ở ranh giới quyết định:

- Java đọc dữ liệu nhưng không giữ quyền trên hạn mức;
- JPA ghi một giá trị tuyệt đối được tính từ ảnh chụp cũ;
- cơ sở dữ liệu không có điều kiện phòng thủ ngay tại câu ghi;
- kiểm tra khóa và chèn khóa là hai thao tác;
- lời gọi từ xa chen giữa quyết định nội bộ và lần chốt.

Ranh giới đúng phải kết hợp:

```text
ràng buộc duy nhất cho cùng một ý định
+ UPDATE có điều kiện cho các ý định khác nhau
+ transaction cho projection, refund, ledger và outbox
```

## 7. MVCC dưới `READ COMMITTED`

PostgreSQL dùng MVCC để các câu đọc không nhất thiết chặn câu ghi. Với
`READ COMMITTED`, mỗi câu lệnh thấy ảnh chụp dữ liệu đã chốt tại đầu câu lệnh.
Vì vậy, hai câu `SELECT` độc lập có thể cùng thấy `allocated = 0`.

Một câu `UPDATE` cùng nhắm tới một dòng thì khác. Bên đến sau phải chờ khóa dòng.
Sau khi bên trước chốt, PostgreSQL đánh giá lại điều kiện cập nhật trên phiên bản
mới nhất của dòng. Thuộc tính này cho phép câu cập nhật có điều kiện trở thành
điểm phân xử.

## 8. Dòng thời gian với cập nhật có điều kiện

Giả sử A giành khóa dòng trước:

| Thời điểm | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| t1 | Chiếm `KEY-A` | Chiếm `KEY-B` |
| t2 | `UPDATE +700.000` trả một dòng | |
| t3 | Giữ khóa `payment_charge` | Gửi `UPDATE +600.000`, phải chờ |
| t4 | Ghi refund, ledger, outbox rồi `COMMIT` | |
| t5 | | Kiểm tra lại `700.000 + 600.000 <= 1.000.000` |
| t6 | | Điều kiện sai, `UPDATE` trả không dòng |
| t7 | | Lưu `LIMIT_EXCEEDED`, rồi `COMMIT` |

A thắng không phải vì đến API trước, mà vì câu cập nhật của A giành được dòng và
chốt trước. Nếu B giành dòng trước, B có thể thắng và A bị từ chối.

## 9. Khi cả hai yêu cầu đều vừa hạn mức

Với hai yêu cầu `600.000` và `400.000`, bên đến sau vẫn chờ nhưng điều kiện được
kiểm tra lại thành:

```text
600.000 + 400.000 <= 1.000.000
```

Điều kiện đúng nên cả hai được chấp nhận. Đây là lý do không được dùng một khóa
lũy đẳng chung theo `charge_id`: hai ý định khác nhau phải được phép cùng thành
công khi tổng vẫn hợp lệ.

## 10. Cách khóa lũy đẳng phân xử lần gửi lại

Lệnh chiếm khóa:

```sql
INSERT INTO refund_request (...)
VALUES (..., 'PROCESSING', ...)
ON CONFLICT (merchant_id, idempotency_key) DO NOTHING;
```

Nếu hai giao dịch cùng chèn một khóa:

- PostgreSQL để một bên chèn trước;
- bên còn lại chờ kết quả của giao dịch đang giữ xung đột duy nhất;
- nếu bên trước chốt, bên sau nhận `0` dòng và đọc kết quả đã lưu;
- nếu bên trước hoàn tác, bên sau có thể chèn và trở thành chủ xử lý.

`PROCESSING` được tạo và đổi thành kết quả cuối trong cùng giao dịch. Nếu tiến
trình dừng trước `COMMIT`, bản ghi chưa chốt bị PostgreSQL hoàn tác; hệ thống
không để lại một quyền xử lý mồ côi chỉ vì tiến trình chết.

## 11. Vì sao phải kiểm tra dấu vân tay

Khóa do phía gọi cung cấp có thể bị dùng nhầm:

```text
KEY-A + CH-1 + 700.000 VND
KEY-A + CH-1 + 600.000 VND
```

Nếu chỉ phát lại theo khóa, lần thứ hai có thể nhận phản hồi của một lệnh khác
mà không biết nội dung bị sai. Dấu vân tay phải được tạo từ dữ liệu đã chuẩn hóa,
ít nhất gồm cửa hàng, giao dịch, số tiền, loại tiền và mục đích nghiệp vụ ổn định.

Cùng khóa nhưng dấu vân tay khác phải trả `IDEMPOTENCY_KEY_REUSED`; không được
thử xử lý như một yêu cầu mới.

## 12. Bảng tổng hợp và sổ lịch sử

`payment_charge` phục vụ quyết định nhanh vì một dòng có thể được khóa và cập
nhật nguyên tử. `refund_ledger_entry` phục vụ giải thích và xây dựng lại:

```text
allocated_refund_amount = SUM(allocation_delta)
completed_refund_amount = SUM(completion_delta)
```

Chỉ tính `SUM` trước mỗi yêu cầu không giải quyết tranh chấp; hai giao dịch vẫn
có thể cùng thấy một tổng cũ. Chỉ lưu bảng tổng hợp cũng không đủ vì không thể
giải thích khoản nào đã chiếm hoặc giải phóng hạn mức. Hai loại dữ liệu phải chốt
cùng nhau.

## 13. Điểm tuyến tính hóa

Mỗi loại quyết định có một điểm tạo thứ tự rõ ràng:

| Quyết định | Điểm tuyến tính hóa |
| --- | --- |
| Ai sở hữu cùng khóa lũy đẳng | `INSERT ... ON CONFLICT` |
| Yêu cầu khác nhau có giành được tiền | `UPDATE ... WHERE allocated + amount <= captured` |
| Hoàn tất khoản hoàn | `UPDATE refund ... WHERE status = 'PENDING_PROVIDER'` |
| Giải phóng khoản thất bại | Cùng câu chuyển trạng thái có điều kiện |

Các bước Java trước điểm này chỉ là chuẩn bị. Phản hồi chỉ được coi là bền vững
sau khi toàn bộ giao dịch đã `COMMIT`.

## 14. Chốt và hoàn tác

### Lỗi sau khi tăng số tiền đã dành

Nếu chèn refund, bút toán hoặc outbox thất bại, ngoại lệ phải làm hoàn tác cả
`allocated_refund_amount`. Khóa lũy đẳng cũng hoàn tác nên lần thử lại có thể
chiếm khóa và chạy lại từ đầu.

### Lỗi trước khi chốt kết quả từ chối

Nếu cập nhật hạn mức trả không dòng nhưng lưu kết quả `LIMIT_EXCEEDED` thất bại,
giao dịch hoàn tác. Lần thử lại được phép đánh giá lại bằng cùng khóa vì chưa có
kết quả bền vững.

### Tiến trình dừng trước `COMMIT`

PostgreSQL hủy giao dịch và nhả khóa. Không có refund, bút toán hay outbox nào
được xem là đã chốt.

### Mất phản hồi sau `COMMIT`

Phía gọi không thể suy ra giao dịch đã thất bại. Nó phải gửi lại cùng khóa và
cùng nội dung. Hệ thống đọc `refund_request` rồi phát lại kết quả, không tăng hạn
mức thêm lần nữa.

## 15. Nhà cung cấp thanh toán là ranh giới khác

PostgreSQL không thể hoàn tác một lời gọi mạng đã thành công. Vì vậy giao dịch
tiếp nhận chỉ ghi `outbox_event`. Bộ phát outbox gửi sau khi chốt và dùng
`refund_id` ổn định làm mã chống lặp ở nhà cung cấp nếu API hỗ trợ.

Nếu nhà cung cấp trả kết quả không rõ, không được tự động giải phóng ngay. Cần
tra cứu hoặc áp dụng quy trình xử lý trạng thái chưa rõ; nếu giải phóng sớm rồi
nhà cung cấp thực sự đã hoàn, yêu cầu khác có thể dùng lại cùng hạn mức.

## 16. Hoàn tất và giải phóng

Một kết quả đến cho refund `R1` trước hết phải giành chuyển trạng thái:

```sql
UPDATE refund
SET status = 'FAILED_RELEASED'
WHERE refund_id = :refundId
  AND status = 'PENDING_PROVIDER'
RETURNING charge_id, amount;
```

Chỉ giao dịch nhận được một dòng mới được giảm `allocated_refund_amount` và thêm
bút toán giải phóng. Nếu bước sau lỗi, toàn bộ giao dịch hoàn tác, bao gồm trạng
thái. Nhánh thành công áp dụng cùng nguyên tắc để tăng
`completed_refund_amount` đúng một lần.

Trường hợp callback thành công, thất bại, đảo ngược và tra cứu đến sai thứ tự cần
máy trạng thái chi tiết hơn; đó là phạm vi của BANK-006.

## 17. Hết thời gian chờ, bế tắc và thử lại

Các lỗi PostgreSQL như hết thời gian chờ khóa, bế tắc hoặc lỗi tuần tự hóa là lỗi
kỹ thuật. Không ánh xạ chúng thành `LIMIT_EXCEEDED`, vì câu lệnh chưa đưa ra
quyết định nghiệp vụ đáng tin cậy.

Nếu thử lại:

- điều phối thử lại phải nằm ngoài giao dịch đã thất bại;
- mỗi lần dùng một giao dịch và persistence context mới;
- giữ nguyên khóa lũy đẳng và dấu vân tay;
- giới hạn số lần và dùng khoảng lùi có nhiễu;
- sau lỗi chưa rõ, ưu tiên tra cứu kết quả trước khi làm lại.

## 18. Thứ tự khóa và bế tắc

Đường tiếp nhận thường khóa `refund_request` rồi `payment_charge`. Đường hoàn tất
hoặc giải phóng khóa `refund` rồi `payment_charge`. Mỗi đường phải giữ thứ tự ổn
định trong cùng loại xử lý.

Nếu một nghiệp vụ chạm nhiều giao dịch thanh toán, hãy sắp xếp `charge_id` trước
khi cập nhật. Không giữ khóa trong lúc gọi mạng, ghi log chậm hoặc thực hiện công
việc không liên quan.

## 19. Nhiều máy chủ

Hai JVM không chia sẻ khóa Java. Chúng vẫn cùng đi qua ràng buộc duy nhất và khóa
dòng PostgreSQL, nên kết quả không phụ thuộc máy chủ nào nhận yêu cầu. Một khóa
phân tán không thay thế được điều kiện hạn mức, sổ lịch sử hay ràng buộc duy nhất.

Hàng đợi có thể giảm tải và tạo thứ tự theo `charge_id`, nhưng cơ sở dữ liệu vẫn
cần bảo vệ bất biến vì thông điệp có thể được giao lại hoặc có nhiều consumer.

## 20. Đối soát

Hai kiểm tra quan trọng:

```sql
SELECT c.charge_id,
       c.allocated_refund_amount,
       COALESCE(SUM(e.allocation_delta), 0) AS ledger_allocated,
       c.completed_refund_amount,
       COALESCE(SUM(e.completion_delta), 0) AS ledger_completed
FROM payment_charge c
LEFT JOIN refund_ledger_entry e ON e.charge_id = c.charge_id
GROUP BY c.charge_id,
         c.allocated_refund_amount,
         c.completed_refund_amount
HAVING c.allocated_refund_amount <> COALESCE(SUM(e.allocation_delta), 0)
    OR c.completed_refund_amount <> COALESCE(SUM(e.completion_delta), 0);
```

```sql
SELECT charge_id
FROM payment_charge
WHERE allocated_refund_amount < 0
   OR completed_refund_amount < 0
   OR completed_refund_amount > allocated_refund_amount
   OR allocated_refund_amount > captured_amount;
```

Kết quả phải rỗng. Đối soát giúp phát hiện và phục hồi; nó không thay thế cơ chế
phòng ngừa trên đường ghi.

## 21. Dữ liệu quan sát cần có

Nên ghi nhận theo mã tương quan, không đưa khóa thô vào nhãn metric:

```text
refund.request.accepted
refund.request.replayed
refund.request.limit_exceeded
refund.request.fingerprint_mismatch
refund.capacity.conditional_update_zero
refund.provider.pending_age
refund.transition.duplicate
refund.ledger.reconciliation_mismatch
refund.database.lock_wait
refund.database.deadlock
```

Log điều tra nên liên kết `refund_id`, `charge_id`, loại kết quả, SQLSTATE và mã
outbox. Dữ liệu nhạy cảm phải được che hoặc kiểm soát theo chính sách vận hành.
