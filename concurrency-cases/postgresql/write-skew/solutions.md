# Giải pháp guard row, locking và serializable

## Mục tiêu thiết kế

Mọi transaction có thể làm thay đổi roster invariant phải gặp cùng authoritative
conflict point hoặc được database kiểm tra như một serializable execution.

Loser behavior phải rõ: block rồi reject, timeout, affected-row `0`, hoặc abort
`40001` và bounded retry.

## Solution 1 — Lock roster guard row

`on_call_roster` đã là stable parent; dùng nó làm guard:

```java
public interface OnCallRosterRepository
    extends JpaRepository<OnCallRoster, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        select r
          from OnCallRoster r
         where r.id = :rosterId
        """)
    Optional<OnCallRoster> findForInvariantChange(UUID rosterId);
}
```

PostgreSQL tương đương:

```sql
select roster_id
from on_call_roster
where roster_id = :rosterId
for update;
```

Service:

```java
@Service
public class GuardedOnCallService {
    private final OnCallRosterRepository rosters;
    private final OnCallAssignmentRepository assignments;

    public GuardedOnCallService(
        OnCallRosterRepository rosters,
        OnCallAssignmentRepository assignments
    ) {
        this.rosters = rosters;
        this.assignments = assignments;
    }

    @Transactional
    public LeaveResult leaveOnCall(UUID rosterId, UUID operatorId) {
        rosters.findForInvariantChange(rosterId)
            .orElseThrow(RosterNotFoundException::new);

        OnCallAssignment own = assignments
            .findAssignment(rosterId, operatorId)
            .orElseThrow(AssignmentNotFoundException::new);
        if (!own.isOnCall()) {
            return LeaveResult.alreadyOffCall();
        }

        long count = assignments.countOnCall(rosterId);
        if (count <= 1) {
            return LeaveResult.lastOperatorRequired();
        }

        own.leave();
        assignments.flush();
        return LeaveResult.accepted();
    }
}
```

Behavior:

1. A locks roster row.
2. B attempts same `FOR UPDATE` and blocks.
3. A counts `2`, updates Alice, commits; guard lock releases.
4. B acquires guard, loads current state/count `1`.
5. B returns `LAST_OPERATOR_REQUIRED` without updating Bob.

> **Nói ngắn gọn:** guard row làm cho roster-level invariant có một physical row
> mà mọi competing transaction buộc phải đi qua.

Mọi add/remove/leave/reactivate path phải lock guard trước. Với nhiều rosters, lock
theo sorted roster ID. Giữ transaction ngắn, dùng bounded `lock_timeout`, không
gọi remote service khi đang giữ lock.

## Solution 2 — Authoritative on-call counter

Thêm counter vào roster:

```sql
alter table on_call_roster
    add column on_call_count integer not null,
    add constraint ck_on_call_count_positive
        check (on_call_count >= 1);
```

Một robust leave attempt update own row rồi conditional-decrement guard trong cùng
transaction:

```java
public interface OnCallAssignmentCommands {
    @Modifying
    @Query(
        value = """
            update on_call_assignment
               set on_call = false,
                   version = version + 1
             where roster_id = :rosterId
               and operator_id = :operatorId
               and on_call = true
            """,
        nativeQuery = true
    )
    int markOffCall(UUID rosterId, UUID operatorId);
}

public interface OnCallRosterCommands {
    @Modifying
    @Query(
        value = """
            update on_call_roster
               set on_call_count = on_call_count - 1
             where roster_id = :rosterId
               and on_call_count > 1
            """,
        nativeQuery = true
    )
    int decrementIfAnotherRemains(UUID rosterId);
}
```

```java
@Transactional
public LeaveResult leaveWithCounter(UUID rosterId, UUID operatorId) {
    int changed = assignmentCommands.markOffCall(rosterId, operatorId);
    if (changed == 0) {
        return LeaveResult.alreadyOffCall();
    }

    int decremented = rosterCommands.decrementIfAnotherRemains(rosterId);
    if (decremented == 0) {
        throw new LastOperatorRequiredException(rosterId);
    }
    return LeaveResult.accepted();
}
```

`LastOperatorRequiredException` là runtime exception được map thành domain response
ngoài transaction; rollback khôi phục assignment update. Concurrent A/B update
different assignments rồi contend on roster counter: first `2 -> 1`, second sees
predicate false và rollback own update.

Tất cả mutation paths phải dùng consistent lock order và giữ counter đồng bộ. Có
reconciliation:

```sql
select r.roster_id, r.on_call_count,
       count(a.*) filter (where a.on_call) as actual
from on_call_roster r
left join on_call_assignment a on a.roster_id = r.roster_id
group by r.roster_id, r.on_call_count
having r.on_call_count <> count(a.*) filter (where a.on_call);
```

## Solution 3 — Lock toàn relevant assignment set

Không có parent row thì lock all existing roster assignments theo deterministic
order:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("""
    select a
      from OnCallAssignment a
     where a.rosterId = :rosterId
     order by a.operatorId
    """)
List<OnCallAssignment> findAllForUpdate(UUID rosterId);
```

A locks Alice/Bob rows; B waits. Sau A commit, B statement ở `READ COMMITTED`
acquire current versions and must evaluate current on-call set before leaving.

Caveat: protocol phải xử lý concurrent INSERT assignment. Lock existing rows
không khóa một future row; stable parent guard thường rõ hơn. Lock order khác nhau
tạo deadlock; set lớn tăng lock/memory/latency.

## Solution 4 — PostgreSQL `SERIALIZABLE`

Attempt:

```java
@Service
public class SerializableLeaveAttempt {
    private final OnCallAssignmentRepository assignments;

    @Transactional(
        isolation = Isolation.SERIALIZABLE,
        propagation = Propagation.REQUIRES_NEW
    )
    public LeaveResult run(UUID rosterId, UUID operatorId) {
        long count = assignments.countOnCall(rosterId);
        OnCallAssignment own = assignments
            .findAssignment(rosterId, operatorId)
            .orElseThrow(AssignmentNotFoundException::new);

        if (!own.isOnCall()) {
            return LeaveResult.alreadyOffCall();
        }
        if (count <= 1) {
            return LeaveResult.lastOperatorRequired();
        }

        own.leave();
        assignments.flush();
        return LeaveResult.accepted();
    }
}
```

Outer bounded retry:

```java
public LeaveResult leave(UUID rosterId, UUID operatorId) {
    for (int attemptNumber = 1; attemptNumber <= 3; attemptNumber++) {
        try {
            return attempt.run(rosterId, operatorId);
        } catch (CannotSerializeTransactionException ex) {
            if (attemptNumber == 3) {
                throw ex;
            }
            backoff.pauseWithJitter(attemptNumber);
        }
    }
    throw new IllegalStateException("unreachable");
}
```

Retry orchestrator và transactional attempt là hai Spring beans để proxy tạo new
physical transaction. Loser rollback; retry reloads winner state và returns
`LAST_OPERATOR_REQUIRED`.

SSI tránh explicit blocking guard nhưng có abort/retry cost. Hot rosters có thể
gây retry amplification; metric attempts/exhaustion là bắt buộc.

## Solution 5 — Constraint trigger với explicit guard

Row-level `CHECK`/unique không diễn đạt at-least-one sibling row. Một constraint
trigger chỉ safe nếu nó lock roster guard trước khi aggregate validation:

```sql
select 1
from on_call_roster
where roster_id = :rosterId
for update;

select count(*)
from on_call_assignment
where roster_id = :rosterId
  and on_call;
```

Trigger không có lock protocol vẫn race. Logic trong database tăng enforcement
coverage cho direct SQL nhưng tăng migration/debug complexity; stored procedure
có thể là boundary rõ hơn.

## Tại sao `@Version` và uniqueness không đủ

Generated SQL:

```sql
update on_call_assignment
set on_call=false, version=version+1
where assignment_id=:id and version=:oldVersion;
```

Affected-row `0` phát hiện stale write cùng assignment; write skew có affected-row
`1` trên Alice và Bob. Unique constraint cũng không biểu diễn lower-bound count.

## Trade-off comparison

| Cách | Conflict point | Loser | Contention | Complexity |
| --- | --- | --- | --- | --- |
| Guard `FOR UPDATE` | Roster row | block rồi reject/timeout | Theo hot roster | Thấp |
| Conditional counter | Roster counter | affected-row `0` + rollback | Theo hot roster | Counter/reconcile |
| Lock all rows | Assignment set | block/timeout | Theo set size | Lock ordering |
| `SERIALIZABLE` | SSI dependencies | `40001`, retry | Retry under conflict | Operational retry |
| JVM lock | Process memory | local wait | Local only | Không multi-instance |

## Failure và recovery

- Guard holder rollback/crash: assignment rollback, row lock release.
- Counter decrement failure: runtime exception rollback prior assignment update.
- Serialization loser: transaction unusable; retry new transaction.
- Timeout/deadlock: rollback whole attempt; bounded safe retry if command
  idempotent.
- Crash after commit/before response: conditional state/idempotency key replays
  result without another decrement.
- External notification: durable outbox only in successful transaction.

## Recommendation

Với stable roster parent, guard-row lock là default dễ audit. Counter phù hợp khi
roster reads cần cheap count và team có reconciliation discipline.
`SERIALIZABLE` phù hợp khi invariant phức tạp hoặc nhiều mutation paths khó quy về
một guard, miễn full-transaction retry được vận hành tốt.

## Production checklist

- [ ] Invariant scope là roster, không nhầm với assignment row.
- [ ] Mọi mutation path dùng cùng guard/counter/SSI contract.
- [ ] Effective isolation được xác minh.
- [ ] `40001`, `40P01`, `55P03` có bounded handling.
- [ ] Retry tạo transaction mới và reload toàn roster.
- [ ] Multi-roster/operator locks có deterministic order.
- [ ] Không remote I/O khi giữ database locks.
- [ ] Counter-versus-rows reconciliation và unsafe-roster alert tồn tại.
- [ ] Duplicate command không apply leave/decrement lần hai.
- [ ] PostgreSQL Testcontainers regression assert final on-call count.
