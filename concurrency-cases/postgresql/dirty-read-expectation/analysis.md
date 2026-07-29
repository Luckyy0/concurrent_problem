# PostgreSQL visibility analysis

## Initial state

```text
job_id          = IMPORT-42
status          = RUNNING
progress        = 20
generation      = 7
last commit     = Tx-0
```

Processor A và Watchdog B dùng independent PostgreSQL transactions/connections.

## Expected theo broken design

```text
A updates 80 but does not commit
B requests READ_UNCOMMITTED
B reads 80
B decides HEALTHY
```

## PostgreSQL actual

```text
A updates 80 but does not commit
B requests READ_UNCOMMITTED
B reads committed 20
B decides START_RECOVERY
```

Nếu A rollback, final vẫn `20`. Nếu A commit sau B read, một subsequent statement
bắt đầu sau commit mới có thể thấy `80`.

## Timeline ba actor

| Bước | Processor A — Tx-A | Watchdog B — Tx-B | Recovery C |
| --- | --- | --- | --- |
| T0 | BEGIN | | |
| T1 | UPDATE progress 80 | | |
| T2 | FLUSH, giữ Tx open | | |
| T3 | | BEGIN requested RU | |
| T4 | | SELECT snapshot thấy 20 | |
| T5 | | Return `startRecovery` | |
| T6 | | COMMIT | Starts duplicate work |
| T7 | A tiếp tục hoặc fail | | |
| T8 | COMMIT 80 hoặc ROLLBACK | | |

Không có dirty read ở T4. Business failure là watchdog dùng absence of
uncommitted progress làm bằng chứng job stalled.

> **Nói ngắn gọn:** committed visibility và liveness signal là hai contracts khác
> nhau; một open transaction không phải heartbeat channel.

## MVCC tuple versions

A UPDATE tạo một new tuple version về mặt MVCC:

```text
old version: progress=20, created by committed Tx-0
new version: progress=80, created by in-progress Tx-A
```

Snapshot của B:

- old version `20` visible;
- Tx-A chưa committed nên version `80` invisible;
- B không cần đọc inconsistent bytes/in-place value;
- B plain SELECT có thể tiếp tục mà không chờ A row lock.

A luôn thấy own writes trong transaction của chính A. Điều đó không có nghĩa B
được thấy.

## PostgreSQL isolation mapping

SQL standard cho phép dirty read ở `READ UNCOMMITTED`, nhưng PostgreSQL cung cấp
stronger guarantee: requested RU behaves as `READ COMMITTED`.

Có hai facts cần tách:

1. **reported/requested label:** `transaction_isolation` hoặc JDBC metadata có thể
   phản ánh `read uncommitted`/alias tùy cách driver/session set level;
2. **effective visibility semantics:** plain SELECT không thấy concurrent
   uncommitted rows.

Vì thế test đúng:

```text
record label for diagnostics
assert actual row visibility for correctness
```

Không viết test chỉ assert string `"read committed"` hoặc `"read uncommitted"` rồi
suy luận phenomenon.

## Statement snapshot

B plain SELECT dùng snapshot tại statement start:

```text
SELECT starts before A commit -> sees 20 for entire statement
A commits
next SELECT starts afterward  -> may see 80
```

Thay đổi giữa hai committed-only reads là non-repeatable read, thuộc DB-003. Nó
không biến read trước thành dirty read.

## Plain SELECT và row lock

A UPDATE giữ row-level lock tới commit/rollback. B plain SELECT:

- không xin incompatible `FOR UPDATE` lock;
- đọc old committed version;
- không block chỉ để đợi current writer.

B locking read:

```sql
select *
from job_run
where job_id = :id
for update;
```

có thể block A. Sau A:

- commit: B tiếp tục với committed new row theo command semantics;
- rollback: B tiếp tục với old row `20`.

Locking read chờ outcome; nó không expose dirty version.

## Rollback và aborted versions

A rollback:

```text
new tuple version 80 -> aborted/dead for visibility
old committed version 20 -> remains visible
```

Physical cleanup có thể diễn ra sau qua vacuum/hint-bit/page maintenance. Logical
visibility không chờ cleanup: B không được đọc aborted version.

Đây là lý do dirty-read progress nguy hiểm về semantics ngay cả trên database cho
phép nó: observer có thể hành động trên value sau đó biến mất.

## Flush/commit boundary

Hibernate flush:

- dirty-check entity;
- execute UPDATE;
- PostgreSQL tạo uncommitted version;
- acquire/retain locks;
- chưa make version visible cross-transaction.

Commit mới publish atomically. `saveAndFlush()` không phải progress broadcasting.

## Root cause theo layer

| Layer | Behavior | Kết luận |
| --- | --- | --- |
| Spring | Request isolation được truyền qua transaction manager | Annotation không hứa cross-DB identical phenomena |
| JDBC/driver | Set accepted isolation/alias | Label không override server MVCC |
| PostgreSQL | RU behavior như RC | No dirty rows |
| Hibernate | Flush không commit | Expected transaction semantics |
| Application | Dùng uncommitted visibility cho liveness | Root design error |

## Portability

Cùng tên isolation không bảo đảm database cho đúng minimum phenomenon set theo
cách application mong đợi; implementation có thể stronger.

Design dựa vào dirty read còn có vấn đề trên database thật sự cho phép:

- đọc partial/inconsistent unit;
- quyết định từ state sẽ rollback;
- UI hiển thị progress lùi;
- duplicate/compensation chạy theo provisional data;
- portability tests pass ở một engine nhưng fail ở engine khác.

Correct contract phải dùng committed state/explicit messaging, không chọn engine
để giữ dirty-read dependency.

## Watchdog semantics

Progress và final business outcome cần tách:

```text
attempt heartbeat/progress:
  independently committed, có attempt/generation/lease identity

final status:
  chỉ committed khi processing result durable
```

Watchdog không chỉ đọc percentage. Nó nên kiểm tra:

- lease owner/token/generation;
- committed heartbeat timestamp;
- allowed staleness/deadline;
- terminal status;
- atomic recovery claim.

Old committed progress không tự động cho phép C start. C phải win a conditional
claim để tránh nhiều watchdogs cùng recover.

## Long transaction effect

Nếu A giữ one transaction qua nhiều processing units:

- progress invisible lâu;
- locks/connections/snapshots giữ lâu;
- rollback mất toàn bộ uncommitted work;
- watchdog stale window lớn.

Chia workflow thành durable checkpoints/state transitions thường phù hợp hơn. Mỗi
checkpoint phải có domain semantics rõ, không giả làm final completion.

## Timeout và polling

Poll nhanh hơn chỉ tạo nhiều SELECTs thấy cùng committed `20` cho tới A commit.
Query timeout không thay visibility.

Nếu watchdog timeout:

- không retry recovery mù;
- attempt atomic lease/claim;
- recheck generation/status trong same conditional write;
- make duplicate processing idempotent nếu delivery có thể lặp.

## Crash behavior

- A crash trước commit: connection close/rollback; progress remains 20.
- A commit 80 rồi crash trước response: watchdog's next statement can see 80.
- Watchdog crash after deciding recovery: claim state phải durable/idempotent.
- Recovery C crash cần lease/command recovery riêng.

Dirty read không giải quyết unknown outcome hoặc crash coordination.

## Sequence caveat

PostgreSQL sequence changes có transactional rules khác row updates và có thể
visible/not rolled back theo cách riêng. Không dùng sequence behavior để suy luận
ordinary table-row dirty reads.

## Multi-instance

In-memory heartbeat/flag trên Processor node A không visible đáng tin cho Watchdog
node B. Coordination cần committed PostgreSQL row, durable message hoặc lease store
với ownership/fencing semantics.

PostgreSQL committed visibility nhất quán theo database transaction rules cho mọi
application instances; JVM local primitive không đổi nó.

## Observability

Ghi nhận:

- requested/reported isolation label và database product/version;
- observed committed progress/generation;
- processor transaction age;
- heartbeat age và lease owner/token;
- watchdog recovery decisions/conditional-claim outcome;
- duplicate recovery count;
- commit/rollback of progress checkpoints.

Không alert chỉ vì in-transaction entity state lớn hơn committed row; đó là expected
MVCC visibility, không phải replication lag hay database corruption.
