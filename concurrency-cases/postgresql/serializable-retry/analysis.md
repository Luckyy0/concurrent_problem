# Phân tích SSI, `40001` và transaction retry

## Trạng thái ban đầu

```text
merchant 7 limit             = 100
committed ACTIVE total       = 60
C1 requested amount          = 30
C2 requested amount          = 30
```

T1 và T2 đều chạy `SERIALIZABLE`, mỗi actor có connection, transaction,
persistence context và command ID riêng.

## Timeline bắt buộc

| Bước | T1 — C1 | T2 — C2 |
| ---: | --- | --- |
| 1 | `BEGIN SERIALIZABLE` | `BEGIN SERIALIZABLE` |
| 2 | không thấy decision C1 | không thấy decision C2 |
| 3 | đọc limit `100`, active total `60` | |
| 4 | | đọc limit `100`, active total `60` |
| 5 | quyết định `60 + 30 <= 100` | quyết định `60 + 30 <= 100` |
| 6 | insert reservation C1 | insert reservation C2 |
| 7 | insert decision `ACCEPTED` | insert decision `ACCEPTED` |
| 8 | flush/commit | flush/commit |
| 9 | ví dụ commit thành công | nhận `40001`, rollback |
| 10 | active total thành `90` | fresh retry đọc `90`, ghi `REJECTED` |

PostgreSQL có thể abort T1 thay vì T2, và failure có thể lộ ra trước `COMMIT`.
Correct code/test không phụ thuộc victim identity hay exact Java line ném lỗi.

> **Nói ngắn gọn:** SSI không chọn sẵn “request thứ hai” làm loser; nó chỉ bảo
> đảm tập transaction đã commit có một serial order hợp lệ.

## Kết quả mong đợi và thực tế

| Khía cạnh | Đúng | Broken |
| --- | --- | --- |
| Invariant | Final active total `90` | `40001` leak hoặc retry sai |
| C1/C2 | Một `ACCEPTED`, một `REJECTED` sau retry | Một command mất outcome |
| Attempt lỗi | Rollback toàn bộ | Application cố tiếp tục trên `25P02` |
| Snapshot | Fresh transaction thấy winner | Reuse snapshot vẫn thấy `60` |
| Side effect | Chỉ publish từ committed decision | Attempt aborted đã gửi notification |

Nếu dùng `READ COMMITTED` hoặc `REPEATABLE READ` cho cùng read/write shape, cả hai
distinct inserts có thể commit và đưa total lên `120`; không có same-row lost
update để tự phát hiện.

## Snapshot của `SERIALIZABLE`

PostgreSQL `SERIALIZABLE` đọc stable transaction snapshot giống nền tảng
snapshot isolation. T1 không tự thấy C2 insert và T2 không tự thấy C1 insert trong
snapshot của mình.

Khác biệt quan trọng là SSI còn theo dõi dependency để ngăn committed history
không thể serialize. Nó không biến mọi predicate read thành blocking lock và
không làm hai transaction tự chạy tuần tự.

Dữ liệu đọc trong một serializable transaction chỉ được xem là kết quả hợp lệ sau
khi transaction commit. Một result object đã được Java method tạo vẫn chưa phải
durable outcome nếu commit advice chưa hoàn tất.

## `SIReadLock` và predicate tracking

Khi hai transactions chạy `SUM ... WHERE merchant_id=7 AND status='ACTIVE'`,
PostgreSQL ghi nhận vùng dữ liệu đã đọc bằng predicate locks. Trong `pg_locks`,
chúng có mode `SIReadLock` và có thể ở tuple, page hoặc relation granularity tùy
query plan và lock promotion.

Các lock này:

- không block writer như `FOR UPDATE`;
- dùng để nhận biết write nào có thể làm thay đổi kết quả read trước đó;
- có thể được giữ sau commit cho tới khi overlapping read-write transactions kết
  thúc;
- có thể coarse hơn dự kiến nếu sequential scan hoặc predicate-lock memory
  pressure xảy ra.

Vì `SIReadLock` không blocking, `lock_timeout` không phải công cụ contain SSI
serialization conflict. Application deadline và bounded retry mới giới hạn tổng
thời gian.

## Read/write dependencies

Ký hiệu `Treader --rw--> Twriter` nghĩa là reader đã đọc state không gồm write
của writer, trong khi write đó ảnh hưởng predicate/version đã đọc.

Trong case:

```text
T1 reads active predicate --rw--> T2 inserts C2 into that predicate
T2 reads active predicate --rw--> T1 inserts C1 into that predicate
```

Hai cạnh tạo cycle trực tiếp. Tổng quát hơn, SSI tìm **cấu trúc nguy hiểm**
(`dangerous structure`) gồm các rw-conflicts kề nhau có thể nằm trong
serialization cycle. PostgreSQL có thể abort để giữ an toàn ngay cả khi
application không thấy một blocking lock wait.

Đây không phải deadlock:

| SSI serialization conflict | Lock deadlock |
| --- | --- |
| Dependency giữa snapshot reads và writes | Wait-for cycle trên incompatible locks |
| `SIReadLock` không block writer | Actor thực sự chờ lock |
| SQLSTATE `40001` | SQLSTATE `40P01` |
| Có thể lộ ở statement/commit | Detector phá lock wait cycle |
| Cả hai cần fresh whole-transaction retry khi policy cho phép | Cần canonical lock order ngoài retry |

## Vì sao loser phải chạy lại toàn bộ logic?

Attempt đầu đã quyết định dựa trên `active=60`. Sau winner commit, current state là
`90`. Chỉ chạy lại `INSERT` sẽ vẫn dùng decision/value cũ và phá invariant.

Fresh attempt phải thực hiện lại:

```text
check durable command decision
→ load limit
→ recompute active predicate
→ compare requested amount
→ write ACCEPTED/REJECTED decision
→ commit
```

PostgreSQL không tự retry vì database không biết application logic nào đã chọn
SQL, values hoặc external actions. Whole-transaction retry là responsibility của
caller/coordinator.

## Transaction failed state

Khi statement/commit trả `40001`:

- attempted writes không commit;
- transaction không được dùng tiếp;
- query tiếp theo trước rollback thường trả `25P02`;
- Spring proxy phải rollback rồi đóng/return connection;
- managed entities/result từ attempt không được dùng làm source cho retry.

`EntityManager.clear()`, savepoint hoặc bắt exception bên trong method không tạo
physical transaction mới. Backoff phải chạy sau rollback, bên ngoài transaction,
để không giữ connection/SIREAD tracking của attempt cũ.

## Nơi conflict có thể xuất hiện

Hibernate có thể gửi SQL ở nhiều thời điểm:

- repository query có thể trigger auto-flush;
- explicit `flush()` có thể nhận `40001`;
- transaction commit flushes pending changes;
- PostgreSQL có thể phát hiện serialization conflict khi xử lý statement hoặc
  lúc transaction kết thúc.

Vì vậy coordinator catch exception từ lời gọi **qua transactional proxy**, không
chỉ catch quanh một repository call. Classifier traverse cause chain để tìm
`SQLException#getSQLState() == "40001"`; message text và Spring wrapper class
không phải contract bền vững bằng SQLSTATE.

## Effective isolation trong Spring

`@Transactional(isolation = Isolation.SERIALIZABLE)` chỉ được áp dụng khi
transaction manager bắt đầu physical transaction mới:

- call phải đi qua Spring proxy;
- inner `REQUIRED` join outer transaction đã có, không nâng isolation;
- self-invocation bỏ qua advice;
- `DEFAULT` dùng datasource/database default;
- test có outer transaction có thể che mất boundary thật.

Mỗi integration test cần query:

```sql
select current_setting('transaction_isolation');
```

và assert `serializable` bên trong attempt.

## Commit, rollback và retry outcomes

### Winner commit

Reservation và command decision của winner commit atomically. `SIReadLock` có thể
còn được giữ nội bộ cho dependency tracking tới khi overlapping transactions
kết thúc; không được xem đó là leaked blocking lock.

### Victim rollback

Reservation/decision của victim biến mất. Nếu không retry, API phải trả
temporary failure rõ ràng; không được báo `ACCEPTED`.

### Fresh retry

Retry giữ nguyên command ID, mở snapshot mới, thấy active total `90` và commit
decision `REJECTED`. Business rejection không được retry tiếp.

### Retry exhaustion

Sau attempt cap/deadline, coordinator trả `LimitContentionException` hoặc một
stable retry-later outcome. Không transaction dở dang nào được để lại. Caller có
thể query command ID trước khi quyết định retry ở tầng ngoài.

## Idempotency và ambiguous outcome

`40001` là known abort: attempt không commit. Khác với connection loss đúng lúc
commit, outcome có thể không rõ từ phía client.

Durable `limit_command_decision(command_id primary key, outcome, ...)` cho phép:

- retry cùng command ID replay `ACCEPTED`/`REJECTED`;
- response loss sau commit không tạo reservation mới;
- concurrent duplicate command được unique constraint phân xử;
- outbox row tham chiếu cùng command/event ID.

Unique violation `23505` không được retry vô điều kiện. Nếu constraint là
`command_id` của chính command này, rollback rồi đọc committed decision trong
transaction mới; nếu là constraint khác, propagate đúng lỗi.

Idempotency không thay SSI. Command IDs khác nhau vẫn cần serialization để bảo vệ
merchant limit.

## Backoff, jitter và deadline

Retry ngay lập tức làm transactions cùng collision window gặp lại nhau. Policy
cần:

- allowlist `40001`;
- attempt cap;
- exponential backoff có random jitter;
- overall deadline/cancellation;
- business revalidation ở mỗi attempt;
- metric attempt, success-after-retry và exhaustion.

Không có con số delay phổ quát. Hot merchant, transaction duration, pool size và
SLO quyết định configuration. Retry amplification có thể làm một conflict ban
đầu thành overload nếu mỗi request tạo nhiều attempts.

## `SERIALIZABLE READ ONLY DEFERRABLE`

Long-running report chỉ đọc có thể dùng:

```sql
begin isolation level serializable read only deferrable;
```

PostgreSQL có thể chờ khi lấy snapshot an toàn; sau đó transaction không có nguy
cơ bị abort vì serialization conflict và giảm SSI overhead. Mode `DEFERRABLE`
không giúp read-write reservation attempt và không thay request deadline.

## Timeout và deadlock

`SERIALIZABLE` không loại bỏ ordinary row/table lock waits:

- `40P01`: deadlock; rollback và retry nếu safe, đồng thời sửa lock order;
- `55P03`: lock timeout; điều tra holder và latency contract;
- `57014`: statement canceled/timeout;
- `40001`: serialization failure.

Classifier và metrics phải giữ các outcome riêng. Backoff policy có thể chia sẻ,
nhưng root cause/remediation khác nhau.

## External side effect

Notification, HTTP call hoặc message gửi trước commit không rollback cùng
database. Một attempt có thể return `ACCEPTED` trong method body, gửi message rồi
nhận `40001` lúc commit.

Ghi outbox trong cùng successful transaction và publish sau commit. Consumer cần
deduplicate bằng event/command ID. Không giữ serializable transaction mở qua
remote I/O.

## Crash và connection loss

Crash trước commit làm PostgreSQL rollback transaction và bỏ attempted decision.
Crash sau server commit nhưng trước response tạo ambiguous outcome; retry cùng
command ID phải replay durable decision.

Nếu process chết trong backoff, không có database transaction đang mở. Caller/
durable command handler có thể tiếp tục theo idempotency contract.

## Multi-instance

SSI chạy tại shared PostgreSQL nên bảo vệ transactions từ App-1, App-2 và các
worker khác — miễn mọi relevant mutation dùng compatible isolation/protocol.
`synchronized` chỉ serialize một JVM và không thay đổi database snapshots.

Một direct SQL path chạy `READ COMMITTED` có thể không tham gia SSI guarantee như
mong đợi. Permission, stored procedure, service ownership và migration discipline
phải giới hạn bypass cho invariant quan trọng.

## Nguyên nhân gốc theo từng layer

| Layer | Vai trò |
| --- | --- |
| Application | Read → decide → insert trên predicate nhiều rows; thiếu retry contract |
| Spring | Proxy/isolation/transaction boundary quyết định có fresh attempt hay không |
| Hibernate | Flush timing quyết định nơi exception lộ ra |
| PostgreSQL | SSI theo dõi dependencies và abort với `40001` |
| JVM local | Không coordinate nhiều instances |

Nguyên nhân không phải PostgreSQL “ngẫu nhiên rollback”. Abort là một phần của
correctness contract khi chọn optimistic serializable execution.

## Khả năng quan sát (`observability`)

Theo dõi:

- `40001` theo operation và attempt number;
- success-after-retry, exhaustion, backoff/deadline;
- effective isolation và transaction duration;
- pool active/pending để thấy retry amplification;
- query plan/index changes ảnh hưởng predicate-lock granularity;
- `pg_locks.mode = 'SIReadLock'` khi điều tra;
- PostgreSQL logs có SQLSTATE/correlation ID nhưng không lộ bind data nhạy cảm.

`pg_stat_database.deadlocks` chỉ đếm deadlocks, không phải SSI failures.
Application metric/log classification là nguồn chính cho serialization failure
rate.
