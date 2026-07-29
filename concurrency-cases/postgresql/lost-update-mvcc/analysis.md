# MVCC and lost-update analysis

## Initial state

```text
job_id          = IMPORT-42
completed_units = 10
total_units     = 100
effective isolation = read committed
```

Worker A và B xử lý disjoint batches; accepted deltas lần lượt là `3` và `4`.

## Expected versus actual

Expected additive invariant:

```text
10 + 3 + 4 = 17
```

Actual khi B commit stale absolute value cuối:

```text
A returned ProgressResult(before=10, after=13)
B returned ProgressResult(before=10, after=14)
database final completed_units=14
```

Cả results đều có vẻ hợp lệ cục bộ. Chỉ global committed invariant phát hiện delta
`3` đã mất.

## Mandatory two-actor timeline

| Bước | Worker A — Tx-A | Worker B — Tx-B |
| --- | --- | --- |
| T0 | BEGIN | |
| T1 | SELECT snapshot S-A thấy 10 | |
| T2 | | BEGIN |
| T3 | | SELECT snapshot S-B thấy 10 |
| T4 | Java tính 13 | Java tính 14 |
| T5 | UPDATE absolute 13; lock row | |
| T6 | COMMIT, version mới visible | |
| T7 | | UPDATE absolute 14; affected 1 |
| T8 | | COMMIT, overwrite 13 |
| T9 | | Final query thấy 14 |

Nếu T5 và B UPDATE overlap:

```text
A holds row lock
B UPDATE waits
A commits
B resumes/re-evaluates target row
B predicate WHERE job_id = ? vẫn true
B writes parameter 14
```

Row lock serialization không merge application deltas.

> **Nói ngắn gọn:** B chờ A không làm B quên value `14` đã tính; predicate không
> hỏi “row có còn ở state tôi đã đọc không?”.

## Snapshot của từng statement

Ở PostgreSQL `READ COMMITTED`, mỗi SELECT lấy statement snapshot:

- S-A thấy committed row `10`;
- S-B cũng thấy committed row `10` vì A chưa commit;
- không actor nào đọc uncommitted value;
- A commit tạo visible row version `13`;
- B không tự chạy lại business SELECT/calculation.

Nếu B thực hiện một SQL SELECT mới sau A commit, statement mới có thể thấy `13`.
Nhưng managed entity đã load trong cùng persistence context thường vẫn là cùng Java
object trừ khi refresh/clear/query behavior buộc reload. Correctness không nên dựa
vào accidental second read.

## PostgreSQL MVCC không tự phát hiện semantic staleness

Database nhận B statement:

```sql
update job_progress
set completed_units = 14,
    total_units = 100
where job_id = :id;
```

Từ PostgreSQL perspective:

- transaction hợp lệ;
- row tồn tại;
- primary-key predicate match;
- no constraint violated;
- affected-row count là `1`.

Database không biết `14` được tính từ earlier `10`, cũng không biết business intent
là `+4`. Không có conflict signal để Hibernate/Spring rollback.

## Hibernate dirty checking

Hibernate giữ loaded state:

```text
loaded completedUnits = 10
current Java state     = 14
```

Tại flush, dirty checking sinh absolute UPDATE. Không có `@Version`, Hibernate chỉ
dựa vào identifier để locate row.

`@DynamicUpdate` có thể giảm columns trong SET clause nhưng không thêm stale-state
predicate. Nó tránh overwrite unrelated unchanged columns trong một số mappings,
không bảo vệ same-column lost update.

## Root cause theo layer

| Layer | Behavior | Có phải bug layer đó? |
| --- | --- | --- |
| JVM | Mỗi actor tính đúng từ local entity `10` | Không tự coordinate DB actors |
| Spring | Mỗi call có physical transaction đúng | Transaction boundary riêng lẻ vẫn chưa đủ |
| Hibernate | Dirty-check và update by ID | Thiếu configured conflict detector |
| PostgreSQL MVCC | Cho concurrent reads, serialize row writes | Không biết semantic delta |
| Domain design | Intent `+delta` bị biểu diễn thành stale absolute write | Root design error |

Non-atomic operation là:

```text
read database state
  -> decide/calculate outside authoritative write
  -> unconditional absolute update
```

## Locks

Plain SELECT không acquire `FOR UPDATE` row lock. UPDATE acquire row lock và giữ
đến transaction end.

Lock bảo đảm two physical UPDATE operations không mutate tuple đồng thời. Nó không
bảo đảm second UPDATE được tính từ state sau first commit. Để dùng blocking
solution, lock phải được acquire trước read/calculate.

## Constraint behavior

Schema constraint:

```sql
check (
    completed_units >= 0
    and completed_units <= total_units
)
```

bảo vệ row range, nhưng lost delta example final `14` vẫn nằm trong range. Database
constraints bảo vệ invariants diễn đạt trong current row; chúng không reconstruct
accepted deltas không được lưu.

Một append-only progress event/command record có thể hỗ trợ reconciliation, nhưng
projection update vẫn cần atomicity/idempotency.

## Effective isolation

Đo trong đúng transaction:

```sql
select current_setting('transaction_isolation');
```

Expected broken experiment trả:

```text
read committed
```

Không suy luận isolation chỉ từ `@Transactional` vì datasource/outer boundary có
thể thay effective value.

## `REPEATABLE READ`

PostgreSQL `REPEATABLE READ` giữ transaction snapshot. Với same-row concurrent
update, một updater có thể bị abort bằng serialization failure thay vì silently
overwrite.

Điều đó đổi contract:

```text
one transaction commits
one transaction fails SQLSTATE 40001
application rolls back and may retry in new transaction
```

Không catch/retry bên trong failed transaction. Case SPR-006 mô tả retry boundary.

## `SERIALIZABLE`

`SERIALIZABLE` có thể ngăn anomaly bằng cách abort transaction không thể đặt vào
một serial order, nhưng:

- abort/retry là expected control path;
- toàn transaction phải idempotent/retry-safe;
- contention tăng retry rate;
- vẫn cần attempt/deadline limits;
- single-row additive mutation thường đơn giản hơn bằng atomic SQL.

Isolation cao hơn là credible option, không phải default answer cho mọi counter.

## Commit, rollback và timeout

Broken path:

- A commit durable `13`;
- B commit durable `14`;
- không transaction rollback;
- không exception ở caller.

Nếu B timeout/rollback trước commit, A `13` còn lại và B delta chưa accepted. Nếu
caller retry B, retry phải có idempotency/command identity để tránh duplicate khi
outcome ambiguous.

Atomic delta/optimistic conflict solutions phải map affected rows/exception thành
explicit success, retry hoặc rejection.

## Crash behavior

- Crash trước commit: PostgreSQL rollback connection transaction.
- Crash sau commit trước response: durable command/result cần cho outcome lookup.
- Crash không phải điều kiện tạo lost update; anomaly xảy ra với hai clean commits.
- Rebuild projection chỉ khả thi nếu accepted progress events/commands được lưu
  durable và deduplicated.

## Retry behavior

Retry unconditional broken service không sửa lần commit đã overwrite; nó có thể
cộng delta lần nữa hoặc overwrite tiếp. Retry chỉ hợp lệ khi:

- conflict được detect;
- failed attempt rollback hoàn tất;
- next attempt reload current state;
- command idempotency rõ;
- attempt/deadline bounded.

Atomic SQL thường không cần retry cho two positive deltas nếu predicate vẫn pass;
row-level execution serializes and composes them.

## Multi-instance

App A `synchronized` không coordinate App B. PostgreSQL row/predicate/version là
shared boundary duy nhất mà mọi instances/direct workers cùng tham gia.

Một distributed mutex bên ngoài database cũng cần failure/lease/fencing analysis
và thường phức tạp hơn atomic SQL cho single-row counter.

## Observability

Lost update im lặng nên exception metrics không giúp. Theo dõi:

- accepted command/event count và summed deltas;
- projection value/reconciliation drift;
- affected-row count cho conditional updates;
- optimistic/serialization conflict rate;
- aggregate key/hot-key distribution;
- effective isolation trong diagnostics;
- command ID replay/duplicate outcome.

Không log payload nhạy cảm. Reconciliation invariant tốt hơn raw SQL success count.
