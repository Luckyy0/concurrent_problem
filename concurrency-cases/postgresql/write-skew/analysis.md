# Phân tích snapshot isolation và write skew

## Initial state

```text
roster R:
  Alice on_call=true, version=0
  Bob   on_call=true, version=0
```

Invariant là một predicate trên set: phải có ít nhất một assignment
`roster_id=R AND on_call=true`.

## Mandatory interleaving

| Bước | Transaction A | Transaction B |
| --- | --- | --- |
| 1 | BEGIN RR | BEGIN RR |
| 2 | Snapshot SA thấy Alice+Bob on-call | |
| 3 | | Snapshot SB thấy Alice+Bob on-call |
| 4 | Quyết định Alice có thể leave | Quyết định Bob có thể leave |
| 5 | UPDATE Alice row false | |
| 6 | | UPDATE Bob row false |
| 7 | COMMIT | COMMIT |
| 8 | | final on-call count = 0 |

Expected: một transaction phải thua/re-evaluate và thấy chỉ còn một operator.
Actual: both commits because writes do not overlap.

> **Nói ngắn gọn:** snapshot của mỗi transaction tự nhất quán, nhưng hai snapshots
> không bao gồm write tương lai của nhau; ghép hai locally-valid commits tạo
> globally-invalid state.

## MVCC view

Mỗi `REPEATABLE READ` transaction thấy old committed tuple versions:

```text
SA: Alice=true, Bob=true
SB: Alice=true, Bob=true
```

A tạo new Alice tuple `false`; B tạo new Bob tuple `false`. A không update Bob và
B không update Alice, nên PostgreSQL snapshot isolation không gặp same-row
concurrent-update rule để abort.

Sau both commits, new transaction snapshot thấy cả hai new versions và count `0`.

## Read/write sets

```text
A read set:  Alice row + Bob row/predicate
A write set: Alice row

B read set:  Alice row + Bob row/predicate
B write set: Bob row
```

Write sets disjoint, nhưng:

- A decision phụ thuộc vào Bob=true mà B thay đổi;
- B decision phụ thuộc vào Alice=true mà A thay đổi.

Đây là hai rw-antidependencies đối chiều, nền tảng của write skew/serialization
anomaly.

## Lock behavior

Plain COUNT/entity SELECT dùng MVCC snapshot; chúng không acquire row locks để
reserve invariant.

A versioned UPDATE:

- locks Alice row;
- predicate `version=0` match;
- affected-row `1`;
- lock release tại A commit/rollback.

B làm tương tự trên Bob row. Hai row locks tương thích vì target khác nhau.

`@Version` không có aggregate roster version để compare.

## `READ COMMITTED`

Write skew cũng xảy ra nếu cả COUNTs chạy trước UPDATE commits. Statement snapshot
mới chỉ giúp khi code thực sự revalidate sau concurrent commit; broken flow không
có authoritative final validation.

Một SELECT sau A commit có thể giúp B thấy count `1`, nhưng không được tự động chèn
bởi `@Transactional`.

## `REPEATABLE READ`

PostgreSQL `REPEATABLE READ` giữ stable transaction snapshot. B query lại sau A
commit vẫn có thể thấy Alice=true từ SB, nên re-read trong same transaction không
phải freshness validation.

Mức này bảo vệ same-row update conflicts mạnh hơn `READ COMMITTED`, nhưng
snapshot isolation cho phép write skew trên disjoint write sets.

## `SERIALIZABLE` SSI

SSI theo dõi predicate/read-write dependencies. Trong case:

```text
A reads Bob=true  --rw--> B writes Bob=false
B reads Alice=true --rw--> A writes Alice=false
```

Dangerous structure không thể có serial order giữ both decisions. PostgreSQL abort
một transaction với SQLSTATE `40001`. Serializable predicate locks (`SIReadLock`)
thường không block như row lock; chúng hỗ trợ conflict detection.

Loser không được dự đoán theo actor identity. Retry transaction mới sẽ thấy winner
đã commit, count `1`, rồi return `LAST_OPERATOR_REQUIRED`.

## Root cause theo layer

### PostgreSQL

`REPEATABLE READ` cung cấp snapshot isolation, không full serializability cho
multi-row predicate invariant.

### Spring

Transaction boundary đúng nhưng isolation/locking protocol chưa đủ. Inner
`SERIALIZABLE REQUIRED` không nâng outer transaction đã mở.

### Hibernate

Dirty checking sinh two correct versioned UPDATEs. Hibernate không biết two entity
instances cùng tham gia roster-level rule.

### Application model

Invariant không có shared authoritative write point. Check và writes nằm trên
different rows/snapshots.

## Tại sao đây không phải lost update

Không committed update nào overwrite update khác:

```text
A owns Alice false
B owns Bob false
both changes remain visible
```

Lost update làm một write biến mất. Write skew giữ cả hai writes nhưng combination
vi phạm constraint.

## So với DB-004

DB-004 dùng predicate count rồi INSERT new rows làm vượt upper capacity. DB-005
UPDATE existing different rows làm vi phạm lower bound. Cả hai cần reasoning trên
read/write sets, nhưng failure shape và representation solutions khác nhau.

## Commit và rollback

- Nếu A rollback, Alice vẫn true; B commit Bob=false, invariant giữ.
- Nếu both commit ở RR, invariant durable sai; database không tự repair.
- Ở SERIALIZABLE, transaction nhận `40001` đã aborted; toàn write/outbox trong nó
  rollback.
- Row/guard locks release ở commit, rollback hoặc connection failure.

Checked exception không mặc định rollback trong Spring. Domain exception dùng để
hủy attempt sau partial mutation phải là runtime hoặc cấu hình `rollbackFor`.

## Retry

Correct retry:

```text
attempt 1 gets 40001 and rolls back
bounded backoff/jitter
attempt 2 starts new physical transaction
reload roster snapshot
rerun count and decision
```

Không reuse managed entities hoặc transaction snapshot cũ. Hot roster có thể gây
retry amplification; cap attempts và trả retryable domain failure khi exhausted.

## Timeout và deadlock

Guard/all-row locking có thể block:

- cấu hình bounded lock/query timeout;
- không remote I/O khi giữ lock;
- lock roster IDs và operator IDs theo deterministic order;
- deadlock SQLSTATE `40P01` cũng cần new-transaction retry khi safe.

Lock timeout không chứng minh invariant failed; current transaction rollback rồi
caller có thể retry/reject theo SLO.

## Crash và duplicate

Crash trước commit rollback own row/guard lock. Crash sau commit trước response có
thể làm caller retry leave request. Command idempotency phải replay result hoặc
conditional `on_call=true -> false` làm duplicate no-op; nó vẫn không thay
roster-invariant coordination.

Outbox event `OPERATOR_LEFT_ON_CALL` chỉ publish cùng successful transaction. SSI
loser không được phát external event.

## Multi-instance

Actors có thể ở hai pods hoặc admin UI. `synchronized`/local cache chỉ cover một
process. Guard row, conditional counter, row locks hoặc SSI sống tại shared
PostgreSQL boundary nên coordinate mọi compliant application nodes.

Direct SQL bypass vẫn có thể phá protocol; permissions/stored procedure/trigger
có thể giới hạn mutation paths cho invariant critical.

## Observability

Log/metric:

```text
rosterId
operatorId
observedOnCallCount
effectiveIsolation
serializationFailure
lockWaitDuration
retryAttempt
leaveOutcome
```

Reconciliation:

```sql
select roster_id
from on_call_assignment
group by roster_id
having count(*) filter (where on_call) = 0;
```

Theo dõi `40001`, `40P01`, `55P03`, retry exhaustion, transaction duration và
unsafe-roster count. Exception count bằng `0` ở RR không phải success signal.
