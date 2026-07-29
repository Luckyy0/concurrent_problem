# LOCK-003 — Pessimistic write lock với `FOR UPDATE`

## Tóm tắt

Hai khách đồng thời giữ cùng ghế `A-10` của một suất chiếu. Quyết định không chỉ
là giảm một counter: hệ thống phải đọc trạng thái ghế, kiểm tra policy, tạo
`seat_hold` và cập nhật chủ sở hữu. Nếu cả hai transaction đọc `AVAILABLE` bằng
plain `SELECT`, cả hai có thể cùng quyết định chấp nhận trước khi writer đầu tiên
commit.

Với khóa ghi bi quan (`pessimistic write lock`), transaction khóa đúng
`show_seat` row trước khi quyết định:

```text
Tx-A: SELECT seat FOR UPDATE → validate → create hold A → COMMIT
Tx-B: SELECT same seat FOR UPDATE → wait
      → A commits → B đọc trạng thái mới HELD → reject
```

> **Nói ngắn gọn:** row lock biến hai quyết định cạnh tranh trên cùng ghế thành
> hai lượt tuần tự; actor thứ hai phải quyết định lại trên trạng thái đã commit.

## Actor và trạng thái dùng chung

| Thành phần | Trạng thái |
| --- | --- |
| `show_seat` | Suất `42`, ghế `A-10`, trạng thái `AVAILABLE` |
| Request A | Customer `501`, command `hold-a`, chạy trên App-1 |
| Request B | Customer `902`, command `hold-b`, chạy trên App-2 |
| `seat_hold` | Lịch sử durable của hold đã được chấp nhận |
| PostgreSQL | Nơi authoritative state và row lock cùng tồn tại |

Điểm tranh chấp (`contention point`) là locking query trên khóa chính
`(show_id, seat_no)`. Đây là một row đã tồn tại và được biết trước; application
không khóa một predicate mở như “bất kỳ ghế trống nào”.

## Invariant

```text
Tại một thời điểm, mỗi show seat có tối đa một ACTIVE hold.

show_seat.hold_id phải trỏ tới đúng ACTIVE seat_hold của ghế.

Một command ID chỉ tạo tối đa một seat_hold.

Caller chỉ nhận HELD sau khi transaction đã commit.
```

Khóa row bảo vệ read–decide–write trên ghế. Unique constraint của `command_id`
bảo vệ duplicate command; hai cơ chế giải quyết hai invariant khác nhau.

## Ranh giới transaction

`SeatHoldCoordinator` không mở transaction. Nó gọi `SeatHoldTx.execute()` qua
Spring proxy. Một attempt duy nhất bao gồm:

1. đặt PostgreSQL `lock_timeout` cục bộ cho transaction;
2. load `show_seat` bằng `PESSIMISTIC_WRITE`;
3. sau khi acquire lock, kiểm tra command replay và trạng thái ghế;
4. tạo `seat_hold`, cập nhật `show_seat`, rồi flush;
5. commit hoặc rollback trước khi trả kết quả qua proxy.

Timeout/deadlock được bắt ở coordinator, sau khi transaction lỗi đã rollback.
Remote I/O, user interaction và backoff không nằm trong transaction giữ lock.

## Lock acquisition và lifetime

Spring Data JPA gắn `LockModeType.PESSIMISTIC_WRITE` vào query. Hibernate dùng
locking clause phù hợp với PostgreSQL; SQL tương đương về ý nghĩa là:

```sql
select show_id, seat_no, state, hold_id, hold_until
from show_seat
where show_id = :showId
  and seat_no = :seatNo
for update;
```

Transaction acquire lock khi statement chạy, không phải khi method được gọi.
Lock tồn tại đến transaction commit/rollback, kể cả khi Java code không còn
chạm entity. Một competitor cần incompatible lock trên cùng row sẽ wait,
fail-fast hoặc timeout theo policy.

Plain MVCC reader không tự động bị chặn bởi row lock này; nó vẫn có thể đọc
committed version phù hợp với snapshot của statement.

## Actor thua cuộc nhận gì?

| Outcome của holder | Outcome của waiter |
| --- | --- |
| Holder commit `HELD` | Waiter thức dậy, đọc `HELD`, trả `ALREADY_HELD` |
| Holder rollback | Waiter acquire lock, đọc `AVAILABLE`, có thể tạo hold |
| Wait vượt `lock_timeout` | Statement lỗi, transaction rollback, trả `BUSY` |
| Deadlock victim | Transaction rollback; chỉ retry nếu policy cho phép |
| Mất connection/crash | PostgreSQL rollback session transaction và release lock |

Blocking không đồng nghĩa sẽ thành công. Sau khi wait, actor luôn phải revalidate
business state.

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong case |
| --- | --- |
| pessimistic locking | Giả định conflict có khả năng cao và reserve row trước quyết định |
| `PESSIMISTIC_WRITE` | JPA lock mode yêu cầu explicit database write lock |
| `FOR UPDATE` | PostgreSQL locking read chặn writer/locking reader cạnh tranh |
| lock holder | Transaction đang giữ incompatible lock |
| waiter | Transaction đang chờ acquire lock |
| lock lifetime | Từ lúc acquire đến transaction end |
| `lock_timeout` | Giới hạn thời gian chờ một database lock |
| revalidation | Kiểm tra lại business state sau khi acquire lock |
| lock ordering | Thứ tự ổn định khi một operation khóa nhiều rows |

## Điều hướng

- [Code read–decide–write bị hỏng](broken-code.md)
- [Timeline, snapshot, timeout và recovery](analysis.md)
- [Spring/JPA solution và các lựa chọn khác](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [Pessimistic locking](../../concepts/pessimistic-locking.md)
- [PostgreSQL locks và lock lifetime](../../concepts/postgresql-locks.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)
- [DB-007 — Row/table lock lifecycle](../../postgresql/row-table-lock-lifecycle/README.md)

## Hậu quả trong production nếu dùng sai

- Hai customer nhận hai hold hợp lệ về mặt application cho cùng ghế.
- `show_seat` chỉ giữ customer ghi cuối nhưng `seat_hold` còn hai ACTIVE rows.
- Lock transaction dài tạo wait queue, giữ connection và tăng tail latency.
- Không có timeout làm request chờ vượt client deadline.
- Catch lock error bên trong cùng transaction rồi tiếp tục gây lỗi “transaction
  is aborted”.
- Khóa nhiều ghế theo input order khác nhau tạo deadlock.
- `synchronized` có vẻ đúng ở một instance nhưng hỏng sau scale-out.

## Khi nên dùng cách này

`PESSIMISTIC_WRITE` phù hợp khi:

- resource row đã tồn tại và biết chính xác;
- quyết định cần nhiều bước nhưng phải dựa trên state mới nhất;
- conflict đủ thường xuyên để optimistic retries tạo wasted work;
- critical section trong database ngắn và có bounded wait;
- loser có domain outcome rõ như `ALREADY_HELD` hoặc `BUSY`.

Nếu invariant diễn đạt được bằng một conditional `UPDATE`, cách đó thường nhỏ
hơn và thuộc `LOCK-004`. Nếu invariant bao phủ rows chưa tồn tại hoặc một
predicate rộng, row lock trên một row không đủ; cần constraint, guard row hoặc
`SERIALIZABLE`.

## Phạm vi

Case chỉ xử lý known-row selection với `PESSIMISTIC_WRITE`/`FOR UPDATE`.
`SKIP LOCKED` cho work claiming thuộc `DB-010`; opposite-order deadlock thuộc
`DB-008`; strategy dưới sustained high contention thuộc `LOCK-005`.
