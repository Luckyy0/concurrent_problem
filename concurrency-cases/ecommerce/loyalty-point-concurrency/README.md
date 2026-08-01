# ECOM-005 — Chi tiêu điểm thưởng đồng thời

## 1. Bài toán

Khách hàng `C-17` có `1.000` điểm. Hai đơn hàng khác nhau cùng yêu cầu dùng
`800` điểm và được xử lý đồng thời trên hai máy chủ.

Nếu mỗi giao dịch đọc số dư, kiểm tra trong Java rồi ghi lại giá trị mới, cả hai
có thể cùng thấy `1.000`, cùng kết luận đủ điểm và cùng chấp nhận. Hai lịch sử
trừ `800` được tạo, nhưng hai lần ghi đè cùng lưu số dư `200`.

```text
số dư ban đầu        =  1.000
tổng bút toán mới    = -1.600
số dư đúng theo sổ   =   -600  ← vi phạm
số dư bị ghi đè      =    200  ← nhìn hợp lệ nhưng sai lịch sử
```

Ngoài tiêu hai lần, cộng điểm và trừ điểm đồng thời cũng có thể ghi đè nhau. Ví
dụ `+500` và `-300` từ số dư `1.000` phải cho kết quả `1.200`, không phải `700`
hoặc `1.500`.

> **Nói ngắn gọn:** Số dư không được tính từ một ảnh chụp cũ trong Java. Phép
> cộng/trừ, bút toán và kết quả lệnh phải cùng chốt trong PostgreSQL.

## 2. Tác nhân và dữ liệu dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Đơn hàng A | Tiêu `800` điểm bằng lệnh `CMD-A` |
| Đơn hàng B | Tiêu `800` điểm bằng lệnh `CMD-B` |
| `loyalty_account` | Giữ số dư điểm hiện tại để quyết định nhanh |
| `loyalty_ledger_entry` | Sổ cái chỉ thêm mới của mọi lần cộng/trừ điểm |
| `loyalty_command` | Chiếm mã lệnh, lưu dấu vân tay và kết quả để phát lại |
| Máy chủ A, B | Chạy hai giao dịch và kết nối riêng |
| PostgreSQL | Nguồn dữ liệu có thẩm quyền đối với số dư và lịch sử |

Điểm tranh chấp là dòng `loyalty_account` của khách hàng `C-17`. Cả hai lệnh là
ý định hợp lệ và khác nhau; chống gửi trùng không được phép gộp chúng thành một.

## 3. Quy tắc bất biến

```text
loyalty_account.points_balance >= 0

loyalty_account.points_balance
    = tổng points_delta của mọi loyalty_ledger_entry đã chốt

Mỗi (customer_id, command_id) có tối đa một kết quả lệnh.

Mỗi lệnh thành công có đúng một bút toán.
Mỗi lệnh bị từ chối vì thiếu điểm không có bút toán.
```

Sổ cái là lịch sử có thể kiểm toán; bảng số dư là dữ liệu tổng hợp dùng trong
đường ghi. Hai bảng cùng nằm trong một giao dịch để số dư không đi trước hoặc đi
sau lịch sử.

## 4. Ranh giới giao dịch khi tiêu điểm

```text
BEGIN
  1. INSERT loyalty_command ... ON CONFLICT DO NOTHING RETURNING command_id
  2. nếu là lệnh mới: trừ điểm bằng UPDATE có điều kiện
  3a. đủ điểm: INSERT bút toán âm, lưu kết quả SUCCEEDED
  3b. thiếu điểm: lưu kết quả REJECTED, không thêm bút toán
COMMIT
```

Nếu chèn bút toán hoặc lưu kết quả thất bại, phép trừ điểm phải hoàn tác. Không
gọi dịch vụ đơn hàng, gửi thông điệp hoặc thực hiện I/O từ xa trong lúc giao dịch
đang giữ khóa dòng số dư.

Kết quả thiếu điểm có thể được chốt vào `loyalty_command`. Lần gửi lại cùng mã sẽ
nhận đúng kết quả từ chối cũ, ngay cả khi khách hàng đã kiếm thêm điểm sau đó.
Muốn thực hiện một ý định mới, phía gọi phải dùng mã lệnh mới.

## 5. Phép trừ có thẩm quyền

```sql
UPDATE loyalty_account
SET points_balance = points_balance - :points,
    revision = revision + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE customer_id = :customerId
  AND :points > 0
  AND points_balance >= :points
RETURNING points_balance, revision;
```

| Kết quả | Ý nghĩa |
| --- | --- |
| Có một dòng | Điểm đã được giữ/trừ trong giao dịch hiện tại |
| Không có dòng | Tài khoản không tồn tại hoặc không đủ điểm |
| Lỗi chờ khóa/bế tắc | Lỗi kỹ thuật; giao dịch phải hoàn tác |

Khi hai giao dịch cùng tiêu `800` từ `1.000`, bên đến trước khóa dòng và giảm còn
`200`. Bên sau chờ; sau khi bên trước chốt, PostgreSQL kiểm tra lại điều kiện
`200 >= 800`, nhận sai và trả `0` dòng.

## 6. Phép cộng không làm mất thay đổi

```sql
UPDATE loyalty_account
SET points_balance = points_balance + :points,
    lifetime_earned = lifetime_earned + :points,
    revision = revision + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE customer_id = :customerId
  AND :points > 0
RETURNING points_balance, revision;
```

Cộng và trừ đều dùng giá trị hiện tại trong PostgreSQL. Bên đến sau tính trên kết
quả đã chốt của bên trước nên không ghi đè chênh lệch.

## 7. Sổ cái chỉ thêm mới

Một bút toán thành công ghi:

```text
entry_id
customer_id
command_id
entry_type: EARN | REDEEM | REVERSAL | ADJUSTMENT
points_delta: số dương hoặc âm
balance_after
account_sequence
created_at
```

Không sửa hoặc xóa bút toán cũ để hoàn điểm. Một lần hoàn trả phải tạo bút toán
dương mới, liên kết tới bút toán bị đảo và có mã lệnh riêng. Điều này giữ lịch sử
đầy đủ để đối soát.

Điểm thưởng là một quyền lợi của chương trình, không phải tiền được thanh toán.
Mô hình một phía trong case này không thay thế sổ cái bút toán kép, quyết toán và
yêu cầu pháp lý của hệ thống tiền thật.

## 8. Hợp đồng kết quả

| Tình huống | Kết quả | Thay đổi bền vững |
| --- | --- | --- |
| Lệnh mới, đủ điểm | `REDEEMED` | Trừ số dư, thêm một bút toán, lưu phản hồi |
| Lệnh mới, thiếu điểm | `INSUFFICIENT_POINTS` | Chỉ lưu kết quả từ chối |
| Cùng mã, cùng dấu vân tay | Phát lại kết quả cũ | Không cộng/trừ thêm |
| Cùng mã, khác dấu vân tay | `IDEMPOTENCY_MISMATCH` | Không thay đổi số dư |
| Lỗi trước `COMMIT` | Lỗi kỹ thuật | Toàn giao dịch hoàn tác |
| Mất phản hồi sau `COMMIT` | Kết quả chốt chưa rõ | Tra cứu bằng cùng mã lệnh |

## 9. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh hoặc API | Ý nghĩa |
| --- | --- | --- |
| sổ cái điểm | loyalty ledger | Lịch sử chỉ thêm mới của mọi thay đổi điểm |
| bút toán | ledger entry / posting | Một dòng tăng hoặc giảm điểm có tham chiếu nghiệp vụ |
| bảng chiếu số dư | balance projection | Dòng số dư hiện tại được tổng hợp từ sổ cái |
| tiêu điểm hai lần | double spending | Hai lệnh cùng được duyệt dựa trên một số dư cũ |
| cập nhật chênh lệch nguyên tử | atomic delta | Cộng/trừ trực tiếp trên giá trị hiện tại trong SQL |
| trừ có điều kiện | conditional debit | Chỉ giảm số dư khi điều kiện đủ điểm còn đúng |
| bút toán bù | compensating entry | Bút toán mới đảo tác động cũ mà không sửa lịch sử |
| đối soát | reconciliation | So sánh số dư hiện tại với tổng các bút toán |

## 10. Chạy trên nhiều máy chủ

`synchronized` hoặc `ReentrantLock` chỉ phối hợp các luồng trong một JVM. Hai đơn
hàng có thể được xử lý trên hai máy chủ khác nhau, nên số dư phải được bảo vệ tại
PostgreSQL bằng câu cập nhật có điều kiện và ràng buộc duy nhất.

Khóa Java có thể giảm tranh chấp cục bộ nhưng không được xem là lớp bảo vệ bất
biến. Một khóa phân tán cũng không thay thế giao dịch giữa số dư, bút toán và kết
quả lệnh.

## 11. Hậu quả khi triển khai sai

- Khách hàng dùng nhiều điểm hơn số điểm đã kiếm.
- Bảng số dư vẫn dương nhưng tổng sổ cái đã âm.
- Lần cộng điểm hoặc trừ điểm bị ghi đè và biến mất khỏi số dư hiển thị.
- Một lần thử lại tạo thêm bút toán và trừ điểm lần nữa.
- Nhân viên sửa/xóa lịch sử làm mất khả năng kiểm toán.
- Đơn hàng đã hưởng giảm giá nhưng giao dịch điểm bị hoàn tác, hoặc ngược lại.
- Một tài khoản nóng làm nhiều giao dịch chờ khóa và chiếm vùng kết nối.
- Việc khắc phục phải xây dựng lại số dư từ lịch sử và xử lý các đơn sai.

## 12. Điều hướng tài liệu

- [Mã nguồn đọc–kiểm tra–ghi gây lỗi](broken-code.md)
- [Phân tích số dư, sổ cái và dòng thời gian](analysis.md)
- [Thiết kế Java và SQL an toàn](solutions.md)
- [Thực nghiệm đồng thời với PostgreSQL](experiments.md)
- [BANK-001 — Chi tiêu đồng thời vượt số dư](../../banking/concurrent-withdrawal-double-spend/README.md)
- [BANK-007 — Bút toán đồng thời và bảng chiếu](../../banking/ledger-posting-projection/README.md)
- [Sổ cái, số dư và khoản giữ chỗ](../../concepts/ledger-balances-and-holds.md)
- [Tính lũy đẳng và tính duy nhất](../../concepts/idempotency-and-uniqueness.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)
- [ECOM-003 — Tạo đơn hàng trùng](../duplicate-checkout-order/README.md)

## 13. Khi nên dùng cách này

Cập nhật có điều kiện phù hợp khi một dòng số dư là nguồn quyết định tức thời và
điều kiện đủ điểm có thể biểu diễn trong `WHERE`. Giao dịch ngắn này xử lý tốt cả
nhiều máy chủ mà không cần khóa ngoài cơ sở dữ liệu.

Dùng `FOR UPDATE` khi quyết định phải đọc nhiều thuộc tính phức tạp không thể đưa
vào một câu SQL. Dùng `@Version` khi xung đột hiếm và nghiệp vụ chấp nhận thử lại
toàn bộ. Dù chọn cách nào, sổ cái, tính duy nhất của mã lệnh và đối soát vẫn cần
được giữ.

## 14. Phạm vi

Case này xử lý điểm thưởng như quyền lợi có thể kiểm toán: cộng, tiêu và giữ số
dư không âm. Nó không mô tả quyết toán tiền, bút toán kép tài chính, tỷ giá hay
tuân thủ kế toán. Giới hạn coupon thuộc ECOM-004; tạo đơn trùng thuộc ECOM-003.
Hết hạn điểm, giữ điểm tạm thời và hoàn điểm đồng thời cần vòng đời trạng thái
riêng nhưng phải dùng cùng nguyên tắc sổ cái và cập nhật có điều kiện.
