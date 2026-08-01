# Mã nguồn xác nhận và hết hạn cùng thắng

## 1. Thực thể không có cơ chế phát hiện xung đột

```java
@Entity
@Table(name = "inventory_reservation")
public class InventoryReservation {

    @Id
    @Column(name = "reservation_id", nullable = false)
    private UUID id;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private ReservationStatus status;

    @Column(name = "expires_at", nullable = false)
    private Instant expiresAt;

    @Column(name = "confirmed_order_id")
    private UUID confirmedOrderId;

    @Column(name = "confirmed_at")
    private Instant confirmedAt;

    @Column(name = "expired_at")
    private Instant expiredAt;

    public boolean canConfirm(Instant now) {
        return status == ReservationStatus.RESERVED
            && now.isBefore(expiresAt);
    }

    public boolean shouldExpire(Instant now) {
        return status == ReservationStatus.RESERVED
            && !now.isBefore(expiresAt);
    }

    public void confirm(UUID orderId, Instant now) {
        status = ReservationStatus.CONFIRMED;
        confirmedOrderId = orderId;
        confirmedAt = now;
    }

    public void expire(Instant now) {
        status = ReservationStatus.EXPIRED;
        expiredAt = now;
    }
}
```

Các phương thức miền nhìn hợp lý khi chạy tuần tự. Tuy nhiên, thực thể không có
`@Version`; phép kiểm tra và phép ghi cũng không nằm trong cùng câu lệnh SQL.
Hai giao dịch có hai bản sao thực thể nên đều có thể vượt qua điều kiện.

## 2. Kho dữ liệu tìm ứng viên hết hạn

```java
public interface InventoryReservationRepository
    extends JpaRepository<InventoryReservation, UUID> {

    List<InventoryReservation> findTop100ByStatusAndExpiresAtLessThanEqualOrderByExpiresAt(
        ReservationStatus status,
        Instant now
    );
}
```

Truy vấn này không khóa dòng và danh sách trả về chỉ là ảnh chụp. Một khoản giữ
có thể được xác nhận sau lúc truy vấn tìm thấy nhưng trước lúc tác vụ xử lý.

Thời gian còn đến từ đồng hồ của máy chạy tác vụ. Nếu đồng hồ máy checkout và
tác vụ lệch nhau, hai bên không thống nhất một khoản giữ đã tới hạn hay chưa.

## 3. Dịch vụ xác nhận bị lỗi

```java
@Service
public class BrokenReservationConfirmationService {

    private final InventoryReservationRepository reservations;
    private final PurchaseOrderRepository orders;
    private final OutboxEventRepository outbox;
    private final Clock clock;

    @Transactional
    public UUID confirm(UUID reservationId, UUID customerId) {
        InventoryReservation reservation = reservations.findById(reservationId)
            .orElseThrow(ReservationNotFoundException::new);

        Instant now = clock.instant();
        if (!reservation.canConfirm(now)) {
            throw new ReservationExpiredException();
        }

        UUID orderId = UUID.randomUUID();
        reservation.confirm(orderId, now);
        orders.save(PurchaseOrder.pendingPayment(
            orderId,
            customerId,
            reservationId
        ));
        outbox.save(OutboxEvent.startPayment(orderId));

        return orderId;
    }
}
```

`@Transactional` bảo đảm ba lần ghi cùng chốt hoặc cùng hoàn tác trong giao dịch
này. Nó không ngăn tác vụ khác đọc cùng trạng thái `RESERVED` trước khi Hibernate
`flush` thay đổi xuống PostgreSQL.

Dịch vụ còn không nhận khóa lũy đẳng. Nếu giao dịch chốt nhưng phản hồi bị mất,
lần gọi lại có thể dùng mã đơn mới và không có hợp đồng phát lại rõ ràng.

## 4. Tác vụ hết hạn bị lỗi

```java
@Component
public class BrokenReservationExpiryJob {

    private final InventoryReservationRepository reservations;
    private final BrokenExpiryTransaction transaction;
    private final Clock clock;

    @Scheduled(fixedDelayString = "${reservation.expiry-delay}")
    public void runBatch() {
        Instant now = clock.instant();
        reservations
            .findTop100ByStatusAndExpiresAtLessThanEqualOrderByExpiresAt(
                ReservationStatus.RESERVED,
                now
            )
            .forEach(reservation -> transaction.expire(
                reservation.getId(),
                now
            ));
    }
}
```

```java
@Service
public class BrokenExpiryTransaction {

    private final InventoryReservationRepository reservations;
    private final InventoryItemRepository inventory;

    @Transactional
    public void expire(UUID reservationId, Instant candidateTime) {
        InventoryReservation reservation = reservations.findById(reservationId)
            .orElseThrow(ReservationNotFoundException::new);

        if (!reservation.shouldExpire(candidateTime)) {
            return;
        }

        reservation.expire(candidateTime);
        reservation.getLines().forEach(line -> {
            InventoryItem item = inventory.findById(line.getProductId())
                .orElseThrow(InventoryItemNotFoundException::new);
            item.release(line.getQuantity());
        });
    }
}
```

Tách mỗi ứng viên thành một giao dịch là hợp lý để tránh một lô dài. Lỗi nằm ở
việc giao dịch mới lại đọc rồi kiểm tra trạng thái trong Java, không khóa và
không có điều kiện trạng thái tại câu `UPDATE`.

## 5. Dòng thời gian gây đơn không còn hàng

Ban đầu:

```text
reservation.status = RESERVED
expires_at          = 10:00:00
inventory           = available 0, reserved 1, on_hand 1
purchase_order      = chưa có
```

Hai giao dịch dùng đồng hồ riêng và cùng thấy mình đủ điều kiện:

```text
giao dịch xác nhận                    giao dịch hết hạn
----------------------------------    ----------------------------------
BEGIN                                 BEGIN
đọc reservation = RESERVED            đọc reservation = RESERVED
đồng hồ thấy chưa quá hạn              đồng hồ thấy đã tới hạn
canConfirm = true                      shouldExpire = true
tạo order + outbox                     release: available +1, reserved -1
đặt status = CONFIRMED                 đặt status = EXPIRED
flush / COMMIT                         flush / COMMIT
```

Hai câu `UPDATE inventory_reservation` chỉ tìm theo khóa chính. Bên ghi sau có
thể ghi đè trạng thái của bên trước. Dù trạng thái cuối là gì, tác dụng phụ ở hai
nhánh đã cùng xuất hiện: có đơn/lệnh thanh toán và tồn kho đã được hoàn lại.

## 6. Vì sao khóa cục bộ không sửa được lỗi

```java
private final ConcurrentMap<UUID, ReentrantLock> locks =
    new ConcurrentHashMap<>();
```

Một khóa theo `reservationId` chỉ phối hợp các luồng trong cùng JVM. Tác vụ và
checkout có thể ở hai máy chủ, hai tiến trình hoặc hai phiên bản ứng dụng khác
nhau. Sau khi khởi động lại, bản đồ khóa cũng mất.

Khóa cục bộ còn dễ bị triển khai lệch: một đường API dùng khóa nhưng tác vụ nền
không dùng, hoặc ngược lại. Nguồn dữ liệu có thẩm quyền vẫn phải tự từ chối lần
chuyển trạng thái thứ hai.

## 7. Chỉ kiểm tra trạng thái ở câu tìm kiếm vẫn sai

```java
List<InventoryReservation> due = repository.findDueReserved(now);
```

Điều kiện `status = 'RESERVED'` ở câu `SELECT` không được giữ đến lúc `UPDATE`.
Khoảng thời gian xử lý danh sách, tải dòng tồn kho và chờ vùng kết nối đủ để
checkout đổi trạng thái.

Sửa câu tìm ứng viên thành `FOR UPDATE SKIP LOCKED` chỉ có tác dụng nếu khóa và
toàn bộ thao tác hết hạn nằm trong cùng giao dịch. Nếu lấy ID, chốt giao dịch rồi
mới hoàn kho ở giao dịch khác, khóa đã được nhả và cuộc đua mở lại.

## 8. Chỉ dùng `@Version` nhưng vẫn tin đồng hồ máy chủ

```java
@Version
private long version;
```

`@Version` có thể làm một giao dịch thất bại thay vì ghi đè trạng thái. Tuy
nhiên, nếu cả hai máy chủ dùng thời gian khác nhau, nghiệp vụ vẫn chưa có một
ranh giới hết hạn thống nhất. Bên thua còn phải hoàn tác, mở giao dịch mới và
tải lại; không được bắt ngoại lệ rồi tiếp tục trong vùng quản lý thực thể cũ.

Với tác vụ theo lô, xung đột lạc quan dày còn tạo nhiều lần tải và hoàn tác hơn
cần thiết. Điều kiện trạng thái cùng thời hạn ngay trong SQL thường rõ ràng hơn.

## 9. Hoàn kho trước rồi mới đổi trạng thái

Một biến thể nguy hiểm khác:

```java
inventory.release(productId, quantity);
reservation.markExpired();
```

Nếu hai thao tác nằm ở hai giao dịch hoặc câu đổi trạng thái thất bại nhưng lỗi
bị nuốt, tồn kho đã tăng mà khoản giữ vẫn `RESERVED` hoặc `CONFIRMED`. Thứ tự câu
lệnh trong Java không đủ; chúng phải cùng giao dịch và chỉ chạy sau khi nhánh hết
hạn giành được trạng thái.

## 10. Gọi cổng thanh toán trong lúc giữ khóa

```java
reservation.confirm(orderId, now);
paymentClient.authorize(orderId); // lời gọi mạng trong giao dịch
```

Lời gọi mạng kéo dài thời gian giữ khóa và tạo kết quả không thể hoàn tác nguyên
tử với PostgreSQL. Nếu thanh toán thành công rồi giao dịch hoàn tác, khách bị thu
tiền mà không có đơn. Nếu giao dịch chốt rồi tiến trình dừng trước lời gọi, đơn
không được thanh toán.

Hãy ghi lệnh thanh toán vào outbox trong giao dịch, rồi phát ra sau khi chốt.

## 11. Dấu hiệu thường gặp

- Đơn tham chiếu một khoản giữ có trạng thái `EXPIRED`.
- `available_quantity` tăng cùng lúc một đơn mới được tạo.
- Cùng `reservation_id` xuất hiện trong nhật ký “confirmed” và “expired”.
- Nhiều tác vụ xử lý cùng một ứng viên do không có `SKIP LOCKED`.
- Khoản giữ vẫn `RESERVED` sau hạn nhưng checkout vẫn xác nhận được.
- Lỗi chỉ xuất hiện khi tác vụ và API nằm trên hai máy chủ khác nhau.
- Hết thời gian chờ khóa bị trả nhầm thành `RESERVATION_EXPIRED`.
