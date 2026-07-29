# Phân tích versioned update và optimistic conflict

## Trạng thái ban đầu

Offer `42`: price `100`, version `7`. A và B có transactions/persistence contexts
riêng, cùng `READ COMMITTED`.

## Timeline có `@Version`

| Bước | Editor A | Editor B |
| ---: | --- | --- |
| 1 | load `100/v7` | |
| 2 | | load `100/v7` |
| 3 | set `90` | set `80` |
| 4 | `UPDATE ... version=7` → `1` | |
| 5 | commit `90/v8` | |
| 6 | | `UPDATE ... version=7` → `0` |
| 7 | | optimistic exception, rollback |

Final state `90/v8`. Loser không được báo success và không có partial write.

> **Nói ngắn gọn:** expected version biến “ai ghi sau thắng” thành compare-and-set:
> chỉ actor còn giữ revision hiện tại mới update được.

## SQL contract

```sql
update product_offer
set price = :newPrice,
    title = :title,
    version = :nextVersion
where offer_id = :offerId
  and version = :expectedVersion;
```

Affected rows:

- `1`: version match, row mới được tạo và version tăng;
- `0`: row bị xóa hoặc version đã đổi; Hibernate coi managed state là stale.

Application không nên tự dựa vào message text. Jakarta Persistence signal là
`OptimisticLockException`; Spring có thể translate thành
`ObjectOptimisticLockingFailureException`.

## PostgreSQL MVCC và row lock

A/B plain SELECT cùng thấy committed tuple `v7`; `@Version` không khóa lúc read.
Khi UPDATE:

- A acquire row lock, tạo tuple `v8`, commit;
- nếu B đến khi A chưa commit, B có thể chờ row lock;
- sau A commit, B command predicate `version=7` không match current row;
- affected rows `0`, không overwrite.

Vì vậy optimistic locking có thể vẫn gặp lock wait ngắn ở write phase. “Optimistic”
nghĩa conflict được detect thay vì giữ long read lock, không nghĩa không có
PostgreSQL row lock khi UPDATE.

## Hibernate dirty checking và flush

Setter chỉ đổi managed Java object. SQL được phát khi:

- explicit `EntityManager.flush()`/`JpaRepository.flush()`;
- auto-flush trước query liên quan;
- transaction commit.

Exception timing phụ thuộc provider/flush path. Nếu service không explicit flush,
method body có thể tạo result trước khi interceptor commit. Boundary ngoài proxy
phải xử lý failure.

## Transaction và persistence context sau conflict

`OptimisticLockException` trong active joined transaction đánh dấu rollback.
Managed entity loser vẫn có price `80`/version state trong memory nhưng không phải
committed truth.

Không:

- catch rồi dùng object để trả success;
- gọi `clear()` rồi tiếp tục trong transaction cũ;
- merge lại stale object mà không new transaction/revalidation.

Rollback đóng attempt. Nếu domain cho retry, transaction/persistence context mới
phải reload current aggregate; chi tiết policy thuộc `LOCK-002`.

## Client version và database version giải quyết hai cửa sổ

Có hai stale windows:

1. **Disconnected window:** user mở form v7; A commit v8 trước khi B request tới.
2. **Transaction window:** service B đã load v8; C commit v9 trước B flush.

Service compare `command.expectedVersion` với entity version giải quyết cửa sổ 1.
Hibernate version predicate giải quyết cửa sổ 2. Bỏ một trong hai khiến stale
intent vẫn có thể overwrite.

HTTP có thể dùng:

```text
GET /offers/42 → ETag: "7"
PUT /offers/42
If-Match: "7"
```

Contract mapping `412 Precondition Failed` hay `409 Conflict` phải nhất quán;
không gửi entity internals/sensitive fields trong conflict response.

## Detached entity và `merge`

Persistence provider phải kiểm tra version của detached entity khi merge, nhưng
timing vẫn có thể tới flush/commit. `merge()` trả một managed copy; detached input
không tự trở thành managed.

Mapping DTO → current managed entity với explicit expected version thường rõ hơn
merge graph lớn: tránh accidental cascade và over-posting. Dù chọn cách nào,
version từ client không được bỏ.

## Aggregate boundary

Version column bảo vệ entity/aggregate root được update. Nếu business invariant
trải trên nhiều independent rows:

- version từng row không tự phát hiện write skew;
- child collection ownership/mapping quyết định root version có tăng hay không;
- bulk JPQL/native UPDATE có thể bypass persistence context/version behavior nếu
  không tự viết predicate/increment.

Test phải cover exact mutation paths, kể cả collection/bulk operations.

## Vì sao không auto-retry edit này?

Command “set price to 80 dựa trên v7” chứa user intent gắn với state cũ. Sau A
commit `90/v8`, tự reload rồi set `80` làm retry thành overwrite có chủ ý mà user
chưa review.

Loser nên nhận conflict, current version và workflow reload/merge. Các command
commutative/delta có thể retry, nhưng cần new attempt, backoff/deadline và
idempotency ở `LOCK-002`.

## Commit, rollback, timeout và crash

- Winner commit: price/version visible atomically.
- Loser conflict: whole transaction rollback; locks release.
- Lock/statement timeout: khác optimistic conflict, không map thành stale edit.
- Crash trước commit: PostgreSQL rollback attempted version/update.
- Crash sau commit trước response: outcome ambiguous; command/audit ID giúp query.
- Delete cạnh tranh: affected rows `0` có thể map not-found-versus-conflict theo
  domain contract sau rollback/fresh read.

## Multi-instance

Version predicate nằm tại PostgreSQL nên App-1/App-2 cùng được bảo vệ. JVM
`synchronized` không cần cho correctness và không cover background/direct writers.

Mọi writer phải duy trì version contract. Native SQL bỏ `version=version+1` hoặc
predicate có thể làm JPA conflict detection sai. Giới hạn permissions và review
migrations/batch jobs.

## So với các cơ chế khác

| Cơ chế | Loser behavior | Phù hợp |
| --- | --- | --- |
| `@Version` | affected-row `0`, fail/rollback | Aggregate edit, conflict hiếm |
| Conditional SQL | affected-row theo business predicate | Counter/stock/delta |
| `FOR UPDATE` | block/timeout | Decision cần serialize trước mutation |
| `SERIALIZABLE` | `40001`, whole-Tx retry | Predicate/multi-row invariant |

## Nguyên nhân gốc theo layer

| Layer | Vai trò |
| --- | --- |
| Application/API | Có hoặc làm mất expected version/user intent |
| Spring | Transaction boundary và exception translation |
| Hibernate/JPA | Dirty checking, version predicate, affected-row check |
| PostgreSQL | MVCC tuple, row lock và row-count result |

## Khả năng quan sát (`observability`)

Theo dõi:

- optimistic conflict theo entity/use case;
- expected/current version trong structured debug log có kiểm soát;
- conflict response và user reload/merge outcome;
- transaction duration, flush location và SQL count;
- retry rate/exhaustion khi case khác cho phép retry;
- hot aggregate IDs trong sampled trace, không metric label high-cardinality.

Alert khi conflict rate tăng sau deploy/bulk job; đó có thể là contention thật,
client giữ form quá lâu hoặc writer bypass version increment.
