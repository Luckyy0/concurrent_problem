# Sổ cái, số dư và khoản giữ chỗ

## 1. Mục đích

Tài liệu này giải thích cách lưu một quyền lợi có số lượng, chẳng hạn tiền, điểm
thưởng hoặc hạn mức, sao cho hệ thống vừa quyết định nhanh vừa giữ được lịch sử
để kiểm toán.

Hai loại dữ liệu thường xuất hiện cùng nhau:

```text
sổ cái:       ghi lại từng thay đổi đã xảy ra
bảng số dư:   giữ kết quả tổng hợp hiện tại để đọc và cập nhật nhanh
```

Nếu chỉ lưu số dư, hệ thống biết “còn bao nhiêu” nhưng khó trả lời “vì sao có con
số này”. Nếu chỉ lưu sổ cái, mỗi lần kiểm tra có thể phải tính tổng rất nhiều
dòng và vẫn cần giải quyết tranh chấp khi hai giao dịch cùng tiêu một số dư.

> **Nói ngắn gọn:** Sổ cái giữ bằng chứng; bảng số dư phục vụ quyết định. Hai
> phần phải được cập nhật trong cùng một ranh giới có thẩm quyền.

## 2. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh | Ý nghĩa |
| --- | --- | --- |
| sổ cái | ledger | Tập hợp các bút toán ghi lại mọi thay đổi giá trị |
| bút toán | ledger entry / posting | Một dòng chỉ thêm mới biểu diễn một thay đổi tăng hoặc giảm |
| chỉ thêm mới | append-only | Không sửa hoặc xóa lịch sử cũ; sai sót được điều chỉnh bằng bút toán mới |
| bảng chiếu số dư | balance projection | Dòng tổng hợp số dư hiện tại để truy vấn nhanh |
| số dư đã ghi nhận | posted balance | Tổng của các bút toán đã chốt |
| khoản giữ chỗ | hold / reservation | Phần giá trị tạm thời chưa chi trả nhưng không còn được dùng cho lệnh khác |
| số dư khả dụng | available balance | Phần có thể dùng sau khi trừ các khoản giữ chỗ và áp dụng quy tắc nghiệp vụ |
| cập nhật chênh lệch nguyên tử | atomic delta | Cộng hoặc trừ trực tiếp trên giá trị hiện tại bằng một câu SQL |
| bút toán bù | compensating entry | Bút toán mới đảo tác động của bút toán cũ mà không sửa lịch sử |
| đối soát | reconciliation | So sánh bảng số dư với tổng sổ cái và xử lý chênh lệch |

## 3. Sổ cái và bảng số dư có vai trò khác nhau

Ví dụ một tài khoản điểm có các bút toán:

```text
+1.000  OPENING
  -200  REDEEM
  +100  EARN
```

Tổng sổ cái là `900`. Bảng số dư cũng phải là `900`, nhưng nó không thay thế ba
dòng lịch sử trên.

Một thiết kế thông dụng:

```sql
CREATE TABLE balance_projection (
    account_id UUID PRIMARY KEY,
    posted_balance BIGINT NOT NULL,
    held_amount BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL,
    CHECK (posted_balance >= 0),
    CHECK (held_amount >= 0),
    CHECK (held_amount <= posted_balance)
);

CREATE TABLE ledger_entry (
    entry_id UUID PRIMARY KEY,
    account_id UUID NOT NULL,
    command_id UUID NOT NULL,
    entry_type VARCHAR(30) NOT NULL,
    amount_delta BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    UNIQUE (account_id, command_id),
    CHECK (amount_delta <> 0)
);
```

Tên cột và công thức thực tế phụ thuộc nghiệp vụ. Không sao chép lược đồ tiền tệ
cho điểm thưởng mà không xem xét đơn vị, thời hạn, hoàn trả và quy tắc pháp lý.

## 4. Số dư đã ghi nhận và số dư khả dụng

Trong mô hình đơn giản:

```text
available_balance = posted_balance - active_holds
```

Nhưng công thức thật có thể còn bao gồm hạn mức tín dụng, khoản bị phong tỏa, phí
đang chờ hoặc giá trị chỉ dùng cho một loại giao dịch. Vì vậy không nên dùng hai
tên `posted` và `available` như thể chúng luôn giống nhau.

Ví dụ:

```text
posted_balance = 1.000
active_holds   =   300
available      =   700
```

Một yêu cầu mới không được tiêu `800` dù sổ cái đã ghi nhận `1.000`. Nếu hệ
thống không có vòng đời giữ chỗ, có thể dùng một số dư duy nhất; tài liệu phải
nói rõ giả định đó.

## 5. Vì sao sổ cái phải chỉ thêm mới

Không sửa một bút toán cũ từ `-200` thành `-100` và không xóa nó để “hoàn tác”.
Thay vào đó, thêm một bút toán bù `+100` hoặc một bút toán đảo `+200`, tùy quy
tắc nghiệp vụ.

Cách này giữ được:

- ai tạo thay đổi;
- thay đổi xảy ra khi nào;
- lệnh nghiệp vụ nào là nguyên nhân;
- bút toán nào được điều chỉnh;
- số dư được hình thành qua những bước nào.

Ứng dụng chạy thật nên dùng tài khoản cơ sở dữ liệu chỉ có quyền `SELECT` và
`INSERT` trên bảng sổ cái:

```sql
REVOKE UPDATE, DELETE ON ledger_entry FROM app_runtime;
GRANT SELECT, INSERT ON ledger_entry TO app_runtime;
```

Quyền migration dùng tài khoản riêng. Trigger có thể tăng bảo vệ, nhưng phân
quyền rõ ràng và quy trình sửa dữ liệu vẫn cần thiết.

## 6. Cập nhật số dư bằng chênh lệch

Không đọc số dư lên Java, cộng/trừ rồi ghi một giá trị tuyệt đối:

```text
đọc 1.000 → tính 200 → UPDATE balance = 200
```

Hai giao dịch có thể cùng đọc `1.000` và cùng ghi `200`, làm mất một thay đổi.
Hãy cộng hoặc trừ ngay trong SQL:

```sql
UPDATE balance_projection
SET posted_balance = posted_balance + :delta,
    updated_at = CURRENT_TIMESTAMP
WHERE account_id = :accountId
RETURNING posted_balance;
```

Khi hai giao dịch cập nhật cùng dòng, PostgreSQL xếp chúng theo khóa dòng. Bên
đến sau tính trên giá trị mới nhất sau khi bên trước chốt, nên các chênh lệch
không ghi đè nhau.

## 7. Trừ có điều kiện để không âm

Một phép tiêu hoặc rút phải đưa điều kiện vào câu ghi:

```sql
UPDATE balance_projection
SET posted_balance = posted_balance - :amount,
    updated_at = CURRENT_TIMESTAMP
WHERE account_id = :accountId
  AND :amount > 0
  AND posted_balance >= :amount
RETURNING posted_balance;
```

Kết quả:

| Số dòng | Ý nghĩa |
| --- | --- |
| `1` | Phép trừ đã xảy ra trên số dư đủ dùng |
| `0` | Tài khoản không tồn tại hoặc số dư không đủ |
| Lỗi chờ khóa/bế tắc | Lỗi kỹ thuật; giao dịch phải hoàn tác |

Nếu hai lệnh cùng tiêu lượt cuối, bên đến sau chờ khóa rồi PostgreSQL kiểm tra
lại `posted_balance >= :amount`. Điều kiện được đánh giá trên giá trị mới nhất,
không phải số dư cũ mà ứng dụng đã đọc.

## 8. Bút toán và số dư phải cùng giao dịch

Một thay đổi thành công thường gồm:

```text
BEGIN
  chiếm mã lệnh duy nhất
  cập nhật bảng số dư
  INSERT bút toán chỉ thêm mới
  lưu kết quả của mã lệnh
COMMIT
```

Nếu chèn bút toán thất bại, phép cập nhật số dư phải hoàn tác. Nếu cập nhật số dư
không thỏa điều kiện, không được chèn bút toán thành công. Không dùng hai giao
dịch rồi hy vọng một tác vụ bù sẽ luôn chạy, vì tiến trình có thể sập giữa chúng.

Nếu sổ cái và bảng số dư ở hai cơ sở dữ liệu hoặc hai dịch vụ khác nhau, không
còn một giao dịch cục bộ bao trùm. Khi đó cần một mô hình phân tán riêng như giữ
chỗ, outbox/inbox, saga hoặc cấp hạn mức; không được tuyên bố hai lần ghi đã
nguyên tử chỉ vì cả hai phương thức Java có `@Transactional`.

## 9. Chống ghi lặp

Một lần thử lại cùng lệnh không được tạo bút toán thứ hai. Đặt ràng buộc duy nhất
trên mã tham chiếu nghiệp vụ:

```sql
UNIQUE (account_id, command_id)
```

Luồng xử lý cần xác định:

- cùng mã và cùng nội dung: trả kết quả đã lưu;
- cùng mã nhưng khác nội dung: từ chối;
- giao dịch trước đang chạy: chờ hoặc báo đang xử lý theo hợp đồng;
- giao dịch trước hoàn tác: lần sau có thể chiếm mã và chạy;
- mất phản hồi sau `COMMIT`: tra cứu bằng cùng mã.

Ràng buộc duy nhất chỉ ngăn dòng trùng. Muốn phát lại kết quả và kiểm tra nội
dung, cần bản ghi lệnh cùng dấu vân tay như mô tả trong tài liệu về tính lũy đẳng.

## 10. Thứ tự bút toán và `balance_after`

Một số hệ thống lưu số dư sau bút toán (`balance_after`) để truy vết thuận tiện.
Giá trị này chỉ đáng tin khi mọi thay đổi của cùng tài khoản được tuần tự hóa bởi
dòng số dư và bút toán được thêm trước khi khóa dòng được giải phóng.

Ví dụ:

```text
khóa/cập nhật dòng số dư → nhận balance_after → thêm bút toán → COMMIT
```

Không dùng thời gian tạo hoặc UUID ngẫu nhiên làm bằng chứng duy nhất cho thứ tự
nghiệp vụ. Nếu cần một chuỗi chặt chẽ, lưu `account_sequence` tăng cùng dòng số
dư và đặt `UNIQUE (account_id, account_sequence)` trên sổ cái.

## 11. Sổ cái đơn và bút toán kép

Điểm thưởng có thể được mô hình hóa như quyền lợi một phía: cộng khi kiếm điểm,
trừ khi sử dụng. Tiền trong hệ thống tài chính thường cần bút toán kép để mỗi
giao dịch cân bằng giữa ít nhất hai tài khoản:

```text
tổng debit = tổng credit
```

Một cột `amount_delta` đơn trên tài khoản khách hàng là ví dụ học tập hữu ích,
nhưng không tự đáp ứng yêu cầu kế toán, tiền tệ, thanh toán hay quy định pháp lý.
Không suy rộng mô hình điểm thưởng thành sổ cái tiền thật mà không có thiết kế
chuyên ngành.

## 12. Khoản giữ chỗ và vòng đời

Khi nghiệp vụ cần giữ quyền sử dụng trước rồi mới chốt, không trừ và cộng lại tùy
tiện. Hãy lưu một thực thể giữ chỗ có trạng thái:

```text
ACTIVE → CAPTURED
ACTIVE → RELEASED
ACTIVE → EXPIRED
```

Mỗi chuyển trạng thái chỉ được xảy ra một lần bằng cập nhật có điều kiện. Số dư
khả dụng phải tính các khoản `ACTIVE`; chốt hoặc giải phóng cập nhật số dư và
lịch sử theo một quy tắc có thể đối soát.

Hết hạn và chốt cùng lúc là một bài toán tranh chấp riêng. Bộ hẹn giờ không được
giải phóng một khoản đã chốt chỉ vì nó đọc trạng thái cũ.

## 13. Bảng số dư là dữ liệu chiếu, nhưng vẫn có thẩm quyền vận hành

Sổ cái thường là nguồn để xây dựng lại lịch sử. Tuy vậy, câu hỏi “lệnh này có đủ
số dư ngay bây giờ không?” cần một dòng có thể khóa và cập nhật nguyên tử. Không
tính `SUM(ledger_entry)` trong mỗi yêu cầu rồi xem kết quả đó là quyền giữ chỗ:

- phép tổng có thể tốn kém;
- hai giao dịch vẫn có thể cùng thấy một tổng;
- kết quả đọc không tự ngăn bút toán khác được thêm;
- điều kiện không âm khó được bảo vệ bằng một lần đọc thông thường.

Vì bảng số dư và sổ cái cùng chốt, bảng số dư là cơ sở quyết định tức thời; sổ
cái là cơ sở kiểm toán và phục hồi. Hai vai trò không mâu thuẫn.

## 14. Đối soát và xây dựng lại

Truy vấn cơ bản:

```sql
SELECT b.account_id,
       b.posted_balance,
       COALESCE(sum(e.amount_delta), 0) AS ledger_balance
FROM balance_projection b
LEFT JOIN ledger_entry e ON e.account_id = b.account_id
GROUP BY b.account_id, b.posted_balance
HAVING b.posted_balance <> COALESCE(sum(e.amount_delta), 0);
```

Kết quả phải rỗng. Khi có chênh lệch:

1. xác định sổ cái hay dữ liệu nhập nguồn nào có thẩm quyền;
2. cô lập hoặc giới hạn ghi lên tài khoản bị ảnh hưởng;
3. tìm lệnh thiếu, lặp hoặc sai thứ tự;
4. sửa bằng bút toán điều chỉnh hoặc xây dựng lại bảng chiếu theo quy trình;
5. lưu dấu vết của thao tác khắc phục;
6. tìm và sửa đường ghi gây lệch.

Không âm thầm sửa số dư cho khớp mà bỏ qua nguyên nhân. Đối soát là lớp phát
hiện và phục hồi, không thay thế bảo vệ đồng thời trong đường ghi.

## 15. Chốt, hoàn tác và sự cố

| Tình huống | Hành vi đúng |
| --- | --- |
| Cập nhật số dư và chèn bút toán cùng chốt | Kết quả bền vững và đối soát được |
| Chèn bút toán thất bại | Hoàn tác cả cập nhật số dư |
| Giao dịch sập trước `COMMIT` | PostgreSQL hoàn tác thay đổi chưa chốt |
| Mất phản hồi sau `COMMIT` | Tra cứu theo mã lệnh; không tạo mã mới |
| Hết thời gian chờ khóa | Không coi là thiếu số dư; thử lại có giới hạn bằng giao dịch mới |
| Bế tắc | PostgreSQL hủy một giao dịch; thống nhất thứ tự khóa và thử lại có giới hạn |

Không gọi dịch vụ từ xa trong lúc giữ khóa dòng số dư. Nếu phải phát thông điệp,
ghi outbox trong cùng giao dịch rồi gửi sau.

## 16. Chạy trên nhiều máy chủ

`synchronized`, `ReentrantLock` hoặc bộ nhớ đệm cục bộ không bảo vệ số dư khi các
yêu cầu chạy trên nhiều JVM. Ràng buộc duy nhất, câu cập nhật có điều kiện và
giao dịch PostgreSQL áp dụng cho mọi kết nối nên là ranh giới có thẩm quyền.

Một khóa phân tán không thay thế ràng buộc và lịch sử. Nó còn thêm các tình huống
hết hạn thuê, tiến trình tạm dừng và chủ sở hữu cũ tiếp tục chạy.

## 17. Kiểm thử bắt buộc

- Hai lệnh trừ khác nhau cùng tranh số dư chỉ cho phép số lệnh phù hợp thành
  công.
- Một lệnh được gửi lại không tạo bút toán thứ hai.
- Cộng và trừ đồng thời không làm mất chênh lệch.
- Lỗi sau khi cập nhật số dư nhưng trước khi chèn sổ cái hoàn tác mọi thay đổi.
- Bên giữ khóa chốt và hoàn tác đều được kiểm tra.
- Bảng số dư luôn bằng tổng sổ cái sau mỗi test.
- Test dùng PostgreSQL Testcontainers, giao dịch/kết nối riêng và các chốt có
  thời gian tối đa.

## 18. Dữ liệu cần theo dõi

```text
ledger.posting.applied
ledger.posting.replayed
ledger.posting.rejected
ledger.posting.fingerprint_mismatch
balance.conditional_update_zero
balance.lock_wait_duration
balance.deadlock
balance.reconciliation_mismatch
hold.stuck_age
```

Không đưa mã tài khoản, mã khách hàng hoặc mã lệnh thô vào nhãn metric có số
lượng giá trị không giới hạn. Dùng log có kiểm soát và mã tương quan phù hợp.

## 19. Liên kết tài liệu

- [BANK-001 — Chi tiêu đồng thời vượt số dư](../banking/concurrent-withdrawal-double-spend/README.md)
- [BANK-007 — Bút toán đồng thời và bảng chiếu số dư](../banking/ledger-posting-projection/README.md)
- [BANK-008 — Số dư khả dụng và khoản giữ](../banking/authorization-available-balance/README.md)
- [ECOM-005 — Chi tiêu điểm thưởng đồng thời](../ecommerce/loyalty-point-concurrency/README.md)
- [Cập nhật dữ liệu an toàn bằng SQL](atomic-database-operations.md)
- [Tính lũy đẳng và tính duy nhất](idempotency-and-uniqueness.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
