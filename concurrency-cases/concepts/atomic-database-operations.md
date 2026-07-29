# Atomic database operations và conditional mutation

## Mục đích

Một mutation nguyên tử có điều kiện (`conditional atomic mutation`) đặt business
guard và state change trong cùng database statement. Pattern này phù hợp khi
invariant có thể biểu diễn trên current row values mà không cần application load
aggregate trước.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| atomic mutation | Predicate check và write không có race window ở giữa |
| conditional `UPDATE` | UPDATE chỉ áp dụng khi `WHERE` còn đúng |
| affected-row count | Số rows statement đã thay đổi |
| predicate recheck | Đánh giá lại search condition sau concurrent target-row update |
| current row version | Tuple version mà updating command thực sự mutate |
| no-op | Statement hoàn tất nhưng không row nào được update |
| `RETURNING` | PostgreSQL trả values từ rows vừa mutate |
| guarded delta | Cộng/trừ dựa trên current column với range predicate |
| bulk DML | Direct database mutation ngoài entity dirty checking |

## Mẫu cơ bản

```sql
update inventory_item
set available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
where product_id = :productId
  and :quantity > 0
  and available_quantity >= :quantity;
```

Contract:

```text
affected rows = 1 → intent được áp dụng
affected rows = 0 → intent không được áp dụng
```

Application bắt buộc xử lý cả hai branches. Nếu bỏ qua return value, guard có thể
đúng ở database nhưng API vẫn ghi side effect/report success sai.

> **Nói ngắn gọn:** `WHERE` là precondition tại thời điểm ghi; affected-row count
> là câu trả lời của database, không phải metadata tùy chọn.

## Current values thay stale absolute values

Không gửi:

```sql
update inventory_item
set available_quantity = :valueCalculatedFromOldRead
where product_id = :productId;
```

Ưu tiên gửi intent:

```sql
update inventory_item
set available_quantity = available_quantity - :quantity
where product_id = :productId
  and available_quantity >= :quantity;
```

Expression dùng database current row values. Guard và delta cùng nằm trong
statement.

## PostgreSQL `READ COMMITTED`

Concurrent UPDATE cùng target row vẫn dùng row-level locking:

1. actor A match predicate, acquire lock và update;
2. actor B tìm same target nhưng chờ A;
3. A commit: B xử lý updated row version và re-evaluate `WHERE`;
4. A rollback: B xử lý original row;
5. B affected `1` hoặc `0` theo current predicate.

Cơ chế này phù hợp simple condition trên predetermined row. Complex conditions
qua nhiều rows có thể không nhìn một globally consistent snapshot ở
`READ COMMITTED`.

`lock_timeout`, deadlock và serialization failure vẫn có thể xảy ra. Chúng là
statement/transaction failures, không phải affected rows `0`.

## `RETURNING`

Khi response cần post-mutation values:

```sql
update inventory_item
set available_quantity = available_quantity - :quantity,
    revision = revision + 1
where product_id = :productId
  and available_quantity >= :quantity
returning available_quantity, revision;
```

One returned row là winner state. Zero rows là no-op. `RETURNING` tránh SELECT
riêng để đoán value sau mutation và là PostgreSQL extension.

Affected-count API thường phù hợp với Spring Data `@Modifying`; result-set
mapping của `RETURNING` thường rõ hơn qua JDBC/jOOQ/custom native DAO.

## Outcome cardinality

Known-key command nên giữ:

```text
changed rows ∈ {0, 1}
```

Nếu `>1`, predicate/query model sai với aggregate boundary. Nếu `0`, application
phải biết những guard nào có thể fail:

- insufficient capacity;
- missing row;
- wrong state/tenant;
- stale token;
- invalid input chưa được validate.

Không map `0` thành một domain result chung nếu SQL contract không bảo đảm nghĩa
đó.

## Transaction composition

Một statement chỉ atomic cho mutation của nó. Business unit thường còn:

```text
command claim
→ conditional mutation
→ durable outcome/audit
→ outbox
→ commit
```

Đặt các database steps trong cùng transaction. Failure sau mutation phải rollback
counters và side effects. External calls/publish không tự tham gia PostgreSQL
transaction; dùng outbox/workflow phù hợp.

Lock do UPDATE giữ đến transaction end. Remote I/O sau atomic statement vẫn kéo
dài lock lifetime.

## Idempotency khác mutation safety

Conditional predicate ngăn distinct commands cùng vượt capacity. Nó không ngăn
same command được áp dụng lại khi state vẫn cho phép.

Unique command claim/replay bảo vệ duplicate command. Nó không ngăn hai unique
commands tranh cùng row. Robust operation thường cần cả hai.

## Constraints là defense in depth

```sql
check (available_quantity >= 0)
check (reserved_quantity >= 0)
check (available_quantity + reserved_quantity = on_hand_quantity)
```

Constraints chặn invalid values từ write paths khác. PostgreSQL `CHECK` chỉ nên
dựa trên current row, không dùng như cross-table assertion. Counter-versus-audit
vẫn cần atomic transaction và reconciliation.

## Spring Data JPA

Affected-count repository:

```java
@Modifying
@Query(value = """
        update inventory_item
        set available_quantity = available_quantity - :quantity
        where product_id = :productId
          and available_quantity >= :quantity
        """, nativeQuery = true)
int reserveIfEnough(long productId, int quantity);
```

Bulk/native DML bypasses managed entity state:

- persistence context có thể giữ stale entity;
- dirty flush/merge sau đó có thể overwrite direct mutation;
- entity callbacks/normal dirty checking không đại diện statement này.

Các lựa chọn:

1. không load target entity trong same transaction;
2. flush pending changes trước và clear sau nếu boundary sở hữu toàn context;
3. refresh exact entity khi cần tiếp tục dùng.

`clearAutomatically` và `flushAutomatically` là lifecycle tools, không thay thế
transaction design.

## Commit, rollback và retry

| Outcome | Ý nghĩa |
| --- | --- |
| affected `1`, commit | Mutation durable |
| affected `0`, commit | Durable no-op/rejection nếu outcome được lưu |
| affected `1`, later rollback | Mutation biến mất |
| `55P03` | Lock wait failure, transaction rollback |
| `40P01` | Deadlock victim, transaction rollback |
| `40001` | Serialization failure, transaction rollback |

Retry technical conflict chỉ khi command replayable, attempt mới dùng transaction
mới và budget/backoff/deadline bounded. Không retry business no-op máy móc.

## Multi-row mutations

Một UPDATE có thể match nhiều rows atomically ở statement level, nhưng:

- row-lock acquisition order/deadlock cần phân tích;
- affected count không còn `{0,1}`;
- partial business meaning của each row phải rõ;
- `UPDATE ... FROM` join phải không tạo nhiều source matches cho một target;
- response/order không tự được đảm bảo.

Nếu command yêu cầu all-or-nothing cho một known set, transaction có thể giữ toàn
statement atomic nhưng vẫn cần validate affected count bằng expected cardinality.

## Missing rows và predicate-wide invariant

Conditional UPDATE không tạo row và không lock missing target. Nó không tự bảo vệ
“không có row thỏa predicate” hoặc capacity tính từ child rows, trừ khi system có
stable guard/counter row làm authority.

Khi invariant trải rộng, cân nhắc:

- unique/check/exclusion constraint;
- stable guard row/counter;
- `SERIALIZABLE`;
- pessimistic lock protocol;
- schema redesign.

## Multi-instance

Predicate, current values, row lock và affected count nằm tại PostgreSQL primary,
nên mọi application instances cùng dùng một coordination boundary. JVM-local
mutex không có thuộc tính này.

Correctness cross-node không bảo đảm throughput. Hot row có thể tạo lock queue;
theo dõi affected `0`, wait latency, pool pressure và hot-key concentration.

## Chọn pattern

Conditional atomic SQL thường là lựa chọn nhỏ nhất khi:

- known target row;
- guard nằm trên same row/current values;
- mutation là delta/state transition;
- loser map được từ affected `0`;
- critical section cần ngắn.

Dùng `FOR UPDATE` khi Java decision cần nhiều state/steps trước write. Dùng
`@Version` khi aggregate edit phức tạp và conflict hiếm. Dùng constraint khi
database có thể biểu diễn invariant trực tiếp.

## Quan sát

- affected rows/returned rows theo operation;
- no-op rate tách khỏi exception rate;
- SQLSTATE `55P03`, `40P01`, `40001`;
- row-lock wait và transaction duration;
- reconciliation giữa projection và audit;
- duplicate replay/fingerprint mismatch;
- stale persistence-context incidents/tests.

Không log raw bind values nhạy cảm. Instrument domain outcome cùng database
outcome để phát hiện code bỏ qua affected count.

## Liên kết

- [LOCK-004 — Conditional atomic UPDATE](../locking/conditional-atomic-update/README.md)
- [DB-001 — Lost update dưới MVCC](../postgresql/lost-update-mvcc/README.md)
- [DB-004 — Phantom capacity check](../postgresql/phantom-capacity-check/README.md)
- [DB-007 — Row/table lock lifecycle](../postgresql/row-table-lock-lifecycle/README.md)
- [LOCK-003 — Pessimistic write lock](../locking/pessimistic-write-for-update/README.md)
- [PostgreSQL MVCC](postgresql-mvcc.md)
- [PostgreSQL locks](postgresql-locks.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Idempotency và uniqueness](idempotency-and-uniqueness.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
