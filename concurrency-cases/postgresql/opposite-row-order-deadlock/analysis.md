# Phân tích chu trình chờ, victim và rollback

## Trạng thái ban đầu

Hai rows đã commit:

```text
account 101 (A): balance = 1_000
account 202 (B): balance = 1_000
```

T1 chạy `transfer(101, 202, 100)`. T2 chạy
`transfer(202, 101, 70)`. Mỗi request đi qua Spring proxy và mở một transaction
`READ COMMITTED` riêng trên một physical connection riêng.

## Timeline tạo deadlock trong PostgreSQL

| Bước | T1 — A→B | T2 — B→A | Wait-for graph |
| ---: | --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` | Chưa có cạnh chờ |
| 2 | khóa row A bằng `FOR UPDATE` | | T1 giữ A |
| 3 | | khóa row B bằng `FOR UPDATE` | T2 giữ B |
| 4 | xin khóa B và block | | T1 → T2 |
| 5 | | xin khóa A và block | T1 → T2 → T1 |
| 6 | detector thấy cycle | detector thấy cycle | PostgreSQL chọn một victim |
| 7 | ví dụ T2 nhận `40P01` | statement lỗi, transaction aborted | T2 phải rollback |
| 8 | T2 rollback, release B | T1 acquire B và tiếp tục | Cycle bị phá |
| 9 | T1 debit A, credit B, commit | | A=`900`, B=`1_100` |

Victim có thể là T1 thay vì T2. Correctness và test chỉ được assert “đúng một
victim cho cycle này”, không được hard-code connection nào thua.

> **Nói ngắn gọn:** PostgreSQL cứu khả năng tiến triển bằng cách hủy một
> transaction; database không thể tự quyết command đó nên được retry hay trả lỗi
> nghiệp vụ.

## Kết quả mong đợi và thực tế

| Khía cạnh | Mong đợi của team | Broken actual |
| --- | --- | --- |
| Locking | Hai transfer tự serialize | Mỗi transfer giữ một row và chờ row kia |
| Progress | Cả hai commit ở lần đầu | Một victim bị abort với `40P01` |
| Atomicity | Debit và credit cùng commit | PostgreSQL rollback victim, nhưng caller phải xử lý failure |
| Retry | Chạy lại vài dòng sau catch | Transaction cũ đã aborted; cần rollback và fresh attempt |
| Scale-out | Local service code đủ bảo vệ | Mọi instance cùng tranh chấp tại PostgreSQL |

Transaction bảo đảm atomic rollback của attempt; nó không bảo đảm attempt luôn
commit và không tự tạo canonical lock order.

## Đồ thị chờ (`wait-for graph`) trong PostgreSQL

Khi T1 chạy:

```sql
select * from account where id = 202 for update;
```

row B đã bị T2 khóa. Backend T1 chờ transaction T2 kết thúc vì outcome của T2
quyết định tuple version/lock có thể dùng. Tương tự, T2 chờ transaction T1 trên
row A. Hai cạnh tạo cycle:

```text
T1 owns row A ── waits for row B owned by T2
 ▲                                      │
 └──────── owns row B, waits for A ─────┘
```

PostgreSQL không chạy full deadlock check cho mọi micro-wait ngay lập tức. Sau
khi wait kéo dài đến `deadlock_timeout`, detector kiểm tra graph. Nếu tìm thấy
cycle, nó abort một transaction để các actor khác có thể tiến tiếp.

`deadlock_timeout` là detection delay/tuning knob, không phải business deadline.
Giảm nó không xóa circular wait và có thể tăng chi phí kiểm tra trong workload có
nhiều lock waits bình thường.

## Các loại lock thực sự tham gia

Mỗi `SELECT ... FOR UPDATE`:

- acquire relation-level `ROW SHARE` lock trên bảng `account`;
- acquire row-level lock trên tuple phù hợp;
- khi row đang do transaction khác khóa, backend thường chờ transaction ID của
  holder hoàn tất.

`ROW SHARE` relation locks của T1 và T2 tương thích với nhau. Cycle của case nằm
ở hai account rows/transaction waits, không phải vì “cả table bị khóa”.

Trong `pg_locks`, một waiter có thể hiện wait trên `transactionid`; row-level
locks không phải lúc nào cũng xuất hiện như hai dòng `tuple` trực quan. Cần nối
`pg_stat_activity`, `pg_locks`, `pg_blocking_pids(pid)` và application
correlation ID, thay vì suy luận chỉ từ một lock row.

## MVCC và statement snapshot

Ở `READ COMMITTED`, mỗi statement lấy snapshot mới. Tuy nhiên
`SELECT ... FOR UPDATE` không chỉ đọc visible version; nó còn phải acquire lock
trên row được chọn trước khi trả kết quả cho application.

Trong timeline:

- statement khóa A của T1 thấy committed balance `1_000`;
- statement khóa B của T2 cũng thấy committed balance `1_000`;
- mỗi statement thứ hai biết row mục tiêu tồn tại nhưng phải chờ incompatible
  lock;
- plain `SELECT` từ dashboard vẫn có thể đọc committed tuple version và không
  phải là cạnh trong cycle;
- sau khi holder kết thúc, locking statement còn sống tiếp tục theo PostgreSQL
  command/snapshot semantics và application phải revalidate state đã khóa.

Nâng isolation không sửa opposite ordering. `REPEATABLE READ`/`SERIALIZABLE` còn
có thêm conflict outcomes riêng; `DB-009` xử lý serialization failure.

## Transaction victim bị abort nghĩa là gì?

Statement nhận `40P01` thất bại và transaction chứa nó chuyển sang trạng thái
failed. Mọi database changes của transaction đó — kể cả thay đổi trước khi xin
lock thứ hai — không được commit.

Nếu application cố chạy:

```sql
select 1;
```

trước rollback, PostgreSQL thường trả:

```text
25P02: current transaction is aborted, commands ignored until end of transaction block
```

Spring transaction interceptor nhận exception đi ra khỏi method và đánh dấu/
thực hiện rollback. Sau rollback:

- row, relation và transaction locks của victim được release;
- persistence context của attempt không được tái sử dụng;
- waiter còn lại có thể acquire lock và tiếp tục;
- retry, nếu policy cho phép, phải mở transaction mới và reload cả hai accounts.

`EntityManager.clear()` không thay thế rollback. Catch exception rồi tiếp tục
trong cùng `@Transactional` method là sai boundary.

## Các kết quả commit và rollback

### Transaction thắng commit

Winner acquire row thứ hai sau victim rollback, validate balance trên state đang
khóa, flush hai `UPDATE` statements rồi commit. Cả debit và credit trở nên
visible cùng nhau; locks được release tại commit.

### Transaction victim rollback, không retry

Caller nhận kết quả technical conflict/temporarily unavailable. Không thay đổi
nào của attempt được giữ. Nếu API đã hứa xử lý bất đồng bộ, command phải được lưu
durably ở boundary khác; không được giả vờ success.

### Transaction victim rollback rồi retry

Attempt mới có snapshot, connection/persistence context và locks mới. Nó có thể
thấy balance do winner vừa commit, nên phải chạy lại business validation. Retry
không chỉ lặp statement thứ hai.

Nếu two opposing commands đều hợp lệ, loser retry sau winner có thể commit:

```text
sau T1: A=900, B=1100
sau T2 retry: A=970, B=1030
total = 2000
```

Giá trị cụ thể chỉ minh họa; invariant quan trọng là atomicity và conservation
trong scope của example.

## Phân biệt deadlock và timeout

| Failure | SQLSTATE thường gặp | Cơ chế | Hướng xử lý |
| --- | --- | --- | --- |
| deadlock detected | `40P01` | detector tìm thấy cycle, abort victim | rollback; bounded whole-transaction retry nếu safe |
| lock timeout | `55P03` | wait vượt `lock_timeout`, có thể không có cycle | rollback current transaction; điều tra holder/latency |
| statement canceled | `57014` | `statement_timeout` hoặc cancel | rollback khi transaction failed; tôn trọng deadline |
| serialization failure | `40001` | SSI/snapshot conflict | fresh whole-transaction retry; xem DB-009 |

Không map tất cả thành “insufficient funds” hoặc HTTP success. Technical
contention và business rejection là hai outcome khác nhau.

## Vì sao canonical order phá cycle?

Đặt total order theo stable unique account ID:

```text
first  = min(fromId, toId)
second = max(fromId, toId)
```

Cả A→B lẫn B→A đều xin A trước. Nếu T1 giữ A, T2 phải chờ A trước khi có thể giữ
B. Vì T2 chưa giữ B, T1 vẫn acquire B được; không thể hình thành cạnh ngược
T1→T2 và T2→T1 trên hai rows này.

Proof phụ thuộc mọi participating code path dùng cùng comparator và cùng
resource identity. Những lỗi sau làm proof mất hiệu lực:

- endpoint sort theo numeric ID nhưng batch sort theo external account number;
- comparator có thể coi hai resource khác nhau là bằng nhau;
- một path khóa account rồi customer, path khác khóa customer rồi account;
- trigger, foreign key hoặc statement khác acquire thêm resources theo order
  không được phân tích.

Vì vậy ordering làm deadlock ít hơn/rõ hơn, không cho phép application bỏ hoàn
toàn error classification và retry policy.

## Ranh giới Spring và Hibernate

`@Lock(PESSIMISTIC_WRITE)` khiến conflict xảy ra ngay tại repository query. Với
deferred dirty checking, balance `UPDATE` thường phát lúc flush/commit, nhưng
deadlock trong case xuất hiện trước mutation, tại locking SELECT thứ hai.

Nếu code mutate source trước rồi mới lock destination, Hibernate có thể
auto-flush trước query thứ hai tùy flush mode/query space. Cycle hoặc conflict có
thể xuất hiện ở SQL khác với dự đoán. Correctness không được phụ thuộc “exception
luôn ném ở dòng repository thứ hai”.

Spring có thể translate Hibernate/JDBC exception thành
`CannotAcquireLockException` hoặc một `DataAccessException` phù hợp. Retry
classifier nên tìm root `SQLException#getSQLState()` là `40P01`, đồng thời giữ
cause chain trong log/trace.

Đặt `@Retryable` và `@Transactional` trên cùng method khiến correctness phụ thuộc
advisor order. Thiết kế dễ kiểm chứng hơn là non-transactional retry coordinator
gọi một transactional attempt worker trên bean khác.

## External side effect và nguy cơ trùng lặp

Database rollback không thu hồi email, HTTP call hoặc message đã phát. Nếu
attempt gọi external system trước commit rồi bị chọn làm victim:

- database state rollback;
- external side effect có thể đã thành công;
- retry phát side effect lần nữa.

Giải pháp là giữ remote I/O ngoài lock-holding transaction và dùng
outbox/idempotency/reconciliation phù hợp. Deadlock retry tự nó không cung cấp
exactly-once.

## Process crash và mất connection

Nếu application process chết hoặc connection bị đóng, PostgreSQL abort
transaction chưa commit và release locks. Waiter có thể tiến tiếp, nhưng caller
không luôn biết commit đã xảy ra hay chưa nếu connection mất đúng lúc commit.
Command quan trọng cần idempotency key/status lookup để giải quyết ambiguous
outcome; blind retry có thể duplicate business operation.

## Multi-instance

Local `synchronized`, `ReentrantLock` hoặc in-memory account mutex không tạo cạnh
coordination giữa App-1 và App-2. Authoritative rows và PostgreSQL locks mới là
boundary dùng chung.

Scale-out làm tăng số transactions có thể cùng nhắm hot accounts. Canonical row
order phải là protocol toàn hệ thống, được dùng bởi API, batch, scheduler và
reconciliation worker. Mỗi instance cũng giữ connection trong lúc lock wait, nên
pool/application deadline phải được theo dõi cùng database locks.

## Nguyên nhân gốc theo từng layer

| Layer | Vai trò |
| --- | --- |
| Application | Chọn lock order theo source/destination, tạo circular wait |
| Spring | Mở transaction đúng nhưng retry cùng boundary có thể sai |
| Hibernate/JPA | Phát locking SELECT và translate/propagate conflict |
| PostgreSQL | Giữ row locks, phát hiện cycle, abort victim với `40P01` |
| JVM-local lock | Không bảo vệ transactions trên application instance khác |

Nguyên nhân gốc là non-atomic multi-resource acquisition không có total order
chung, không phải “PostgreSQL chạy chậm” hay “hai request đến cùng lúc”.

## Khả năng quan sát (`observability`)

Cần thu thập:

- metric `deadlock_detected_total` theo operation, không gắn high-cardinality
  account ID;
- retry attempt/outcome/exhaustion và elapsed deadline;
- SQLSTATE, transaction/correlation ID, canonical resource order;
- `pg_stat_database.deadlocks`;
- PostgreSQL deadlock log với `log_lock_waits`/query context phù hợp;
- `pg_stat_activity`, `pg_locks` và `pg_blocking_pids` khi điều tra live waits;
- connection pool active/pending/acquisition timeout.

Không log balance, token hoặc bind value nhạy cảm. Deadlock log là bằng chứng về
cycle đã xảy ra; lock wait metric giúp thấy contention trước khi thành incident.
