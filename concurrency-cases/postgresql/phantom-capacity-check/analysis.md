# Phân tích MVCC predicate và capacity invariant

## Initial state

```text
processing_pool(P-42)
  capacity = 10

slot_allocation
  9 rows where pool_id=P-42 and status=ACTIVE
```

Request A và B là commands hợp lệ, khác request/allocation IDs. Mỗi command riêng
lẻ có thể dùng chỗ cuối cùng; cả hai cùng accepted thì sai.

## Hai timelines cần phân biệt

### Timeline 1 — Visible phantom ở `READ COMMITTED`

| Bước | Reader A | Writer B |
| --- | --- | --- |
| 1 | `BEGIN READ COMMITTED` | |
| 2 | COUNT snapshot S1 = `9` | |
| 3 | | INSERT một ACTIVE row |
| 4 | | `COMMIT` |
| 5 | COUNT snapshot S2 = `10` | |

Result set thay đổi vì row B mới commit thỏa cùng predicate. Đây là phantom row
nhìn thấy khi A query lại.

### Timeline 2 — Capacity violation

| Bước | Allocator A | Allocator B |
| --- | --- | --- |
| 1 | BEGIN | BEGIN |
| 2 | COUNT = 9 | |
| 3 | | COUNT = 9 |
| 4 | quyết định ACCEPT | quyết định ACCEPT |
| 5 | INSERT row A | INSERT row B |
| 6 | COMMIT | COMMIT |
| 7 | | final active = 11 |

Capacity race không cần một actor query lại. Root cause là hai check decisions đọc
cùng predicate state rồi ghi hai rows không conflict.

> **Nói ngắn gọn:** phantom mô tả result set thay đổi; invariant violation mô tả
> hai decisions cùng dựa trên “chỗ trống” không được đại diện bằng một
> authoritative write conflict.

## Predicate là shared state thật sự

Application nhìn capacity qua:

```text
COUNT(rows matching pool=P-42 AND status=ACTIVE) < pool.capacity
```

Không có một physical “slot còn lại” row trong broken model. PostgreSQL chỉ thấy:

- A đọc nhiều tuple versions;
- B đọc nhiều tuple versions;
- A insert một new tuple;
- B insert một new tuple.

Primary keys khác nhau nên row/index locks không xung đột. `CHECK` trên một
allocation row cũng không thể tự query aggregate rows khác.

## MVCC snapshots ở `READ COMMITTED`

Mỗi COUNT statement lấy snapshot lúc statement bắt đầu. Nếu cả hai COUNT hoàn tất
trước bất kỳ INSERT commit nào:

```text
A snapshot SA -> 9 committed ACTIVE rows
B snapshot SB -> same 9 committed ACTIVE rows
```

A uncommitted INSERT không visible cho B plain SELECT. Sau A commit, một new
statement của B có thể thấy row A, nhưng broken service đã ra quyết định từ count
cũ và không revalidate.

## Lock behavior

### Plain predicate read

COUNT không acquire a row lock để ngăn inserts. PostgreSQL table-level
`ACCESS SHARE` lock của SELECT chỉ bảo vệ schema-level compatibility; nó không
serialize application inserts.

### INSERT

Mỗi INSERT giữ locks cho row/index entries của chính nó tới commit/rollback.
Different primary/request keys không conflict.

### Lock existing rows

`SELECT ... FOR UPDATE` lock những rows đã tồn tại và được chọn. New qualifying
row chưa tồn tại nên không có row identity để lock. Một shared parent row hoặc
pre-created slot rows mới tạo điểm contention hữu hình.

## PostgreSQL `REPEATABLE READ` nuance

PostgreSQL `REPEATABLE READ` dùng một transaction snapshot và không cho repeated
predicate query thấy phantom đã commit sau snapshot:

```text
A COUNT #1 -> 9
B INSERT + COMMIT
A COUNT #2 -> vẫn 9
```

Nhưng hai allocators có thể cùng snapshot count `9`, rồi insert different rows.
Snapshot isolation có serializability anomaly vì mỗi transaction writes dựa trên
predicate read, nhưng không có same-row write conflict buộc abort.

Do đó:

```text
“Không thấy phantom khi đọc lại”
không suy ra
“aggregate capacity luôn đúng”.
```

## PostgreSQL `SERIALIZABLE` và SSI

Serializable Snapshot Isolation (`SSI`) theo dõi read/write dependencies bằng
predicate-style `SIReadLock` metadata. Các locks này thường không block writer như
`FOR UPDATE`; chúng giúp PostgreSQL phát hiện dangerous dependency structure.

Trong capacity race:

```text
A reads predicate excluding future B row
B inserts row matching A predicate

B reads predicate excluding future A row
A inserts row matching B predicate
```

Hai rw-antidependencies tạo cycle/dangerous structure. PostgreSQL abort ít nhất một
transaction với serialization failure, SQLSTATE `40001`, để committed outcome có
serial order hợp lệ.

Application contract:

- loser transaction đã rollback;
- retry phải bắt đầu physical transaction mới;
- reload count/current state;
- rerun toàn decision;
- retry bounded với backoff/jitter;
- nếu capacity đã full, return domain `FULL`, không tiếp tục retry.

Không dựa vào exact transaction nào sẽ abort.

## Root cause theo layer

### PostgreSQL

Ở `READ COMMITTED`/`REPEATABLE READ`, different inserts không tạo conflict bảo vệ
aggregate predicate. Đây là permitted concurrency behavior.

### Spring

`@Transactional` xác định atomic commit boundary nhưng default isolation không
biến `count -> compare -> insert` thành một statement/lock protocol.

Inner method khai báo `SERIALIZABLE` nhưng join outer `REQUIRED` transaction vẫn
dùng effective isolation của outer transaction.

### Hibernate

`save()` đưa entity vào persistence context; INSERT có thể xảy ra ở flush/commit.
Hibernate không biết pool capacity để tự thêm predicate hoặc parent lock.

### Application model

Capacity chỉ tồn tại như phép tính từ child rows. Không có authoritative counter,
lock row, finite slot identity hoặc serializable retry boundary.

## Expected và actual

Expected:

```text
accepted=1
full/retried=1
final active=10
```

Broken:

```text
accepted=2
full=0
final active=11
no exception
```

## Commit và rollback

### Một INSERT rollback

Row không visible và final active có thể còn `10`. Nhưng correctness không được
phụ thuộc vào một actor tình cờ fail.

### Cả hai commit

Violation durable. Transaction atomicity không có bước global validation sau
commit để tự sửa.

### Counter claim rồi allocation INSERT fail

Ở correct atomic-counter solution, cả counter increment và INSERT phải ở cùng
transaction. Runtime/DataAccess exception rollback cả hai; nếu exception bị catch
và nuốt, counter có thể drift.

### Release

Release phải transition allocation `ACTIVE -> RELEASED` conditionally rồi giảm
counter đúng một lần trong cùng transaction. Duplicate release không được
double-decrement.

## Timeout và deadlock

Parent-row locking hoặc atomic counter làm contenders chờ cùng row. Cần:

- short transaction;
- bounded `lock_timeout`/query timeout;
- không remote I/O khi giữ lock;
- deterministic parent lock order nếu một request dùng nhiều pools;
- retry deadlock/serialization trong transaction mới.

Hot pool có thể tạo queue và connection-pool pressure. `SKIP LOCKED` trên
pre-created slots có thể giảm wait nhưng loser phải hiểu “no row available”.

## Crash và duplicate

Crash trước commit rollback claim/allocation. Crash sau commit nhưng trước response
khiến caller retry; unique `(pool_id, request_id)` hoặc idempotency record phải
replay accepted result.

Duplicate prevention không thay capacity safety:

```text
same request twice          -> uniqueness/idempotency
two distinct requests race -> capacity coordination
```

## Multi-instance

A và B có thể chạy ở hai pods, scheduled batch và admin endpoint. JVM monitor chỉ
serialize code path trong một process. PostgreSQL phải là coordination boundary
vì pool/allocation state là shared ở đó.

Một local cache của active count còn yếu hơn: invalidation delay làm predicate
stale ngay cả khi requests không overlap chính xác.

## Observability

Business metrics:

```text
pool.capacity
pool.used_slots
pool.active_allocation_count
allocation.accepted
allocation.full
allocation.duplicate
capacity.invariant_violation
```

Database metrics:

```text
conditional_update_affected_zero
lock_wait_duration
lock_timeout
serialization_failure_40001
deadlock_40P01
retry_attempt
transaction_duration
```

Theo dõi `pg_stat_activity`, `pg_locks` và `pg_stat_database_conflicts` phù hợp,
nhưng reconciliation query `active_count <= capacity` mới trực tiếp kiểm tra
business invariant.
