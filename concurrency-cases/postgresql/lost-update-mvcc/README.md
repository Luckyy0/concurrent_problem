# DB-001 — Lost update dưới PostgreSQL MVCC

## Tóm tắt

Job `IMPORT-42` đã hoàn tất `10` units. Worker A báo thêm `3`; Worker B báo thêm
`4`. Cả hai Spring transactions load cùng JPA entity ở `completedUnits = 10`, tự
tính absolute values `13` và `14`, rồi Hibernate dirty checking flush hai UPDATE
chỉ có predicate `WHERE job_id = ?`.

A commit `13`; B commit sau và overwrite thành `14`. Delta `+3` của A biến mất dù
cả hai callers đều nhận success.

Invariant:

```text
final completedUnits
  = initial completedUnits + tổng delta của mọi accepted command

Với initial=10, deltaA=3, deltaB=4:
expected final = 17
```

> **Nói ngắn gọn:** MVCC cho phép cả hai actors đọc cùng committed version; nếu
> application ghi absolute stale value, PostgreSQL không biết cách merge deltas.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Worker A | Transaction cộng batch delta `3` |
| Worker B | Transaction cộng batch delta `4` |
| `job_progress` | Authoritative completed/total counters |
| Hibernate persistence context | Giữ entity snapshot và dirty-check absolute value |
| PostgreSQL | Cung cấp statement snapshots, row locks và commit visibility |

Initial row:

```text
job_id          = IMPORT-42
completed_units = 10
total_units     = 100
```

Broken final:

```text
completed_units = 14
both operations returned success
```

## Transaction boundary và contention point

Mỗi `addCompletedUnits()` call đi qua Spring proxy và tạo independent transaction
ở PostgreSQL `READ COMMITTED`.

Non-atomic sequence:

```text
SELECT current value
  -> calculate new absolute value in JVM
  -> UPDATE by primary key only
```

Contention point là row `job_progress(job_id='IMPORT-42')`. UPDATEs acquire row
lock, nên writes được serialize về thời gian; nhưng serialization của stale
absolute writes không bảo vệ additive invariant.

## Expected và actual

| | A | B | Final |
| --- | --- | --- | --- |
| Read | 10 | 10 | |
| Calculate | 13 | 14 | |
| Commit order | First | Second | |
| Expected merge | +3 | +4 | 17 |
| Broken absolute write | 13 | overwrite 14 | 14 |

Không exception, timeout hoặc affected-row conflict xảy ra: mỗi UPDATE match đúng
một row vì không có version/old-value predicate.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| lost update | Một committed write bị stale write sau ghi đè âm thầm |
| MVCC | Multi-Version Concurrency Control |
| statement snapshot | Snapshot cho từng statement ở `READ COMMITTED` |
| read-modify-write | Đọc, tính trong application, rồi ghi |
| dirty checking | Hibernate phát hiện managed entity đổi và sinh UPDATE |
| absolute write | Ghi value đã tính trước, ví dụ `SET completed_units = 14` |
| atomic delta | Database tự tính `SET completed_units = completed_units + 4` |
| version predicate | Điều kiện `WHERE version = expected` phát hiện stale write |

## Điều hướng

- [Broken JPA read-modify-write](broken-code.md)
- [MVCC and lost-update analysis](analysis.md)
- [Atomic, optimistic and pessimistic solutions](solutions.md)
- [Deterministic PostgreSQL experiments](experiments.md)
- [Atomic database operations](../../concepts/atomic-database-operations.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Isolation levels](../../concepts/isolation-levels.md)
- [Optimistic locking](../../concepts/optimistic-locking.md)
- [PostgreSQL locks](../../concepts/postgresql-locks.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Progress/counters thấp hơn tổng accepted work.
- Job completion trigger có thể không chạy hoặc chạy muộn.
- Audit log/worker responses nói success nhưng projection thiếu delta.
- Reconciliation phải rebuild counter từ event records nếu có.
- Nhiều application instances làm JVM-local lock vô hiệu.
- Tăng load tăng xác suất overwrite mà không tạo error signal.

## Hướng sửa khuyến nghị

Với commutative counter delta, dùng atomic conditional SQL:

```sql
update job_progress
set completed_units = completed_units + :delta
where job_id = :jobId
  and completed_units + :delta <= total_units;
```

Kiểm tra affected-row count để map success/rejection. Nếu aggregate mutation phức
tạp, dùng `@Version` cùng bounded retry trong transaction mới; hoặc `FOR UPDATE`
khi blocking trade-off phù hợp.

## Phạm vi

Đây là generic anomaly. Financial balance và ecommerce stock semantics nằm ở
`BANK-002` và `ECOM-001`; case này không dùng counter pattern để đơn giản hóa các
domain invariants đó.
