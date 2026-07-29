# Phân tích statement snapshot và business decision

## Initial state

Trước khi hai actors bắt đầu, PostgreSQL có committed row:

```text
merchant_refund_policy(M-42)
  auto_refund_limit = 100.00
  active            = true
  revision          = 7
```

Refund request có amount `80.00`. Revision `7` cho phép auto-approve; revision
`8` với limit `50.00` thì không.

## Expected theo broken design

Developer thường suy luận:

```text
A đã BEGIN transaction
  -> policy không đổi trong mọi lần đọc của A
  -> revision dùng để quyết định và revision dùng để audit giống nhau
```

Suy luận này đúng với stable transaction snapshot phù hợp, nhưng không đúng cho
hai plain SELECT ở PostgreSQL `READ COMMITTED`.

## Actual interleaving

| Bước | Refund evaluator A | Risk administrator B |
| --- | --- | --- |
| 1 | `BEGIN READ COMMITTED` | |
| 2 | SELECT #1 bắt đầu với snapshot S1 | |
| 3 | Đọc limit `100`, revision `7` | |
| 4 | Quyết định `APPROVED` cho `80` | `BEGIN` |
| 5 | | UPDATE limit `50`, revision `8` |
| 6 | | `COMMIT` |
| 7 | SELECT #2 bắt đầu với snapshot S2 | |
| 8 | Đọc limit `50`, revision `8` | |
| 9 | INSERT approved, evaluated limit `100`, revision `8` | |
| 10 | `COMMIT` | |

Final state:

```text
current policy: limit=50, revision=8
decision:       amount=80, APPROVED, evaluated_limit=100, policy_revision=8
```

Cả A và B commit thành công. Không row nào bị hai transactions cùng update, nên
không có lost-update conflict.

> **Nói ngắn gọn:** PostgreSQL bảo đảm mỗi statement không thấy state nửa commit;
> nó không bảo đảm nhiều statements ở `READ COMMITTED` ghép lại thành một view
> duy nhất.

## Snapshot chính xác của từng statement

Ở `READ COMMITTED`, mỗi command lấy snapshot khi command bắt đầu:

```text
S1 created before B commit
  -> visible policy tuple: revision 7

B commits revision 8

S2 created after B commit
  -> visible policy tuple: revision 8
```

SELECT #1 không trở thành dirty read hay “sai” sau B commit. Nó là committed view
hợp lệ tại S1. Lỗi xuất hiện khi application kết hợp decision từ S1 với evidence
từ S2 mà không revalidate.

## MVCC tuple versions

B UPDATE tạo tuple version mới:

```text
old tuple: limit=100, revision=7
new tuple: limit=50,  revision=8
```

Sau B commit:

- snapshot S1 vẫn đã đọc old committed version;
- statement mới của A lấy S2 và chọn new committed version;
- old tuple còn có thể tồn tại vật lý cho active snapshots/vacuum rules;
- application vẫn nhìn cùng một logical row.

MVCC giảm read/write blocking; nó không đóng băng business object cho toàn
transaction ở mọi isolation level.

## Lock behavior

B UPDATE acquire row-level lock trên policy row và giữ tới commit. A plain SELECT:

- không acquire `FOR SHARE`/`FOR UPDATE` row lock;
- thường không block sau B vì đọc committed tuple theo snapshot;
- không giữ quyền ưu tiên cho lần đọc sau;
- không ngăn B commit giữa hai statements.

A INSERT lock row/index entries của `refund_decision`, không conflict với B UPDATE
policy row. Đây là lý do database không tự phát hiện evidence mismatch.

## Non-atomic sequence

Root sequence:

```text
read policy version V1
  -> decide in Java using V1
  -> concurrent commit changes policy to V2
  -> read audit fields from V2
  -> write decision combining V1 and V2
```

Transaction làm các writes của A atomic với nhau; nó không làm chuỗi nhiều reads
thành một atomic observation ở `READ COMMITTED`.

## Root cause theo layer

### PostgreSQL

`READ COMMITTED` intentionally dùng statement snapshots. Non-repeatable read là
behavior được phép, không phải database corruption.

### Spring

`@Transactional(isolation = READ_COMMITTED)` tạo đúng physical transaction nhưng
không yêu cầu stable transaction snapshot. Inner `REQUIRED` method cũng join
isolation đang có.

### Hibernate/JPA

Scalar projection phát SELECT mới; first-level cache không hợp nhất hai result
sets. Hibernate dirty checking/flush không thêm policy revision predicate vào
INSERT của decision.

### Application

Decision và evidence không có một explicit policy-version contract. Code đọc lại
một authoritative row nhưng không recompute, compare revision hoặc lock.

## Persistence context nuance

Managed entity identity có thể làm cùng entity instance được trả về hai lần:

```text
findById #1 -> managed entity revision 7
findById #2 -> same Java object revision 7, possibly no SQL
```

Điều này chỉ che interleaving. Một `refresh`, scalar query, native query hoặc
persistence context khác lại expose revision `8`. Correctness không nên phụ thuộc
vào query plan/cache accident.

Nếu query entity được thực thi lại, Hibernate vẫn có thể trả managed instance và
merge behavior phụ thuộc operation/flush mode. Test phải log SQL và dùng query
form có semantics rõ.

## `REPEATABLE READ`

PostgreSQL `REPEATABLE READ` dùng transaction snapshot:

```text
SELECT #1 -> revision 7
B commits revision 8
SELECT #2 -> vẫn revision 7
```

Nó loại non-repeatable read trong A. Tuy nhiên B vẫn có thể commit vì A chỉ đọc
policy và insert một row khác. Execution có thể được hiểu theo serial order:

```text
A decision theo revision 7
sau đó B đổi current policy thành revision 8
```

Nếu business cho phép decision hợp lệ theo policy tại snapshot, đây là acceptable.
Nếu requirement là “phải dùng policy mới nhất tại thời điểm A commit”,
`REPEATABLE READ` một mình chưa đủ.

## `SERIALIZABLE`

PostgreSQL Serializable Snapshot Isolation tìm serialization anomalies và có thể
abort transactions khi có dangerous dependency structure. Nhưng stronger
isolation không đồng nghĩa “mọi reader phải thấy commit mới nhất”.

Với dependency đơn giản A đọc policy/insert decision, B chỉ update policy, serial
order A trước B có thể hợp lệ nên cả hai vẫn có thể commit. Muốn policy update và
decision có ordering chặt, model read/write set hoặc explicit lock phải diễn đạt
ordering đó.

## Revalidation semantics

Có ba hợp đồng khác nhau:

1. **Snapshot-at-evaluation:** decision hợp lệ theo một policy version đã đọc.
2. **Current-at-write:** policy revision phải chưa đổi khi decision được insert.
3. **Locked-through-commit:** policy không được update cho tới decision commit.

Một final conditional statement có thể bảo vệ contract 2 tại statement boundary.
`FOR SHARE` giữ ordering tới commit cho contract 3. Không nên gọi cả ba là
“consistent” mà không định nghĩa thời điểm.

## Commit và rollback

### B rollback

Revision `8` không visible. SELECT #2 của A vẫn thấy revision `7`; anomaly không
xảy ra trong interleaving đó.

### A rollback

Decision INSERT bị rollback. Policy revision `8` của B vẫn committed độc lập.

### A commit trước B

Decision revision `7` có thể hợp lệ theo serial order A rồi B, miễn policy history
và audit contract cho phép.

## Timeout và lock wait

Broken plain SELECT không cần chờ policy UPDATE đã commit. Nếu chuyển sang
`FOR SHARE`, concurrent updater có thể:

- block tới A commit/rollback;
- fail vì `lock_timeout`;
- tham gia deadlock nếu nhiều policies được lock không theo deterministic order.

Timeout phải map thành domain retry/rejection phù hợp; không tăng timeout để che
transaction dài.

## Retry

Non-repeatable read ở `READ COMMITTED` không tự ném exception để trigger retry.
Application phải phát hiện revision mismatch hoặc affected-row `0`.

Retry đúng:

```text
attempt 1 transaction rolls back
bounded backoff/jitter
attempt 2 starts a new transaction
reload current policy
recompute the whole decision
```

Chỉ chạy lại INSERT với decision cũ là sai. Retry trong cùng transaction snapshot
cũng không đảm bảo freshness.

## Crash behavior

Nếu process A crash trước commit, PostgreSQL rollback transaction và không lưu
decision. Nếu crash sau commit nhưng trước response, caller có thể retry command;
unique `command_id`/idempotency xử lý duplicate delivery, nhưng không thay thế
policy consistency mechanism.

Policy revision và decision evidence phải durable trong cùng database contract để
reconciliation không phụ thuộc log trong process đã chết.

## Multi-instance

A có thể chạy ở App Node 1 và B ở App Node 2 hoặc admin tool. `synchronized`,
`ReentrantLock` và in-memory cache trên Node 1 không coordinate các actors đó.

Authoritative protection phải nằm ở PostgreSQL:

- immutable policy version + foreign key;
- conditional write/revision validation;
- row lock;
- hoặc isolation/model đủ diễn đạt invariant.

## Observability

Ghi structured fields:

```text
commandId
merchantId
amount
policyRevisionRead
policyRevisionWritten
decisionOutcome
effectiveIsolation
retryAttempt
```

Theo dõi:

- revision mismatch/conditional no-op count;
- lock wait và `lock_timeout`;
- serialization/deadlock retry count;
- decisions không join được policy history;
- approved amount vượt stored evaluated limit;
- transaction duration và connection pool pressure.

SQL/trace span cho hai SELECT phải có cùng transaction identifier để phân biệt
non-repeatable read với hai transactions hoàn toàn khác.
