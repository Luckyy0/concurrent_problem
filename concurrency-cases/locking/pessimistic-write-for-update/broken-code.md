# Phản Mẫu Thiết Kế (Anti-Patterns): Rủi Ro Từ Quá Trình "Đọc - Quyết Định - Ghi" Không Đồng Bộ

## 1. Cấu Trúc Lược Đồ (Schema Minimal)

Xây dựng hệ thống quản lý đặt chỗ rạp phim với cấu trúc CSDL cơ bản:

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

Phân tích: Bảng `seat_hold` thiết lập Unique Constraint cho cột `command_id` để loại bỏ các yêu cầu lặp (Idempotency). Tuy nhiên, không có ràng buộc tương hỗ nào ngăn cản việc ghi nhận hai giao dịch cấp phát độc lập (`hold_id` khác biệt) hướng tới cùng một bản ghi tại `show_seat`. Hệ thống yêu cầu mã nguồn tầng Ứng dụng phải đảm nhận việc duy trì tính toàn vẹn này.

## 2. Mô Hình Thực Thể Đơn Thuần (Unprotected Entity)

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

    // Phương thức kiểm định điều kiện cấp phát
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

Mô hình thực thể trên thiếu vắng sự hiện diện của cơ chế Khóa Lạc Quan (`@Version`) và không khai báo phương thức áp dụng Khóa Bi Quan. Nó hoàn toàn hoạt động như một lớp Java thông thường mà không có bất kỳ hàng rào bảo vệ tương tranh nào.

## 3. Kiến Trúc Dịch Vụ Mắc Lỗi Nghiêm Trọng (Broken Service)

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
        // Bước 1: Truy xuất bản ghi thông qua lệnh SELECT tiêu chuẩn
        ShowSeat seat = seats.findById(command.seatId())
                .orElseThrow(SeatNotFoundException::new);

        Instant now = clock.instant();
        // Bước 2: Thẩm định nghiệp vụ (Kiểm tra trạng thái)
        if (!seat.isAvailable(now)) {
            return HoldResult.alreadyHeld(seat.getHoldId());
        }

        // Bước 3: Thay đổi trạng thái đối tượng
        UUID holdId = UUID.randomUUID();
        seat.hold(holdId, command.customerId(), now.plusSeconds(120));
        holds.save(SeatHold.active(holdId, command, now));

        return HoldResult.held(holdId);
    }
}
```

Kiến trúc này thoạt nhìn đảm bảo nguyên tắc xử lý tuần tự (Atomicity) đối với các lệnh cập nhật, nhưng lại mắc lỗi hệ thống nghiêm trọng tại Bước 1 khi tiến hành truy xuất bản ghi mà không cấp phát Khóa (`SELECT` thuần).

## 4. Hội Tụ Các Yếu Tố Kích Hoạt Lỗi

- Mức cách ly của CSDL mặc định là `READ COMMITTED`.
- Tài nguyên ghế `(42, A-10)` duy trì trạng thái `AVAILABLE`.
- Hai yêu cầu đặt chỗ từ phía gọi A và phía gọi B được phân luồng xử lý song song.
- Cả hai thao tác đọc dữ liệu (`SELECT`) diễn ra hoàn tất trước khi quá trình xả lệnh Ghi (Flush) tác động tới CSDL.
- Hoàn toàn vắng bóng các cơ chế khóa hỗ trợ.

Tại thời điểm lệnh `UPDATE` được gửi đi, cơ chế Khóa Dòng nội tại của PostgreSQL sẽ thiết lập việc xử lý tuần tự. Tuy nhiên, luồng Logic tại Tầng Ứng Dụng đã hoàn thành khâu đánh giá và xác nhận thành công cho CẢ HAI giao dịch dựa trên cùng một trạng thái trong quá khứ.

## 5. Hiện Tượng Phân Bổ Trùng Lặp (Double Booking)

| Quá Trình Thực Thi | Luồng Giao Dịch A | Luồng Giao Dịch B |
| --- | --- | --- |
| 1 | Khởi tạo Giao dịch | |
| 2 | Truy xuất trạng thái `AVAILABLE` | |
| 3 | | Khởi tạo Giao dịch |
| 4 | | Nhận trạng thái `AVAILABLE` (Từ Snapshot) |
| 5 | Ghi nhận yêu cầu cấp phát (Memory) | Ghi nhận yêu cầu cấp phát (Memory) |
| 6 | Gửi lệnh lưu trạng thái (A) | |
| 7 | Cấp nhận Commit | |
| 8 | | Gửi lệnh lưu trạng thái (B) - GHI ĐÈ KẾT QUẢ CỦA A |
| 9 | | Cấp nhận Commit |

Hậu quả hệ thống: Bảng lịch sử `seat_hold` chứa hai bản ghi cấp phát độc lập (Active). Bảng tham chiếu `show_seat` bị ghi đè bởi Giao dịch xử lý cuối cùng (Phía gọi B). Giao dịch của phía gọi A bị loại bỏ trên hệ thống lưu trữ mặc dù đã nhận phản hồi Thành công từ API. PostgreSQL Row Lock chỉ đóng vai trò phân xử thứ tự Ghi Đè, không cung cấp tính năng tái xác thực điều kiện nghiệp vụ (`isAvailable()`).

> **Nguyên tắc kỹ thuật:** Cấp phát Khóa tại thời điểm cập nhật (Ghi) là hành động chậm trễ vô giá trị, bởi phán quyết nghiệp vụ đã được ấn định dựa trên tập dữ liệu lịch sử (Stale Snapshot) từ pha Đọc ban đầu.

## 6. Phản Mẫu 1 — Nhận Thức Sai Lệch Về Ranh Giới `@Transactional`

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    ShowSeat seat = seats.findById(command.seatId()).orElseThrow();
    // Bỏ quên mệnh đề FOR UPDATE tại khâu nạp dữ liệu.
    return decideAndWrite(seat, command);
}
```

Annotation `@Transactional` quản trị ranh giới Commit/Rollback, đảm bảo các tác vụ Ghi gắn kết với nhau (All-or-nothing). Nó KHÔNG tự động chuyển đổi lệnh truy xuất `SELECT` thành lệnh Yêu Cầu Khóa, do đó không cản trở các giao dịch khác tiếp cận trạng thái dữ liệu cũ.

## 7. Phản Mẫu 2 — Tách Rời Pha Xin Khóa Khỏi Chu Kỳ Xử Lý (Premature Lock Release)

```java
public HoldResult hold(HoldSeatCommand command) {
    // Yêu cầu FOR UPDATE tại tầng Repository
    ShowSeat seat = seats.findForUpdate(command.seatId()).orElseThrow();
    // Kết thúc phạm vi Giao dịch của Repository, Khóa lập tức bị Hủy!
    return mutateInAnotherTransaction(seat, command); // Xử lý trên Giao dịch độc lập
}
```

Khóa bảo vệ gắn liền với vòng đời Giao dịch vật lý. Khi quá trình gọi Repository trả về kết quả, Giao dịch ngắn hạn bị đóng, kéo theo sự giải phóng Khóa. Việc thẩm định và cập nhật tại một Giao dịch kế tiếp sẽ phải đối mặt với trạng thái không an toàn.

## 8. Phản Mẫu 3 — Quy Trình Tái Thẩm Định Trên Dữ Liệu Cũ

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    // 1. Tải bản sao Snapshot không bảo vệ
    ShowSeat stale = seats.findById(command.seatId()).orElseThrow();

    // 2. Yêu cầu Khóa (Bỏ qua phiên bản Mới nhất trả về)
    seats.findForUpdate(command.seatId());

    // 3. Tiến hành đánh giá điều kiện dựa trên phiên bản Snapshot ban đầu
    if (stale.isAvailable(clock.instant())) {
        return decideAndWrite(stale, command);
    }
    return HoldResult.alreadyHeld(stale.getHoldId());
}
```

Yêu cầu Cấp Khóa (`FOR UPDATE`) đồng nghĩa với việc tái nhận bản cập nhật mới nhất. Tiến trình cần tiến hành Revalidation (tái thẩm định) trên chính Đối Tượng Nhận Về Từ Lệnh Khóa, không tiếp tục khai thác thông số trên phiên bản chưa được cập nhật (`stale`).

## 9. Phản Mẫu 4 — Bắt Bỏ Qua Ngoại Lệ Của Giao Dịch Hỏng (Illegal State Recovery)

```java
@Transactional
public HoldResult hold(HoldSeatCommand command) {
    try {
        return decideAndWrite(seats.findForUpdate(command.seatId()).orElseThrow(),
                command);
    } catch (PessimisticLockException ex) {
        // Giao dịch đã bị CSDL đánh dấu Hủy (Rollback-only), vẫn cố gắng truy vấn tiếp
        return holds.findByCommandId(command.commandId())
                .map(HoldResult::replayed)
                .orElseGet(HoldResult::busy);
    }
}
```

Khi phát sinh lỗi Timeout hoặc xung đột Khóa từ CSDL, Giao dịch đang chạy bị chuyển trạng thái sang `aborted`. Mọi tương tác truy vấn phát sinh thêm đều sẽ ném ngoại lệ (lỗi dây chuyền). Lập trình viên phải thoát khối lệnh và cho phép Spring hoàn tất quá trình Rollback trước khi xử lý tiếp.

## 10. Phản Mẫu 5 — Nguy Cơ Bóp Nghẹt Hệ Thống Do Tích Hợp Cuộc Gọi Ngoại Vi (Lock Convoy via Remote Call)

```java
@Transactional
public HoldResult holdAndCharge(HoldSeatCommand command) {
    ShowSeat seat = seats.findForUpdate(command.seatId()).orElseThrow();
    // Giao dịch đang nắm Khóa độc quyền, nhưng bị đình trệ bởi tiến trình External I/O
    PaymentReply reply = paymentClient.authorize(command.payment());
    return updateSeatAfterReply(seat, command, reply);
}
```

Cấp Khóa nhưng tạm ngừng thực thi (Blocking I/O) sẽ dồn ứ các giao dịch cạnh tranh tại cửa ngõ CSDL. Sự gia tăng độ trễ mạng dẫn đến việc bòn rút tài nguyên Connection Pool, gây ra tình trạng khóa liên hoàn (Lock convoy) và sụp đổ dịch vụ.

## 11. Phản Mẫu 6 — Thiếu Đồng Bộ Thứ Tự Khóa (Deadlock Vulnerability)

```java
@Transactional
public void holdPair(List<ShowSeatId> requestedOrder) {
    for (ShowSeatId id : requestedOrder) {
        // Khóa từng tài nguyên tùy ý theo thứ tự đầu vào
        ShowSeat seat = seats.findForUpdate(id).orElseThrow();
        validate(seat);
    }
}
```

Luồng A truy vấn `[A-10, A-11]`, Luồng B truy vấn `[A-11, A-10]` → Phát sinh bẫy kẹt cứng (Deadlock). Các bộ tham số đầu vào đa phần tử bắt buộc phải trải qua tiến trình Sắp Xếp (Sort) thành một trình tự chuẩn nhất (VD: Sắp xếp theo ID định danh) trước khi đi vào vòng lặp Yêu Cầu Khóa.

## 12. Phản Mẫu 7 — Áp Dụng Khóa Mức Ngôn Ngữ Tại Hệ Thống Phân Tán

```java
public synchronized HoldResult hold(HoldSeatCommand command) {
    return transactionalWorker.hold(command);
}
```

Khung tham chiếu rào cản `synchronized` giới hạn kiểm soát luồng hoạt động trong nội bộ một JVM (Java Virtual Machine) duy nhất. Mô hình đa máy chủ phân tán (Clustered servers) khiến cơ chế này mất khả năng ngăn chặn hiện tượng đụng độ dữ liệu vật lý tại CSDL.

## 13. Phản Mẫu 8 — Tùy Biến Lệnh Bỏ Qua Đối Với Định Danh Cụ Thể (`SKIP LOCKED`)

Sử dụng cờ cấu hình `SKIP LOCKED` cho một tài nguyên (ID) cụ thể. Nếu tài nguyên này đang bị khóa bởi Giao dịch phụ trợ, CSDL sẽ trực tiếp bỏ qua và trả về kết quả rỗng (Empty result). API sẽ xử lý chuỗi logic và trả thông báo sai lệnh: "Bản ghi không tồn tại" (Not Found). Kỹ thuật này chỉ ứng dụng trong mô hình tiêu thụ đa thành phần (Tiến trình xử lý hàng đợi).

## 14. Phản Mẫu 9 — Yêu Cầu Thiết Lập Khóa Cho Tập Thực Thể Rỗng (Phantom Locks)

```sql
select *
from show_seat
where show_id = 42 and seat_no = 'A-99'
for update;
```

Cú pháp không phát sinh ngoại lệ, nhưng kết quả trả về bằng 0 dòng dẫn đến CSDL không thiết lập bất kỳ Khóa Dòng nào. Phương thức `FOR UPDATE` không thể phòng chống hiện tượng một luồng khác thực thi lệnh `INSERT` để tạo mới bản ghi `A-99` (Xung đột Phantom Row). Yêu cầu này cần được xử lý thông qua Ràng Buộc Cơ Sở (Table Constraints).

## 15. Dấu Hiệu Nhận Diện Sự Cố (Observability Signals)

- Bất thường cấu trúc dữ liệu (Data Integrity): Bản ghi lịch sử và Bản ghi đại diện trạng thái không khớp quan hệ.
- Luồng truy xuất: Phát hiện khối lệnh `SELECT` không mang theo cờ Khóa phân lập độc lập trước khối lệnh `UPDATE` trong Giao dịch.
- Đặc tả hiệu năng: Chỉ số Wait-event tập trung vào chu kỳ Xả đồng bộ (Flush/Update) thay vì pha tải dữ liệu, chứng tỏ Khóa Dòng diễn ra ngầm định quá muộn.
- Bùng nổ ngoại lệ nội bộ (Lỗi máy chủ nội bộ 500) đè lên các phản hồi mã lỗi Kinh doanh, báo hiệu lỗi Rollback không được bắt lỗi (Catch) chuẩn xác.
- Vòng đời dịch vụ suy giảm trầm trọng (Latency degradation) khi cấp quyền hệ thống chạy dưới mô hình mở rộng số lượng App Nodes.

## 16. Phân Định Lỗi Cấu Trúc (Core Resolution)

Bản chất của các sự cố kể trên không phải do sự thiếu vắng của ranh giới Giao dịch, mà do sự phân rã **Tính Liền Mạch (Atomicity)** của chuỗi nghiệp vụ:

```text
Luồng Hỏng: Đọc Trạng Thái → Đánh Giá Nghiệp Vụ → Kiến Tạo Lịch Sử → Cập Nhật Tài Tài Nguyên
```

Quy trình đúng đắn yêu cầu: Thiết lập Quyền Sở Hữu Độc Quyền (Authoritative Lock) tại CSDL **TRƯỚC KHI** khởi tạo tiến trình Đánh Giá Nghiệp Vụ, và duy trì tính toàn vẹn này liên tục cho tới quá trình Commit cuối cùng.
