# Code hỏng — quyết định trước khi reserve row

## Schema tối thiểu

Legacy schema dùng một row làm seat projection và một bảng hold audit:

```sql
create table show_seat (
    show_id bigint not null,
    seat_no varchar(16) not null,
    state varchar(16) not null,
    hold_id uuid,
    holder_customer_id bigint,
    hold_until timestamptz,
    primary key (show_id, seat_no),
    check (state in ('AVAILABLE', 'HELD', 'SOLD'))
);

create table seat_hold (
    hold_id uuid primary key,
    command_id uuid not null unique,
    show_id bigint not null,
    seat_no varchar(16) not null,
    customer_id bigint not null,
    status varchar(16) not null,
    created_at timestamptz not null
);
```

Unique `command_id` ngăn replay cùng request. Schema này chưa có unique constraint
cho ACTIVE hold trên cùng seat, nên application phải bảo vệ invariant seat.

## Entity không có conflict detection

```java
@Entity
@Table(name = "show_seat")
public class ShowSeat {
    @EmbeddedId
    private ShowSeatId id;

    @Enumerated(EnumType.STRING)
    private SeatState state;

    private UUID holdId;
    private Long holderCustomerId;
    private Instant holdUntil;

    protected ShowSeat() {}

    boolean isAvailable(Instant now) {
        return state == SeatState.AVAILABLE
                || state == SeatState.HELD && !holdUntil.isAfter(now);
    }

    void hold(UUID newHoldId, long customerId, Instant until) {
        state = SeatState.HELD;
        holdId = newHoldId;
        holderCustomerId = customerId;
        holdUntil = until;
    }
}
```

Entity không có `@Version`. Plain load cũng không acquire explicit row lock.

## Broken service

```java
@Service
public class BrokenSeatHoldService {
    private final ShowSeatRepository seats;
    private final SeatHoldRepository holds;
    private final Clock clock;

    public BrokenSeatHoldService(
            ShowSeatRepository seats,
            SeatHoldRepository holds,
            Clock clock
    ) {
        this.seats = seats;
        this.holds = holds;
        this.clock = clock;
    }

    @Transactional
    public HoldResult hold(HoldSeatCommand command) {
        ShowSeat seat = seats.findById(command.seatId())
                .orElseThrow(SeatNotFoundException::new);

        Instant now = clock.instant();
        if (!seat.isAvailable(now)) {
            return HoldResult.alreadyHeld(seat.getHoldId());
        }

        UUID holdId = UUID.randomUUID();
        seat.hold(holdId, command.customerId(), now.plusSeconds(120));
        holds.save(SeatHold.active(holdId, command, now));

        return HoldResult.held(holdId);
    }
}
```

Code hợp lý nếu chỉ đọc tuần tự: method có `@Transactional`, entity managed và
Hibernate flush mutation lúc commit. Nhưng transaction không reserve row ở bước
đọc.

## Preconditions để lỗi xảy ra

- PostgreSQL dùng `READ COMMITTED`.
- Ghế `(42, A-10)` đang `AVAILABLE`.
- Request A và B dùng hai command/customer khác nhau.
- Cả hai plain `SELECT` hoàn tất trước khi một transaction flush.
- Không có `@Version` hoặc partial unique constraint cho ACTIVE hold.

Khi flush, PostgreSQL serialize hai `UPDATE` statements trên row, nhưng thời
điểm đó cả hai application decision đã là “accept”.

## Interleaving tạo double hold

| Bước | Tx-A | Tx-B |
| --- | --- | --- |
| 1 | `BEGIN` | |
| 2 | đọc `AVAILABLE` | |
| 3 | | `BEGIN` |
| 4 | | đọc `AVAILABLE` |
| 5 | tạo `seat_hold A` | tạo `seat_hold B` |
| 6 | update seat → holder A | |
| 7 | commit | |
| 8 | | update seat → holder B, rồi commit |

Final `show_seat` trỏ tới B, nhưng cả hold A và B đều là `ACTIVE`. Database row
lock tự động của `UPDATE` chỉ serialize writes; nó không quay lại chạy
`isAvailable()` cho Tx-B.

> **Nói ngắn gọn:** khóa ở lúc ghi là quá muộn; business decision đã được đưa ra
> từ hai lần đọc không reserve cùng resource.

## Sai lầm 1 — Tin rằng `@Transactional` tự khóa entity

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    ShowSeat seat = seats.findById(command.seatId()).orElseThrow();
    // Không có FOR UPDATE. Transaction chỉ tạo atomic commit boundary.
    return decideAndWrite(seat, command);
}
```

Transaction bảo đảm các writes commit/rollback cùng nhau. Nó không biến plain
`SELECT` thành locking read và không ngăn transaction khác đọc cùng committed
version.

## Sai lầm 2 — Lock trong repository transaction rồi mới mutate

```java
public HoldResult hold(HoldSeatCommand command) {
    ShowSeat seat = seats.findForUpdate(command.seatId()).orElseThrow();
    // Repository transaction đã kết thúc tại đây nếu outer service không có Tx.
    return mutateInAnotherTransaction(seat, command);
}
```

Lock lifetime gắn với database transaction. Nếu query transaction kết thúc
trước decision/write, lock đã release và không bảo vệ critical section.

## Sai lầm 3 — Plain-load trước, lock sau nhưng dùng stale object

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    ShowSeat stale = seats.findById(command.seatId()).orElseThrow();

    // Có thể phát thêm locking query, nhưng return value bị bỏ qua.
    seats.findForUpdate(command.seatId());

    if (stale.isAvailable(clock.instant())) {
        return decideAndWrite(stale, command);
    }
    return HoldResult.alreadyHeld(stale.getHoldId());
}
```

Đừng thiết kế correctness dựa trên giả định lock call luôn refresh một managed
instance đã load trước đó. Lock ngay từ query đầu tiên và quyết định trên object
được trả về; nếu phải nâng lock, refresh state một cách explicit.

## Sai lầm 4 — Bắt timeout rồi tiếp tục cùng transaction

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    try {
        return decideAndWrite(seats.findForUpdate(command.seatId()).orElseThrow(),
                command);
    } catch (PessimisticLockException ex) {
        return holds.findByCommandId(command.commandId())
                .map(HoldResult::replayed)
                .orElseGet(HoldResult::busy);
    }
}
```

Sau PostgreSQL statement error, transaction thường đã ở aborted state. Query
fallback trong cùng transaction không phải recovery. Hãy để exception thoát qua
transaction interceptor để rollback, rồi map/retry ở boundary bên ngoài.

## Sai lầm 5 — Giữ lock trong remote call

```java
@Transactional
public HoldResult holdAndCharge(HoldSeatCommand command) {
    ShowSeat seat = seats.findForUpdate(command.seatId()).orElseThrow();
    PaymentReply reply = paymentClient.authorize(command.payment());
    return updateSeatAfterReply(seat, command, reply);
}
```

Network latency, provider timeout và retry kéo dài lock lifetime. Waiters vẫn giữ
connection; một slow holder có thể tạo lock convoy và cạn pool.

## Sai lầm 6 — Khóa nhiều ghế theo request order

```java
@Transactional
public void holdPair(List<ShowSeatId> requestedOrder) {
    for (ShowSeatId id : requestedOrder) {
        ShowSeat seat = seats.findForUpdate(id).orElseThrow();
        validate(seat);
    }
}
```

Request `[A-10, A-11]` và `[A-11, A-10]` có thể tạo wait-for cycle. Mọi code
path phải canonicalize cùng một deterministic lock order trước khi acquire.

## Sai lầm 7 — `synchronized` quanh database call

```java
public synchronized HoldResult hold(HoldSeatCommand command) {
    return transactionalWorker.hold(command);
}
```

App-1 và App-2 có monitor khác nhau. JVM lock không serialize requests sau load
balancer và không sống qua process crash/deployment.

## Sai lầm 8 — Dùng `SKIP LOCKED` cho ghế khách đã chọn

Với interactive booking, “row không trả về vì đang locked” khác “seat không tồn
tại”. `SKIP LOCKED` phù hợp hơn với pool công việc thay thế được; nó có thể làm
API báo sai rằng ghế `A-10` biến mất.

## Sai lầm 9 — Lock một row không tồn tại

```sql
select *
from show_seat
where show_id = 42 and seat_no = 'A-99'
for update;
```

Nếu query trả zero rows thì không có seat row nào được khóa. `FOR UPDATE` cũng
không khóa toàn bộ predicate chống một insert tương lai. Known resource cần row
ổn định; predicate-wide invariant cần constraint, guard row hoặc isolation phù
hợp.

## Dấu hiệu quan sát được

- Hai ACTIVE holds nhưng seat projection chỉ trỏ tới một hold.
- Hibernate SQL log chỉ có plain select trước update.
- Wait xuất hiện ở UPDATE/flush, muộn hơn business decision.
- `pg_stat_activity.wait_event_type = 'Lock'` tăng khi remote call nằm trong Tx.
- Lock timeout/deadlock được log như lỗi kỹ thuật 500 thay vì domain outcome.
- Lỗi chỉ xuất hiện sau khi chạy nhiều instance.

## Điều code hỏng thực sự thiếu

Không phải “thiếu transaction”, mà thiếu atomicity cho chuỗi:

```text
read current seat → decide eligibility → create hold → update seat
```

Correct path phải acquire authoritative lock trước bước đọc dùng cho decision,
giữ lock đến khi toàn bộ database mutation commit/rollback, và bound thời gian
chờ.
