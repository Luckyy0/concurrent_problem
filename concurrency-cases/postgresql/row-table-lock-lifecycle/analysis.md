# Phân tích lock acquisition, visibility và release

## Initial state

```text
tenant_quota(T-42)
  quota    = 10
  revision = 5
```

Ba actors có physical transactions/connections riêng.

## Timeline 1 — Plain SELECT không reserve row

| Bước | Admin A | Admin B |
| --- | --- | --- |
| 1 | BEGIN RC | |
| 2 | plain SELECT quota `10` | |
| 3 | local validation | BEGIN |
| 4 | | UPDATE quota `8` |
| 5 | | COMMIT |
| 6 | A vẫn giữ stale entity `10` | |

A transaction open không đủ. Không row-level lock được acquire bởi ordinary
SELECT.

## Timeline 2 — Explicit row lock

| Bước | Admin A | Admin B | Dashboard C |
| --- | --- | --- | --- |
| 1 | BEGIN | | |
| 2 | SELECT FOR UPDATE quota `10` | | |
| 3 | UPDATE uncommitted `12` | | |
| 4 | | UPDATE/FOR UPDATE waits | |
| 5 | | | plain SELECT returns committed `10` |
| 6 | COMMIT | | |
| 7 | lock release | B continues/re-evaluates | next C SELECT sees `12` |

> **Nói ngắn gọn:** lock compatibility quyết định ai chờ; MVCC visibility quyết
> định plain reader thấy tuple version nào.

## Lock acquisition

PostgreSQL statements acquire relation/table-level modes automatically:

- plain SELECT: `ACCESS SHARE`;
- `SELECT ... FOR UPDATE`: relation `ROW SHARE` cùng row locks trên selected rows;
- UPDATE/DELETE/INSERT: relation `ROW EXCLUSIVE` và row/index locks phù hợp.

Tên `ROW SHARE`/`ROW EXCLUSIVE` là table-level mode names, không có nghĩa cả table
rows đều bị khóa. Phải tách relation lock khỏi tuple/row lock.

Explicit:

```sql
lock table tenant_quota in access exclusive mode;
```

`ACCESS EXCLUSIVE` conflict cả `ACCESS SHARE`, nên plain SELECT khác phải wait.
Đây khác hoàn toàn row lock trên một quota row.

## Row-lock compatibility trong case

A `FOR UPDATE` cùng row conflict với:

- B UPDATE/DELETE;
- B `FOR UPDATE`;
- B `FOR NO KEY UPDATE`;
- các row-lock modes incompatible khác theo PostgreSQL matrix.

A `FOR UPDATE` không làm plain MVCC SELECT C đợi. C không xin row lock, nên snapshot
chọn last committed tuple version.

`FOR SHARE`/`FOR KEY SHARE` có weaker compatibility semantics; không thay tất cả
thành `FOR UPDATE` mà không hiểu mutation cần ngăn.

## MVCC reader khi writer uncommitted

A UPDATE tạo new tuple version quota `12` nhưng chưa commit. C statement snapshot:

```text
new version by A -> invisible
old committed version 10 -> visible
```

C không dirty-read `12`. Sau A commit, statement C mới ở `READ COMMITTED` có thể
thấy `12`; C transaction ở `REPEATABLE READ` có thể tiếp tục thấy old snapshot.

## Waiter sau holder outcome

### Holder commit

B UPDATE chờ A. Khi A commit, B xử lý current row version và command predicate theo
isolation semantics. Nếu B dùng conditional predicate:

```sql
update tenant_quota
set quota=:newQuota
where tenant_id=:id
  and revision=:expectedRevision;
```

affected-row có thể thành `0`; B phải map conflict thay vì overwrite.

### Holder rollback

A uncommitted version biến mất, row lock release. B tiếp tục trên previous
committed state.

Không manual unlock row giữa transaction. Savepoint rollback có nuanced behavior;
case contract dùng outer commit/rollback làm release boundary rõ.

## Lock lifetime

Lock acquire tại statement thực sự chạy và giữ tới physical transaction end:

```text
Spring proxy opens transaction
  repository emits FOR UPDATE -> lock acquired
  Java validation/serialization/remote call -> lock still held
  Hibernate flush -> lock still held
proxy commits/rolls back -> lock released
```

`readOnly=true`, Java block exit, repository return hoặc `EntityManager.flush()`
không tự release transaction locks.

Self-invocation có thể bỏ qua Spring proxy, khiến expected transaction/lock
lifetime khác code annotation. Locking query ngoài transaction thường fail hoặc
có autocommit lifetime quá ngắn để protect later work.

## Table lock nuance

Mọi query đều có một table-level lock mode theo PostgreSQL vocabulary, nhưng
ordinary SELECT không “lock whole table” theo nghĩa chặn DML. `ACCESS SHARE` chủ
yếu conflict `ACCESS EXCLUSIVE`.

Explicit table lock cần:

- exact mode;
- acquisition order nếu nhiều tables;
- statement/transaction timeout;
- impact lên unrelated rows;
- migration/DDL coordination.

Dùng phrase “table is locked” mà thiếu mode là insufficient design review.

## PostgreSQL observability

```sql
select pid,
       locktype,
       relation::regclass,
       mode,
       granted,
       transactionid,
       virtualxid
from pg_locks
where relation = 'tenant_quota'::regclass
   or not granted;
```

```sql
select pid,
       pg_blocking_pids(pid) as blockers,
       wait_event_type,
       wait_event,
       xact_start,
       query_start,
       state,
       query
from pg_stat_activity
where datname = current_database();
```

Row lock representation in `pg_locks` is not a simple “one tuple row per held
lock” inventory in every situation; combine waiters, transaction-ID locks,
activity and controlled experiment.

## Root cause theo layer

### PostgreSQL

Lock modes/compatibility và MVCC work as designed. Plain SELECT is not a
pessimistic reservation.

### Spring

`@Transactional` sets boundary; repository `@Lock`/query sets lock mode. Mixing
them incorrectly creates no lock or unnecessarily long lock.

### Hibernate/JPA

Managed entity identity is not database lock ownership. Pessimistic lock requires
SQL locking clause/dialect execution.

### Application

Team conflates transaction, persistence context, row lock, table lock và
visibility.

## Timeout, deadlock và aborted state

`lock_timeout` limits time waiting for a lock. SQLSTATE commonly `55P03`; current
transaction must rollback after statement error.

`statement_timeout` limits whole statement, not just lock wait. Application/client
deadline có scope khác.

Multiple-row locks in inconsistent order can form cycle; PostgreSQL aborts a
deadlock victim (`40P01`). DB-008 handles full pattern. Retry must start new
transaction and reload.

## Crash và connection loss

Backend connection termination causes PostgreSQL rollback active transaction and
release locks. Application process crash không để permanent row lock, nhưng
failover/network detection delay có thể kéo dài observed outage.

`idle in transaction` session vẫn giữ locks/snapshot/resources. Connection pool
leak hoặc missing timeout biến short lock thành operational incident.

## Multi-instance

Database row locks coordinate all connections across app instances. JVM monitor
không coordinate Node B/direct SQL. Conversely, a database lock only protects
mutation paths that target compatible lock/resource; it không bảo vệ external API
hay another database.

## Expected và actual summary

| Assumption | Actual |
| --- | --- |
| Transactional SELECT locks row | Plain SELECT only MVCC-reads |
| Row lock blocks every read | Plain SELECT reads committed version |
| Flush releases lock | Transaction end releases |
| Table lock means all queries block | Depends exact table mode |
| Timeout fixes race | Timeout bounds wait, not invariant |
| JVM lock equals DB lock | Scope/process differs |
