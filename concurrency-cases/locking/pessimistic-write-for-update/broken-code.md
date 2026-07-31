# Hiện trường vụ án: Tội ác của việc "Cầm đèn chạy trước ô tô"

## 1. Thiết kế Bảng (Schema tối thiểu)

Giả sử chúng ta có 2 bảng để quản lý rạp phim:

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

Nhìn kỹ nhé: Bảng `seat_hold` có cột Unique để chặn khách nhấn đúp `command_id`, nhưng KHÔNG có Ràng Buộc (Constraint) nào cản 2 cái `hold_id` khác nhau cùng trỏ vào 1 cái ghế. Nghĩa là code Java phải tự đứng ra bảo vệ chiếc ghế này.

## 2. Entity Ngây Thơ (Không có chống đạn)

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

    // Hàm kiểm tra ghế còn trống không
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

Không có `@Version` (Khóa Lạc Quan). Cũng chẳng có dòng nào đòi `FOR UPDATE` (Khóa Bi Quan). Chỉ là một Class Java ngoan hiền.

## 3. Đoạn Code Gây Thảm Họa (Broken service)

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
        // Bước 1: Tải ghế lên bằng SELECT bình thường
        ShowSeat seat = seats.findById(command.seatId())
                .orElseThrow(SeatNotFoundException::new);

        Instant now = clock.instant();
        // Bước 2: Hỏi ghế "Còn trống không em?"
        if (!seat.isAvailable(now)) {
            return HoldResult.alreadyHeld(seat.getHoldId());
        }

        // Bước 3: Còn trống thì anh Xí chỗ nhé!
        UUID holdId = UUID.randomUUID();
        seat.hold(holdId, command.customerId(), now.plusSeconds(120));
        holds.save(SeatHold.active(holdId, command, now));

        return HoldResult.held(holdId);
    }
}
```

Đoạn code này đọc cứ như một bài thơ, logic cực kỳ trơn tru... nếu chạy 1 mình 1 ngựa. Khổ nỗi nó dùng `SELECT` chay (không Khóa) ở Bước 1.

## 4. Thiên Thời Địa Lợi Cho Lỗi Nổ Ra

- Dưới gầm đang xài `READ COMMITTED`.
- Ghế `(42, A-10)` đang rảnh `AVAILABLE`.
- Anh A và Chị B cùng nhào vô bấm Đặt Ghế bằng 2 Request riêng biệt.
- Cả hai câu `SELECT` đều chạy xong trước khi có lệnh Xả (`flush`) ghi vào DB.
- Không hề có Khóa gì bảo vệ (Không `@Version`, không Constraint).

Đến lúc xả lệnh UPDATE xuống DB, tụi nó sẽ bị DB bắt xếp hàng tuần tự. Nhưng xui cái là lúc đó não bộ của Java đã chốt: Cả A và B ĐỀU ĐƯỢC CHẤP NHẬN trên App mất rồi!

## 5. Thảm Cảnh 1 Ghế 2 Chủ

| Bước | Máy của A | Máy của B |
| --- | --- | --- |
| 1 | Mở Giao Dịch | |
| 2 | Nhìn thấy `AVAILABLE` | |
| 3 | | Mở Giao Dịch |
| 4 | | Cũng thấy `AVAILABLE` |
| 5 | Lập tờ Đăng Ký cho A | Lập tờ Đăng Ký cho B |
| 6 | Sửa tên chủ ghế thành A | |
| 7 | Chốt sổ (Commit) | |
| 8 | | Sửa tên chủ ghế thành B (ĐÈ LÊN A) rồi Chốt sổ! |

Kết quả: Bảng `seat_hold` đẻ ra 2 tờ Đăng Ký (đều ACTIVE) rành rành. Bảng `show_seat` thì chỉ lưu tên B. A đến rạp và bị đuổi về dù App đã báo thành công. Cảm ơn chức năng Row Lock ngầm của UPDATE, nó chỉ cản đè Data chứ KHÔNG quay lại chạy hàm `isAvailable()` cho B!

> **Bài học xương máu:** Khóa lúc Ghi là QUÁ MUỘN; vì phán quyết (Business decision) đã được đưa ra dựa trên cái "Hình chụp Dĩ vãng" từ 2 câu đọc không Khóa trước đó rồi.

## 6. Sai lầm 1 — Ảo tưởng sức mạnh của `@Transactional`

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    ShowSeat seat = seats.findById(command.seatId()).orElseThrow();
    // Thiếu FOR UPDATE.
    return decideAndWrite(seat, command);
}
```

`@Transactional` chỉ làm mỗi việc: Ép các lệnh Ghi (Write) phải đi chung 1 xuồng, chìm cùng chìm, nổi cùng nổi. Chứ nó KHÔNG HỀ biến lệnh `SELECT` thành lệnh Khóa, và cản không cho kẻ khác đọc dữ liệu cũ.

## 7. Sai lầm 2 — Xin Khóa trong cái Hẻm, rồi mang data ra Đường Lớn xài

```java
public HoldResult hold(HoldSeatCommand command) {
    // Xin FOR UPDATE trong Repository
    ShowSeat seat = seats.findForUpdate(command.seatId()).orElseThrow();
    // Ra đến đây là HẾT Giao Dịch của Repository, Khóa đã BỊ VỨT SỌT RÁC!
    return mutateInAnotherTransaction(seat, command);
}
```

Khóa sống chết theo Giao Dịch DB. Giao Dịch đứt thì Khóa mất hiệu lực. Nếu Giao Dịch chết trước khi bạn đưa ra phán quyết và cập nhật, thì cái Khóa đó hoàn toàn Vô Dụng.

## 8. Sai lầm 3 — Đọc đồ ôi thiu rồi mới chạy đi Xin Khóa

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    // 1. Tải lên 1 cục data ôi thiu
    ShowSeat stale = seats.findById(command.seatId()).orElseThrow();

    // 2. Chạy hàm xin Khóa (nhưng vứt data Mới vào sọt rác)
    seats.findForUpdate(command.seatId());

    // 3. Phán quyết dựa trên... cục data ôi thiu ở bước 1
    if (stale.isAvailable(clock.instant())) {
        return decideAndWrite(stale, command);
    }
    return HoldResult.alreadyHeld(stale.getHoldId());
}
```

Xin khóa xong thì phải lấy cái Object Sạch Sẽ Mới Tinh đó mà dùng! Đừng giả định là gọi hàm Khóa xong thì cái Object cũ mèm `stale` kia tự động được gột rửa (refresh).

## 9. Sai lầm 4 — Bắt Lỗi (Catch) Timeout rồi Cố Đấm Ăn Xôi

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    try {
        return decideAndWrite(seats.findForUpdate(command.seatId()).orElseThrow(),
                command);
    } catch (PessimisticLockException ex) {
        // DB đã cắm cờ Phá Sản mà vẫn ráng chạy tiếp
        return holds.findByCommandId(command.commandId())
                .map(HoldResult::replayed)
                .orElseGet(HoldResult::busy);
    }
}
```

Một khi DB đã chửi vì Timeout hoặc Lỗi Khóa, cái Giao Dịch đó coi như ĐÃ Ô UẾ (aborted state). Gọi lệnh DB gì tiếp trong đó cũng nổ lỗi bung bét. Phải buông xuôi cho nó thoát ra ngoài Proxy (để Rollback) rồi tính sau!

## 10. Sai lầm 5 — Vừa giữ Khóa, vừa gọi điện thoại buôn chuyện (Remote call)

```java
@Transactional
public HoldResult holdAndCharge(HoldSeatCommand command) {
    ShowSeat seat = seats.findForUpdate(command.seatId()).orElseThrow();
    // Đang ôm cái Khóa mà rảnh rỗi gọi API thanh toán (mất xừ nó 5 giây)
    PaymentReply reply = paymentClient.authorize(command.payment());
    return updateSeatAfterReply(seat, command, reply);
}
```

Bao nhiêu thằng khác đang đứng chờ Khóa mòn mỏi. Gọi API lỡ nó lag thì cả ngàn Connection đi tong, tạo thành một "Bãi xe kẹt cứng" (lock convoy).

## 11. Sai lầm 6 — Khóa nhiều ghế lộn xộn không Xếp Hàng

```java
@Transactional
public void holdPair(List<ShowSeatId> requestedOrder) {
    for (ShowSeatId id : requestedOrder) {
        // Khóa A-10 xong qua khóa A-11
        ShowSeat seat = seats.findForUpdate(id).orElseThrow();
        validate(seat);
    }
}
```

Khách mua `[A-10, A-11]`. Khách khác truyền `[A-11, A-10]` -> Kẹt cứng ngắc (Deadlock). Phải luôn SORT (sắp xếp) danh sách ghế theo một trật tự duy nhất trước khi chạy Vòng Lặp xin Khóa.

## 12. Sai lầm 7 — Xài "Khóa Nội Bộ" `synchronized`

```java
public synchronized HoldResult hold(HoldSeatCommand command) {
    return transactionalWorker.hold(command);
}
```

Từ khóa này xịn, nhưng chỉ có tác dụng trong MỘT cái Máy ảo Java (JVM). Khách gọi vào Máy 1, nó chặn được. Khách gọi vào Máy 2, máy 2 chả biết gì. Chạy Đa Máy Chủ là toang!

## 13. Sai lầm 8 — Xài "Bỏ Qua" `SKIP LOCKED` cho 1 cái Ghế Cụ Thể

Giả sử khách đích danh đòi mua ghế `A-10`. Trùng hợp thằng khác đang giữ Khóa để xem. Nếu bạn xài `SKIP LOCKED`, DB lơ luôn cái `A-10` đó, hàm trả về kết quả Rỗng. API của bạn ngây ngô báo lại: "Cái ghế này KHÔNG TỒN TẠI!"
Thật ngớ ngẩn. Trò này chỉ dành cho Bọn Worker tự động hốt hàng ngẫu nhiên thôi.

## 14. Sai lầm 9 — Đòi Khóa một thứ KHÔNG TỒN TẠI

```sql
select *
from show_seat
where show_id = 42 and seat_no = 'A-99'
for update;
```

Ghế `A-99` làm gì có! Query về 0 dòng, DB chả khóa cái gì sất. Nếu mục đích của bạn là "Ngăn không cho ai đẻ ra cái ghế A-99", thì `FOR UPDATE` bó tay nhé. Phải xài các món võ khác như Constraint hoặc Rào chắn (guard row).

## 15. Bắt Bệnh Qua Triệu Chứng (Dấu hiệu)

- Dữ liệu rác: Có 2 vé ACTIVE nhưng ghế chỉ ghi tên 1 người.
- Bật Log SQL lên chỉ thấy câu `SELECT` trần trụi trước câu `UPDATE`.
- Dấu hiệu đứng chờ (Wait) toàn rớt vào lúc `UPDATE/flush` thay vì lúc tải dữ liệu lên.
- Quá trình "gọi mạng ngoài" lỡ lọt vào trong Giao Dịch làm Lỗi Kẹt Xe (`pg_stat_activity.wait_event_type = 'Lock'`) nhảy dựng đứng.
- Log thấy báo lỗi 500 hầm bà lằng thay vì trả lỗi Kinh Doanh sạch sẽ.
- Chạy 1 máy ảo không sao, bung nhiều máy (scale) thì nát.

## 16. Tóm Lại Đoạn Code Bị Bệnh Gì?

Không phải là nó "thiếu Giao dịch", mà là nó **Thiếu Tính Liền Mạch (Atomicity)** của chuỗi hành động:

```text
Đọc Trạng Thái Hiện Tại → Suy Nghĩ Phán Quyết → Làm Tờ Đăng Ký → Cập Nhật Lại Ghế
```

Con đường đúng đắn là: Phải giật cái Khóa Độc Quyền Dưới DB (Authoritative Lock) **TRƯỚC KHI** đem não ra Suy Nghĩ, giữ khư khư nó cho đến khi Xả sạch sẽ xuống DB và Chốt sổ (Commit). Và nhớ cài đồng hồ hẹn giờ cho cái Khóa đó!
