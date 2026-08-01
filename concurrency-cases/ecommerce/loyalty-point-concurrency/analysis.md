# Phân tích số dư điểm và sổ cái khi chạy đồng thời

## 1. Trạng thái ban đầu

```text
loyalty_account(C-17):
  points_balance = 1.000
  revision       = 10

loyalty_ledger_entry:
  OPENING +1.000, account_sequence = 10

CMD-A và CMD-B chưa tồn tại.
```

A và B là hai giao dịch, hai kết nối, hai đơn hàng và hai mã lệnh khác nhau. Mỗi
lệnh muốn tiêu `800` điểm.

## 2. Kết quả mong đợi và thực tế

| Nội dung | Mong đợi | Thiết kế bị lỗi |
| --- | --- | --- |
| Lệnh thành công | `1` | `2` |
| Lệnh thiếu điểm | `1` | `0` |
| Số dư cuối | `200` | `200` nhìn bề ngoài |
| Tổng sổ cái | `200` | `-600` |
| Khả năng kiểm toán | Số dư khớp từng bút toán | Lịch sử và số dư mâu thuẫn |

## 3. Hai bất biến độc lập nhưng phải cùng giao dịch

### An toàn số dư

```text
points_balance >= 0
```

Điều kiện này quyết định một lệnh có được phép tiêu hay không.

### Tính đầy đủ của lịch sử

```text
points_balance = SUM(loyalty_ledger_entry.points_delta)
```

Điều kiện này cho phép giải thích và xây dựng lại số dư.

Một câu cập nhật có điều kiện bảo vệ bất biến đầu. Việc thêm bút toán trong cùng
giao dịch bảo vệ quan hệ thứ hai. Chỉ dùng một trong hai là chưa đủ.

## 4. Dòng thời gian bị lỗi

| Bước | Giao dịch A | Giao dịch B | Dữ liệu bền vững |
| --- | --- | --- | --- |
| 1 | Đọc số dư `1.000` | | `1.000` |
| 2 | | Đọc số dư `1.000` | `1.000` |
| 3 | Quyết định đủ `800` | Quyết định đủ `800` | `1.000` |
| 4 | Chèn bút toán `-800` | Chèn bút toán `-800` | Chưa chốt |
| 5 | Ghi số dư `200` | | A giữ khóa dòng |
| 6 | `COMMIT` | | Số dư `200`, một bút toán mới |
| 7 | | Ghi số dư cũ đã tính `200`; `COMMIT` | Hai bút toán, số dư vẫn `200` |

Nguyên nhân chính xác là chuỗi `đọc → kiểm tra → tính → ghi giá trị tuyệt đối`
không nguyên tử. Việc có nhiều luồng chỉ làm lộ khoảng hở đó.

## 5. Ảnh chụp MVCC và câu ghi tuyệt đối

Ở `READ COMMITTED`, mỗi câu `SELECT` nhìn thấy dữ liệu đã chốt trước khi câu lệnh
bắt đầu. A và B có thể cùng đọc `1.000` vì chưa bên nào chốt thay đổi.

Khi hai câu `UPDATE` cùng sửa một dòng, PostgreSQL buộc B chờ A. Sau A chốt, B
tiếp tục trên phiên bản dòng mới nhất, nhưng mệnh đề gán vẫn là:

```sql
SET points_balance = 200
```

PostgreSQL không biết `200` được tính từ giá trị cũ nào trong Java. Nó chỉ thực
hiện phép gán hợp lệ và làm mất tác động của A trên bảng số dư.

## 6. Phép trừ có điều kiện phân xử hai lệnh

```sql
UPDATE loyalty_account
SET points_balance = points_balance - 800
WHERE customer_id = 'C-17'
  AND points_balance >= 800
RETURNING points_balance;
```

Dòng thời gian an toàn:

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | Khóa dòng tài khoản | |
| 2 | Kiểm tra `1.000 >= 800`, giảm còn `200` | Chờ khóa |
| 3 | Thêm bút toán `-800`; `COMMIT` | Được đánh thức |
| 4 | | PostgreSQL kiểm tra lại `200 >= 800` |
| 5 | | Điều kiện sai, nhận `0` dòng; lưu kết quả từ chối |

B không cần ném ngoại lệ để biểu diễn thiếu điểm. Nó có thể lưu
`INSUFFICIENT_POINTS` vào bảng lệnh rồi chốt một giao dịch không có bút toán.

## 7. Khi bên giữ khóa hoàn tác

Nếu A giảm số dư rồi chèn bút toán thất bại:

```text
A ROLLBACK
→ số dư bền vững trở lại 1.000
→ bút toán của A không tồn tại
→ B được đánh thức
→ B kiểm tra 1.000 >= 800 và có thể thành công
```

Khóa được giữ tới khi giao dịch chốt hoặc hoàn tác. Bên đang chờ không quan sát
một số dư tạm thời như dữ liệu đã chốt.

## 8. Cộng và trừ đồng thời

Với số dư `1.000`, một lệnh cộng `500` và một lệnh trừ `300` dùng SQL tương đối:

```text
nếu cộng trước: 1.000 + 500 = 1.500; 1.500 - 300 = 1.200
nếu trừ trước:  1.000 - 300 =   700;   700 + 500 = 1.200
```

Hai thứ tự tuần tự đều cho cùng kết quả vì đây là hai chênh lệch hợp lệ. Cả hai
bút toán cùng tồn tại và bảng số dư bằng tổng sổ cái.

Nếu lệnh trừ lớn đến mức chỉ thành công sau khi lệnh cộng chốt, kết quả có thể
phụ thuộc thứ tự lấy khóa. Hai yêu cầu thật sự đồng thời không có một thứ tự kinh
doanh định sẵn; mỗi kết quả tương ứng một thứ tự tuần tự hợp lệ. Nếu nghiệp vụ
cần ưu tiên, phải có hàng đợi hoặc quy tắc sắp thứ tự rõ ràng.

## 9. Mã lệnh và tính lũy đẳng

Hai bản sao của cùng `command_id` không phải hai lần tiêu hợp lệ. Bảng
`loyalty_command` dùng:

```sql
UNIQUE (customer_id, command_id)
```

Luồng chính chèn quyền xử lý trước khi đụng số dư:

- bên thắng tiếp tục;
- bên thua chờ kết quả giao dịch đang giữ khóa duy nhất;
- bên thắng chốt: bên thua đọc và phát lại;
- bên thắng hoàn tác: bên chờ có thể trở thành bên thắng;
- cùng mã nhưng khác dấu vân tay: từ chối.

Ràng buộc trên sổ cái là lớp phòng thủ thứ hai, không phải nơi đầu tiên phát hiện
lệnh trùng sau khi số dư đã bị trừ.

## 10. Kết quả thiếu điểm được lưu

Khi phép trừ trả `0` dòng, hệ thống có thể chốt:

```text
loyalty_command.status  = COMPLETED
loyalty_command.outcome = INSUFFICIENT_POINTS
ledger_entry            = không có
points_balance          = không đổi
```

Sau đó khách hàng kiếm thêm điểm. Gửi lại cùng mã lệnh vẫn nhận kết quả thiếu
điểm ban đầu, vì đó là cùng ý định đã hoàn tất. Một lần thử mới dùng mã mới.

Nếu không lưu kết quả từ chối, hợp đồng phải nói rõ lần thử lại có thể được đánh
giá lại và cho kết quả khác. Không được để hành vi này phụ thuộc ngẫu nhiên vào
máy chủ xử lý.

## 11. Thứ tự tài khoản và bút toán

Giải pháp cập nhật dòng tài khoản trước, nhận `points_balance` và `revision` mới,
sau đó thêm bút toán chứa:

```text
balance_after    = points_balance mới
account_sequence = revision mới
```

Khóa dòng tài khoản chưa được giải phóng cho tới `COMMIT`, nên giao dịch khác
không thể nhận cùng `account_sequence`. Ràng buộc
`UNIQUE (customer_id, account_sequence)` bảo vệ chuỗi lịch sử.

Không sắp thứ tự chỉ bằng `created_at` hoặc UUID. Hai đồng hồ có thể trùng, lệch,
hoặc không phản ánh thứ tự khóa thực tế.

## 12. Vì sao sổ cái không tự ngăn tiêu quá mức

Sổ cái chỉ thêm mới bảo đảm lịch sử không bị ghi đè. Nó không tự quyết định một
bút toán âm có được chấp nhận hay không.

Hai giao dịch có thể cùng tính `SUM(points_delta) = 1.000` rồi cùng chèn `-800`.
Kết quả sổ cái là chính xác về những gì ứng dụng đã làm, nhưng ứng dụng đã cho
phép một hành động sai. Dòng số dư với cập nhật có điều kiện là điểm tuần tự hóa
quyết định.

## 13. Hibernate và câu SQL trực tiếp

Câu cập nhật JDBC bỏ qua ngữ cảnh lưu trữ của Hibernate. Nếu `LoyaltyAccount` đã
được tải trong cùng giao dịch, thực thể giữ số dư cũ và có thể ghi đè khi
`flush()`.

Không tải thực thể số dư trong giao dịch dùng JDBC cho phép cộng/trừ. Nếu buộc
phải trộn, cần `flush()` trước và `clear()` sau câu SQL, nhưng thiết kế đó dễ bị
sai hơn việc dùng một đường ghi duy nhất.

## 14. Chốt, hoàn tác, hết thời gian và sự cố

| Tình huống | Trạng thái bền vững | Hành vi lần sau |
| --- | --- | --- |
| Thành công và chốt | Số dư, bút toán, kết quả lệnh cùng tồn tại | Phát lại |
| Thiếu điểm và chốt | Chỉ có kết quả từ chối | Phát lại từ chối |
| Lỗi sau phép trừ | Toàn bộ thay đổi hoàn tác | Có thể thử lại cùng mã |
| Tiến trình sập trước chốt | PostgreSQL hoàn tác | Bên chờ có thể tiếp tục |
| Mất phản hồi sau chốt | Kết quả có thể đã tồn tại | Tra cứu cùng mã; không tạo mã mới |
| Hết thời gian chờ khóa | Không đồng nghĩa thiếu điểm | Giao dịch mới, thử lại có giới hạn |
| Bế tắc | Một giao dịch bị PostgreSQL hoàn tác | Sửa thứ tự khóa, thử lại có giới hạn |

## 15. Hoàn điểm và điều chỉnh

Không cập nhật bút toán `REDEEM -800` thành `0`. Tạo một lệnh mới và một bút toán
`REVERSAL +800` liên kết tới bút toán gốc.

Để ngăn hoàn hai lần:

```sql
UNIQUE (reverses_entry_id)
```

Phép cộng hoàn điểm và bút toán bù cùng giao dịch. Nếu đơn hàng, điểm và hoàn
tiền nằm ở các dịch vụ khác nhau, cần quy trình phân tán và đối soát riêng.

## 16. Điểm hết hạn và khoản giữ

Hết hạn điểm là một bút toán âm có quy tắc chọn lô điểm, không phải lệnh sửa số
dư trực tiếp. Tiêu điểm và tác vụ hết hạn chạy đồng thời phải cùng tranh số dư và
không được làm âm.

Nếu checkout chỉ giữ điểm rồi chốt sau, cần thực thể giữ chỗ với trạng thái
`ACTIVE`, `CAPTURED`, `RELEASED`, `EXPIRED`. Số dư khả dụng phải trừ các khoản
`ACTIVE`; chốt và hết hạn phải dùng chuyển trạng thái có điều kiện. Case này tập
trung vào phép tiêu chốt ngay.

## 17. Tranh chấp trên tài khoản nóng

Mọi thay đổi của một khách hàng đi qua một dòng số dư, nên một tài khoản hoạt
động mạnh có thể thành dòng nóng. Câu cập nhật ngắn giữ dữ liệu đúng nhưng không
cho cùng một dòng được ghi song song.

Không tăng vùng kết nối vô hạn. Cần giới hạn thời gian chờ, giữ giao dịch ngắn và
theo dõi hàng đợi khóa. Phân mảnh một số dư thành nhiều dòng làm điều kiện không
âm trở thành bất biến nhiều dòng và cần một mô hình hạn mức mới.

## 18. Chạy trên nhiều máy chủ

PostgreSQL phân xử mọi kết nối bằng cùng khóa dòng và chỉ mục duy nhất. JVM nào
gửi câu SQL trước không ảnh hưởng tới tính đúng đắn.

`synchronized` và bộ nhớ đệm không bảo vệ máy chủ khác. Khóa phân tán không thay
thế ràng buộc, phép trừ có điều kiện hoặc giao dịch sổ cái.

## 19. Nguyên nhân gốc theo tầng

| Tầng | Vấn đề |
| --- | --- |
| Nghiệp vụ | Không tách quyết định số dư khỏi lịch sử kiểm toán và kết quả lệnh |
| Ứng dụng | Dùng `read → check → write`, hoặc tạo mã mới khi thử lại |
| Spring | Hiểu nhầm `@Transactional` là khóa giữa hai yêu cầu |
| Hibernate | Ghi giá trị tuyệt đối từ thực thể cũ và trì hoãn SQL tới `flush()` |
| PostgreSQL | Lược đồ thiếu `CHECK`, `UNIQUE` và câu trừ có điều kiện |
| Vận hành | Sửa số dư trực tiếp, xóa bút toán hoặc không chạy đối soát |

## 20. Đối soát và phục hồi

Đối soát so sánh:

```sql
loyalty_account.points_balance
    với
SUM(loyalty_ledger_entry.points_delta)
```

Khi lệch, không tự động ghi đè số dư mà không điều tra. Cần xác định bút toán
thiếu/lặp, khóa đường ghi bị lỗi, tạo điều chỉnh có dấu vết hoặc xây dựng lại bảng
chiếu theo quy trình được kiểm soát.

Sổ cái giúp phục hồi bảng số dư, nhưng không tự sửa những đơn hàng đã được hưởng
ưu đãi do tiêu điểm sai. Phần đó cần đối soát với hệ thống đơn hàng.

## 21. Dữ liệu cần theo dõi

```text
loyalty.command.claimed
loyalty.command.replayed
loyalty.command.fingerprint_mismatch
loyalty.points.redeemed
loyalty.points.insufficient
loyalty.points.earned
loyalty.balance.lock_wait_duration
loyalty.balance.rollback
loyalty.balance.reconciliation_mismatch
```

Chỉ số từ chối thiếu điểm là tín hiệu nghiệp vụ. Lỗi chờ khóa, bế tắc và chênh
lệch đối soát là tín hiệu kỹ thuật; không gộp chúng thành một mã lỗi chung.

## 22. Phạm vi phân tích

Case này dùng một sổ cái điểm đơn giản, không phải sổ cái tiền bút toán kép. Tạo
đơn trùng thuộc ECOM-003; giới hạn coupon thuộc ECOM-004. Vòng đời giữ điểm, hết
hạn theo lô và hoàn điểm đồng thời cần mở rộng mô hình trạng thái nhưng vẫn phải
giữ lịch sử chỉ thêm mới.
