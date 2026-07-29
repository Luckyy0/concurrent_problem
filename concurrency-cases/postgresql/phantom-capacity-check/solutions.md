# Giải pháp authoritative capacity

## Mục tiêu thiết kế

Correct solution phải biến “còn chỗ” thành một conflict/claim mà PostgreSQL có
thể enforce atomically. Kết quả loser phải rõ:

```text
accepted, full, duplicate, retryable conflict, hoặc technical failure
```

Capacity safety và duplicate-command safety là hai invariants riêng.

## Solution 1 — Atomic conditional counter

Thêm `used_slots` vào authoritative parent row:

```sql
alter table processing_pool
    add column used_slots integer not null default 0,
    add constraint ck_processing_pool_capacity
        check (
            capacity >= 0
            and used_slots >= 0
            and used_slots <= capacity
        );
```

Claim một slot bằng single conditional UPDATE:

```sql
update processing_pool
set used_slots = used_slots + 1
where pool_id = :poolId
  and used_slots < capacity;
```

Repository:

```java
public interface ProcessingPoolRepository
    extends JpaRepository<ProcessingPool, UUID> {

    @Modifying
    @Query(
        value = """
            update processing_pool
               set used_slots = used_slots + 1
             where pool_id = :poolId
               and used_slots < capacity
            """,
        nativeQuery = true
    )
    int claimOne(UUID poolId);

    @Modifying
    @Query(
        value = """
            update processing_pool
               set used_slots = used_slots - 1
             where pool_id = :poolId
               and used_slots > 0
            """,
        nativeQuery = true
    )
    int releaseOne(UUID poolId);
}
```

Allocation service:

```java
@Service
public class AtomicPoolAllocationService {
    private final ProcessingPoolRepository pools;
    private final SlotAllocationRepository allocations;

    public AtomicPoolAllocationService(
        ProcessingPoolRepository pools,
        SlotAllocationRepository allocations
    ) {
        this.pools = pools;
        this.allocations = allocations;
    }

    @Transactional
    public AllocationResult allocate(UUID poolId, UUID requestId) {
        int claimed = pools.claimOne(poolId);
        if (claimed == 0) {
            return pools.existsById(poolId)
                ? AllocationResult.full()
                : AllocationResult.poolNotFound();
        }

        SlotAllocation saved = allocations.saveAndFlush(
            SlotAllocation.active(poolId, requestId)
        );
        return AllocationResult.accepted(saved.getId());
    }
}
```

Concurrency behavior với `used_slots=9`, `capacity=10`:

1. A UPDATE acquire parent row lock, predicate true, value thành `10`.
2. B UPDATE cùng row chờ A.
3. A commit, row lock release.
4. B tiếp tục trên current row; PostgreSQL re-evaluate predicate.
5. `10 < 10` false, affected-row `0`; B trả `FULL`.

Nếu allocation INSERT/flush fail, Runtime/DataAccess exception rollback cả counter
increment. `saveAndFlush()` đưa unique violation vào method boundary; không catch
rồi commit counter.

> **Nói ngắn gọn:** counter đúng không chỉ là cache để hiển thị; conditional
> UPDATE biến phần capacity cuối cùng thành một database row conflict có
> affected-row contract.

### Release đúng một lần

Tránh load status rồi decrement riêng. Transition allocation conditionally:

```sql
with released as (
    update slot_allocation
       set status = 'RELEASED'
     where allocation_id = :allocationId
       and status = 'ACTIVE'
    returning pool_id
)
update processing_pool p
   set used_slots = p.used_slots - 1
  from released r
 where p.pool_id = r.pool_id
   and p.used_slots > 0;
```

Affected-row `1` nghĩa release mới xảy ra; `0` nghĩa already released/not found.
Một statement giữ allocation transition và decrement cùng outcome.

### Counter reconciliation

Counter là denormalized authoritative state nên cần:

```sql
select p.pool_id,
       p.capacity,
       p.used_slots,
       count(a.*) filter (where a.status = 'ACTIVE') as actual_active
from processing_pool p
left join slot_allocation a on a.pool_id = p.pool_id
group by p.pool_id, p.capacity, p.used_slots
having p.used_slots <>
       count(a.*) filter (where a.status = 'ACTIVE');
```

Migration phải backfill dưới write quiescence/lock hoặc dual-write protocol; không
khởi tạo `used_slots` bằng stale COUNT khi allocations vẫn đổi.

## Solution 2 — Lock parent rồi count

Nếu không muốn counter, dùng một stable parent row làm mutex ở PostgreSQL:

```java
public interface ProcessingPoolRepository
    extends JpaRepository<ProcessingPool, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        select p
          from ProcessingPool p
         where p.id = :poolId
        """)
    Optional<ProcessingPool> findForAllocation(UUID poolId);
}
```

PostgreSQL SQL tương đương:

```sql
select pool_id, capacity
from processing_pool
where pool_id = :poolId
for update;
```

Service:

```java
@Transactional
public AllocationResult allocateWithParentLock(
    UUID poolId,
    UUID requestId
) {
    ProcessingPool pool = pools.findForAllocation(poolId)
        .orElseThrow(PoolNotFoundException::new);

    long active = allocations.countByPoolAndStatus(
        poolId,
        AllocationStatus.ACTIVE
    );
    if (active >= pool.getCapacity()) {
        return AllocationResult.full();
    }

    SlotAllocation saved = allocations.saveAndFlush(
        SlotAllocation.active(poolId, requestId)
    );
    return AllocationResult.accepted(saved.getId());
}
```

Mọi allocate/release/capacity-change path phải lock same parent row trước khi
đụng predicate. B blocks tới A commit/rollback, rồi acquire lock và recount current
committed rows. B thấy `10` và reject.

Trade-off: implementation đơn giản và source of truth vẫn là child rows, nhưng
hot pool serialize requests; bypass path phá protocol. Lock nhiều pools theo UUID
order; dùng bounded `lock_timeout` và handle deadlock.

## Solution 3 — Pre-created finite slot rows

Khi capacity thật sự là tập finite slots, model chúng trực tiếp:

```sql
create table processing_slot (
    pool_id uuid not null,
    slot_no integer not null,
    allocation_id uuid,
    primary key (pool_id, slot_no),
    unique (allocation_id),
    foreign key (pool_id) references processing_pool(pool_id)
);
```

Claim một free slot:

```sql
with candidate as (
    select pool_id, slot_no
    from processing_slot
    where pool_id = :poolId
      and allocation_id is null
    order by slot_no
    for update skip locked
    limit 1
)
update processing_slot s
set allocation_id = :allocationId
from candidate c
where s.pool_id = c.pool_id
  and s.slot_no = c.slot_no
returning s.slot_no;
```

Mỗi slot là một lockable/unique capacity unit. `SKIP LOCKED` tránh chờ slot đang
được contender khác giữ; no returned row map thành `FULL` hoặc short retry theo
contract.

Allocation row và slot claim phải commit cùng transaction, với foreign key/unique
constraint phù hợp. Capacity decrease cần xác định xử lý occupied high-numbered
slots, không xóa mù quáng.

## Solution 4 — PostgreSQL `SERIALIZABLE`

Phù hợp khi invariant predicate phức tạp, không dễ quy về một parent counter/finite
slot:

```java
@Service
public class SerializableAllocationAttempt {
    private final ProcessingPoolRepository pools;
    private final SlotAllocationRepository allocations;

    @Transactional(
        isolation = Isolation.SERIALIZABLE,
        propagation = Propagation.REQUIRES_NEW
    )
    public AllocationResult run(UUID poolId, UUID requestId) {
        ProcessingPool pool = pools.findById(poolId)
            .orElseThrow(PoolNotFoundException::new);
        long active = allocations.countByPoolAndStatus(
            poolId,
            AllocationStatus.ACTIVE
        );
        if (active >= pool.getCapacity()) {
            return AllocationResult.full();
        }

        SlotAllocation saved = allocations.saveAndFlush(
            SlotAllocation.active(poolId, requestId)
        );
        return AllocationResult.accepted(saved.getId());
    }
}
```

Outer retry:

```java
@Service
public class AllocationRetrier {
    private final SerializableAllocationAttempt attempt;
    private final RetryBackoff backoff;

    public AllocationRetrier(
        SerializableAllocationAttempt attempt,
        RetryBackoff backoff
    ) {
        this.attempt = attempt;
        this.backoff = backoff;
    }

    public AllocationResult allocate(UUID poolId, UUID requestId) {
        int maxAttempts = 3;
        for (int number = 1; number <= maxAttempts; number++) {
            try {
                return attempt.run(poolId, requestId);
            } catch (CannotSerializeTransactionException ex) {
                if (number == maxAttempts) {
                    throw ex;
                }
                backoff.pauseWithJitter(number);
            }
        }
        throw new IllegalStateException("unreachable");
    }
}
```

SSI conflict thường được phát hiện tại statement/flush/commit. Loser transaction
rollback toàn bộ. Retry proxy phải gọi bean khác để `REQUIRES_NEW` thật sự được
intercept; mỗi attempt reload/recount/recompute. Backoff bounded và interrupt-aware.

Hot predicate có thể tạo retry amplification. Sau retry, pool có thể full; đó là
domain result, không phải serialization failure cần retry tiếp.

## Solution 5 — Transaction-scoped advisory lock

Khi không có suitable parent row nhưng key ổn định, có thể:

```sql
select pg_advisory_xact_lock(hashtextextended(:poolId::text, 0));
```

Sau đó count/insert trong cùng transaction. Mọi writers phải dùng exact same key
derivation và lock order. Advisory lock không có foreign-key relationship với
data, dễ bị bypass/collision/design error; parent row thường rõ và an toàn hơn.

Không dùng session-scoped lock nếu connection pool làm ownership/release khó kiểm
soát. Transaction-scoped lock tự release khi commit/rollback.

## Tại sao constraint thông thường chưa đủ

`CHECK` không được dùng để query COUNT rows khác như một general cross-row
constraint. Unique constraint chỉ giới hạn duplicate key. Trigger muốn enforce
aggregate capacity vẫn phải acquire authoritative parent/advisory lock; nếu chỉ
COUNT rồi reject, trigger cũng có race.

Table lock có thể serialize inserts:

```sql
lock table slot_allocation in share row exclusive mode;
```

nhưng contention scope rộng hơn pool, giảm throughput và tăng deadlock/latency.
Đây hiếm khi là lựa chọn tốt khi parent row hoặc slots model được.

## Trade-off comparison

| Cách | Correctness boundary | Loser behavior | Contention | Complexity |
| --- | --- | --- | --- | --- |
| Conditional counter | Parent row + check | affected-row `0` | Theo hot pool | Counter/reconciliation |
| Parent `FOR UPDATE` | Shared parent lock protocol | block, timeout hoặc FULL | Theo hot pool | Thấp |
| Pre-created slots | Slot rows + unique keys | skip/no slot | Phân tán theo slots | Schema/lifecycle cao hơn |
| `SERIALIZABLE` | SSI predicate dependencies | `40001`, retry | Retry dưới conflict | Retry/operations |
| Advisory xact lock | Agreed numeric key | block/timeout | Theo key | Dễ bypass |
| JVM lock | Process memory | local block | Local only | Không multi-instance |

## Failure và recovery

- Allocation INSERT fail sau counter claim: rollback transaction, counter trở lại.
- Parent lock holder crash: PostgreSQL rollback connection transaction, release
  lock.
- Serializable loser: transaction unusable; start new attempt.
- Client timeout không chứng minh rollback; inspect committed result bằng
  request/idempotency key trước replay.
- Duplicate request: return stored allocation/outcome; không claim thêm capacity.
- Release retry: conditional status transition làm duplicate release no-op.

## Recommendation cho case này

Với generic single-pool capacity, ưu tiên:

1. conditional counter khi claim/release lifecycle có thể giữ counter chính xác;
2. parent `FOR UPDATE` khi count child rows là source of truth và throughput cho
   mỗi pool chấp nhận serialization;
3. pre-created slots khi capacity units có identity tự nhiên;
4. `SERIALIZABLE` khi predicate phức tạp và team vận hành bounded retry tốt.

Không dùng `REPEATABLE READ` chỉ vì repeated COUNT không thấy phantom.

## Production checklist

### Invariant

- [ ] Mọi accepted allocation consume đúng một authoritative unit.
- [ ] Active count/counter không vượt capacity.
- [ ] Duplicate request không consume thêm unit.
- [ ] Release chỉ trả unit đúng một lần.

### Transactions

- [ ] Claim và allocation INSERT commit/rollback cùng nhau.
- [ ] Effective isolation được xác nhận trên physical transaction.
- [ ] Retry bắt đầu transaction mới và recount.
- [ ] `lock_timeout`, deadlock và `40001` có domain mapping.
- [ ] Multi-pool lock order deterministic.

### Operations

- [ ] Reconciliation counter-versus-active rows chạy định kỳ.
- [ ] Có metrics affected-row `0`, lock wait, retry và invariant violation.
- [ ] Không remote I/O trong transaction giữ lock.
- [ ] Test dùng PostgreSQL Testcontainers và controlled interleaving.
- [ ] Capacity migration/change protocol đã được document.
