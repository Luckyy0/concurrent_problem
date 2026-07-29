# Phân tích — reserve trước khi quyết định

## Initial state

```text
show_seat(42, "A-10")
  state              = AVAILABLE
  hold_id            = null
  holder_customer_id = null
  hold_until         = null

seat_hold = []
```

Request A và B chạy trên hai Spring instances, dùng hai database connections và
hai transaction độc lập. Cả hai muốn tạo ACTIVE hold cho cùng primary key.

## Timeline code hỏng

| Bước | Tx-A / App-1 | Tx-B / App-2 | PostgreSQL |
| --- | --- | --- | --- |
| 1 | `BEGIN` | | A có statement snapshot |
| 2 | plain SELECT → `AVAILABLE` | | Không acquire explicit row lock |
| 3 | | `BEGIN` | B có statement snapshot |
| 4 | | plain SELECT → `AVAILABLE` | Cùng committed tuple version |
| 5 | quyết định ACCEPT A | quyết định ACCEPT B | Hai decisions đã tách khỏi write |
| 6 | insert hold A, update seat | | UPDATE acquire row lock |
| 7 | `COMMIT` | | Seat tạm thời trỏ A |
| 8 | | insert hold B, update seat | B có thể overwrite sau khi wait |
| 9 | | `COMMIT` | Seat trỏ B, hai ACTIVE holds |

PostgreSQL serialize conflicting updates, nhưng không biết method
`isAvailable()` là predicate cần chạy lại. Không có version predicate, conditional
SQL hay unique ACTIVE-seat constraint để biến stale decision thành conflict.

## Expected và actual

| | Expected | Actual với plain SELECT |
| --- | --- | --- |
| Số ACTIVE holds | 1 | 2 |
| Seat projection | Trỏ unique ACTIVE hold | Chỉ trỏ writer cuối |
| Request loser | `ALREADY_HELD` | Có thể nhận `HELD` |
| Audit consistency | Một accepted decision | Hai accepted decisions |

## Timeline với `FOR UPDATE`

| Bước | Tx-A / App-1 | Tx-B / App-2 | PostgreSQL |
| --- | --- | --- | --- |
| 1 | `BEGIN` | | |
| 2 | SELECT A-10 `FOR UPDATE` | | A acquire row lock |
| 3 | đọc `AVAILABLE`, tạo hold A | | Lock vẫn thuộc Tx-A |
| 4 | | SELECT A-10 `FOR UPDATE` | B block trên incompatible lock |
| 5 | flush update/insert | wait | |
| 6 | `COMMIT` | wait | A changes visible, lock release |
| 7 | | acquire rồi nhận row `HELD` | B tiếp tục với current row |
| 8 | | revalidate → `ALREADY_HELD` | Không tạo hold B |
| 9 | | `COMMIT` | Lock B release |

Điểm tuyến tính hóa nghiệp vụ (`linearization point`) là lúc transaction acquire
lock và đọc row dùng cho decision. Các actor cạnh tranh cùng row không còn đi qua
decision đồng thời.

> **Nói ngắn gọn:** waiter không tiếp tục với quyết định cũ; nó chỉ được quyết
> định sau khi thấy outcome đã commit hoặc rollback của holder.

## Snapshot tại `READ COMMITTED`

Mỗi statement có snapshot riêng. Plain reader chỉ cần visible committed tuple
version, nên không mặc định chờ `FOR UPDATE` holder.

Locking read thì khác:

1. statement tìm row theo primary key;
2. nếu row đang có incompatible lock, backend chờ;
3. sau khi holder kết thúc, PostgreSQL lock và trả updated row, hoặc không trả
   row nếu nó đã bị delete;
4. application phải đánh giá `state`, `hold_until` và policy trên row đó.

Vì query theo primary key luôn còn qualify nếu row không bị delete, Tx-B nhận
state `HELD` sau A commit. Nếu A rollback, B nhận `AVAILABLE`.

`REPEATABLE READ` không phải thay thế mặc định cho workflow này. Snapshot và
concurrent update behavior khác, có thể tạo serialization failure thay vì đúng
wait-and-revalidate contract đang thiết kế.

## Lock nào thực sự được lấy?

`SELECT ... FOR UPDATE` lấy relation lock cần thiết cho statement và row-level
lock trên selected tuple. Transaction khác muốn `UPDATE`, `DELETE` hoặc
incompatible locking read trên cùng row phải wait/fail.

Row lock không có nghĩa “khóa cả table” và không chặn plain SELECT thông thường.
Nó cũng không reserve row không tồn tại hay toàn bộ search predicate.

Hibernate `PESSIMISTIC_WRITE` yêu cầu explicit database lock. Dialect và query
shape quyết định locking clause chính xác; hãy kiểm tra emitted SQL thay vì suy
luận từ annotation name. Correctness cần một active database transaction bao
trùm cả query lẫn mutation.

## Lock lifetime

```text
BEGIN
  set local lock_timeout
  SELECT ... FOR UPDATE  ← acquire
  revalidate
  insert/update/flush
COMMIT or ROLLBACK        ← release
```

Lock không release khi:

- repository method return;
- entity rời một Java block;
- application bắt đầu remote call;
- Hibernate đã phát xong UPDATE nhưng transaction chưa end.

Do đó transaction duration chính là upper bound lý tưởng của lock lifetime,
trừ phần wait trước khi acquire.

## Spring proxy và Hibernate flush

`@Lock` chỉ gắn lock mode lên repository query. Transaction boundary thường nằm
ở public service method được gọi qua Spring proxy:

```text
caller
  → transaction interceptor BEGIN
  → repository locking query
  → managed entity mutation
  → Hibernate flush
  → database COMMIT
  → interceptor returns
```

Nếu self-invocation bỏ qua proxy hoặc outer method không có transaction, query
có thể fail vì transaction-required hoặc lock release trước business mutation.

Dirty checking thường phát UPDATE lúc flush/commit. Gọi `flush()` giúp conflict
hoặc constraint error xuất hiện bên trong attempt method, nhưng success chỉ chắc
chắn sau khi proxy commit thành công.

## Holder commit và rollback

### Holder commit

Tx-A changes trở nên visible rồi lock được release. Tx-B acquire lock và đọc
`HELD`; kết quả đúng là domain rejection/no-op. Blocking không cho phép B dùng
snapshot business cũ.

### Holder rollback

Uncommitted hold A biến mất và row quay về committed state trước đó. Tx-B acquire
lock, đọc `AVAILABLE`, tạo hold B và commit. Không cần cleanup lock riêng.

### Crash hoặc connection loss

Khi backend session chấm dứt, PostgreSQL abort transaction và release locks.
Application không được giả định response mất nghĩa là transaction chắc chắn
rollback: nếu connection mất sau commit nhưng trước response, caller cần replay
theo stable command ID để phân giải ambiguous outcome.

## Timeout và transaction aborted state

`lock_timeout` chỉ đo thời gian chờ acquire từng database lock. Khi vượt hạn,
PostgreSQL phát SQLSTATE `55P03` (`lock_not_available`). Hibernate/Spring dịch
exception theo provider và statement path; production classifier nên giữ
SQLSTATE/cause để phân biệt timeout với lỗi business.

Sau database statement error, transaction không còn là nơi an toàn để query
fallback. Luồng đúng:

```text
locking statement fails
→ exception leaves transactional method
→ Spring marks/executes rollback
→ outer coordinator maps to BUSY or starts a bounded fresh attempt
```

`statement_timeout`, Spring transaction timeout, client deadline và pool
acquisition timeout có phạm vi khác. Lock wait budget phải nhỏ hơn overall
request deadline và chừa thời gian rollback/response.

## Wait, `NOWAIT` hay `SKIP LOCKED`

| Policy | Hành vi | Phù hợp |
| --- | --- | --- |
| Bounded wait | Chờ holder rồi revalidate | Interactive exact resource, short Tx |
| `NOWAIT` | Fail ngay nếu resource bận | Caller có fallback rõ |
| `SKIP LOCKED` | Bỏ qua locked rows | Worker chọn item thay thế được |
| Unbounded wait | Tail latency phụ thuộc holder | Không nên là default API |

Với ghế `A-10` do user chọn, `SKIP LOCKED` làm mất khác biệt giữa “đang bận” và
“không tồn tại”. Bounded wait hoặc `NOWAIT` tạo contract rõ hơn.

## Multi-row operation và deadlock

Giữ hai ghế liền nhau cần lock cùng order ở mọi code path:

```text
sort by (show_id, seat_no)
→ acquire all rows in that order
→ validate all
→ mutate all
```

Nếu Tx-X lock A-10 rồi A-11 trong khi Tx-Y làm ngược lại, có thể xuất hiện
wait-for cycle. PostgreSQL deadlock detector chọn một victim với SQLSTATE
`40P01`; victim rollback. Deterministic order giảm cycle nhưng không loại mọi
nguồn deadlock trong hệ thống.

Retry deadlock chỉ hợp lệ khi:

- attempt cũ đã rollback;
- command idempotent/replayable;
- transaction mới reload state;
- attempt/deadline/backoff đều bounded.

Chi tiết detector/retry thuộc `DB-008` và shared deadlock concept.

## Duplicate command khác concurrent mutation

Unique `seat_hold.command_id` ngăn cùng command tạo hai audit rows. Nó không
ngăn `hold-a` và `hold-b` cùng chọn A-10.

Row lock serialize hai commands trên một seat. Nó không tự phát hiện cùng
`command_id` được gửi cho seat/customer khác; replay handler vẫn cần so request
fingerprint hoặc các trường identity.

## Multi-instance

`FOR UPDATE` thuộc PostgreSQL transaction nên App-1 và App-2 cùng tranh chấp tại
authoritative boundary. Đây là khác biệt cốt lõi với `synchronized` hoặc
`ReentrantLock`, vốn chỉ có hiệu lực trong một JVM.

Scale-out không phá correctness của row lock, nhưng tăng số potential waiters và
tổng connection capacity. Admission control, pool sizing và lock timeout vẫn là
vấn đề vận hành.

## Lock convoy và fairness

Một slow holder tạo queue trên hot seat. Waiters thường giữ connection trong lúc
chờ. PostgreSQL không cung cấp business fairness guarantee như “request đến
trước chắc chắn thắng”; timeout, cancellation và scheduler có thể thay đổi
outcome.

Metrics cần tách:

- lock acquisition latency;
- count/sum của SQLSTATE `55P03` và `40P01`;
- transaction duration sau khi acquire;
- pool active/pending/acquisition timeout;
- `HELD`, `ALREADY_HELD`, `BUSY` theo seat hotness;
- long/idle-in-transaction sessions.

## Quan sát holder và waiter

```sql
select a.pid,
       a.application_name,
       a.state,
       a.wait_event_type,
       a.wait_event,
       a.xact_start,
       pg_blocking_pids(a.pid) as blocking_pids,
       a.query
from pg_stat_activity a
where a.datname = current_database()
  and a.application_name like 'lock003-%';
```

Row-level waits thường hiện qua transaction ID lock; không kỳ vọng mỗi tuple
luôn có một row dễ đọc trong `pg_locks`. Correlation/application name dành cho
test và production diagnostics cần tránh chứa dữ liệu nhạy cảm.

## Root cause theo layer

### Application

Chuỗi `read → decide → write` không atomic. Hai distinct commands đều được
accept trên cùng state cũ.

### Spring

`@Transactional` chỉ định transaction boundary; nó không tự chọn pessimistic
lock. Proxy boundary sai có thể release lock quá sớm.

### Hibernate/JPA

Plain entity load không yêu cầu `PESSIMISTIC_WRITE`. Dirty checking flush xảy ra
sau decision, nên automatic UPDATE lock đến quá muộn.

### PostgreSQL

`READ COMMITTED` cho phép hai plain SELECT đọc cùng committed version. Automatic
row locking của UPDATE serialize statements nhưng không chạy lại Java predicate.

## Scope boundary

Known `show_seat` row là lock target ổn định. Nếu business rule là “tối đa 100
holds thỏa một predicate” hoặc “không có row nào trong khoảng thời gian”, khóa
một row trả về hiện tại không bảo vệ phantom/missing row. Khi đó cần constraint,
stable guard row, atomic counter hoặc `SERIALIZABLE` tùy invariant.
