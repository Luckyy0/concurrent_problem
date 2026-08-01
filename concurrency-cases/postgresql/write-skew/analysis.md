# Phân tích snapshot isolation và write skew

## Trạng thái ban đầu

```text
roster R:
  Alice on_call=true, version=0
  Bob   on_call=true, version=0
```

Quy tắc bất biến là một điều kiện (predicate) trên một tập hợp: phải có ít nhất một phân công `roster_id=R AND on_call=true`.

## Trình tự xen kẽ bắt buộc

| Bước | Transaction A | Transaction B |
| --- | --- | --- |
| 1 | BEGIN RR | BEGIN RR |
| 2 | Snapshot SA thấy Alice+Bob on-call | |
| 3 | | Snapshot SB thấy Alice+Bob on-call |
| 4 | Quyết định Alice có thể rời ca | Quyết định Bob có thể rời ca |
| 5 | UPDATE Alice row false | |
| 6 | | UPDATE Bob row false |
| 7 | COMMIT | COMMIT |
| 8 | | số lượng on-call cuối = 0 |

Mong đợi: một transaction phải thua/đánh giá lại và thấy chỉ còn một nhân viên.
Thực tế: cả hai commits do các thao tác ghi không chồng chéo.

> **Nói ngắn gọn:** snapshot của mỗi transaction tự nhất quán, nhưng hai snapshots không bao gồm dữ liệu ghi trong tương lai của nhau; ghép hai commits hợp lệ cục bộ tạo ra một trạng thái không hợp lệ toàn cục.

## Góc nhìn MVCC

Mỗi transaction `REPEATABLE READ` thấy các phiên bản tuple đã commit cũ:

```text
SA: Alice=true, Bob=true
SB: Alice=true, Bob=true
```

A tạo tuple Alice mới `false`; B tạo tuple Bob mới `false`. A không cập nhật Bob và B không cập nhật Alice, nên PostgreSQL snapshot isolation không vi phạm quy tắc cập nhật đồng thời trên cùng row để tiến hành abort.

Sau khi cả hai commits, một transaction snapshot mới thấy cả hai phiên bản mới và số lượng là `0`.

## Tập hợp đọc/ghi (Read/write sets)

```text
A read set:  Alice row + Bob row/predicate
A write set: Alice row

B read set:  Alice row + Bob row/predicate
B write set: Bob row
```

Các write sets rời rạc nhau, nhưng:

- Quyết định của A phụ thuộc vào Bob=true mà B sẽ thay đổi;
- Quyết định của B phụ thuộc vào Alice=true mà A sẽ thay đổi.

Đây là hai rw-antidependencies đối chiều, nền tảng của write skew và các lỗi tuần tự hóa (serialization anomaly).

## Hành vi lock

Các câu lệnh COUNT hoặc SELECT entity thông thường dùng MVCC snapshot; chúng không lấy row locks để giữ quy tắc bất biến.

Thao tác UPDATE có version của A:

- locks Alice row;
- điều kiện `version=0` khớp;
- số row bị ảnh hưởng là `1`;
- lock được giải phóng tại thời điểm A commit hoặc rollback.

B làm tương tự trên Bob row. Hai row locks tương thích vì mục tiêu khác nhau.

`@Version` không có phiên bản tổng hợp của danh sách trực để so sánh.

## `READ COMMITTED`

Write skew cũng xảy ra nếu cả hai câu lệnh COUNT chạy trước các thao tác UPDATE commits. Snapshot của câu lệnh mới chỉ giúp khi mã nguồn thực sự kiểm tra lại sau commit đồng thời; luồng xử lý bị lỗi không có bước kiểm tra chốt cuối cùng.

Một lệnh SELECT sau khi A commit có thể giúp B thấy số lượng là `1`, nhưng không được tự động chèn bởi `@Transactional`.

## `REPEATABLE READ`

PostgreSQL `REPEATABLE READ` giữ một transaction snapshot ổn định. B truy vấn lại sau khi A commit vẫn có thể thấy Alice=true từ SB, nên việc đọc lại trong cùng transaction không phải là bước xác thực độ mới.

Mức độ này bảo vệ các xung đột cập nhật trên cùng row mạnh hơn `READ COMMITTED`, nhưng snapshot isolation cho phép write skew trên các write sets rời rạc.

## `SERIALIZABLE` SSI

SSI theo dõi các phụ thuộc của điều kiện và read-write. Trong trường hợp này:

```text
A reads Bob=true  --rw--> B writes Bob=false
B reads Alice=true --rw--> A writes Alice=false
```

Cấu trúc nguy hiểm này không thể có thứ tự tuần tự hóa nào giữ được cả hai quyết định. PostgreSQL abort một transaction với SQLSTATE `40001`. Các predicate locks tuần tự hóa (`SIReadLock`) thường không gây block như row lock; chúng hỗ trợ phát hiện xung đột.

Transaction thua không được dự đoán theo định danh tiến trình. Transaction thử lại mới sẽ thấy bên thắng đã commit, số lượng là `1`, rồi trả về `LAST_OPERATOR_REQUIRED`.

## Nguyên nhân gốc rễ theo từng lớp

### PostgreSQL

`REPEATABLE READ` cung cấp snapshot isolation, không phải full serializability cho quy tắc bất biến đa điều kiện trên nhiều row.

### Spring

Ranh giới transaction đúng nhưng giao thức isolation/locking chưa đủ. Thuộc tính `SERIALIZABLE REQUIRED` bên trong không nâng cấp transaction bên ngoài đã mở.

### Hibernate

Dirty checking sinh ra hai thao tác cập nhật versioned hợp lệ. Hibernate không biết hai đối tượng entity cùng tham gia một quy tắc mức danh sách trực.

### Mô hình ứng dụng

Quy tắc bất biến không có một điểm ghi xác thực chung. Việc kiểm tra và ghi nằm trên các rows và snapshots khác nhau.

## Tại sao đây không phải lost update

Không có cập nhật đã commit nào ghi đè cập nhật khác:

```text
A owns Alice false
B owns Bob false
cả hai thay đổi đều hiện hữu
```

Lost update làm một thao tác ghi biến mất. Write skew giữ cả hai thao tác ghi nhưng tổ hợp của chúng vi phạm ràng buộc.

## So với DB-004

DB-004 dùng lệnh đếm điều kiện rồi INSERT rows mới làm vượt sức chứa tối đa. DB-005 UPDATE các rows có sẵn khác nhau làm vi phạm giới hạn tối thiểu. Cả hai đều cần lý luận trên read/write sets, nhưng hình dạng lỗi và các giải pháp biểu diễn khác nhau.

## Commit và rollback

- Nếu A rollback, Alice vẫn true; B commit Bob=false, quy tắc bất biến được giữ.
- Nếu cả hai cùng commit ở RR, quy tắc bất biến bị sai vĩnh viễn; database không tự sửa chữa.
- Ở SERIALIZABLE, transaction nhận `40001` đã bị aborted; toàn bộ thao tác ghi và sự kiện outbox trong đó bị rollback.
- Các row/guard locks giải phóng tại thời điểm commit, rollback hoặc rớt kết nối.

Checked exception không mặc định rollback trong Spring. Domain exception dùng để hủy nỗ lực sau khi đã thay đổi một phần phải là runtime exception hoặc có cấu hình `rollbackFor`.

## Thử lại (Retry)

Trình tự thử lại đúng:

```text
lần 1 nhận 40001 và rolls back
chờ một khoảng thời gian backoff/jitter có giới hạn
lần 2 bắt đầu một physical transaction mới
tải lại danh sách trực snapshot
chạy lại việc đếm và ra quyết định
```

Không tái sử dụng các managed entities hoặc transaction snapshot cũ. Danh sách trực quá tải có thể gây khuếch đại thử lại (retry amplification); cần giới hạn số lần thử và trả về domain failure có thể thử lại khi đã cạn kiệt.

## Timeout và deadlock

Việc lấy guard lock hoặc khóa tất cả các row có thể gây block:

- cấu hình giới hạn thời gian chờ lock hoặc truy vấn;
- không gọi I/O từ xa khi giữ lock;
- lock ID danh sách trực và ID nhân viên theo một thứ tự tất định;
- deadlock SQLSTATE `40P01` cũng cần thử lại transaction mới khi an toàn.

Lock timeout không chứng minh quy tắc bất biến bị hỏng; current transaction rollback rồi phía gọi có thể thử lại hoặc từ chối theo SLO.

## Sự cố (Crash) và trùng lặp

Crash trước khi commit sẽ rollback lock của row hoặc guard. Crash sau khi commit nhưng trước khi có phản hồi có thể làm phía gọi thử lại yêu cầu rời ca. Thuộc tính lũy đẳng (idempotency) của lệnh phải phát lại kết quả hoặc tiến hành chuyển đổi có điều kiện `on_call=true -> false` để yêu cầu trùng lặp không có tác dụng; nó vẫn không thay thế việc điều phối quy tắc danh sách trực.

Sự kiện outbox `OPERATOR_LEFT_ON_CALL` chỉ được phát hành cùng successful transaction. Giao dịch thua SSI không được phát sự kiện ra bên ngoài.

## Đa tiến trình (Multi-instance)

Các tiến trình có thể ở hai pods khác nhau hoặc từ giao diện admin. Việc dùng `synchronized` hoặc cache cục bộ chỉ bao phủ một process. Guard row, bộ đếm có điều kiện, row locks hoặc SSI sống tại ranh giới PostgreSQL chia sẻ nên có thể điều phối mọi nodes ứng dụng tuân thủ.

Việc truy vấn SQL trực tiếp vẫn có thể phá vỡ giao thức; các quyền, stored procedure, hoặc trigger có thể giới hạn các luồng biến đổi đối với dữ liệu quan trọng.

## Khả năng quan sát (Observability)

Log/metric:

```text
rosterId
operatorId
observedOnCallCount
effectiveIsolation
serializationFailure
lockWaitDuration
retryAttempt
leaveOutcome
```

Đối soát (Reconciliation):

```sql
select roster_id
from on_call_assignment
group by roster_id
having count(*) filter (where on_call) = 0;
```

Theo dõi `40001`, `40P01`, `55P03`, cạn kiệt số lần thử lại, thời gian transaction và số lượng danh sách trực không an toàn. Số exception bằng `0` ở mức RR không phải là tín hiệu thành công.
