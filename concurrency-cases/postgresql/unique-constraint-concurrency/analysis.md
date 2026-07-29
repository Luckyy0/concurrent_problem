# Phân tích absent key, unique index và flush

## Initial state

```text
business key = (tenant T-42, reference CASE-9001)
matching rows = 0
```

A/B có different request threads, connections, transactions và random entity IDs.

## Expected theo broken design

Developer kỳ vọng:

```text
A checks absent -> A owns right to insert
B checks later -> B sees A and returns duplicate
```

Không có mechanism bảo đảm B check sau A commit. Scheduler có thể đặt both checks
trước either insert.

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

Expected final count `1`; actual `2`. Không exception hay affected-row conflict.

> **Nói ngắn gọn:** absence là observation, không phải ownership; unique index mới
> biến business key thành object mà concurrent INSERTs phải tranh chấp.

## MVCC và lock behavior không constraint

Plain existence SELECT:

- đọc committed snapshot;
- không thấy future/uncommitted row;
- không row-lock một key chưa tồn tại;
- table `ACCESS SHARE` lock không chặn ordinary INSERT.

A/B INSERTs lock different heap rows và primary-key index entries. Vì random IDs
khác, PostgreSQL không biết hai rows đại diện cùng logical object.

`REPEATABLE READ` còn giữ absent snapshot ổn định. `SERIALIZABLE` có thể phát hiện
predicate conflict, nhưng unique constraint là direct, cheaper invariant
representation và bảo vệ mọi isolation levels/mutation clients.

## Khi unique constraint tồn tại

```sql
alter table work_item
add constraint uk_work_item_tenant_reference
unique (tenant_id, external_reference);
```

PostgreSQL unique-index arbitration:

```text
A INSERT key K -> index claim, uncommitted
B INSERT key K -> waits for A transaction outcome
```

Nếu A commit:

```text
B wakes -> duplicate key -> SQLSTATE 23505 -> B transaction aborted
```

Nếu A rollback:

```text
A index/row effects disappear
B can continue INSERT -> B becomes winner
```

Exact internal locking/speculative insertion details phụ thuộc statement form,
nhưng application contract là one durable winner.

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

Ở `READ COMMITTED`, conflict arbitration có thể dựa trên row chưa visible trong
statement snapshot. Sau `DO NOTHING` empty, một subsequent SELECT statement mới
thường thấy winner đã commit.

Ở `REPEATABLE READ`, concurrent conflict có thể được chuyển thành serialization
failure, hoặc stable snapshot không phù hợp để đọc winner mới. Application phải
handle `40001`; path đọc existing nên dùng transaction/snapshot mới thay vì assume
same-snapshot visibility.

## Hibernate persistence context

`save()` không đồng nghĩa SQL executed. Conflict có thể surface:

- `saveAndFlush()`;
- explicit `EntityManager.flush()`;
- query-triggered autoflush;
- transaction commit.

Nếu response được xây trước commit, caller chỉ thực sự nhận response sau Spring
interceptor commit; commit exception thay đổi outcome.

## PostgreSQL aborted transaction

Unique violation là statement error làm current transaction unusable tới rollback:

```text
INSERT -> 23505
SELECT existing in same transaction -> 25P02 current transaction is aborted
```

Spring/Hibernate có thể wrap root cause thành `DataIntegrityViolationException`.
Catch block bên trong `@Transactional` không xóa rollback-only state.

Correct topology:

```text
outer coordinator (no transaction)
  -> insert attempt (REQUIRES_NEW, saveAndFlush)
  -> catch classified duplicate outside failed transaction
  -> reader transaction loads existing
```

## Exception classification

Không chỉ kiểm tra Java wrapper. Traverse cause chain để xác nhận:

```text
SQLSTATE = 23505
constraint = uk_work_item_tenant_reference
```

Một FK violation `23503`, check `23514`, not-null `23502` hoặc different unique
constraint phải propagate như technical/data error phù hợp.

Constraint name là operational API; migration rename cần đồng bộ classifier.

## Root cause theo layer

### PostgreSQL

Broken schema không có business invariant. MVCC cho both absent reads và distinct
inserts.

### Spring

Transaction proxy atomic theo attempt, nhưng exception mapping đặt sai boundary
nếu catch trong aborted transaction.

### Hibernate

Deferred flush làm conflict xuất hiện muộn. Entity annotation không bảo đảm
production migration đã tạo index.

### Application

Check được dùng như claim và primary key bị nhầm với business key.

## Commit, rollback và timeout

- Winner commit: loser duplicate/no-op.
- Winner rollback: waiter có thể become winner.
- Loser `23505`: whole loser transaction rollback, không partial side effects.
- Lock timeout while waiting unique conflict: outcome chưa chắc duplicate; rollback
  rồi retry/read theo contract.
- Commit response timeout: caller không biết commit thành công; lookup/replay bằng
  business/idempotency key.

Không giữ remote call trong insert transaction chỉ để “giữ claim”; commit durable
claim trước workflow hoặc dùng outbox/state machine phù hợp.

## Duplicate command và payload

DB-006 bảo vệ unique logical record. Nếu same external reference đi kèm payload
khác, system phải chọn:

- reject key reuse/fingerprint mismatch;
- treat reference as resource key and ignore mutable fields;
- hoặc define explicit update/upsert.

Silently overwrite original row bằng `DO UPDATE` có thể biến duplicate create
thành unauthorized mutation.

## Crash behavior

Crash trước commit rollback insert; waiter có thể win. Crash sau commit trước
response leaves durable row; retry must find it instead of create new random-key
row.

Nếu downstream event cần publish, write outbox trong same successful transaction.
Unique row một mình không làm external call exactly-once.

## Multi-instance

Unique index nằm ở shared PostgreSQL, nên coordinate App A, App B, batch và direct
SQL. `synchronized`, local map hoặc cache chỉ cover subset actors.

DB permissions và migration validation cần ngăn bypass table/constraint.

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

Log constraint name, scoped key hash và correlation ID; tránh log raw sensitive
payload.
