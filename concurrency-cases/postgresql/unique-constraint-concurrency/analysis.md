# Phân tích absent key, unique index và flush

## Initial state

```text
business key = (tenant T-42, reference CASE-9001)
matching rows = 0
```

A và B có các thread yêu cầu, connections, transactions và các ID thực thể ngẫu nhiên khác nhau.

## Expected theo broken design

Lập trình viên kỳ vọng:

```text
A kiểm tra thấy chưa tồn tại -> A sở hữu quyền insert
B kiểm tra sau -> B thấy A và trả về kết quả trùng lặp
```

Không có cơ chế nào bảo đảm B kiểm tra sau khi A đã commit. Trình lập lịch (scheduler) có thể đặt cả hai thao tác kiểm tra trước bất kỳ thao tác insert nào.

## Mandatory timeline không constraint

| Bước | Transaction A | Transaction B |
| --- | --- | --- |
| 1 | BEGIN RC | BEGIN RC |
| 2 | EXISTS snapshot SA = false | |
| 3 | | EXISTS snapshot SB = false |
| 4 | INSERT id A/key K | |
| 5 | | INSERT id B/key K |
| 6 | COMMIT | COMMIT |
| 7 | | final count(K)=2 |

Số lượng mong đợi cuối cùng là `1`; thực tế là `2`. Không có ngoại lệ (exception) hay xung đột hàng bị ảnh hưởng (affected-row conflict).

> **Nói ngắn gọn:** sự vắng mặt (absence) chỉ là một trạng thái quan sát (observation), không phải quyền sở hữu (ownership); unique index mới biến business key thành một đối tượng mà các thao tác INSERT đồng thời phải tranh chấp.

## MVCC và lock behavior không constraint

Lệnh SELECT kiểm tra tồn tại thông thường:

- đọc snapshot đã commit;
- không thấy row trong tương lai hoặc chưa commit;
- không khóa hàng (row-lock) một key chưa tồn tại;
- khóa table `ACCESS SHARE` không chặn lệnh INSERT thông thường.

Các lệnh INSERT của A và B khóa các heap rows và các mục primary-key index khác nhau. Vì các ID ngẫu nhiên khác nhau, PostgreSQL không biết hai rows đại diện cho cùng một đối tượng logic.

`REPEATABLE READ` còn giữ snapshot chưa tồn tại một cách ổn định. `SERIALIZABLE` có thể phát hiện xung đột vị từ (predicate conflict), nhưng unique constraint là biểu diễn bất biến trực tiếp, rẻ hơn và bảo vệ mọi mức độ cô lập (isolation levels) hoặc các client thực hiện thay đổi (mutation clients).

## Khi unique constraint tồn tại

```sql
alter table work_item
add constraint uk_work_item_tenant_reference
unique (tenant_id, external_reference);
```

Phân xử của unique-index trong PostgreSQL:

```text
A INSERT key K -> index claim, uncommitted
B INSERT key K -> chờ kết quả transaction của A
```

Nếu A commit:

```text
B thức dậy -> khóa trùng lặp -> SQLSTATE 23505 -> transaction của B bị hủy
```

Nếu A rollback:

```text
Các hiệu ứng trên index/row của A biến mất
B có thể tiếp tục INSERT -> B trở thành bên thắng
```

Các chi tiết về khóa nội bộ (internal locking) hoặc chèn dự đoán (speculative insertion) phụ thuộc vào định dạng của statement, nhưng hợp đồng ứng dụng (application contract) luôn là một bên thắng bền bỉ duy nhất.

## `ON CONFLICT DO NOTHING`

```sql
insert into work_item(...)
values (...)
on conflict (tenant_id, external_reference) do nothing
returning work_item_id;
```

Outcome:

- returned ID: this transaction inserted;
- empty result: conflicting key won.

Ở mức `READ COMMITTED`, phân xử xung đột có thể dựa trên row chưa hiển thị (visible) trong snapshot của statement. Sau khi `DO NOTHING` trả về rỗng (empty), một lệnh SELECT tiếp theo mới thường thấy bên thắng đã commit.

Ở mức `REPEATABLE READ`, xung đột đồng thời có thể được chuyển thành lỗi tuần tự hóa (serialization failure), hoặc snapshot ổn định không phù hợp để đọc bên thắng mới. Ứng dụng phải xử lý lỗi `40001`; luồng (path) đọc bản ghi đã tồn tại nên dùng một transaction/snapshot mới thay vì giả định khả năng hiển thị trên cùng một snapshot.

## Hibernate persistence context

`save()` không đồng nghĩa câu lệnh SQL đã được thực thi. Xung đột có thể xuất hiện (surface):

- `saveAndFlush()`;
- explicit `EntityManager.flush()`;
- query-triggered autoflush;
- transaction commit.

Nếu kết quả phản hồi (response) được xây dựng trước khi commit, phía gọi (caller) chỉ thực sự nhận được response sau khi Spring interceptor thực hiện commit; ngoại lệ trong lúc commit sẽ thay đổi kết quả (outcome).

## PostgreSQL aborted transaction

Vi phạm tính duy nhất (Unique violation) là một lỗi statement làm cho transaction hiện tại không thể sử dụng tiếp cho đến khi rollback:

```text
INSERT -> 23505
SELECT existing in same transaction -> 25P02 current transaction is aborted
```

Spring/Hibernate có thể bọc (wrap) nguyên nhân gốc thành `DataIntegrityViolationException`. Khối catch bên trong `@Transactional` không xóa trạng thái chỉ-rollback (rollback-only state).

Correct topology:

```text
outer coordinator (no transaction)
  -> insert attempt (REQUIRES_NEW, saveAndFlush)
  -> catch classified duplicate outside failed transaction
  -> reader transaction loads existing
```

## Exception classification

Không chỉ kiểm tra Java wrapper. Duyệt qua chuỗi nguyên nhân (cause chain) để xác nhận:

```text
SQLSTATE = 23505
constraint = uk_work_item_tenant_reference
```

Một vi phạm FK `23503`, check `23514`, not-null `23502` hoặc unique constraint khác phải được lan truyền (propagate) như một lỗi kỹ thuật/dữ liệu phù hợp.

Tên của constraint là một API vận hành; việc đổi tên khi migration cần đồng bộ hóa trình phân loại (classifier).

## Root cause theo layer

### PostgreSQL

Lỗi schema không có bất biến nghiệp vụ. MVCC cho phép cả thao tác đọc chưa tồn tại (absent reads) và thao tác chèn phân biệt (distinct inserts).

### Spring

Transaction proxy đảm bảo tính nguyên tử theo lần thử (attempt), nhưng việc ánh xạ ngoại lệ đặt sai ranh giới nếu catch bên trong transaction đã bị hủy.

### Hibernate

Việc trì hoãn flush (Deferred flush) làm xung đột xuất hiện muộn. Annotation trên Entity không bảo đảm rằng quá trình migration trên production đã thực sự tạo index.

### Application

Lệnh kiểm tra được dùng như một quyền sở hữu (claim) và primary key bị nhầm lẫn với business key.

## Commit, rollback và timeout

- Bên thắng commit: bên thua trùng lặp/không làm gì (no-op).
- Bên thắng rollback: bên chờ (waiter) có thể trở thành bên thắng.
- Bên thua bị lỗi `23505`: toàn bộ transaction của bên thua bị rollback, không có tác dụng phụ một phần (partial side effects).
- Quá thời gian chờ khóa (Lock timeout) khi đang chờ xung đột unique: kết quả chưa chắc là trùng lặp; cần rollback rồi thử lại/đọc theo hợp đồng.
- Quá thời gian phản hồi commit (Commit response timeout): phía gọi (caller) không biết commit có thành công hay không; cần tra cứu/phát lại (lookup/replay) bằng business key hoặc khóa lũy đẳng (idempotency key).

Không giữ lệnh gọi từ xa (remote call) bên trong transaction insert chỉ để "giữ quyền"; hãy commit quyền sở hữu bền bỉ trước quy trình (workflow) hoặc dùng outbox/cỗ máy trạng thái (state machine) phù hợp.

## Duplicate command và payload

DB-006 bảo vệ bản ghi logic duy nhất. Nếu cùng một tham chiếu bên ngoài đi kèm với tải trọng (payload) khác, hệ thống phải chọn:

- từ chối việc tái sử dụng key/sai lệch dấu vân tay;
- coi tham chiếu như khóa tài nguyên (resource key) và bỏ qua các trường có thể thay đổi;
- hoặc định nghĩa rõ ràng thao tác update/upsert.

Ghi đè âm thầm (Silently overwrite) hàng gốc bằng `DO UPDATE` có thể biến một yêu cầu tạo trùng lặp thành một sự thay đổi trái phép.

## Crash behavior

Sự cố (Crash) trước khi commit sẽ rollback thao tác insert; bên chờ có thể thắng. Sự cố sau khi commit nhưng trước khi phản hồi sẽ để lại một hàng bền bỉ; lần thử lại (retry) phải tìm thấy nó thay vì tạo ra hàng mới với key ngẫu nhiên.

Nếu sự kiện ở hạ nguồn cần được phát hành, hãy ghi vào outbox trong cùng một transaction thành công. Bản thân một hàng unique không làm cho cuộc gọi ra bên ngoài đạt được tính chính xác một lần (exactly-once).

## Multi-instance

Unique index nằm ở PostgreSQL chia sẻ, nên sẽ điều phối được App A, App B, công việc hàng loạt (batch) và thao tác SQL trực tiếp. Các cơ chế như `synchronized`, bản đồ (map) cục bộ hoặc cache chỉ bao phủ một nhóm nhỏ các chủ thể (actors).

Quyền truy cập cơ sở dữ liệu và xác thực migration cần ngăn chặn việc bỏ qua bảng/ràng buộc.

## Observability

Theo dõi:

```text
work_item.created
work_item.duplicate
unique_violation{constraint}
upsert_noop
unique_wait_duration
transaction_rollback
payload_mismatch
```

Reconciliation trước migration:

```sql
select tenant_id, external_reference, count(*)
from work_item
group by tenant_id, external_reference
having count(*) > 1;
```

Log tên ràng buộc, mã băm (hash) của khóa và correlation ID; tránh log trực tiếp tải trọng nhạy cảm (sensitive payload).
