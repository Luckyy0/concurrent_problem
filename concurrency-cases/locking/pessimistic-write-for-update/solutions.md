# Giải Pháp Thiết Kế: Ranh Giới Giao Dịch, Giới Hạn Chờ Và Tái Thẩm Định

## 1. Tiêu Chuẩn Kiến Trúc Tối Ưu (Design Principles)

Kiến trúc xử lý tương tranh dựa trên cơ chế Khóa Bi Quan cần đảm bảo 7 nguyên lý toàn vẹn:

1. **Khóa Độc Quyền (Lock Acquisition):** Khởi tạo `FOR UPDATE` trước khi chuyển hướng luồng xử lý nghiệp vụ.
2. **Nguyên Tắc Kèm Cặp (Atomicity):** Pha Đọc Khóa và Ghi Dữ Liquy bắt buộc phải hoạt động bên trong 1 Giao dịch duy nhất.
3. **Tái Thẩm Định (Revalidation):** Tiến trình tiếp nhận Khóa (Waiter) phải truy xuất dữ liệu từ Snapshot mới nhất sau khi tiếp quản.
4. **Giới Hạn Chờ Kẹt (Bounded Wait):** Bắt buộc cấu hình ngưỡng Timeout CSDL (`lock_timeout`) để triệt tiêu hiện tượng treo hệ thống.
5. **Cấm Kết Nối Ngoại Vi (No Remote I/O):** Loại bỏ mọi thao tác gọi API ngoài biên phạm vi Giao dịch đang giữ Khóa.
6. **Thứ Tự Sắp Xếp Đơn Điệu (Stable Lock Ordering):** Sắp xếp chuỗi tham số (Ví dụ: Định danh tăng dần) để loại trừ triệt để Khóa Chéo (Deadlock).
7. **Phân Tách Xử Lý Trùng (Idempotency):** Áp dụng giải pháp Cấu trúc Ràng buộc riêng biệt cho các yêu cầu lặp (Duplicate Request), tách bạch với xung đột Cạnh tranh.

## 2. Thiết Lập Ràng Buộc Cơ Sở (Schema Defense in Depth)

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

-- Ràng buộc cốt lõi ngăn ngừa trạng thái Cấp Phát Trùng
create unique index uq_seat_hold_one_active
    on seat_hold (show_id, seat_no)
    where status = 'ACTIVE';
```

Chỉ mục `Unique Index` đóng vai trò phòng ngự tuyến cuối (Technical conflict). Nếu một lỗi phần mềm bỏ qua bước Yêu Cầu Khóa, CSDL vật lý sẽ tự động đình chỉ tiến trình. Việc này hạn chế khả năng kiểm soát mã lỗi định hướng người dùng (`ALREADY_HELD`), do vậy cơ chế Yêu Cầu Khóa (Lock) vẫn là lớp bảo vệ thiết yếu số 1.

Yêu cầu giữ chỗ quá hạn (Expired hold) cần được cập nhật trạng thái `EXPIRED` trước khi hệ thống chấp thuận chèn dữ liệu giữ chỗ mới cho cùng tài nguyên.

## 3. Kiến Trúc Thực Thể Hợp Nhất (Aggregate State)

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
            throw new IllegalStateException("Cấu trúc sai lệch: Bản ghi chưa được cấp phát");
        }
        state = SeatState.AVAILABLE;
        holdId = null;
        holderCustomerId = null;
        holdUntil = null;
    }

    void hold(UUID newHoldId, long customerId, Instant until) {
        if (state != SeatState.AVAILABLE) {
            throw new IllegalStateException("Cấu trúc sai lệch: Bản ghi không khả dụng");
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

Ghi chú: Việc áp dụng đồng thời Khóa Lạc Quan (`@Version`) trong hệ thống này là dư thừa. Cơ chế Khóa Bi Quan đã cung cấp lớp cô lập đủ mức an toàn, việc bổ sung thuộc tính Version chỉ gia tăng chi phí vận hành (Overhead) mà không mang lại giá trị tương xứng.

## 4. Tích Hợp Repository Mức JPA (Locking Repository)

```java
public interface ShowSeatRepository
        extends JpaRepository<ShowSeat, ShowSeatId> {

    // Kỹ thuật 1: Áp dụng Khóa đối với dữ liệu Đơn Bản Ghi
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from ShowSeat s where s.id = :id")
    Optional<ShowSeat> findForUpdate(@Param("id") ShowSeatId id);

    // Kỹ thuật 2: Xử lý Đa Bản Ghi (SQL Thuần kèm cấu trúc Mệnh Lệnh Sắp Xếp)
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

Sử dụng `PESSIMISTIC_WRITE` cho bản ghi đơn. Đối với tập bản ghi, sử dụng `ORDER BY ... FOR UPDATE` trong cấu trúc Native Query để triệt tiêu nguy cơ Deadlock.
Truy vấn này bắt buộc phải thi hành trong một khung Giao dịch (Transaction) đang hoạt động. Việc thiếu vắng cấu trúc Transaction sẽ vô hiệu hóa hoàn toàn cơ chế Khóa.

## 5. Cấu Hình Giới Hạn Thời Gian Chờ (Lock Timeout)

Nhằm đảm bảo cơ chế giới hạn thời gian (Timeout) được vận hành ổn định trên nền tảng PostgreSQL (không phụ thuộc vào Provider), hệ thống sử dụng cấu hình cục bộ thông qua Native Query:

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
            throw new IllegalArgumentException("Khung thời gian Timeout ngoài giới hạn an toàn");
        }

        // Thiết lập Timeout giới hạn trong phiên Giao dịch hiện hành
        entityManager.createNativeQuery("""
                select set_config('lock_timeout', :value, true)
                """)
                .setParameter("value", millis + "ms")
                .getSingleResult();
    }
}
```

Thuộc tính `true` (IS_LOCAL) quy định mức độ áp dụng của thiết lập này chỉ dành riêng cho ngữ cảnh Giao dịch đang được kết nối. Tham số Timeout cần được định nghĩa hệ thống (Hardcoded/Config), không mở cho phía gọi can thiệp.

## 6. Thiết Kế Proxy Điều Phối Giao Dịch (Transactional Worker)

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
        // 1. Áp dụng cơ chế Timeout giới hạn (Ví dụ: 750ms)
        lockTimeout.apply(Duration.ofMillis(750));

        // 2. Yêu cầu Cấp Khóa Độc Quyền (Chuyển sang trạng thái Wait nếu đụng độ)
        ShowSeat seat = seats.findForUpdate(command.seatId())
                .orElseThrow(SeatNotFoundException::new);

        // 3. Tái Thẩm Định: Chống tấn công Lặp Lệnh (Idempotency Check)
        Optional<SeatHold> replay =
                holds.findByCommandId(command.commandId());
        if (replay.isPresent()) {
            SeatHold previous = replay.orElseThrow();
            previous.requireSameRequest(command);
            return HoldResult.replayed(previous.holdId());
        }

        Instant now = clock.instant();
        
        // 4. Giải phóng Trạng Thái Giữ Chỗ Đã Hết Hạn
        if (seat.hasExpiredHold(now)) {
            holds.expire(seat.currentHoldId(), now);
            seat.releaseExpiredHold();
        }

        // 5. Kiểm định Điều Kiện (Báo lỗi nếu đã bị Cấp Phát)
        if (seat.isHeldAndNotExpired(now)) {
            return HoldResult.alreadyHeld(seat.currentHoldId());
        }
        if (seat.isSold()) {
            return HoldResult.sold();
        }

        // 6. Thực thi quy trình Cập Nhật Trạng Thái
        UUID holdId = UUID.randomUUID();
        Instant holdUntil = now.plus(Duration.ofMinutes(2));

        seat.hold(holdId, command.customerId(), holdUntil);
        holds.save(SeatHold.active(holdId, command, holdUntil, now));
        
        // Đồng bộ hệ thống nhằm phát hiện tức thì các Ràng buộc Database
        entityManager.flush(); 

        return HoldResult.held(holdId, holdUntil);
    }
}
```

Kiến trúc liên kết quá trình cấp phát và tái thẩm định (Revalidate) thành một khối chặt chẽ (Persistence context). Yêu cầu gọi `flush()` ngay lập tức để đẩy độ trễ ngoại lệ (Exception Catch) về phạm vi nội hàm.

Tham số `Propagation.REQUIRED` yêu cầu không được mở rộng Giao dịch tại phía gọi bên ngoài (Coordinator), để bảo đảm Vòng đời Giao dịch là nhỏ nhất.

## 7. Xử Lý Phân Loại Ngoại Lệ (Exception Mapping)

Tiến trình điều phối vòng ngoài hứng chịu các sự cố từ khối Giao dịch nội tại, tiến hành ánh xạ lỗi (Map) tới hệ thống HTTP:

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
        // Áp đặt ràng buộc: Không thực thi dưới khung Giao dịch đã tồn tại
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            throw new IllegalStateException(
                    "Cấu trúc sai lệch: Coordinator không được bao bọc bởi Transaction"
            );
        }

        try {
            return worker.execute(command);
        } catch (RuntimeException failure) {
            // Chuyển đổi trạng thái Lỗi Vật Lý sang Lỗi Dịch Vụ
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

Nguyên tắc hệ thống: Chỉ tiến hành ánh xạ (Catch & Map) đối với lỗi Liên quan Tới Khóa. Các lỗi phát sinh không mong muốn khác (Network Drop, System Crash) bắt buộc phải trả về mã lỗi 500 (Server Error). Việc lập lịch Thử lại (Retry) với lỗi `55P03` phải áp dụng các chiến lược Back-off độc lập tương đương bài toán `LOCK-002`.

## 8. Cấu Trúc Truy Vấn Nguyên Thủy Tương Đương

Quy trình hoạt động thông qua lệnh SQL vật lý dưới tầng CSDL:

```sql
begin;
-- Khởi tạo Timeout cục bộ:
select set_config('lock_timeout', '750ms', true);

-- Thiết lập Khóa Độc Quyền (Row Lock):
select *
from show_seat
where show_id = 42 and seat_no = 'A-10'
for update;

-- Nếu vượt quá thời gian Hold, hệ thống Hủy yêu cầu cũ:
update seat_hold
set status = 'EXPIRED'
where hold_id = :oldHoldId and status = 'ACTIVE';

-- Chèn dữ liệu cấp phát mới (Cơ chế Unique Constraint kích hoạt):
insert into seat_hold (
    hold_id, command_id, show_id, seat_no, customer_id,
    status, request_fingerprint, created_at
) values (
    :holdId, :commandId, 42, 'A-10', :customerId,
    'ACTIVE', :fingerprint, now()
);

-- Cập nhật bản ghi đối chiếu (Root Entity):
update show_seat
set state = 'HELD',
    hold_id = :holdId,
    holder_customer_id = :customerId,
    hold_until = :holdUntil
where show_id = 42 and seat_no = 'A-10';

commit; -- Kết thúc chu kỳ, Giải phóng Khóa
```

## 9. Xử Lý Tương Tranh Nhóm (Lô Nhiều Ghế)

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public GroupHoldResult holdTogether(GroupHoldCommand command) {
    // 1. Áp đặt thuật toán Sắp Xếp đơn điệu (Deterministic Sorting)
    List<String> canonicalSeatNos = command.seatNos().stream()
            .distinct()
            .sorted()
            .toList();

    lockTimeout.apply(Duration.ofMillis(750));
    
    // 2. Yêu cầu Khóa toàn cụm theo thứ tự đã ấn định (ORDER BY)
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

    // 3. Tiến hành Lưu Trữ đồng loạt
    return createGroupHold(command, locked);
}
```

Tích hợp cấu trúc `ORDER BY` vào truy vấn để bảo vệ hệ thống khỏi hiện tượng Deadlock do nghịch đảo tuần tự Khóa. Mọi tác vụ nghiệp vụ phát sinh sau (Hủy, Chuyển Nhượng) đều phải tuân thủ chuẩn Sắp xếp Dữ Liệu `canonicalSeatNos`.

## 10. Nguyên Tắc Phân Rã Xử Lý Tiền Tệ (Remote Payment Workflow)

Nghiêm cấm đưa tác vụ Thanh Toán Hóa Đơn (Payment I/O) vào bên trong biên giới Khóa Dòng. Quy trình tiêu chuẩn:
1. Giao dịch Tạm thời: Khóa Ghế → Đánh dấu trạng thái HELD tạm giữ (Ví dụ 10 phút) → Thoát và Nhả Khóa.
2. Tác vụ Độc lập (Asynchronous): Gửi yêu cầu HTTP đến Payment Gateway (Có thể trễ hệ thống 5-10s).
3. Giao dịch Xác nhận: Nhận phản hồi thanh toán → Xin Lại Khóa Dòng → Kiểm định Ghế vẫn thuộc sở hữu HELD của User → Chuyển `SOLD` → Nhả Khóa.
4. Worker Dọn Dẹp: Tác vụ ngầm quét các ghế có trạng thái HELD quá hạn và chuyển về `EXPIRED`.

Không được phép đồng nhất "Thời hạn chờ quyết định của User" (10 phút) với "Vòng Đời Của Row Lock dưới CSDL" (2 phút).

## 11. Bảng Chỉ Số Rủi Ro Môi Trường (Failure Matrix)

| Biến Cố Phát Sinh | Hệ Quả Dưới CSDL | Cách Thức API Phản Hồi |
| --- | --- | --- |
| Tài nguyên đã `HELD` | Không thao tác dữ liệu | Phản hồi từ chối `ALREADY_HELD` |
| Báo lỗi `55P03` (Timeout) | Rollback Giao Dịch | Báo lỗi quá tải `BUSY` |
| Báo lỗi `40P01` (Deadlock) | Rollback Giao Dịch | Báo lỗi quá tải `BUSY`, cấu hình Retry nhẹ |
| Yêu cầu trùng (Command_ID) | Hủy thao tác nội tại | Phản hồi lịch sử cấp phát cũ |
| Yêu cầu đính kèm sai thông tin | Rollback (Lỗi Logic) | Báo mã `IDEMPOTENCY_MISMATCH` |
| Lỗi Cấp Phát Trùng Diện Rộng | Rollback (Constraint) | Báo lỗi Sự Cố Kỹ Thuật (Hệ thống thiết kế lỗi) |
| Ứng dụng Crash trước Commit | Tự Hủy (Auto Rollback) | Cho phép ứng dụng tái xử lý sau khi hồi phục |
| Ứng dụng Crash sau Commit | Dữ liệu bền vững | Tiến hành truy vấn hệ thống dựa trên ID |

## 12. Phân Tích Phương Án Chọn Lựa (Architectural Alternatives)

| Công Cụ | Nhược Điểm Đối Với Tiến Trình Chờ | Nguy Cơ Tiềm Ẩn Cần Thiết Tránh | Rủi Trạng Deadlock | 
| --- | --- | --- | --- | 
| `FOR UPDATE` | Áp lực xếp hàng chờ đồng bộ | Dồn cục hệ thống nếu Transaction xử lý chậm | Yêu cầu sắp xếp đa bản ghi |
| `FOR UPDATE NOWAIT` | Loại bỏ ngay tức khắc | Làm tăng lượng Lỗi Trả về (Trải nghiệm người dùng kém) | Thấp |
| Condition Update | Phức tạp trong Audit/Lịch sử | Không áp dụng được cho cấu trúc đa Entity phức tạp | Rất thấp |
| Khóa Lạc Quan (`@Version`) | Từ chối ở khâu cuối cùng (Flush) | Tốn chi phí xử lý nghiệp vụ dư thừa | Cực thấp |
| Khóa Phân Tán (ZooKeeper/Redis) | Chỉ phù hợp môi trường ngoại vi | Phức tạp (Dual-write), thiếu đồng bộ vật lý | Đặc thù |

## 13. Khuyến Nghị Sử Dụng Kỹ Thuật (Final Verdict)

Ưu tiên áp dụng `PESSIMISTIC_WRITE` (Khóa Dòng Độc Quyền) cho các kịch bản nhắm đích danh một bản ghi vật lý cố định. Phương pháp này yêu cầu tuân thủ cấu trúc Transaction ngắn hạn (Short-lived), triển khai Timeout, xử lý Revalidate tại thời điểm tiếp nhận, và cơ chế sắp xếp (Sort) cho các nhóm tài nguyên.

Không cố gắng mở rộng Timeout để đối phó với hiện tượng Hotspots (Sự kiện quá tải). Khi hệ thống tắc nghẽn ở mức cực đại (Ví dụ: Flash Sale), cần phân rã kiến trúc sang các mô hình Hàng đợi (Queue), thay vì lạm dụng khả năng chịu đựng của Connection Pool.

## 14. Bộ Tiêu Chuẩn Triển Khai (Production Checklist)

- [ ] Lệnh `FOR UPDATE` phải được chỉ định cho các bản ghi xác thực tồn tại (ID).
- [ ] Lệnh Khóa là tác vụ đầu tiên tiếp cận dữ liệu để ngăn chặn ảo giác (Stale read).
- [ ] Khối logic nằm trọn trong một phương thức Spring Proxy kích hoạt `@Transactional`.
- [ ] Xác minh tệp Log SQL có ghi nhận mệnh đề `FOR UPDATE` trong truy vấn.
- [ ] Cấu hình `lock_timeout` cục bộ thấp hơn đáng kể so với Request Timeout tổng thể.
- [ ] Tắt mọi tác vụ giao tiếp I/O qua mạng trong biên độ Giao dịch chứa Khóa.
- [ ] Mã định danh cần được sắp xếp (Sorted) trước khi Yêu cầu Cấp đa Khóa.
- [ ] Triển khai Constraint Unique Fingerprint chống hiện tượng nhấp đúp của phía gọi.
- [ ] Thiết lập hệ thống Monitor cho mã `55P03`, `40P01` và tài nguyên kết nối Idle.
