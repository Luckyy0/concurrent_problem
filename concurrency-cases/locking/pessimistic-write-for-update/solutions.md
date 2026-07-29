# Giải pháp — short transaction, bounded wait, revalidate

## Mục tiêu thiết kế

Correct path phải đồng thời đạt các điều kiện:

1. acquire database lock trước business decision;
2. locking query và mutations nằm trong cùng transaction;
3. waiter revalidate state sau khi acquire;
4. wait/deadlock có bounded failure contract;
5. remote I/O không kéo dài lock lifetime;
6. multi-row operation dùng deterministic order;
7. duplicate command và same-seat contention được xử lý riêng.

## Schema và defense in depth

```sql
create table show_seat (
    show_id bigint not null,
    seat_no varchar(16) not null,
    state varchar(16) not null,
    hold_id uuid,
    holder_customer_id bigint,
    hold_until timestamptz,
    primary key (show_id, seat_no),
    check (state in ('AVAILABLE', 'HELD', 'SOLD')),
    check (
        (state = 'AVAILABLE'
            and hold_id is null
            and holder_customer_id is null
            and hold_until is null)
        or (state = 'HELD'
            and hold_id is not null
            and holder_customer_id is not null
            and hold_until is not null)
        or state = 'SOLD'
    )
);

create table seat_hold (
    hold_id uuid primary key,
    command_id uuid not null unique,
    show_id bigint not null,
    seat_no varchar(16) not null,
    customer_id bigint not null,
    status varchar(16) not null,
    request_fingerprint varchar(64) not null,
    created_at timestamptz not null,
    check (status in ('ACTIVE', 'EXPIRED', 'CANCELLED', 'CONVERTED'))
);

create unique index uq_seat_hold_one_active
    on seat_hold (show_id, seat_no)
    where status = 'ACTIVE';
```

Partial unique index là safety net nếu một writer path quên lock. Nó không thay
thế domain flow: constraint loser nhận technical conflict muộn, còn locking path
có thể đọc state mới và trả `ALREADY_HELD` có chủ đích.

Expired hold phải được đổi khỏi `ACTIVE` trong cùng locked transaction trước khi
insert hold mới.

## Aggregate state

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

    boolean isHeldAndNotExpired(Instant now) {
        return state == SeatState.HELD && holdUntil.isAfter(now);
    }

    boolean hasExpiredHold(Instant now) {
        return state == SeatState.HELD && !holdUntil.isAfter(now);
    }

    void releaseExpiredHold() {
        if (state != SeatState.HELD) {
            throw new IllegalStateException("seat is not held");
        }
        state = SeatState.AVAILABLE;
        holdId = null;
        holderCustomerId = null;
        holdUntil = null;
    }

    void hold(UUID newHoldId, long customerId, Instant until) {
        if (state != SeatState.AVAILABLE) {
            throw new IllegalStateException("seat is not available");
        }
        state = SeatState.HELD;
        holdId = newHoldId;
        holderCustomerId = customerId;
        holdUntil = until;
    }

    UUID currentHoldId() {
        return holdId;
    }
}
```

Không cần `@Version` để pessimistic path hoạt động. Có thể giữ `@Version` như
defense cho writers khác, nhưng khi đó mọi write path phải hiểu cả optimistic
conflict; không thêm annotation chỉ để “cho chắc”.

## Locking repository

```java
public interface ShowSeatRepository
        extends JpaRepository<ShowSeat, ShowSeatId> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from ShowSeat s where s.id = :id")
    Optional<ShowSeat> findForUpdate(@Param("id") ShowSeatId id);

    @Query(value = """
            select *
            from show_seat
            where show_id = :showId
              and seat_no in (:seatNos)
            order by show_id, seat_no
            for update
            """, nativeQuery = true)
    List<ShowSeat> findAllForUpdateOrdered(
            @Param("showId") long showId,
            @Param("seatNos") Collection<String> seatNos
    );
}
```

Single-seat query dùng JPA `PESSIMISTIC_WRITE`. Với multi-row query, native SQL
làm `ORDER BY ... FOR UPDATE` explicit và dễ kiểm tra trong plan/log. Input vẫn
phải deduplicate; kết quả phải chứa đúng số rows requested trước khi mutate.

Locking query phải được gọi bên trong active transaction. Đừng đặt `@Transactional`
chỉ ở repository rồi dùng entity sau khi repository method kết thúc.

## PostgreSQL lock timeout cục bộ

JPA có `jakarta.persistence.lock.timeout`, nhưng mức hỗ trợ millisecond phụ thuộc
provider/dialect/driver. Với PostgreSQL-specific case, `set_config` tạo policy
kiểm chứng được:

```java
@Component
public class PostgreSqlLockTimeout {
    private final EntityManager entityManager;

    public PostgreSqlLockTimeout(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    public void apply(Duration timeout) {
        long millis = timeout.toMillis();
        if (millis < 1 || millis > 5_000) {
            throw new IllegalArgumentException("lock timeout out of range");
        }

        entityManager.createNativeQuery("""
                select set_config('lock_timeout', :value, true)
                """)
                .setParameter("value", millis + "ms")
                .getSingleResult();
    }
}
```

Tham số `true` làm setting chỉ tồn tại trong current transaction. Duration là
server configuration đã validate, không nhận tùy ý từ request.

## Transactional worker

```java
@Service
public class SeatHoldTx {
    private final ShowSeatRepository seats;
    private final SeatHoldRepository holds;
    private final PostgreSqlLockTimeout lockTimeout;
    private final EntityManager entityManager;
    private final Clock clock;

    public SeatHoldTx(
            ShowSeatRepository seats,
            SeatHoldRepository holds,
            PostgreSqlLockTimeout lockTimeout,
            EntityManager entityManager,
            Clock clock
    ) {
        this.seats = seats;
        this.holds = holds;
        this.lockTimeout = lockTimeout;
        this.entityManager = entityManager;
        this.clock = clock;
    }

    @Transactional(
            propagation = Propagation.REQUIRED,
            isolation = Isolation.READ_COMMITTED
    )
    public HoldResult execute(HoldSeatCommand command) {
        lockTimeout.apply(Duration.ofMillis(750));

        ShowSeat seat = seats.findForUpdate(command.seatId())
                .orElseThrow(SeatNotFoundException::new);

        Optional<SeatHold> replay =
                holds.findByCommandId(command.commandId());
        if (replay.isPresent()) {
            SeatHold previous = replay.orElseThrow();
            previous.requireSameRequest(command);
            return HoldResult.replayed(previous.holdId());
        }

        Instant now = clock.instant();
        if (seat.hasExpiredHold(now)) {
            holds.expire(seat.currentHoldId(), now);
            seat.releaseExpiredHold();
        }

        if (seat.isHeldAndNotExpired(now)) {
            return HoldResult.alreadyHeld(seat.currentHoldId());
        }
        if (seat.isSold()) {
            return HoldResult.sold();
        }

        UUID holdId = UUID.randomUUID();
        Instant holdUntil = now.plus(Duration.ofMinutes(2));

        seat.hold(holdId, command.customerId(), holdUntil);
        holds.save(SeatHold.active(holdId, command, holdUntil, now));
        entityManager.flush();

        return HoldResult.held(holdId, holdUntil);
    }
}
```

Entity được load và locked trong cùng persistence context. Revalidation diễn ra
sau wait. `flush()` phát hiện constraint/SQL error trước khi method body return;
caller chỉ thấy result sau transaction interceptor commit.

Nếu caller đã mở một outer transaction dài, `REQUIRED` sẽ kéo lock vào boundary
đó. Public API phải quy định coordinator không có transaction; có thể thêm guard
bằng `TransactionSynchronizationManager.isActualTransactionActive()` nếu codebase
có nguy cơ gọi sai.

> **Nói ngắn gọn:** annotation đặt trên query chỉ có ý nghĩa khi service
> transaction giữ lock liên tục từ lần đọc đến commit.

## Coordinator map lỗi sau rollback

```java
@Component
public class SeatHoldCoordinator {
    private final SeatHoldTx worker;
    private final LockFailureClassifier classifier;

    public SeatHoldCoordinator(
            SeatHoldTx worker,
            LockFailureClassifier classifier
    ) {
        this.worker = worker;
        this.classifier = classifier;
    }

    public HoldResult hold(HoldSeatCommand command) {
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            throw new IllegalStateException(
                    "seat hold coordinator must run outside a transaction"
            );
        }

        try {
            return worker.execute(command);
        } catch (RuntimeException failure) {
            return classifier.classify(failure)
                    .map(kind -> HoldResult.busy(kind.name()))
                    .orElseThrow(() -> failure);
        }
    }
}

@Component
public class LockFailureClassifier {
    public Optional<LockFailure> classify(Throwable failure) {
        for (Throwable current = failure;
             current != null;
             current = current.getCause()) {
            if (current instanceof SQLException sql) {
                return switch (sql.getSQLState()) {
                    case "55P03" -> Optional.of(LockFailure.LOCK_TIMEOUT);
                    case "40P01" -> Optional.of(LockFailure.DEADLOCK);
                    default -> Optional.empty();
                };
            }
        }
        return Optional.empty();
    }
}
```

Exception rời `SeatHoldTx`, nên Spring rollback trước khi coordinator xử lý.
Không đổi mọi `RuntimeException` thành `BUSY`; corruption, connection failure
và programming error phải tiếp tục fail.

Retry `55P03`/`40P01` không phải mặc định. Nếu product cho phép retry, mỗi
attempt cần transaction mới, reload, stable command ID, attempt cap, deadline và
backoff như `LOCK-002`.

## SQL mà solution dựa vào

Semantic SQL của một successful attempt:

```sql
begin;
select set_config('lock_timeout', '750ms', true);

select *
from show_seat
where show_id = 42 and seat_no = 'A-10'
for update;

-- Nếu hold cũ expired:
update seat_hold
set status = 'EXPIRED'
where hold_id = :oldHoldId and status = 'ACTIVE';

insert into seat_hold (
    hold_id, command_id, show_id, seat_no, customer_id,
    status, request_fingerprint, created_at
) values (
    :holdId, :commandId, 42, 'A-10', :customerId,
    'ACTIVE', :fingerprint, now()
);

update show_seat
set state = 'HELD',
    hold_id = :holdId,
    holder_customer_id = :customerId,
    hold_until = :holdUntil
where show_id = 42 and seat_no = 'A-10';

commit;
```

Lock acquisition nằm ở SELECT. Partial unique index và command uniqueness được
kiểm tra lúc insert/flush. Cả projection và audit commit atomically.

## Multi-seat operation

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public GroupHoldResult holdTogether(GroupHoldCommand command) {
    List<String> canonicalSeatNos = command.seatNos().stream()
            .distinct()
            .sorted()
            .toList();

    lockTimeout.apply(Duration.ofMillis(750));
    List<ShowSeat> locked = seats.findAllForUpdateOrdered(
            command.showId(),
            canonicalSeatNos
    );

    if (locked.size() != canonicalSeatNos.size()) {
        throw new SeatNotFoundException();
    }
    if (locked.stream().anyMatch(seat -> !seat.isAvailable(clock.instant()))) {
        return GroupHoldResult.unavailable();
    }

    // Mutate all locked rows, insert one group audit, then flush.
    return createGroupHold(command, locked);
}
```

Một SQL statement với stable `ORDER BY` dễ audit hơn vòng lặp queries. Mọi path
khóa cùng resources phải dùng cùng canonical order, kể cả cancel/sell/admin
operations.

## Remote workflow

Không giữ seat row lock trong payment authorization. Một boundary hợp lý:

1. short transaction acquire seat lock, tạo durable timed hold và outbox event;
2. commit/release lock;
3. payment workflow chạy ngoài transaction;
4. capture/cancel dùng transaction ngắn khác, lock seat và revalidate hold owner;
5. expiration worker chuyển ACTIVE → EXPIRED idempotently.

Timed hold là business state, không phải database lock kéo dài hai phút.

## Failure contract

| Failure | Transaction | API/domain result |
| --- | --- | --- |
| Seat đã HELD | Commit read-only/no-op | `ALREADY_HELD` |
| `55P03` lock timeout | Rollback | `BUSY`, có thể retry theo policy |
| `40P01` deadlock victim | Rollback | `BUSY`, bounded fresh retry nếu an toàn |
| Duplicate command, same fingerprint | Commit/no-op | Replay stored hold result |
| Duplicate command, different fingerprint | Rollback/reject | `IDEMPOTENCY_MISMATCH` |
| Partial unique conflict | Rollback | Điều tra missing lock path hoặc race |
| Crash trước commit | PostgreSQL rollback | Replay command để xác định outcome |
| Response mất sau commit | State vẫn committed | Replay theo command ID |

## Alternatives

### Conditional atomic `UPDATE`

Nếu rule chỉ là `AVAILABLE → HELD`, một guarded update có thể nhỏ hơn:

```sql
update show_seat
set state = 'HELD', hold_id = :holdId
where show_id = :showId
  and seat_no = :seatNo
  and state = 'AVAILABLE';
```

Affected rows `1/0` map thành win/lose. Khi policy, expired-hold transition và
audit composition phức tạp, explicit lock có thể dễ đọc hơn. Chi tiết thuộc
`LOCK-004`.

### Optimistic `@Version`

Không block tại read; loser conflict lúc flush rồi có thể reload/reject/retry.
Phù hợp contention thấp, nhưng repeated expensive decision tạo wasted work.

### Unique constraint

Partial unique index là safety net mạnh cho “một ACTIVE hold”. Nó không tự trả
đầy đủ current holder/policy outcome và không serialize arbitrary seat state
transitions.

### `SERIALIZABLE` hoặc guard row

Dùng khi invariant là predicate-wide hoặc rows có thể chưa tồn tại. Đừng giả
rằng `FOR UPDATE` trên zero selected rows khóa một gap.

### JVM/distributed lock

JVM lock không bảo vệ multi-instance. External distributed lock thêm lease,
fencing và dual-write failure trong khi state đã nằm ở PostgreSQL; không phải
lựa chọn đầu tiên cho single-database invariant.

## So sánh định tính

| Strategy | Loser | Throughput/latency | Retry | Deadlock risk | Multi-instance |
| --- | --- | --- | --- | --- | --- |
| `FOR UPDATE` bounded wait | Block rồi revalidate/timeout | Tốt khi Tx ngắn; convoy trên hot row | Không bắt buộc | Có khi multi-row | Có |
| `FOR UPDATE NOWAIT` | Fail ngay | Low wait, nhiều BUSY | Product-dependent | Thấp hơn wait dài, không bằng zero | Có |
| Conditional `UPDATE` | `0 rows` | Ít round trips | Thường không | Thấp cho single row | Có |
| `@Version` | Fail lúc flush | Không block read; wasted attempts | Có thể cần | Thấp cho single row | Có |
| Unique constraint | Fail insert | Gọn cho invariant expressible | Thường không | Thấp | Có |
| JVM lock | Block local | Không bảo vệ cross-node | Không | Local only | Không |

## Khuyến nghị

Dùng `PESSIMISTIC_WRITE` cho known seat row khi decision nhiều bước thực sự cần
serialize trước mutation. Giữ transaction ngắn, đặt `lock_timeout`, revalidate
sau lock, map technical failure ngoài boundary và dùng stable order cho nhiều
rows.

Không chữa contention bằng cách tăng timeout/pool vô hạn. Nếu hot-key load bền
vững, đo wait queue và đánh giá conditional SQL, admission control hoặc serialized
processing trong `LOCK-005`.

## Production checklist

- [ ] Lock target là row tồn tại và được biết chính xác.
- [ ] Locking query là lần load đầu tiên dùng cho decision.
- [ ] Service được gọi qua Spring proxy trong một transaction ngắn.
- [ ] Hibernate SQL đã được kiểm tra trên PostgreSQL thật.
- [ ] `lock_timeout` nhỏ hơn overall deadline.
- [ ] Timeout/deadlock để transaction rollback trước khi map/retry.
- [ ] Không có remote I/O hoặc executor wait khi giữ lock.
- [ ] Multi-row paths dùng cùng deterministic order.
- [ ] Duplicate command có unique constraint/fingerprint riêng.
- [ ] Metrics phân biệt holder duration, wait duration, timeout và deadlock.
