# Phạm vi logic, transaction vật lý và dòng thời gian commit

## Trạng thái ban đầu

```text
order 42 status = PENDING
payment_audit     = empty
```

Tx-O bắt đầu và cập nhật order 42 thành `PAID` nhưng chưa commit.

## Dòng thời gian REQUIRES_NEW partial commit

| Bước | Transaction ngoài (Tx-O) | Transaction trong (Tx-A) | Góc nhìn của phía đọc sau commit |
| --- | --- | --- | --- |
| 1 | cập nhật order→PAID | | order=PENDING, không có audit |
| 2 | tạm dừng, giữ tài nguyên/lock | bắt đầu bằng một connection khác | |
| 3 | | chèn PAYMENT_COMPLETED | |
| 4 | | commit Tx-A | order=PENDING + audit=COMPLETED |
| 5 | tiếp tục, dừng ở probe | | thấy được trạng thái ngữ nghĩa một phần |
| 6 | lỗi khối ngoài, rollback | | order=PENDING + bản ghi audit vẫn còn |

`REQUIRES_NEW` bảo đảm việc commit độc lập; chính sự đảm bảo đó phá vỡ bất biến nếu
bản ghi bên trong tuyên bố kết quả của transaction ngoài trong khi nó chưa được quyết định.

> **Nói ngắn gọn:** propagation chạy đúng kỹ thuật nhưng ngữ nghĩa nghiệp vụ chọn
> sai ranh giới commit.

## Dòng thời gian REQUIRED rollback-only

| Bước | Phạm vi logic khối ngoài | Phạm vi logic khối trong (REQUIRED) | Transaction vật lý (Tx-O) |
| --- | --- | --- | --- |
| 1 | bắt đầu | | đang hoạt động |
| 2 | gọi bean bên trong | tham gia Tx-O | đang hoạt động |
| 3 | | ngoại lệ runtime vượt interceptor | bị đánh dấu rollback-only |
| 4 | bắt ngoại lệ, tiếp tục | | vẫn rollback-only |
| 5 | trả về, yêu cầu commit | | rollback |
| 6 | interceptor khối ngoài | | ném `UnexpectedRollbackException` |

Việc bắt ngoại lệ ở khối ngoài chỉ xử lý luồng điều khiển của Java, không đặt lại trạng thái transaction. Spring ném
ngoại lệ để phía gọi không tin rằng việc commit đã thành công.

## Dự kiến và thực tế

| Khía cạnh | Dự kiến | Propagation bị lỗi |
| --- | --- | --- |
| Bản ghi thành công | Commit cùng/sau kết quả nghiệp vụ | Commit trước kết quả của khối ngoài |
| Rollback khối ngoài | Xóa mọi trạng thái thành công | Bản ghi audit vẫn tồn tại |
| Lỗi bên trong tùy chọn | Kết quả rõ ràng | Ngoại lệ runtime đánh dấu rollback-only |
| Phản hồi cho phía gọi | Phản ánh commit thực sự | Có thể bắt lỗi nhầm và báo thành công |
| Sử dụng tài nguyên | Một connection nếu cùng ranh giới | Khối ngoài giữ connection, khối trong cần connection thứ hai |

## Ngữ nghĩa của Propagation

### REQUIRED

Nếu có transaction, tham gia cùng transaction vật lý. Mỗi phương thức được cấu hình là một phạm vi
logic với các quy tắc rollback, nhưng thao tác commit chỉ xảy ra một lần ở ranh giới ngoài cùng. Trạng thái
rollback-only ở bên trong sẽ ảnh hưởng toàn bộ transaction vật lý.

### REQUIRES_NEW

Tạm dừng transaction ngoài, tạo transaction vật lý mới, commit/rollback độc lập,
rồi tiếp tục transaction ngoài. Khối trong không thấy những thay đổi chưa commit của khối ngoài theo cách “cùng
transaction”; truy vấn có thể thấy phiên bản đã commit cũ hoặc bị chặn (block) bởi lock.

### NESTED

Thường dùng savepoint trong cùng transaction vật lý: khối trong có thể rollback về
savepoint, nhưng khi khối ngoài rollback thì sẽ rollback tất cả. Hỗ trợ này phụ thuộc vào transaction
manager/driver; `JpaTransactionManager` không mặc định biến mọi thao tác JPA thành
quy trình savepoint. Cần phải kiểm thử tích hợp trên stack thực.

## Flush, lock và connection pool

Quá trình flush của khối ngoài có thể giữ các lock trên dòng (row lock) khi bị tạm dừng. Transaction bên trong truy cập cùng
các dòng đó có thể tự chặn lại để chờ khối ngoài — trong khi khối ngoài lại đang chờ khối trong trả về, tạo ra tình trạng chờ tài nguyên.
Ngoài ra, mỗi quá trình `REQUIRES_NEW` chạy đồng thời có thể cần một connection thứ hai; việc cạn kiệt pool
làm mọi khối ngoài giữ connection và chờ connection cho khối trong. Chi tiết về cạn kiệt pool
thuộc `SPR-007`.

## Các quy tắc ngoại lệ và rollback

Ngoại lệ runtime vượt qua transactional interceptor mặc định sẽ đánh dấu rollback. Các checked
exception chỉ hoạt động theo cấu hình `rollbackFor`. Việc dùng `noRollbackFor` phải dựa trên trạng thái
có an toàn để commit hay không, chứ không phải do mong muốn “đừng thấy ngoại lệ”.

Nếu khối bên trong tự chuyển nhánh dự kiến thành một giá trị (`RiskOutcome.REJECTED`) mà không
có thay đổi dữ liệu dang dở hoặc ngoại lệ, khối ngoài có thể quyết định commit/rollback một cách rõ ràng. Nếu lỗi
kỹ thuật đã làm transaction vật lý không còn tin cậy, đừng cố cứu vãn bằng cách bắt ngoại lệ (catch).

## Thử lại, tính lũy đẳng và trùng lặp

Việc thử lại (retry) ở khối ngoài sau một partial commit có thể chèn bản ghi audit lần thứ hai. Bản ghi độc lập
bên trong cần khóa duy nhất (unique key) như `(operation_id, event_type)` và ngữ nghĩa phát lại (replay). Nhưng
tính lũy đẳng chỉ chặn dữ liệu trùng lặp; nó không sửa bản ghi `COMPLETED` sai sự thật.

Không thử lại `UnexpectedRollbackException` một cách mù quáng; hãy phân loại nguyên nhân,
dọn dẹp transaction cũ và bắt đầu lần thử mới ở ranh giới proxy.

## Nhiều instance và ranh giới phân tán

Propagation là chính sách của ứng dụng cục bộ nhưng transaction vật lý được
PostgreSQL điều phối qua các node. Nó không tạo ra tính nguyên tử (atomicity) với dịch vụ thanh toán từ xa hay message
broker. Quá trình compensation/saga cho nhiều tài nguyên thuộc `DIST-004`; việc xuất bản sự kiện bền vững (durable event)
thuộc các trường hợp messaging/outbox.

## Sự cố (crash) và khả năng quan sát

Sự cố sau khi Tx-A commit nhưng trước khi Tx-O rollback/commit sẽ để lại bản ghi audit độc lập đúng
như ngữ nghĩa. Cần theo dõi ID của transaction trong/ngoài, propagation, thời gian tạm dừng,
việc cấp phát connection, rollback-only/UnexpectedRollback, sự sai lệch giữa audit và nghiệp vụ
cũng như ID thao tác bị trùng lặp.

Kiến thức nền: [Ranh giới transaction trong Spring](../../concepts/spring-transaction-boundaries.md).
