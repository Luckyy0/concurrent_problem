# Pessimistic locking và `FOR UPDATE`

## Mục đích

Khóa bi quan (`pessimistic locking`) reserve database resource trước khi
application đưa ra một quyết định phụ thuộc state. Tài liệu này định nghĩa cơ
chế chung; mỗi case vẫn phải nêu row nào được khóa, invariant nào được bảo vệ và
failure contract của waiter.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| pessimistic lock | Lock được acquire trước mutation vì concurrent conflict được xem là có khả năng xảy ra |
| locking read | Query vừa đọc vừa yêu cầu database lock |
| `PESSIMISTIC_WRITE` | JPA lock mode cho explicit write-oriented lock |
| `FOR UPDATE` | PostgreSQL row-locking clause mạnh cho selected rows |
| holder | Transaction đang giữ lock |
| waiter | Transaction đang chờ incompatible lock |
| lock lifetime | Từ lock acquisition đến commit/rollback |
| bounded wait | Chờ có timeout/deadline hữu hạn |
| revalidation | Đánh giá lại state sau khi acquire lock |
| lock order | Thứ tự nhất quán khi khóa nhiều resources |

## Cơ chế cốt lõi

```text
BEGIN
  locking SELECT
  → acquire row lock
  → read current state
  → decide
  → mutate
COMMIT/ROLLBACK
  → release lock
```

Một transaction khác yêu cầu incompatible lock trên cùng row sẽ block,
fail-fast hoặc timeout. Sau khi holder kết thúc, waiter không được tiếp tục với
state đã đọc trước đó; nó phải dùng state trả về từ locking read và revalidate
business rule.

> **Nói ngắn gọn:** pessimistic lock đúng không chỉ “làm request thứ hai chờ”;
> nó buộc request đó quyết định sau outcome của request thứ nhất.

## PostgreSQL `FOR UPDATE`

```sql
select *
from show_seat
where show_id = :showId
  and seat_no = :seatNo
for update;
```

`FOR UPDATE` lock selected rows như rows sắp được update. Competing `UPDATE`,
`DELETE` và incompatible locking reads chờ transaction hiện tại kết thúc. Plain
MVCC `SELECT` thường không bị row lock này chặn; nó đọc committed tuple version
theo snapshot.

Row locks tồn tại đến transaction end, không release khi repository method
return. Một transaction giữ lock trong remote I/O vẫn giữ database connection
và làm wait queue dài hơn.

`FOR UPDATE` chỉ lock rows thực sự được select. Query trả zero rows không tạo row
lock trên một object còn thiếu và không khóa toàn bộ predicate chống future
insert.

## JPA/Hibernate mapping

Spring Data JPA cho phép gắn lock mode vào query:

```java
public interface ShowSeatRepository
        extends JpaRepository<ShowSeat, ShowSeatId> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from ShowSeat s where s.id = :id")
    Optional<ShowSeat> findForUpdate(@Param("id") ShowSeatId id);
}
```

Hibernate chuyển lock mode thành database locking syntax phù hợp với dialect.
SQL clause cụ thể có thể phụ thuộc Hibernate version và query shape, nên
integration test/log phải xác nhận behavior trên PostgreSQL thật.

Query cần active transaction. Boundary đúng bao trùm:

```text
locking query → revalidation → all database mutations → flush → commit
```

Lock annotation không sửa self-invocation, transaction quá rộng hoặc object đã
được load stale trước locking query.

## Wait policy

### Bounded wait

Chờ holder trong thời gian ngắn rồi revalidate. Phù hợp interactive operation khi
holder transaction được giữ ngắn.

PostgreSQL `lock_timeout` giới hạn thời gian chờ acquire lock. Nó khác:

- `statement_timeout`: toàn statement;
- transaction/application deadline: toàn unit of work;
- connection acquisition timeout: chờ pool;
- client timeout: caller chờ response.

Các giới hạn phải lồng nhau hợp lý và chừa thời gian rollback/response.

### `NOWAIT`

Fail ngay nếu row đang locked. Phù hợp khi product có outcome `BUSY` hoặc một
fallback khác, không phù hợp nếu caller mặc định hiểu failure là resource không
tồn tại.

### `SKIP LOCKED`

Bỏ qua locked rows thay vì wait. Hữu ích cho work queue nơi worker có thể lấy
item khác; không phải default cho exact resource do user chỉ định.

## Timeout, deadlock và retry

Lock timeout hoặc deadlock là transaction failure. Sau PostgreSQL statement
error, để transaction rollback trước khi map outcome hoặc retry.

Retry chỉ an toàn khi:

- command có replay/idempotency contract;
- attempt mới dùng transaction và persistence context mới;
- state được reload/revalidated;
- attempt cap, overall deadline và backoff đều bounded.

Không retry business rejection như “seat đã held”. Không blanket-retry mọi
`DataAccessException`.

## Multi-row lock order

Khi một operation khóa nhiều rows:

1. deduplicate resource IDs;
2. canonicalize bằng stable key;
3. acquire theo cùng order ở mọi code path;
4. validate đủ rows trước mutation;
5. giữ transaction ngắn.

Opposite order có thể tạo wait-for cycle. PostgreSQL deadlock detector abort một
victim, nhưng detector không thay thế deterministic design.

Một ordered SQL locking query thường dễ audit hơn loop phụ thuộc request order:

```sql
select *
from show_seat
where show_id = :showId
  and seat_no = any(:seatNos)
order by show_id, seat_no
for update;
```

## Known row và predicate-wide invariant

Pessimistic row locking phù hợp khi resource row ổn định và biết trước: account,
inventory item, seat, aggregate guard row.

Nó không tự bảo vệ:

- row chưa tồn tại;
- “không có record nào thỏa predicate”;
- capacity tính từ một tập rows có thể nhận phantom inserts;
- invariant trải trên nhiều tables nhưng không có common lock protocol.

Các trường hợp đó có thể cần unique/check/exclusion constraint, conditional
mutation, stable guard row hoặc `SERIALIZABLE`.

## Commit, rollback và crash

| Holder outcome | Waiter/recovery |
| --- | --- |
| Commit | Waiter acquire rồi đọc committed state mới |
| Rollback | Waiter acquire rồi đọc state trước attempt |
| Connection loss trước commit | PostgreSQL abort và release locks |
| Response loss sau commit | State vẫn committed; phân giải bằng replay/idempotency |

Application-level unlock call thường không tồn tại cho row lock; transaction end
là release boundary. Vì vậy connection leak/idle-in-transaction là operational
risk nghiêm trọng.

## Multi-instance

Database row lock coordinate mọi application instance dùng cùng PostgreSQL
primary. JVM-local lock không có thuộc tính này.

Scale-out vẫn làm tăng:

- số waiters trên hot row;
- tổng connections có thể bị giữ trong lock wait;
- timeout/cancellation traffic;
- nhu cầu admission control và observability.

Correctness không đồng nghĩa throughput tốt dưới sustained high contention.

## Quan sát

Kết hợp:

- `pg_stat_activity.wait_event_type`, `wait_event`, `xact_start`;
- `pg_blocking_pids(pid)` để nối waiter với holder;
- `pg_locks` để phân tích lock graph;
- SQLSTATE `55P03` cho lock-not-available/timeout;
- SQLSTATE `40P01` cho deadlock victim;
- pool active/pending/acquisition timeout;
- application correlation ID và domain outcome.

Row-level lock information không phải lúc nào cũng hiện thành một tuple-lock row
dễ hiểu trong `pg_locks`; waiter thường chờ transaction ID của holder.

## Chọn pessimistic lock khi nào?

Ưu tiên khi decision nhiều bước cần current state, target row đã biết, conflict
đủ thường xuyên và critical section ngắn. So sánh với:

- conditional atomic SQL cho invariant diễn đạt trong một mutation;
- optimistic `@Version` cho contention thấp;
- constraint cho invariant database biểu diễn trực tiếp;
- queue/admission control cho hot key bền vững.

Không chọn primitive trước khi viết invariant và loser outcome.

## Liên kết

- [LOCK-003 — Pessimistic write lock với FOR UPDATE](../locking/pessimistic-write-for-update/README.md)
- [DB-007 — Row/table lock lifecycle](../postgresql/row-table-lock-lifecycle/README.md)
- [DB-008 — Opposite row order deadlock](../postgresql/opposite-row-order-deadlock/README.md)
- [DB-010 — Work claiming với SKIP LOCKED](../postgresql/skip-locked-work-queue/README.md)
- [SPR-007 — Long transaction và connection pool](../spring/connection-pool-long-transaction/README.md)
- [PostgreSQL locks và lock lifetime](postgresql-locks.md)
- [Deadlock và retry an toàn](deadlocks-and-retries.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
