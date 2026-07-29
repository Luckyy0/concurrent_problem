# Broken count-then-insert allocation

## Pool entity

```java
@Entity
@Table(name = "processing_pool")
public class ProcessingPool {
    @Id
    @Column(name = "pool_id", nullable = false)
    private UUID id;

    @Column(nullable = false)
    private int capacity;

    protected ProcessingPool() {
    }

    public UUID getId() {
        return id;
    }

    public int getCapacity() {
        return capacity;
    }
}
```

## Allocation entity

```java
@Entity
@Table(
    name = "slot_allocation",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_slot_allocation_request",
        columnNames = {"pool_id", "request_id"}
    )
)
public class SlotAllocation {
    @Id
    @Column(name = "allocation_id", nullable = false)
    private UUID id;

    @Column(name = "pool_id", nullable = false)
    private UUID poolId;

    @Column(name = "request_id", nullable = false)
    private UUID requestId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private AllocationStatus status;

    protected SlotAllocation() {
    }

    public static SlotAllocation active(UUID poolId, UUID requestId) {
        SlotAllocation allocation = new SlotAllocation();
        allocation.id = UUID.randomUUID();
        allocation.poolId = poolId;
        allocation.requestId = requestId;
        allocation.status = AllocationStatus.ACTIVE;
        return allocation;
    }

    public UUID getId() {
        return id;
    }
}
```

Unique `(pool_id, request_id)` ngăn cùng command tạo duplicate row. A và B có
request IDs khác nhau, nên constraint này không bảo vệ capacity.

## Repository predicate query

```java
public interface SlotAllocationRepository
    extends JpaRepository<SlotAllocation, UUID> {

    @Query("""
        select count(a)
          from SlotAllocation a
         where a.poolId = :poolId
           and a.status = :status
        """)
    long countByPoolAndStatus(
        UUID poolId,
        AllocationStatus status
    );
}
```

## Broken service

```java
@Service
public class PoolAllocationService {
    private final ProcessingPoolRepository pools;
    private final SlotAllocationRepository allocations;

    public PoolAllocationService(
        ProcessingPoolRepository pools,
        SlotAllocationRepository allocations
    ) {
        this.pools = pools;
        this.allocations = allocations;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public AllocationResult allocate(UUID poolId, UUID requestId) {
        ProcessingPool pool = pools.findById(poolId)
            .orElseThrow(PoolNotFoundException::new);

        long active = allocations.countByPoolAndStatus(
            poolId,
            AllocationStatus.ACTIVE
        );

        if (active >= pool.getCapacity()) {
            return AllocationResult.full();
        }

        SlotAllocation saved = allocations.save(
            SlotAllocation.active(poolId, requestId)
        );
        return AllocationResult.accepted(saved.getId());
    }
}
```

Implementation realistic: capacity check nằm cùng transaction với INSERT và
duplicate request có unique constraint. Phần thiếu là database coordination cho
aggregate predicate.

> **Nói ngắn gọn:** transaction làm mỗi allocation attempt atomic, nhưng hai
> attempts vẫn có thể cùng quyết định từ count `9` và commit hai rows hợp lệ riêng
> lẻ.

## SQL Hibernate thực thi

A và B đều chạy:

```sql
begin;

select p.pool_id, p.capacity
from processing_pool p
where p.pool_id = :poolId;

select count(*)
from slot_allocation a
where a.pool_id = :poolId
  and a.status = 'ACTIVE';
-- both observe 9

insert into slot_allocation(
    allocation_id,
    pool_id,
    request_id,
    status
)
values (:differentAllocationId, :poolId, :differentRequestId, 'ACTIVE');

commit;
```

Hibernate có thể defer INSERT tới flush/commit. Flush sớm chỉ làm row của actor đó
visible sau commit; nó không khóa predicate gap cho actor còn lại.

## Concrete interleaving

```text
A BEGIN
B BEGIN
A COUNT ACTIVE -> 9
B COUNT ACTIVE -> 9
A persist allocation A-101
B persist allocation B-202
A flush INSERT A-101 -> success
B flush INSERT B-202 -> success
A COMMIT
B COMMIT
final COUNT ACTIVE -> 11
```

Hai inserts acquire locks trên row/index entries khác nhau. Không constraint nào
biết `COUNT(ACTIVE) <= capacity`.

## Visible phantom khi query lại

Một transaction `READ COMMITTED` có thể tự chứng kiến:

```sql
select count(*) ...; -- 9
-- B inserts ACTIVE row and commits
select count(*) ...; -- 10
```

SELECT sau dùng statement snapshot mới. Đây là visible phantom. Broken service
không cần query lần hai để fail; stale first count đã đủ tạo check-then-insert
race.

## Tại sao `REPEATABLE READ` chưa đủ

Nếu cả A và B chạy `REPEATABLE READ`, mỗi transaction giữ snapshot thấy `9`.
Concurrent inserts có different keys và không update cùng row:

```text
A snapshot: 9 -> INSERT A
B snapshot: 9 -> INSERT B
both may commit -> final 11
```

PostgreSQL ngăn visible phantom trong repeated query ở mức này, nhưng không có
authoritative write conflict cho capacity. Stable stale snapshots thậm chí làm
mỗi actor tiếp tục tin count `9`.

## Preconditions tái hiện

1. Initial active count là `capacity - 1`.
2. A và B có connections/physical transactions độc lập.
3. Cả hai hoàn tất COUNT trước khi actor nào commit INSERT.
4. Allocation/request IDs khác nhau.
5. Không có parent row lock, authoritative counter hoặc serializable retry.
6. Test method không có outer transaction che commits.
7. PostgreSQL thật được dùng để xác nhận MVCC/isolation.

## Những cách sửa chưa đủ

### Chỉ dùng `@Transactional`

Đã có transaction; non-atomic predicate decision vẫn trải qua nhiều statements.

### Chỉ dùng unique allocation ID/request ID

Unique constraint giải quyết duplicate identity, không giới hạn số distinct rows.

### Lock các rows hiện có

Ngay cả khi select existing allocations để lock, một transaction khác vẫn có thể
insert một new row không tồn tại để bị row-lock. `COUNT(*) FOR UPDATE` cũng không
phải cách khóa một predicate range hợp lệ trong PostgreSQL.

### Chỉ nâng lên `REPEATABLE READ`

Giữ result set ổn định cho reader nhưng không tạo conflict giữa different inserts.

### Dùng `synchronized`

Chỉ hoạt động trong một JVM. Node khác, admin tool và batch process không tham gia
cùng monitor.

### Flush trước khi trả result

Flush có thể expose constraint violation sớm hơn, nhưng không có capacity
constraint để vi phạm. Visibility cho actor khác vẫn cần commit.

### Retry mọi exception

Race thường không ném exception ở `READ COMMITTED`. Retry không idempotent còn có
thể tạo duplicate effects.
