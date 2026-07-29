# PostgreSQL MVCC và row visibility

## Mục đích

Tài liệu định nghĩa vocabulary chung cho tuple versions, snapshots và visibility
trong PostgreSQL. Các case cụ thể vẫn phải chỉ ra statement nào đọc version nào,
write conflict/lock nào xảy ra và invariant nào bị ảnh hưởng.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| MVCC | Multi-Version Concurrency Control, duy trì nhiều row versions |
| tuple version | Physical row version có transaction visibility metadata |
| snapshot | Quy tắc xác định transactions/tuple versions visible |
| statement snapshot | Snapshot mới cho mỗi statement ở `READ COMMITTED` |
| transaction snapshot | Snapshot ổn định hơn ở `REPEATABLE READ` |
| committed version | Tuple version từ transaction đã commit và visible |
| dead tuple | Old/deleted version không còn visible cho active snapshots phù hợp |
| vacuum | Reclaim/maintain dead tuples khi không còn snapshot cần chúng |
| read-modify-write | Application đọc value, tính value mới, rồi ghi lại |

## Tuple versions, không phải in-place read illusion

PostgreSQL lưu visibility metadata như transaction IDs trên tuple versions. Một
UPDATE tạo row version mới về mặt MVCC; old version còn tồn tại cho snapshots cần
nó cho tới khi có thể cleanup.

Application vẫn nhìn một logical row. Snapshot quyết định version nào visible.

> **Nói ngắn gọn:** nhiều transactions có thể cùng đọc một committed version cũ;
> MVCC giảm read/write blocking nhưng không tự merge business calculations.

## `READ COMMITTED`

Mỗi statement lấy snapshot tại statement start:

```text
SELECT #1 -> snapshot S1
concurrent transaction commits
SELECT #2 -> snapshot S2, có thể thấy version mới
```

PostgreSQL không cho dirty read; `READ UNCOMMITTED` được xử lý như `READ
COMMITTED`.

Một plain SELECT không acquire row lock để dành quyền update sau. Hai transactions
có thể cùng đọc value `10`, tính absolute values khác nhau và lần lượt write.

## UPDATE, row locks và predicate recheck

UPDATE acquire row-level lock. Nếu target row đang bị transaction khác update,
statement có thể wait. Sau transaction kia kết thúc, PostgreSQL xử lý current row
version và re-evaluate command predicate theo isolation semantics.

Atomic delta:

```sql
update job_progress
set completed_units = completed_units + :delta
where job_id = :jobId;
```

Waiter tính expression trên current row version khi UPDATE thực thi, nên concurrent
deltas có thể compose.

Application absolute write:

```sql
update job_progress
set completed_units = :valueComputedEarlier
where job_id = :jobId;
```

Predicate chỉ match ID; PostgreSQL không biết `:valueComputedEarlier` dựa trên
stale read và không tự merge delta.

## Anomalies

MVCC visibility/isolation liên quan đến:

- lost update từ unprotected read-modify-write;
- non-repeatable read ở statement snapshots;
- phantom/predicate changes;
- write skew trên nhiều rows/predicates;
- serialization failure ở stronger isolation.

Không gom mọi anomaly thành “race condition”. Mỗi case phải nêu concrete
interleaving, snapshot và write predicate.

## Isolation cao hơn

PostgreSQL `REPEATABLE READ` giữ transaction snapshot và có thể abort concurrent
updater thay vì silently overwrite same-row update. `SERIALIZABLE` còn phát hiện
serialization anomalies bằng predicate/dependency tracking.

Stronger isolation đổi failure mode thành abort/retry; nó không làm operation
idempotent và không loại requirement về new transaction per retry.

## Long-lived snapshots

Long transactions giữ snapshots/resources lâu, làm dead tuple cleanup bị trì hoãn
và tăng operational pressure. Không bao quanh remote I/O hoặc unbounded wait bằng
database transaction chỉ để “giữ snapshot”.

## Chọn coordination mechanism

- Atomic conditional/delta SQL cho single-row mutation diễn đạt được ở database.
- Optimistic `@Version` để phát hiện stale aggregate write.
- `FOR UPDATE` khi blocking/serialization trên row là phù hợp.
- Constraint để enforce invariant độc lập application race.
- `SERIALIZABLE` với bounded full-transaction retry cho predicate invariants.

Chọn từ business invariant, contention và failure contract.

## Kiểm thử

PostgreSQL Testcontainers test cần:

1. tạo transactions/connections độc lập;
2. điều phối actors cùng đọc một known version;
3. kiểm soát commit/write order;
4. assert effective isolation;
5. assert final committed business state;
6. không dùng H2 làm bằng chứng MVCC.

## Quan sát

`pg_stat_activity`, `pg_locks`, transaction age và SQL/affected-row metrics giúp
phân tích. System columns như `xmin` hữu ích cho diagnostics nhưng không nên thay
một explicit business/JPA version contract một cách tùy tiện.

## Liên kết

- [DB-001 — Lost update under MVCC](../postgresql/lost-update-mvcc/README.md)
- [LOCK-001 — Optimistic locking with @Version](../locking/optimistic-version-conflict/README.md)
- [DB-002 — Dirty-read expectations](../postgresql/dirty-read-expectation/README.md)
- [DB-003 — Non-repeatable read](../postgresql/non-repeatable-read/README.md)
- [DB-004 — Phantom capacity check](../postgresql/phantom-capacity-check/README.md)
- [DB-005 — Write skew](../postgresql/write-skew/README.md)
- [DB-007 — Row/table lock lifecycle](../postgresql/row-table-lock-lifecycle/README.md)
- [Isolation levels](isolation-levels.md)
- [PostgreSQL locks](postgresql-locks.md)
- [Optimistic locking](optimistic-locking.md)
- [Concurrency testing](concurrency-testing.md)
