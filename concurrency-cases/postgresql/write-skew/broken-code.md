# Broken versioned-row implementation

## Assignment entity

```java
@Entity
@Table(
    name = "on_call_assignment",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_roster_operator",
        columnNames = {"roster_id", "operator_id"}
    )
)
public class OnCallAssignment {
    @Id
    @Column(name = "assignment_id", nullable = false)
    private UUID id;

    @Column(name = "roster_id", nullable = false)
    private UUID rosterId;

    @Column(name = "operator_id", nullable = false)
    private UUID operatorId;

    @Column(name = "on_call", nullable = false)
    private boolean onCall;

    @Version
    @Column(nullable = false)
    private long version;

    protected OnCallAssignment() {
    }

    public void leave() {
        onCall = false;
    }

    public UUID getRosterId() {
        return rosterId;
    }

    public boolean isOnCall() {
        return onCall;
    }
}
```

`@Version` bảo vệ stale write trên cùng assignment row. Nó không tạo một shared
version cho toàn roster.

## Repository

```java
public interface OnCallAssignmentRepository
    extends JpaRepository<OnCallAssignment, UUID> {

    @Query("""
        select count(a)
          from OnCallAssignment a
         where a.rosterId = :rosterId
           and a.onCall = true
        """)
    long countOnCall(UUID rosterId);

    @Query("""
        select a
          from OnCallAssignment a
         where a.rosterId = :rosterId
           and a.operatorId = :operatorId
        """)
    Optional<OnCallAssignment> findAssignment(
        UUID rosterId,
        UUID operatorId
    );
}
```

## Broken service

```java
@Service
public class OnCallService {
    private final OnCallAssignmentRepository assignments;

    public OnCallService(OnCallAssignmentRepository assignments) {
        this.assignments = assignments;
    }

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public LeaveResult leaveOnCall(UUID rosterId, UUID operatorId) {
        long onCallCount = assignments.countOnCall(rosterId);
        if (onCallCount <= 1) {
            return LeaveResult.lastOperatorRequired();
        }

        OnCallAssignment own = assignments
            .findAssignment(rosterId, operatorId)
            .orElseThrow(AssignmentNotFoundException::new);

        if (!own.isOnCall()) {
            return LeaveResult.alreadyOffCall();
        }

        own.leave();
        return LeaveResult.accepted();
    }
}
```

`REPEATABLE_READ` thường được thêm với ý định “giữ count ổn định”. Điều đó đúng
cho repeated reads trong từng transaction, nhưng chính stable snapshots cho phép A
và B cùng tiếp tục tin rằng count là `2`.

> **Nói ngắn gọn:** cả hai Hibernate UPDATEs có version predicate và affected-row
> `1`; optimistic locking không báo lỗi vì Alice và Bob là hai rows khác nhau.

## SQL khi Hibernate flush

A:

```sql
update on_call_assignment
set on_call = false,
    version = 1
where assignment_id = :aliceAssignmentId
  and version = 0;
-- affected rows = 1
```

B:

```sql
update on_call_assignment
set on_call = false,
    version = 1
where assignment_id = :bobAssignmentId
  and version = 0;
-- affected rows = 1
```

Hibernate chỉ ném `OptimisticLockException`/Spring optimistic-lock exception khi
versioned UPDATE affected-row là `0`. Ở đây cả hai là `1`, nên không có conflict
signal để retry.

## Concrete interleaving

```text
A BEGIN REPEATABLE READ
B BEGIN REPEATABLE READ
A COUNT on_call=true -> 2
B COUNT on_call=true -> 2
A load Alice version=0
B load Bob version=0
A UPDATE Alice false, version=1 -> affected 1
B UPDATE Bob false, version=1   -> affected 1
A COMMIT
B COMMIT
final COUNT on_call=true -> 0
```

Row locks:

- A UPDATE locks Alice row tới commit;
- B UPDATE locks Bob row tới commit;
- locks không conflict;
- plain COUNT không lock roster predicate.

## Schema tương đương

```sql
create table on_call_roster (
    roster_id uuid primary key,
    name varchar(100) not null,
    active boolean not null
);

create table on_call_assignment (
    assignment_id uuid primary key,
    roster_id uuid not null references on_call_roster(roster_id),
    operator_id uuid not null,
    on_call boolean not null,
    version bigint not null,
    unique (roster_id, operator_id)
);
```

Unique constraint ngăn duplicate operator assignment; nó không enforce “ít nhất
một row on_call=true”.

## Preconditions tái hiện

1. Roster ban đầu có đúng hai on-call assignments.
2. A và B có physical transactions/connections độc lập.
3. Cả hai COUNT trước khi actor nào commit UPDATE.
4. A update Alice row, B update Bob row.
5. Effective isolation là `READ COMMITTED` hoặc PostgreSQL `REPEATABLE READ`.
6. Không có guard-row lock/counter/serializable retry.
7. Test không có outer transaction che commit.

## Những cách sửa chưa đủ

### Chỉ thêm `@Version`

Đã có; version scopes theo assignment row, trong khi invariant scopes theo roster.

### Nâng từ `READ COMMITTED` lên `REPEATABLE READ`

Ngăn non-repeatable read/visible phantom cho transaction, không phát hiện mọi
serialization anomaly của different-row writes.

### Lock row của chính operator

A và B vẫn lock different rows. Shared invariant cần shared guard hoặc lock toàn
relevant set theo cùng protocol.

### Dùng unique/check constraint đơn giản

Row-level `CHECK` không query được count của sibling rows. Unique constraint không
biểu diễn “at least one true”.

### Dùng `synchronized`

Không coordinate app node khác, admin SQL hay batch process.

### Retry optimistic exception

Không exception nào xảy ra trong interleaving này. Retry same decision mà không
reload toàn roster cũng không bảo vệ invariant.

### Gửi notification trước commit

Nếu transaction sau đó rollback/serialization-fail, notification trở thành
orphan side effect. Publish durable outbox trong successful transaction nếu cần.
