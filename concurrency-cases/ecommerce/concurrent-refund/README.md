# ECOM-008 — Các yêu cầu hoàn tiền đồng thời

## 1. Bài toán

Một giao dịch thanh toán đã thu `1.000.000` đồng. Gần như cùng lúc:

- nhân viên chăm sóc khách hàng yêu cầu hoàn `700.000` đồng;
- quy trình tự động yêu cầu hoàn `600.000` đồng;
- phía gọi gửi lại một yêu cầu vì không nhận được phản hồi trước đó.

Nếu mỗi luồng tự đọc số tiền đã hoàn, tính phần còn lại trong Java rồi ghi kết
quả, cả hai yêu cầu khác nhau có thể cùng được chấp nhận. Tổng số tiền hoàn khi
đó là `1.300.000` đồng, lớn hơn số tiền đã thu. Một yêu cầu bị gửi lại còn có
thể tạo khoản hoàn thứ hai nếu hệ thống chỉ kiểm tra trùng bằng một câu `SELECT`.

> **Nói ngắn gọn:** Cùng một yêu cầu phải được phát lại kết quả; các yêu cầu
> khác nhau phải cùng tranh một hạn mức hoàn tiền được cập nhật nguyên tử.

## 2. Tác nhân và dữ liệu dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Chăm sóc khách hàng | Tạo yêu cầu hoàn tiền thủ công |
| Quy trình tự động | Hoàn tiền khi hủy đơn, thiếu hàng hoặc áp dụng chính sách |
| Yêu cầu gửi lại | Dùng lại cùng khóa lũy đẳng sau lỗi mạng hoặc hết thời gian chờ |
| Máy chủ A, B | Có thể xử lý các yêu cầu trên hai JVM khác nhau |
| `payment_charge` | Giữ số tiền đã thu và phần đã dành cho hoàn tiền |
| `refund_request` | Lưu khóa lũy đẳng, dấu vân tay và kết quả đã chốt |
| `refund` | Theo dõi vòng đời của từng khoản hoàn được chấp nhận |
| `refund_ledger_entry` | Lưu lịch sử phân bổ, hoàn tất và giải phóng theo cách chỉ thêm mới |
| `outbox_event` | Lưu lệnh gửi khoản hoàn sang nhà cung cấp thanh toán |
| PostgreSQL | Nguồn có thẩm quyền để phân xử hạn mức và tính duy nhất |

Điểm tranh chấp chính là dòng `payment_charge` của cùng một `charge_id`. Khóa
lũy đẳng chỉ phân xử các lần gửi lại của cùng một ý định; nó không được gom hai
yêu cầu hoàn khác nhau thành một.

## 3. Ba số tiền cần phân biệt

```text
captured_amount
    số tiền đã thu của giao dịch

allocated_refund_amount
    tổng tiền đã dành cho các khoản hoàn đang chờ hoặc đã thành công

completed_refund_amount
    phần tiền hoàn đã được nhà cung cấp xác nhận thành công
```

Số tiền còn có thể nhận thêm yêu cầu hoàn được tính bằng:

```text
refundable_amount = captured_amount - allocated_refund_amount
```

Không dùng `completed_refund_amount` để quyết định hạn mức. Một khoản hoàn đang
chờ nhà cung cấp xử lý vẫn phải chiếm hạn mức; nếu không, nhiều yêu cầu đang chờ
có thể cùng vượt số tiền đã thu.

## 4. Quy tắc bất biến

```text
0 <= completed_refund_amount
completed_refund_amount <= allocated_refund_amount
allocated_refund_amount <= captured_amount

allocated_refund_amount
    = SUM(refund_ledger_entry.allocation_delta)

completed_refund_amount
    = SUM(refund_ledger_entry.completion_delta)
```

Ngoài các công thức trên:

- mỗi `(merchant_id, idempotency_key)` chỉ có một kết quả đã chốt;
- cùng khóa và cùng dấu vân tay phải phát lại cùng kết quả;
- cùng khóa nhưng khác nội dung phải bị từ chối;
- mỗi yêu cầu được chấp nhận chỉ tạo một `refund`, một bút toán phân bổ và một
  lệnh outbox;
- một khoản hoàn chỉ đi tới một kết quả kết thúc;
- bút toán cũ không bị sửa hoặc xóa; việc giải phóng dùng bút toán bù.

## 5. Hai loại tranh chấp khác nhau

### Cùng một yêu cầu bị gửi lại

Hai lần gọi cùng mang khóa `RF-2026-001` và cùng nội dung không phải hai ý định
hoàn tiền. Ràng buộc duy nhất trên khóa cho phép một bên chiếm quyền xử lý. Bên
còn lại chờ kết quả rồi phát lại cùng `refund_id` hoặc cùng lý do từ chối.

### Hai yêu cầu hoàn tiền độc lập

Yêu cầu `RF-A` hoàn `700.000` đồng và `RF-B` hoàn `600.000` đồng là hai ý định
hợp lệ khác nhau. Chúng phải có khóa riêng, nhưng cùng tranh
`allocated_refund_amount` của giao dịch. Điều kiện giới hạn phải nằm trong câu
`UPDATE`, không nằm trong một lần đọc trước đó.

## 6. Cập nhật hạn mức có điều kiện

```sql
UPDATE payment_charge
SET allocated_refund_amount = allocated_refund_amount + :amount,
    revision = revision + 1,
    updated_at = CURRENT_TIMESTAMP
WHERE charge_id = :chargeId
  AND merchant_id = :merchantId
  AND currency = :currency
  AND :amount > 0
  AND allocated_refund_amount + :amount <= captured_amount
RETURNING captured_amount,
          allocated_refund_amount,
          completed_refund_amount,
          revision;
```

PostgreSQL khóa dòng khi cập nhật. Nếu hai yêu cầu khác nhau cùng tranh phần tiền
cuối, bên đến sau chờ bên trước. Sau khi khóa được nhả, PostgreSQL kiểm tra lại
điều kiện trên giá trị mới nhất:

- trả về một dòng: số tiền đã được dành cho yêu cầu này;
- trả về không dòng: yêu cầu không giành được hạn mức;
- lỗi chờ khóa, bế tắc hoặc kết nối: lỗi kỹ thuật, không phải kết quả “vượt hạn
  mức”.

Danh tính giao dịch, quyền của cửa hàng và loại tiền phải được kiểm tra riêng.
Case này giả định bản ghi giao dịch đã thu là dữ liệu bền vững, không bị xóa
trong lúc xử lý hoàn tiền.

## 7. Ranh giới giao dịch khi tiếp nhận yêu cầu

```text
BEGIN
  1. chiếm (merchant_id, idempotency_key) và lưu dấu vân tay
  2. kiểm tra giao dịch, loại tiền và quyền hoàn
  3. cộng allocated_refund_amount bằng UPDATE có điều kiện
  4. nếu thành công, tạo refund ở trạng thái PENDING_PROVIDER
  5. thêm bút toán REFUND_ALLOCATED
  6. thêm outbox_event yêu cầu nhà cung cấp hoàn tiền
  7. lưu kết quả ACCEPTED hoặc LIMIT_EXCEEDED vào refund_request
COMMIT
```

Tất cả bước ghi cơ sở dữ liệu nằm trong cùng một giao dịch. Không gọi nhà cung
cấp thanh toán qua mạng trong lúc giữ khóa dòng `payment_charge`.

Kết quả `LIMIT_EXCEEDED` cũng được lưu. Lần gửi lại cùng khóa nhận đúng kết quả
đó, kể cả khi một khoản hoàn khác được giải phóng sau này. Một ý định mới phải
dùng khóa mới.

## 8. Vòng đời của một khoản hoàn

```text
                       nhà cung cấp xác nhận
PENDING_PROVIDER ─────────────────────────────> SUCCEEDED
        │
        │ từ chối vĩnh viễn, đã cho phép giải phóng
        └─────────────────────────────────────> FAILED_RELEASED
```

Khi chuyển sang `SUCCEEDED`, hệ thống tăng `completed_refund_amount` và thêm bút
toán `REFUND_SUCCEEDED` trong cùng giao dịch. Khi chuyển sang
`FAILED_RELEASED`, hệ thống giảm `allocated_refund_amount` và thêm bút toán
`REFUND_RELEASED` trong cùng giao dịch.

Mỗi chuyển trạng thái phải có điều kiện `status = 'PENDING_PROVIDER'`. Nhờ đó,
một thông báo được gửi lại không hoàn tất hoặc giải phóng hai lần. Quy tắc xử lý
thứ tự các thông báo phức tạp từ nhà cung cấp là phạm vi của BANK-006; case này
chỉ đặt ranh giới tối thiểu để bảo vệ hạn mức nội bộ.

## 9. Sổ cái trong case này có ý nghĩa gì

`refund_ledger_entry` là sổ lịch sử vận hành cho hạn mức hoàn tiền:

| Loại bút toán | `allocation_delta` | `completion_delta` |
| --- | ---: | ---: |
| `REFUND_ALLOCATED` | `+amount` | `0` |
| `REFUND_SUCCEEDED` | `0` | `+amount` |
| `REFUND_RELEASED` | `-amount` | `0` |

Bảng này giúp giải thích số tiền tổng hợp và chạy đối soát. Nó không tự trở
thành sổ kế toán tiền thật hay bút toán kép. Hạch toán thanh toán thực tế cần mô
hình chuyên ngành riêng.

## 10. Kết quả của bên thắng và bên thua

| Tình huống | Kết quả |
| --- | --- |
| Cùng khóa, cùng nội dung | Phát lại cùng kết quả, không phân bổ lần hai |
| Cùng khóa, khác nội dung | `IDEMPOTENCY_KEY_REUSED` |
| Hai khóa khác nhau, tổng còn trong hạn mức | Cả hai có thể được chấp nhận |
| Hai khóa khác nhau, tổng vượt hạn mức | Chỉ tập hợp yêu cầu phù hợp với hạn mức được chấp nhận |
| Cập nhật trả về không dòng | `LIMIT_EXCEEDED`, không tạo refund, ledger hay outbox |
| Hết thời gian chờ khóa | Lỗi kỹ thuật; toàn bộ giao dịch hoàn tác |
| Mất phản hồi sau `COMMIT` | Gửi lại cùng khóa để phát lại kết quả đã chốt |
| Nhà cung cấp từ chối vĩnh viễn | Chuyển trạng thái và giải phóng bằng bút toán bù đúng một lần |

## 11. Chạy trên nhiều máy chủ

`synchronized`, `ReentrantLock` hoặc một bản đồ khóa trong bộ nhớ chỉ bảo vệ một
JVM. Chúng không ngăn máy chủ khác cùng xử lý giao dịch và cũng mất trạng thái
khi tiến trình khởi động lại.

Ràng buộc duy nhất, câu cập nhật có điều kiện và giao dịch PostgreSQL áp dụng cho
mọi kết nối. Vì vậy, chúng là ranh giới có thẩm quyền cho hệ thống nhiều máy chủ.

## 12. Hậu quả khi triển khai sai

- Hoàn nhiều hơn số tiền đã thu.
- Cùng một yêu cầu tạo nhiều khoản hoàn ở nhà cung cấp.
- Số tổng hợp cho thấy còn hạn mức dù các khoản hoàn đang chờ đã dùng hết.
- Giao dịch đã tăng hạn mức sử dụng nhưng thiếu refund, bút toán hoặc outbox.
- Thông báo lỗi giải phóng cùng một khoản nhiều lần.
- Không giải thích được số tiền tổng hợp khi đối soát hoặc khiếu nại.
- Giữ kết nối và khóa cơ sở dữ liệu trong lúc chờ mạng từ nhà cung cấp.

## 13. Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh hoặc API | Ý nghĩa |
| --- | --- | --- |
| tính lũy đẳng | idempotency | Gửi lại cùng ý định nhưng tác dụng bền vững chỉ xảy ra một lần |
| khóa lũy đẳng | idempotency key | Mã ổn định đại diện cho một ý định hoàn tiền |
| dấu vân tay yêu cầu | request fingerprint | Giá trị dùng để phát hiện cùng khóa nhưng khác nội dung |
| số tiền đã dành | allocated refund amount | Phần tiền không còn cấp cho yêu cầu hoàn khác |
| cập nhật có điều kiện | conditional update | Câu ghi chỉ thành công khi điều kiện hạn mức vẫn đúng |
| bút toán chỉ thêm mới | append-only ledger entry | Dòng lịch sử mới; không sửa lịch sử đã có |
| bút toán bù | compensating entry | Dòng mới đảo tác động của một thay đổi cũ |
| hộp thư đi bền vững | transactional outbox | Bảng ghi lệnh gửi ra ngoài cùng giao dịch nghiệp vụ |
| kết quả chốt chưa rõ | ambiguous commit outcome | Phía gọi mất phản hồi và không biết giao dịch đã chốt hay chưa |

## 14. Điều hướng tài liệu

- [Mã nguồn đọc–kiểm tra–ghi gây lỗi](broken-code.md)
- [Phân tích dòng thời gian và hành vi khóa](analysis.md)
- [Thiết kế Java và SQL an toàn](solutions.md)
- [Thực nghiệm đồng thời với PostgreSQL](experiments.md)
- [BANK-005 — Tạo thanh toán có tính lũy đẳng](../../banking/idempotent-payment-creation/README.md)
- [BANK-007 — Bút toán đồng thời và bảng chiếu số dư](../../banking/ledger-posting-projection/README.md)
- [Cập nhật dữ liệu an toàn bằng SQL](../../concepts/atomic-database-operations.md)
- [Tính lũy đẳng và tính duy nhất](../../concepts/idempotency-and-uniqueness.md)
- [Sổ cái, số dư và khoản giữ chỗ](../../concepts/ledger-balances-and-holds.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## 15. Khi nên dùng cách này

Dùng mô hình này khi một cơ sở dữ liệu PostgreSQL có thể giữ giao dịch đã thu,
yêu cầu hoàn, bảng tổng hợp, sổ lịch sử và outbox trong cùng giao dịch. Đây là
cách phù hợp khi nhiều nguồn được phép tạo các yêu cầu hoàn độc lập trên cùng
một giao dịch.

Nếu quyết định hoàn tiền trải qua nhiều dịch vụ hoặc nhiều cơ sở dữ liệu, cần
thêm saga, inbox/outbox và quy tắc bù. Dù vậy, mỗi dịch vụ vẫn phải bảo vệ bất
biến cục bộ của mình bằng điều kiện và ràng buộc cơ sở dữ liệu.

## 16. Phạm vi

Case này xử lý việc tiếp nhận yêu cầu hoàn, chống gửi lặp, không vượt số tiền đã
thu, lưu lịch sử và tạo lệnh gửi nhà cung cấp. Nó minh họa tối thiểu cách hoàn
tất hoặc giải phóng một khoản đã nhận.

Thứ tự callback, webhook đến muộn, tra cứu trạng thái nhà cung cấp và khôi phục
sau khi nhà cung cấp đã xử lý nhưng hệ thống chưa nhận được thông báo thuộc
BANK-006. Quy tắc kế toán và bút toán kép cho tiền thật cũng nằm ngoài phạm vi.
