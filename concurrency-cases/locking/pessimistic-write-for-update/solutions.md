# Giải Pháp Tối Thượng: Đánh Nhanh Rút Gọn, Chờ Có Giới Hạn và Xác Nhận Lại

## 1. Mục tiêu tối thượng (Mục tiêu thiết kế)

Để code chạy mượt mà không dẫm đạp lên nhau, con đường chân lý phải thỏa mãn 7 điều răn sau:

1. **Giật Khóa Dưới DB** trước khi bắt não Suy Nghĩ phán quyết.
2. Lệnh Xin Khóa và Lệnh Lưu Dữ Liệu phải nằm **chung 1 Giao Dịch (Transaction)**.
3. Kẻ đi sau (waiter) vớ được Khóa thì phải **Đọc lại Dữ Liệu Mới** (Revalidate) chứ không xài đồ cũ.
4. Kẹt Khóa thì phải **có Hẹn Giờ** (Bounded wait), quá giờ là tự hủy, không để treo máy.
5. Tuyệt đối **KHÔNG gọi API ngoài** (Remote I/O) khi đang ôm Khóa.
6. Mua nhiều ghế phải **Xếp Hàng từ bé đến lớn** (Stable order) để chống kẹt chéo (Deadlock).
7. Bấm lưu đúp 2 lần (Duplicate command) và Tranh nhau 1 ghế (Contention) là 2 bài toán hoàn toàn khác nhau, phải trị riêng!

## 2. Lưới Phòng Ngự Cuối Cùng Dưới DB (Schema & Defense in Depth)

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

Cái `Unique Index` kia chính là "Bùa Hộ Mệnh". Lỡ may code Java quên xin Khóa, thằng nào chạy update chậm hơn sẽ bị DB vả vỡ mặt (Technical conflict). Nhưng đừng ỷ lại vào nó, vì nó quăng lỗi "DB nổ" chứ không phải lỗi "Ghế đã bán" lịch sự đâu! Kẻ thua cuộc (loser) chết muộn quá, trong khi con đường Xin Khóa xịn xò sẽ giúp ta đọc được Dữ liệu mới và trả về `ALREADY_HELD` nhẹ nhàng hơn.

Ghế đã giữ quá hạn (Expired hold) phải được quét dọn về KHÔNG ACTIVE ngay trong cái Transaction đó trước khi chèn thêm vé giữ ghế mới.

## 3. Class Java Chống Đạn (Aggregate state)

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

Nhắc lại: KHÔNG CẦN `@Version` (Khóa Lạc Quan) để cách làm Bi Quan này hoạt động. Việc nhét bừa `@Version` vào "cho chắc ăn" chả giải quyết được gì ngoài việc làm cho hệ thống thêm rối rắm (vì mọi code viết ra đều phải bắt được lỗi Lạc Quan). Khóa Bi Quan tự nó đã đủ mạnh rồi!

## 4. Xin Khóa Qua Spring Data JPA (Locking repository)

```java
public interface ShowSeatRepository
        extends JpaRepository<ShowSeat, ShowSeatId> {

    // Chiêu thức 1: Xin khóa 1 ghế (JPA tự sinh FOR UPDATE)
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from ShowSeat s where s.id = :id")
    Optional<ShowSeat> findForUpdate(@Param("id") ShowSeatId id);

    // Chiêu thức 2: Xin khóa NHIỀU GHẾ (Viết SQL thuần để ép Sort chống kẹt)
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

Lệnh `findForUpdate` chạy ghế đơn dùng JPA `PESSIMISTIC_WRITE` cực tiện. Với query hốt nhiều dòng, ta viết hẳn SQL thuần có chữ `ORDER BY ... FOR UPDATE` cho rõ ràng, dễ log ra mà theo dõi. Danh sách ghế đầu vào cũng phải loại bỏ trùng lặp đi nhé.

Lệnh này BẮT BUỘC phải nằm bên trong 1 Giao Dịch (Transaction) đang chạy. Đừng có gắn `@Transactional` hời hợt ở mức Repository, lấy data ra là Khóa bị tuột mất tiêu luôn đấy!

## 5. Hẹn Giờ Bom Nổ Dưới Postgres (`lock_timeout`)

JPA có cái thẻ cấu hình Timeout, nhưng nó chạy hên xui tùy Driver/Provider. Để chắc cú với PostgreSQL, ta phang thẳng câu lệnh này:

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

        // Cài đồng hồ đếm ngược cho cái Giao Dịch hiện tại
        entityManager.createNativeQuery("""
                select set_config('lock_timeout', :value, true)
                """)
                .setParameter("value", millis + "ms")
                .getSingleResult();
    }
}
```

Tham số `true` cực kỳ quan trọng: Nó bảo DB "chỉ cài đồng hồ cho cái Giao dịch hiện tại thôi, Giao dịch khác kệ nó". Con số Timeout phải là tham số thiết lập sẵn đàng hoàng, đừng lấy linh tinh từ Payload của User ném xuống.

## 6. Người Hùng Đứng Mũi Chịu Sào (Transactional worker)

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
        // 1. Gài bom 750 mili-giây
        lockTimeout.apply(Duration.ofMillis(750));

        // 2. Phá cửa xin Khóa! (Nếu kẹt sẽ phải đứng chờ ở đây)
        ShowSeat seat = seats.findForUpdate(command.seatId())
                .orElseThrow(SeatNotFoundException::new);

        // 3. Đọc được Data Mới rồi, chống Spam Click trước:
        Optional<SeatHold> replay =
                holds.findByCommandId(command.commandId());
        if (replay.isPresent()) {
            SeatHold previous = replay.orElseThrow();
            previous.requireSameRequest(command);
            return HoldResult.replayed(previous.holdId());
        }

        Instant now = clock.instant();
        
        // 4. Nếu vé cũ đã Hết Hạn -> Đá văng thằng cũ ra
        if (seat.hasExpiredHold(now)) {
            holds.expire(seat.currentHoldId(), now);
            seat.releaseExpiredHold();
        }

        // 5. Nếu ghế đang bận (người đi trước đã ăn mất) -> Buông tay
        if (seat.isHeldAndNotExpired(now)) {
            return HoldResult.alreadyHeld(seat.currentHoldId());
        }
        if (seat.isSold()) {
            return HoldResult.sold();
        }

        // 6. Ghế trống! Lên đỉnh thôi!
        UUID holdId = UUID.randomUUID();
        Instant holdUntil = now.plus(Duration.ofMinutes(2));

        seat.hold(holdId, command.customerId(), holdUntil);
        holds.save(SeatHold.active(holdId, command, holdUntil, now));
        
        entityManager.flush(); // Đẩy xuống DB test thử Bùa Hộ Mệnh

        return HoldResult.held(holdId, holdUntil);
    }
}
```

Nhìn kỹ nhé: Entity được Tải Lên và Xin Khóa trong CÙNG MỘT HƠI THỞ (Persistence context). Nếu lỡ kẹt Khóa phải chờ, lúc chui vào được thì dữ liệu cũng được Đọc Mới (Revalidate). Gọi `flush()` để dụ cho các Lỗi DB bung bét ra ngay trong ruột cái hàm này, đừng để ra khỏi Proxy mới nổ.

Nếu kẻ gọi nó có sẵn Transaction dài ngoằng, từ khóa `REQUIRED` sẽ giật ngược cái Khóa này ra ăn vạ. Vì vậy, tốt nhất cái lớp ở ngoài API (Coordinator) KHÔNG ĐƯỢC PHÉP mở Transaction nào cả.

> **Nói ngắn gọn:** Annotation nhét ở Repository chỉ thiêng khi cái Service gọi nó tạo 1 cái Transaction trùm ôm trọn từ lúc Đọc Khóa đến lúc Lưu Xong (Commit).

## 7. Người Dọn Rác (Coordinator map lỗi)

Một khi bị Postgres chửi vì Lố Giờ hoặc Kẹt Cứng, cái Transaction coi như nát. Thằng `SeatHoldTx` cứ thế vứt lỗi thẳng ra ngoài cho Spring Rollback. Đứng ngoài hứng rác là ông Coordinator này:

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
        // Cấm tuyệt đối mở Transaction từ ngoài này!
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            throw new IllegalStateException(
                    "seat hold coordinator must run outside a transaction"
            );
        }

        try {
            return worker.execute(command);
        } catch (RuntimeException failure) {
            // Lính chết thì dịch lỗi DB thành lỗi Kinh doanh lịch sự:
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

Tuyệt đối đừng điên khùng biến mọi cái `RuntimeException` thành lỗi "Busy" giả trân. Bị đứt cáp DB hay Code Ngốc (Programming error) thì vẫn phải quăng lỗi 500 chửi sấp mặt! 

Và đừng nghĩ Retry lỗi `55P03` là khôn. Nếu muốn Retry, bạn phải code cơ chế Retry y chang cái vụ Khóa Lạc Quan `LOCK-002`: Reload mới, Backoff hở thời gian, Limit số lần đàng hoàng!

## 8. Túm Lại Đoạn SQL Thực Chạy Trông Ra Sao?

Nếu chạy suôn sẻ, dòng SQL dưới DB sẽ như thế này:

```sql
begin;
-- Bấm giờ:
select set_config('lock_timeout', '750ms', true);

-- Đòi Khóa:
select *
from show_seat
where show_id = 42 and seat_no = 'A-10'
for update;

-- Suy nghĩ, nếu cũ quá hạn thì Hủy:
update seat_hold
set status = 'EXPIRED'
where hold_id = :oldHoldId and status = 'ACTIVE';

-- Tạo vé mới:
insert into seat_hold (
    hold_id, command_id, show_id, seat_no, customer_id,
    status, request_fingerprint, created_at
) values (
    :holdId, :commandId, 42, 'A-10', :customerId,
    'ACTIVE', :fingerprint, now()
);

-- Cập nhật ghế mới:
update show_seat
set state = 'HELD',
    hold_id = :holdId,
    holder_customer_id = :customerId,
    hold_until = :holdUntil
where show_id = 42 and seat_no = 'A-10';

commit; -- Chốt sổ nhả Khóa!
```

Xin Khóa ngay lệnh Đọc. Cái Bùa Hộ Mệnh Unique Index sẽ phát huy tác dụng chém đầu khi Insert/Flush. Cả khối Logic dính chùm vào 1 Giao Dịch duy nhất.

## 9. Mua Nhiều Ghế Cùng Lúc Thì Sao? (Multi-seat)

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public GroupHoldResult holdTogether(GroupHoldCommand command) {
    // 1. Sắp Xếp Danh Sách Ghế Từ A -> Z!!!
    List<String> canonicalSeatNos = command.seatNos().stream()
            .distinct()
            .sorted()
            .toList();

    lockTimeout.apply(Duration.ofMillis(750));
    
    // 2. Chọt 1 phát ăn n Khóa! (Nhờ cái ORDER BY trong hàm nativeQuery)
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

    // Ghi đè vào các row bị khóa, chèn audit, rồi flush.
    return createGroupHold(command, locked);
}
```

Viết 1 câu SQL gom chùm có lệnh `ORDER BY` dễ kiểm soát hơn là móc vòng lặp for chọt từng dòng. Mọi ngóc ngách code từ Mua, Bán, Hủy, Đổi đều phải dùng cái trật tự Sort ghế `canonicalSeatNos` này. Thằng khóa A->B, thằng khóa B->A là cắn chéo (Deadlock) nổ tung đầu.

## 10. Chuyện Gọi API Thanh Toán (Remote workflow)

TUYỆT ĐỐI KHÔNG ôm Khóa DB đi gọi API Thanh toán! Hãy tách nó ra:
1. Giao Dịch Ngắn: Xin Khóa -> Đánh dấu Ghế HELD trong 10 phút, ném Event -> Nhả Khóa.
2. Xử lý Trôi dạt bên ngoài Transaction: Gọi API Thanh toán (Thoải mái lag 5 giây 10 giây).
3. Giao Dịch Ngắn Khác: Thanh toán xong -> Xin Khóa lại -> Kiểm tra xem ta có còn là Chủ Ghế không -> Đổi Ghế thành SOLD -> Nhả Khóa.
4. Một thằng Robot (Worker) quét rác dọn các ghế HELD quá giờ về EXPIRED.

Thời gian giữ ghế (Timed hold) là câu chuyện của Kinh doanh (10 phút), Đừng gán nó thành cái Database Row Lock chặn họng ngâm suốt 2 phút!

## 11. Bảng Xử Lý Thảm Họa (Failure contract)

| Bị Gì? | DB Phải Làm Gì | Trả về App thế nào |
| --- | --- | --- |
| Ghế đã HELD | Đọc xong lơ đi (No-op) | Báo `ALREADY_HELD` lịch sự |
| Nổ `55P03` (Chờ lố giờ) | Rollback cháy máy | Báo `BUSY`, khách tự bấm lại |
| Nổ `40P01` (Deadlock xui xẻo) | Rollback cháy máy | Báo `BUSY`, thử chọc lại nếu an toàn |
| Khách bấm lộn đúp 2 lần cùng lệnh | Lơ đi, chả làm gì | Trả về kết quả HELD cũ |
| Khách bấm lộn 2 lần khác dấu vân tay | Rollback | Chửi `IDEMPOTENCY_MISMATCH` |
| Dính Lưới `uq_seat_hold_one_active` | Rollback cháy máy | Điều tra ngay, có thằng code quên Xin Khóa! |
| App sụp nguồn trước khi Commit | Postgres Rollback | App sống lại húp command id chơi tiếp |
| App sụp nguồn sau khi Commit | DB Đã Ghi Chặt | Húp command id tự chấn chỉnh lại |

## 12. Chọn Khí Giới Nào? (Alternatives & So sánh định tính)

| Vũ Khí | Kẻ thua cuộc | Điểm Trừ | Mức độ rủi ro Deadlock | 
| --- | --- | --- | --- | 
| Khóa Bi Quan `FOR UPDATE` | Đứng xếp hàng đợi | Dồn cục nếu Transaction dài; kẹt xe ghế Hot | Có, nếu khóa nhiều dòng lộn xộn |
| `FOR UPDATE NOWAIT` | Bị sút văng liền | App chửi Lỗi BUSY liên tục (Poor UX) | Thấp hơn 1 chút |
| Update chay (`where state=AVAILABLE`) | Sửa trả về 0 dòng | Khó code khi quy trình Hủy vé, Ghi Audit lằng nhằng | Thấp cho 1 dòng |
| Khóa Lạc Quan (`@Version`) | Sút văng lúc Flush | Hao tài nguyên Não nghĩ, vứt kết quả uổng phí | Thấp |
| Index Unique | Dính lưới Insert | Rất ngon làm bùa hộ mệnh, không thay thế logic | Thấp |
| `SERIALIZABLE` / Giữ bằng Dòng Giả | Chết ngắc | Xài khi ghế chưa được tạo ra | Tránh |
| Khóa ngoài JVM/ZooKeeper | Khóc nhè nội bộ | Không chơi được kiểu nhiều máy chủ; Dual-write đau đầu | Chỉ local |

## 13. Khuyên Thật Lòng

Hãy xài `PESSIMISTIC_WRITE` khi bạn biết RÕ cái Dòng Dữ Liệu đó Tồn Tại, và logic "Suy nghĩ phán quyết" của bạn dài dằng dặc cần bảo kê trước khi sửa đổi.
Phải chốt kèo thật nhanh (Short Transaction), gài mìn hẹn giờ (`lock_timeout`), Đọc đồ mới sau khi giữ Khóa, Đóng gói lỗi kỹ thuật đẩy ra ngoài, và Sắp Xếp thứ tự khi khóa nhiều dòng.

Nếu cái ghế đó Hot đến mức xếp hàng dồn cục dài lê thê làm banh Server (như săn vé BlackPink), thì chớ dại mà đi tăng Timeout! Hãy dùng trò "Cổng Kiểm Soát" (Admission control) hay các chiêu thức Độc Lập để giải quyết, đó không phải việc của cái Row Lock này!

## 14. Bảng Kiểm Tra Sức Khỏe Cuối Cùng (Production checklist)

- [ ] Lệnh Khóa có đúng là trỏ thẳng vào cái Dòng Tồn Tại, chỉ mặt điểm tên chưa?
- [ ] Lệnh Khóa có phải là lệnh Đọc ĐẦU TIÊN để lấy dữ liệu làm Não Suy Nghĩ không?
- [ ] Hàm được gọi thông qua Spring Proxy và chạy trong đúng 1 Giao Dịch ngắn hủn chưa?
- [ ] Đã bật Log SQL kiểm chứng xem có chữ `FOR UPDATE` xịn xò sinh ra trên PostgreSQL chưa?
- [ ] `lock_timeout` (DB) có được cấu hình KHỎE (nhỏ) hơn Deadline chung của App không?
- [ ] Lỗi Timeout / Deadlock bị quăng ra có được để cho DB Rollback bung bét rồi App mới lượm lại bắt lỗi không?
- [ ] Chắc chắn KHÔNG GỌI MẠNG hay làm tác vụ ì ạch trong lúc ôm Khóa chứ?
- [ ] Mua nhiều vé cùng lúc đã được hàm `.sorted()` sắp xếp y chang nhau cho mọi chức năng chưa?
- [ ] Có xài cột Unique Fingerprint để chống vụ Khách Bấm Đúp liên tọi chưa?
- [ ] Đã dựng đồ thị bóc tách đếm: Lỗi văng Timeout, Lỗi văng Deadlock, Số giây đợi Khóa, Số giây xử lý Khóa rõ ràng chưa?
