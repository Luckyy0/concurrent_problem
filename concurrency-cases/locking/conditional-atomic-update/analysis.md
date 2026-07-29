# Phân tích — predicate và mutation phải là một operation

## Initial state

```text
inventory_item(product_id=77)
  on_hand_quantity   = 5
  available_quantity = 5
  reserved_quantity  = 0
  revision           = 10

inventory_reservation = []
```

Command A và B là hai orders hợp lệ, mỗi command giữ `4`. Chúng chạy trên hai
application instances, connections và transactions độc lập.

## Timeline read–check–write

| Bước | Tx-A / App-1 | Tx-B / App-2 | PostgreSQL |
| --- | --- | --- | --- |
| 1 | plain SELECT → `5/0` | | Statement snapshot A |
| 2 | | plain SELECT → `5/0` | Statement snapshot B |
| 3 | Java check pass | Java check pass | Database chưa thấy business guard |
| 4 | tính absolute `1/4` | tính absolute `1/4` | |
| 5 | insert reservation A | insert reservation B | Distinct command IDs |
| 6 | UPDATE row → `1/4` | | A giữ row lock |
| 7 | commit | UPDATE waits rồi ghi `1/4` | B predicate chỉ là primary key |
| 8 | | commit | Cả UPDATEs affected rows `1` |

Final row vẫn thỏa constraints, nhưng hai durable reservations tổng `8`. Đây vừa
là lost update trên counter, vừa là accepted-decision inconsistency.

## Expected và actual

| | Expected | Actual |
| --- | --- | --- |
| Accepted commands | 1 | 2 |
| Reserved audit sum | 4 | 8 |
| `reserved_quantity` | 4 | 4 |
| `available_quantity` | 1 | 1 |
| Reconciliation | Khớp | Lệch `4` |
| Loser outcome | `OUT_OF_STOCK` | `RESERVED` |

## Atomic statement

```sql
update inventory_item
set available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity,
    revision = revision + 1
where product_id = :productId
  and :quantity > 0
  and available_quantity >= :quantity
returning available_quantity, reserved_quantity, revision;
```

`WHERE` là business guard; `SET` là business intent. Cả hai được PostgreSQL xử lý
như một updating command trên current row version.

## Timeline conditional UPDATE

| Bước | Tx-A | Tx-B | PostgreSQL |
| --- | --- | --- | --- |
| 1 | UPDATE quantity `4` | | Predicate `5 >= 4`, A update → `1/4` |
| 2 | giữ transaction mở | UPDATE quantity `4` | B wait trên row |
| 3 | commit | wait | A release lock |
| 4 | | predicate recheck | Current row có available `1` |
| 5 | | zero returned rows | `1 >= 4` false |
| 6 | | lưu `OUT_OF_STOCK`, commit | Không counter mutation B |

> **Nói ngắn gọn:** actor B không thất bại vì application “đọc lại”; chính
> updating statement đánh giá guard trên row đã được A commit.

## `READ COMMITTED` và predicate recheck

PostgreSQL tìm candidate row theo statement snapshot. Nếu một concurrent
transaction đã update/lock row đó, updating command chờ holder commit/rollback.

### Holder commit

PostgreSQL thử áp operation lên updated row version và đánh giá lại `WHERE`.
Nếu predicate còn đúng, B update current values. Nếu predicate sai, B bỏ qua row
và affected-row count là `0`.

### Holder rollback

A effects bị hủy. B xử lý originally found row với available `5`; predicate pass,
B update `1/4` và affected rows `1`.

Điều này phù hợp cho simple condition trên một predetermined row. Không suy rộng
sang complex predicate/join nhiều rows: một statement ở `READ COMMITTED` có thể
thấy concurrent effects trên target row mà không thấy một snapshot đồng nhất của
mọi row khác.

## Row lock vẫn tồn tại

“Lock-free application code” không có nghĩa database không lock. `UPDATE` acquire
row-level lock khi target row match. Competitor có thể:

- wait rồi re-evaluate predicate;
- vượt `lock_timeout` và fail SQLSTATE `55P03`;
- tham gia wait-for cycle và thành deadlock victim `40P01`.

Khác với pre-read `FOR UPDATE`, conditional statement chỉ acquire row lock trong
mutation round trip. Lock vẫn được giữ đến transaction commit/rollback, nên
remote I/O sau UPDATE vẫn kéo dài contention.

## Affected-row count là protocol

PostgreSQL command tag `UPDATE n` và Spring Data modifying return value không chỉ
là metric:

```text
n = 1 → exactly one known product row passed guard and was mutated
n = 0 → no row was mutated; map theo documented domain contract
n > 1 → query/cardinality bug với single-product command
```

`0` không phải exception. Caller phải branch trước khi tạo accepted reservation
hoặc outbox.

Trigger có thể thay đổi affected-row behavior, và một broad predicate có thể
match nhiều rows. Production schema/query review phải giữ single-row cardinality
assumption rõ ràng.

## `RETURNING` tránh read-after-write ambiguity

`UPDATE ... RETURNING` trả new values của chính rows được update:

```text
one returned row → RESERVED cùng remaining quantities
zero returned rows → no mutation
```

Một SELECT riêng sau UPDATE có statement snapshot mới và có thể thấy changes
khác. Dùng `RETURNING` khi response cần exact post-mutation state của winner.

JPA bulk-update API thường phù hợp với affected count. Với PostgreSQL
`RETURNING`, custom JDBC/native DAO cho result mapping rõ hơn.

## Zero row không luôn là `OUT_OF_STOCK`

Cùng `0` có thể đại diện:

- product missing;
- invalid quantity;
- product disabled/tenant mismatch;
- insufficient available;
- optimistic token mismatch.

Case validate `quantity > 0` trước SQL và command được tạo từ product row ổn
định. Vì vậy zero row được map `OUT_OF_STOCK`. Nếu product có thể bị delete hoặc
API phải phân biệt, cần thêm outcome design:

- foreign key/lifecycle bảo đảm target tồn tại;
- separate read chỉ để phân loại sau no-op, không để quyết định mutation;
- stored procedure/function trả typed outcome;
- hoặc chấp nhận một combined `NOT_AVAILABLE` contract.

Không thêm `exists → update` rồi dùng exists làm correctness guard.

## Persistence context của Hibernate

Native/JPQL bulk DML chạy trực tiếp ở database, không đi qua managed entity dirty
checking. Nếu persistence context đã chứa `InventoryItem(77)`:

- object vẫn có old counters;
- response đọc object có thể stale;
- later dirty flush/merge có thể overwrite, tùy dirty fields/mapping;
- entity callbacks không tự chạy như entity-state transition.

Ba thiết kế an toàn:

1. primary mutation DAO không load inventory entity trong transaction;
2. flush pending changes trước bulk DML rồi clear persistence context sau đó;
3. refresh exact entity nếu thật sự cần dùng tiếp.

`clearAutomatically=true` có thể detach mọi managed entities và làm mất pending
changes nếu chưa flush. Vì vậy không bật flag máy móc; transaction/service design
phải sở hữu persistence-context lifecycle.

## Transaction composition

Correctness không dừng ở counter:

```text
BEGIN
  claim command ID
  conditional stock UPDATE
  write RESERVED/OUT_OF_STOCK outcome
  write outbox only for RESERVED
COMMIT
```

Nếu reservation/outbox insert fail sau UPDATE, transaction rollback hoàn nguyên
counter. Nếu code catch database error rồi tiếp tục cùng aborted transaction,
atomic composition bị hiểu sai.

Publish external message trước commit tạo dual-write gap. Outbox row trong cùng
transaction giữ database state và publication intent nhất quán.

## Duplicate command

Conditional stock predicate ngăn oversell giữa distinct orders nhưng không ngăn
same request decrement hai lần khi stock vẫn đủ.

Durable command claim dùng unique `command_id`:

```sql
insert into inventory_reservation (...)
values (...)
on conflict (command_id) do nothing
returning command_id;
```

Concurrent duplicate insert được unique index arbitrate. Same fingerprint replay
stored outcome; different fingerprint reject. Claim và final outcome nằm trong
cùng transaction để không commit permanent `PROCESSING` row.

Idempotency và mutation safety bổ sung nhau, không thay thế nhau.

## Commit, rollback và ambiguous response

| Failure point | Database outcome | Caller recovery |
| --- | --- | --- |
| Trước UPDATE | Không stock change | Có thể fresh attempt |
| Sau UPDATE, trước commit | Rollback counter/claim/outbox | Replay stable command |
| Commit thành công, response mất | Reservation vẫn durable | Replay stored outcome |
| Affected rows `0` | Rejection row có thể commit | Replay `OUT_OF_STOCK` |
| Lock timeout/deadlock | Transaction rollback | Fresh bounded retry nếu policy cho phép |

Không thể suy ra “response mất = rollback”. Stable command ID phân giải outcome
sau crash/network failure.

## Isolation khác

Primary contract dùng PostgreSQL `READ COMMITTED`. Ở `REPEATABLE READ`, concurrent
target-row update có thể tạo serialization failure `40001` thay vì wait rồi
affected rows `0`. Nếu application đổi isolation, test/outcome/retry contract
phải đổi theo.

`SERIALIZABLE` giải quyết lớp anomaly rộng hơn nhưng vẫn yêu cầu xử lý `40001`.
Không tăng isolation chỉ để thay một predicate có thể viết rõ trong UPDATE.

## Constraint là lớp phòng thủ

```sql
check (available_quantity >= 0)
check (reserved_quantity >= 0)
check (available_quantity + reserved_quantity = on_hand_quantity)
```

Constraints chặn invalid row states từ mọi write path. Chúng không tự reconcile
counter với reservation rows ở table khác; PostgreSQL `CHECK` không phải
cross-table assertion.

Reconciliation job vẫn cần so:

```sql
select i.product_id,
       i.reserved_quantity,
       coalesce(sum(r.quantity) filter (where r.outcome = 'RESERVED'), 0)
from inventory_item i
left join inventory_reservation r using (product_id)
group by i.product_id, i.reserved_quantity
having i.reserved_quantity
       <> coalesce(sum(r.quantity) filter (where r.outcome = 'RESERVED'), 0);
```

## Multi-instance

Cả App-1 và App-2 gửi statement tới cùng PostgreSQL primary. Row lock, current
version, predicate và affected count nằm ở authoritative boundary, nên không phụ
thuộc JVM nào xử lý request.

Scale-out có thể tăng waiters trên hot product. Correctness vẫn giữ nhưng
throughput/tail latency có thể giảm; connection pool, lock wait và no-op rate cần
được đo.

## Root cause theo layer

### Application

Business guard và mutation bị tách bởi race window; absolute state được tính từ
stale read.

### Spring

`@Transactional` không hợp nhất hai SQL statements thành một atomic condition.
Return value từ modifying query bị bỏ qua.

### Hibernate/JPA

Managed entity dirty checking ghi snapshot-derived values. Bulk DML lại có nguy
cơ stale persistence context nếu application trộn hai model.

### PostgreSQL

Plain SELECT cho hai transactions thấy cùng committed tuple. Unconditional
UPDATEs đều hợp lệ. Guarded UPDATE mới cung cấp predicate recheck/affected-row
outcome trên current target row.

## Observability

Theo dõi:

- conditional attempts, affected rows `1` và `0`;
- returned remaining quantity;
- row-lock wait duration, SQLSTATE `55P03`, `40P01`, `40001`;
- transaction duration sau successful UPDATE;
- duplicate replay/fingerprint mismatch;
- reconciliation mismatch;
- pool active/pending và hot product IDs đã hash/bucket hóa.

Affected rows `0` tăng có thể là stock pressure bình thường; lock wait tăng là
contention signal khác. Không gộp hai metric.

## Scope boundary

Một known product row chứa counter cần thiết, nên invariant diễn đạt bằng một
predicate. Nếu capacity được tính từ tập child rows hoặc new rows có thể xuất
hiện, conditional UPDATE chỉ đúng khi có stable authoritative counter/guard row
và reconciliation. Predicate-wide design thuộc các isolation/constraint cases.
