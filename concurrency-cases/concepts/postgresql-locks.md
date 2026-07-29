# PostgreSQL locks và thời gian giữ khóa

## Mục đích

Tài liệu định nghĩa vocabulary chung cho row/table locks, lock wait, timeout và
lock lifetime trong PostgreSQL. Mỗi case vẫn phải chỉ ra invariant, statement
acquire lock và actor nào block/fail.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| row-level lock | Lock trên tuple/row để coordinate concurrent mutation |
| table-level lock | Lock mode trên relation; không đồng nghĩa khóa toàn bộ rows |
| lock wait | Backend chờ incompatible lock được release |
| lock holder | Transaction đang giữ lock |
| waiter | Transaction chưa acquire được lock |
| lock queue | Nhiều waiters xếp sau holder trên cùng resource |
| lock lifetime | Thời gian từ acquire đến transaction commit/rollback |
| `lock_timeout` | Giới hạn thời gian chờ acquire lock |
| `statement_timeout` | Giới hạn tổng thời gian statement chạy |
| deadlock detector | Cơ chế phát hiện wait-for cycle và abort victim |

## Row locks

`SELECT ... FOR UPDATE`, `UPDATE` và `DELETE` acquire row-level locks phù hợp.
Locks thường được giữ tới transaction end, không release khi Java method rời một
block hoặc khi remote call bắt đầu.

```sql
select *
from payment_order
where id = :id
for update;
```

Transaction khác muốn incompatible lock trên cùng row sẽ block, timeout hoặc bị
deadlock detector chọn làm victim.

> **Nói ngắn gọn:** lock duration là transaction duration; code chạy giữa SELECT
> và COMMIT kéo dài lock lifetime dù không chạm database.

## PostgreSQL visibility và wait

Ở `READ COMMITTED`, reader thường thấy committed tuple version theo statement
snapshot. Locking writer có thể chờ transaction trước kết thúc, sau đó predicate/
tuple được re-evaluate theo command semantics.

Một waiter vẫn thường giữ database connection và transaction resources trong lúc
chờ. Nhiều waiters vì vậy có thể làm connection pool cạn trước khi database đạt
giới hạn CPU.

## Table lock modes

PostgreSQL dùng nhiều table-level lock modes cho DML/DDL. DML bình thường acquire
relation locks tương thích với concurrent DML nhưng có thể conflict với schema
changes. Không suy luận “table lock” luôn chặn mọi query; phải nêu exact lock modes
và compatibility.

## Timeout layers

Các timeout giải quyết phạm vi khác nhau:

- pool acquisition timeout: chờ mượn connection từ application pool;
- `lock_timeout`: chờ database lock;
- `statement_timeout`: toàn statement;
- transaction/application deadline: toàn business unit;
- remote/client timeout: network operation ngoài database.

Timeout chỉ contain wait; nó không sửa transaction boundary hoặc capacity model.
Timeout phải nằm trong một overall deadline và chừa thời gian cleanup/response.

Sau PostgreSQL statement error, transaction có thể ở aborted state và phải
rollback trước retry. Retry phải dùng transaction mới khi policy cho phép.

## Deadlock khác starvation

Deadlock cần wait-for cycle. Pool starvation có thể xảy ra không có cycle: mọi
connection chỉ đang chờ remote I/O hoặc một lock holder chậm. Deadlock detector
không cứu được resource starvation không có cycle.

## Giảm lock lifetime

1. Đặt remote I/O và executor wait ngoài transaction.
2. Chỉ acquire lock ngay trước short database mutation.
3. Revalidate state sau khi acquire lock.
4. Dùng deterministic lock order khi khóa nhiều rows.
5. Áp dụng bounded timeout/deadline.
6. Không tăng pool size như giải pháp duy nhất.

## Quan sát

PostgreSQL:

```sql
select pid,
       state,
       wait_event_type,
       wait_event,
       xact_start,
       query_start,
       query
from pg_stat_activity
where datname = current_database();
```

`pg_locks`, `pg_blocking_pids(pid)` và application correlation ID giúp nối holder
với waiters. Pool metrics cần active, idle, pending acquisition, acquisition
timeout và connection usage duration.

Không log bind values nhạy cảm. Long `idle in transaction` sessions là tín hiệu
boundary/cleanup cần điều tra, không phải bằng chứng duy nhất của remote wait.

## Multi-instance

Mỗi application instance có pool riêng nhưng cùng tiêu thụ PostgreSQL connection
budget. Scale-out làm total potential connections tăng:

```text
total configured capacity
  = instances × max pool size per instance
```

Cần chừa capacity cho migrations, operations và failover behavior. Pool sizing
phải đi cùng admission control, transaction duration và database capacity.

## Liên kết

- [LOCK-003 — Pessimistic write lock với FOR UPDATE](../locking/pessimistic-write-for-update/README.md)
- [LOCK-004 — Conditional atomic UPDATE](../locking/conditional-atomic-update/README.md)
- [DB-007 — Row/table lock lifecycle](../postgresql/row-table-lock-lifecycle/README.md)
- [DB-008 — PostgreSQL opposite row order deadlock](../postgresql/opposite-row-order-deadlock/README.md)
- [DB-010 — Concurrent workers with SKIP LOCKED](../postgresql/skip-locked-work-queue/README.md)
- [SPR-007 — Long transaction pool exhaustion](../spring/connection-pool-long-transaction/README.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Deadlock và retry an toàn](deadlocks-and-retries.md)
- [Concurrency testing](concurrency-testing.md)
