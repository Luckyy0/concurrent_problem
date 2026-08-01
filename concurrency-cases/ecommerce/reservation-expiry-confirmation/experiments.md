# Thực nghiệm hết hạn và xác nhận đồng thời với PostgreSQL

## 1. Mục tiêu

Mỗi test phải kiểm tra trạng thái bền vững sau khi mọi giao dịch kết thúc:

```text
available_quantity >= 0
reserved_quantity >= 0
on_hand_quantity >= 0
available_quantity + reserved_quantity = on_hand_quantity

reserved_quantity
    = tổng quantity của các reservation_line có reservation.status = RESERVED

mỗi reservation chỉ có một trạng thái kết thúc
mỗi reservation có tối đa một purchase_order
EXPIRED không có order hoặc outbox thanh toán mới
CONFIRMED không được hoàn lại available
cùng checkout request không tác động lần hai
```

Không kết luận hệ thống đúng chỉ vì một tác nhân nhận ngoại lệ. Phải đọc khoản
giữ, bộ đếm, đơn, outbox và bảng lũy đẳng sau `COMMIT` hoặc `ROLLBACK`.

## 2. Cấu hình Testcontainers

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("postgres-test")
class ReservationExpiryConfirmationTest {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
        registry.add("spring.flyway.enabled", () -> "true");
    }

    @Autowired ReservationCheckoutTransaction checkout;
    @Autowired ReservationExpiryTransaction expiry;
    @Autowired ReservationFixtures fixtures;
    @Autowired ReservationProbe probe;
}
```

Migration test phải giống môi trường thật, gồm điều kiện trạng thái, chỉ mục một
phần, khóa ngoại, ràng buộc duy nhất của đơn và mọi `CHECK` bộ đếm. H2 không đại
diện cho MVCC, khóa dòng, `SKIP LOCKED`, `clock_timestamp()` hoặc SQLSTATE của
PostgreSQL.

## 3. Bộ chạy hai tác nhân

```java
final class TwoActors implements AutoCloseable {

    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    <T> List<Outcome<T>> runTogether(
        Callable<T> firstAction,
        Callable<T> secondAction
    ) throws Exception {
        CyclicBarrier start = new CyclicBarrier(2);

        Future<Outcome<T>> first = executor.submit(
            () -> capture(start, firstAction)
        );
        Future<Outcome<T>> second = executor.submit(
            () -> capture(start, secondAction)
        );

        return List.of(
            first.get(10, TimeUnit.SECONDS),
            second.get(10, TimeUnit.SECONDS)
        );
    }

    private static <T> Outcome<T> capture(
        CyclicBarrier start,
        Callable<T> action
    ) {
        try {
            start.await(5, TimeUnit.SECONDS);
            return Outcome.success(action.call());
        } catch (Throwable error) {
            return Outcome.failure(error);
        }
    }

    @Override
    public void close() {
        executor.shutdownNow();
    }
}
```

Mọi `await` và `get` có thời gian tối đa. Không dùng `Thread.sleep` làm cách
chính để đoán một giao dịch đã đọc, khóa hoặc ghi dữ liệu.

## 4. Điểm hẹn tái hiện bản lỗi

Thêm một cổng chỉ dành cho test ngay sau khi bản lỗi đã kiểm tra trạng thái và
thời gian, trước khi tạo đơn hoặc hoàn kho:

```java
@FunctionalInterface
interface AfterReservationCheck {
    void reached(UUID reservationId, ReservationStatus observedStatus);
}
```

```java
CyclicBarrier bothChecked = new CyclicBarrier(2);

AfterReservationCheck rendezvous = (id, status) -> {
    assertThat(status).isEqualTo(ReservationStatus.RESERVED);
    await(bothChecked);
};
```

Cổng này không nằm trong mã chạy thật. Nó buộc hai giao dịch cùng vượt qua một
ảnh chụp cũ nên test không phụ thuộc may rủi của bộ lập lịch.

## 5. Tái hiện đơn được tạo sau khi tồn kho đã hoàn

Dữ liệu thử nghiệm của bản lỗi cho phép hai đồng hồ JVM lệch nhau quanh cùng thời hạn:

```java
@Test
void staleChecksAllowConfirmationAndExpiryToBothCommit() throws Exception {
    UUID reservationId = fixtures.reserved(
        expiresAt("2026-08-01T10:00:00Z"),
        line(101L, 1)
    );
    fixtures.inventory(101L, 1, 0, 1);

    brokenCheckout.setClock(fixed("2026-08-01T09:59:59Z"));
    brokenExpiry.setClock(fixed("2026-08-01T10:00:01Z"));
    brokenCheckout.setAfterCheck(rendezvous);
    brokenExpiry.setAfterCheck(rendezvous);

    try (TwoActors actors = new TwoActors()) {
        List<Outcome<Object>> outcomes = actors.runTogether(
            () -> brokenCheckout.confirm(reservationId, customerId()),
            () -> {
                brokenExpiry.expire(reservationId);
                return ExpiryAttempt.EXPIRED;
            }
        );
        assertThat(outcomes).allMatch(Outcome::isSuccess);
    }

    assertThat(probe.orderCount(reservationId)).isEqualTo(1);
    assertThat(probe.paymentOutboxCount(reservationId)).isEqualTo(1);
    assertThat(probe.inventory(101L).availableQuantity()).isEqualTo(1);
    assertThat(probe.inventory(101L).reservedQuantity()).isZero();
}
```

Test cố ý chứng minh hai tác dụng phụ mâu thuẫn cùng tồn tại. Giá trị trạng thái
cuối có thể phụ thuộc bên ghi sau; nó không làm kết quả trên trở thành hợp lệ.

## 6. Khoản giữ đã quá hạn: tác vụ thắng, checkout bị từ chối

```java
@Test
void expiredReservationCannotBeConfirmedWhileWorkerReleasesIt()
    throws Exception {

    UUID reservationId = fixtures.reservedAlreadyExpired(
        line(101L, 1),
        line(202L, 2)
    );
    ConfirmReservationCommand command = fixtures.confirmCommand(reservationId);

    List<Outcome<Object>> outcomes;
    try (TwoActors actors = new TwoActors()) {
        outcomes = actors.runTogether(
            () -> checkout.confirm(command),
            expiry::expireNext
        );
    }

    ReservationConfirmationResult confirmation = outcomes.stream()
        .filter(Outcome::isSuccess)
        .map(Outcome::value)
        .filter(ReservationConfirmationResult.class::isInstance)
        .map(ReservationConfirmationResult.class::cast)
        .findFirst()
        .orElseThrow();

    assertThat(confirmation.outcome())
        .isEqualTo(ReservationConfirmationOutcome.RESERVATION_EXPIRED);
    assertThat(probe.status(reservationId)).isEqualTo(EXPIRED);
    assertThat(probe.orderCount(reservationId)).isZero();
    assertThat(probe.paymentOutboxCount(reservationId)).isZero();
    probe.assertReleasedExactlyOnce(reservationId);
    probe.assertInventoryConserved();
    probe.assertReservedProjectionMatchesReservations();
}
```

Không giả định checkout hay tác vụ được lập lịch trước. Vì thời hạn đã qua theo
PostgreSQL, xác nhận không đủ điều kiện. Tác vụ cuối cùng hoàn kho đúng một lần.

Nếu checkout trả kết quả trước khi tác vụ kịp hoàn kho, test phải chờ cả hai
kết quả bất đồng bộ (Future) hoàn tất rồi mới đọc trạng thái cuối.

## 7. Khoản giữ còn hạn: xác nhận thắng, tác vụ không hoàn kho

```java
@Test
void activeReservationConfirmsWithoutBeingReleased() throws Exception {
    UUID reservationId = fixtures.reservedFarInFuture(
        line(101L, 1),
        line(202L, 2)
    );
    ConfirmReservationCommand command = fixtures.confirmCommand(reservationId);

    try (TwoActors actors = new TwoActors()) {
        List<Outcome<Object>> outcomes = actors.runTogether(
            () -> checkout.confirm(command),
            expiry::expireNext
        );

        assertThat(outcomes.stream()
            .filter(Outcome::isFailure))
            .isEmpty();
    }

    assertThat(probe.status(reservationId)).isEqualTo(CONFIRMED);
    assertThat(probe.orderCount(reservationId)).isEqualTo(1);
    assertThat(probe.paymentOutboxCount(reservationId)).isEqualTo(1);
    probe.assertConsumedExactlyOnce(reservationId);
    probe.assertInventoryConserved();
    probe.assertReservedProjectionMatchesReservations();
}
```

Tác vụ không tìm thấy dòng đến hạn. Quan trọng hơn, nếu một danh sách ứng viên cũ
vô tình chứa ID này, điều kiện `status = 'RESERVED' AND expires_at <= now` của
`tryExpire` vẫn trả `0` và không được hoàn kho.

## 8. Hai xác nhận cùng phiên bản chỉ tạo một đơn

Khoản giữ còn hạn để cả hai câu xác nhận đều đủ điều kiện thời gian. Hai lệnh dùng
hai khóa checkout khác nhau nhằm kiểm tra chính trạng thái khoản giữ, không phải
gộp bằng cùng khóa lũy đẳng:

```java
@Test
void twoDifferentConfirmationRequestsCreateOneOrder() throws Exception {
    UUID reservationId = fixtures.reservedFarInFuture(line(101L, 1));
    ConfirmReservationCommand first = fixtures.confirmCommand(
        reservationId,
        idempotencyKey("KEY-A")
    );
    ConfirmReservationCommand second = fixtures.confirmCommand(
        reservationId,
        idempotencyKey("KEY-B")
    );

    try (TwoActors actors = new TwoActors()) {
        List<Outcome<ReservationConfirmationResult>> outcomes =
            actors.runTogether(
                () -> checkout.confirm(first),
                () -> checkout.confirm(second)
            );

        assertThat(outcomes.stream()
            .filter(Outcome::isSuccess)
            .map(Outcome::value)
            .filter(result -> result.outcome() == CONFIRMED))
            .hasSize(1);
    }

    assertThat(probe.status(reservationId)).isEqualTo(CONFIRMED);
    assertThat(probe.orderCount(reservationId)).isEqualTo(1);
    assertThat(probe.paymentOutboxCount(reservationId)).isEqualTo(1);
    probe.assertConsumedExactlyOnce(reservationId);
}
```

Bên thua có thể nhận `ReservationAlreadyConfirmedException` hoặc kết quả
`ALREADY_CONFIRMED` theo hợp đồng công khai. Nó không được tạo đơn thứ hai.

Test này cũng chứng minh bên chờ đánh giá lại `status = 'RESERVED'` sau khi bên
thắng chốt.

## 9. Hai tác vụ chỉ hoàn kho một lần

```java
@Test
void concurrentExpiryWorkersReleaseReservationOnce() throws Exception {
    UUID reservationId = fixtures.reservedAlreadyExpired(line(101L, 3));

    try (TwoActors actors = new TwoActors()) {
        List<Outcome<ExpiryAttempt>> outcomes = actors.runTogether(
            expiry::expireNext,
            expiry::expireNext
        );

        assertThat(outcomes).allMatch(Outcome::isSuccess);
        assertThat(outcomes)
            .extracting(Outcome::value)
            .containsExactlyInAnyOrder(
                ExpiryAttempt.EXPIRED,
                ExpiryAttempt.NO_DUE_RESERVATION
            );
    }

    assertThat(probe.status(reservationId)).isEqualTo(EXPIRED);
    probe.assertReleasedExactlyOnce(reservationId);
    probe.assertInventoryConserved();
}
```

Kết quả không phụ thuộc tác vụ nào thắng. `SKIP LOCKED` cho tác vụ còn lại tiếp
tục thay vì chờ cùng dòng.

## 10. Chứng minh `SKIP LOCKED` không chờ dòng đã có chủ

Thêm cổng test ngay sau `lockNextDue`, trước `tryExpire`:

```java
CountDownLatch firstLocked = new CountDownLatch(1);
CountDownLatch allowFirstToFinish = new CountDownLatch(1);

expiryHooks.afterCandidateLocked(reservationId -> {
    firstLocked.countDown();
    await(allowFirstToFinish);
});
```

Kịch bản:

1. tác vụ A khóa `R-1`, báo `firstLocked` rồi dừng;
2. tác vụ B gọi `expireNext()`;
3. B phải lấy `R-2` hoặc trả `NO_DUE_RESERVATION`, không đứng chờ `R-1`;
4. cho A tiếp tục và chốt;
5. xác nhận mỗi khoản giữ được hoàn đúng một lần.

```java
assertThat(second.get(5, TimeUnit.SECONDS))
    .isNotEqualTo(blockedOnFirstReservation());
```

Không dùng thời gian thực thi ngắn làm bằng chứng duy nhất. Kết hợp `CountDownLatch`, `Future`
có thời gian tối đa và kiểm tra ID mà từng tác vụ đã xử lý.

## 11. PostgreSQL quyết định thời gian, không phải `Clock` Java

```java
@Test
void staleApplicationClockCannotConfirmPastDatabaseExpiry() {
    UUID reservationId = fixtures.reservedAlreadyExpired(line(101L, 1));
    applicationClock.set(fixedFarBeforeExpiry());

    ReservationConfirmationResult result = checkout.confirm(
        fixtures.confirmCommand(reservationId)
    );

    assertThat(result.outcome()).isEqualTo(RESERVATION_EXPIRED);
    assertThat(probe.orderCount(reservationId)).isZero();
    assertThat(probe.status(reservationId)).isEqualTo(RESERVED);
}
```

Nếu tác vụ chưa chạy, trạng thái có thể vẫn `RESERVED`; điều cần chứng minh là
checkout vẫn từ chối vì câu SQL dùng `clock_timestamp()`. Sau đó chạy tác vụ và
kiểm tra trạng thái thành `EXPIRED` cùng tồn kho được hoàn.

Test đối xứng có thể đặt đồng hồ ứng dụng sau hạn trong khi PostgreSQL thấy khoản
giữ còn hạn; checkout vẫn được phép xác nhận. Mã chạy thật không được đọc
`applicationClock` để phân xử.

## 12. Lỗi bộ đếm làm hoàn tác trạng thái hết hạn

Tạo dữ liệu hỏng có kiểm soát trong test tích hợp, chẳng hạn khoản giữ yêu cầu
`2` nhưng `reserved_quantity = 1`. Nếu ràng buộc migration ngăn dữ liệu thử nghiệm này, dùng
một điểm chèn lỗi dành cho test ném ngoại lệ ngay sau `tryExpire`:

```java
@Test
void failureAfterExpiryTransitionRollsBackEverything() {
    UUID reservationId = fixtures.reservedAlreadyExpired(line(101L, 2));
    InventorySnapshot before = probe.inventory(101L);
    expiryHooks.failAfterTransition(reservationId);

    assertThatThrownBy(expiry::expireNext)
        .isInstanceOf(SimulatedInventoryWriteFailure.class);

    assertThat(probe.status(reservationId)).isEqualTo(RESERVED);
    assertThat(probe.inventory(101L)).isEqualTo(before);
    assertThat(probe.orderCount(reservationId)).isZero();
}
```

Điểm chèn lỗi phải ném ngoại lệ không được bắt bên trong phương thức `@Transactional`.
Mục tiêu là chứng minh trạng thái và bộ đếm cùng hoàn tác.

## 13. Lỗi tạo đơn hoặc outbox làm hoàn tác xác nhận

```java
@Test
void outboxFailureRollsBackConfirmationAndInventoryConsumption() {
    UUID reservationId = fixtures.reservedFarInFuture(line(101L, 1));
    InventorySnapshot before = probe.inventory(101L);
    fixtures.makeOutboxInsertFailFor(reservationId);

    assertThatThrownBy(
        () -> checkout.confirm(fixtures.confirmCommand(reservationId))
    ).isInstanceOf(DataIntegrityViolationException.class);

    assertThat(probe.status(reservationId)).isEqualTo(RESERVED);
    assertThat(probe.inventory(101L)).isEqualTo(before);
    assertThat(probe.orderCount(reservationId)).isZero();
    assertThat(probe.paymentOutboxCount(reservationId)).isZero();
    assertThat(probe.checkoutRequestCountForLastKey()).isZero();
}
```

Quyền checkout vừa chiếm cũng hoàn tác theo ECOM-003. Lần gửi lại cùng khóa có
thể thử lại từ đầu.

## 14. Mất phản hồi không tạo đơn thứ hai

```java
@Test
void retryAfterAmbiguousCommitReplaysConfirmedOrder() {
    UUID reservationId = fixtures.reservedFarInFuture(line(101L, 1));
    ConfirmReservationCommand command = fixtures.confirmCommand(reservationId);

    ReservationConfirmationResult first = checkout.confirm(command);
    ReservationConfirmationResult replay = checkout.confirm(command);

    assertThat(first.outcome()).isEqualTo(CONFIRMED);
    assertThat(replay.outcome()).isEqualTo(CONFIRMED);
    assertThat(replay.orderId()).isEqualTo(first.orderId());
    assertThat(replay.replayed()).isTrue();

    assertThat(probe.orderCount(reservationId)).isEqualTo(1);
    assertThat(probe.paymentOutboxCount(reservationId)).isEqualTo(1);
    probe.assertConsumedExactlyOnce(reservationId);
}
```

Một test riêng gửi cùng khóa nhưng dấu vân tay khác và phải nhận
`IdempotencyKeyReusedException` mà không thay đổi dữ liệu.

## 15. Kết quả từ chối cũng được phát lại

```java
@Test
void expiredCheckoutResultIsStableForSameIdempotencyKey() {
    UUID reservationId = fixtures.reservedAlreadyExpired(line(101L, 1));
    ConfirmReservationCommand command = fixtures.confirmCommand(reservationId);

    ReservationConfirmationResult first = checkout.confirm(command);
    expiry.expireNext();
    ReservationConfirmationResult replay = checkout.confirm(command);

    assertThat(first.outcome()).isEqualTo(RESERVATION_EXPIRED);
    assertThat(replay.outcome()).isEqualTo(RESERVATION_EXPIRED);
    assertThat(replay.replayed()).isTrue();
    assertThat(probe.orderCount(reservationId)).isZero();
    assertThat(probe.paymentOutboxCount(reservationId)).isZero();
    probe.assertReleasedExactlyOnce(reservationId);
}
```

Việc tác vụ hoàn kho sau phản hồi đầu không biến cùng checkout request thành một
ý định mới.

## 16. Nhiều sản phẩm vẫn giữ bất biến

```java
@Test
void multiLineConfirmationConsumesEveryHeldLine() {
    UUID reservationId = fixtures.reservedFarInFuture(
        line(303L, 2),
        line(101L, 1),
        line(202L, 3)
    );

    ReservationConfirmationResult result = checkout.confirm(
        fixtures.confirmCommand(reservationId)
    );

    assertThat(result.outcome()).isEqualTo(CONFIRMED);
    probe.assertInventoryLockedInOrder(101L, 202L, 303L);
    probe.assertConsumedExactlyOnce(reservationId);
    probe.assertInventoryConserved();
    probe.assertReservedProjectionMatchesReservations();
}
```

Không chỉ kiểm tra tổng lượng. Mỗi dòng sản phẩm phải thay đổi đúng theo dòng giữ
tương ứng; không được chốt nếu thiếu một `inventory_item`.

## 17. Hết thời gian chờ khóa là lỗi kỹ thuật

Dùng hai kết nối điều khiển rõ ràng:

1. kết nối A bắt đầu giao dịch và khóa dòng khoản giữ còn hạn;
2. `CountDownLatch` báo khóa đã được giữ;
3. kết nối B đặt `SET LOCAL lock_timeout = '100ms'` rồi gọi câu xác nhận;
4. xác nhận B nhận SQLSTATE `55P03` hoặc ngoại lệ Spring tương ứng;
5. hoàn tác A trong khối `finally`;
6. xác nhận B không nhận `RESERVATION_EXPIRED` và không có checkout response giả.

Thời lượng `100ms` chỉ là giới hạn kỹ thuật giúp test kết thúc, không phải kết
quả đo hiệu năng. Latch và quyền chốt giao dịch tạo lịch chạy; không dùng sleep
để đoán khóa đã được giữ.

## 18. Tác vụ hoàn tác nhả khóa cho tác vụ khác

```java
@Test
void secondWorkerCanProcessAfterFirstRollsBack() throws Exception {
    UUID reservationId = fixtures.reservedAlreadyExpired(line(101L, 1));
    CountDownLatch firstTransitioned = new CountDownLatch(1);
    CountDownLatch forceRollback = new CountDownLatch(1);

    Future<?> first = executor.submit(() ->
        expiryTestDriver.transitionThenRollback(
            reservationId,
            firstTransitioned,
            forceRollback
        )
    );

    assertThat(firstTransitioned.await(5, TimeUnit.SECONDS)).isTrue();
    Future<ExpiryAttempt> second = executor.submit(expiry::expireNext);

    forceRollback.countDown();
    first.get(5, TimeUnit.SECONDS);
    assertThat(second.get(5, TimeUnit.SECONDS)).isEqualTo(EXPIRED);

    probe.assertReleasedExactlyOnce(reservationId);
    probe.assertInventoryConserved();
}
```

Nếu tác vụ đầu dừng tiến trình thay vì chủ động hoàn tác, đóng kết nối thử nghiệm
tạo cùng hành vi PostgreSQL: giao dịch hoàn tác và khóa được nhả.

## 19. Hai máy chủ ứng dụng

```java
@Test
void databaseCoordinatesCheckoutAndExpiryOnDifferentInstances()
    throws Exception {

    UUID reservationId = fixtures.reservedAlreadyExpired(line(101L, 1));
    ReservationClient appA = instanceA.reservationClient();
    ExpiryClient appB = instanceB.expiryClient();

    try (TwoActors actors = new TwoActors()) {
        actors.runTogether(
            () -> appA.confirm(fixtures.confirmCommand(reservationId)),
            appB::expireNext
        );
    }

    assertThat(probe.status(reservationId)).isEqualTo(EXPIRED);
    assertThat(probe.orderCount(reservationId)).isZero();
    probe.assertReleasedExactlyOnce(reservationId);
}
```

Hai ứng dụng dùng JVM và vùng kết nối riêng nhưng cùng PostgreSQL container. Test
này chứng minh không có giả định ẩn về khóa cục bộ hoặc bộ lập lịch duy nhất.

## 20. Truy vấn đối soát bất biến

```sql
SELECT item.product_id,
       item.reserved_quantity,
       COALESCE(SUM(
           CASE WHEN reservation.status = 'RESERVED' THEN line.quantity ELSE 0 END
       ), 0) AS projected_reserved
FROM inventory_item AS item
LEFT JOIN inventory_reservation_line AS line
       ON line.product_id = item.product_id
LEFT JOIN inventory_reservation AS reservation
       ON reservation.reservation_id = line.reservation_id
      AND reservation.status = 'RESERVED'
GROUP BY item.product_id, item.reserved_quantity
HAVING item.reserved_quantity <> COALESCE(SUM(
    CASE WHEN reservation.status = 'RESERVED' THEN line.quantity ELSE 0 END
), 0);
```

Kết quả phải rỗng. Một truy vấn riêng tìm đơn tham chiếu khoản giữ không được xác
nhận:

```sql
SELECT orders.order_id, orders.reservation_id, reservation.status
FROM purchase_order AS orders
JOIN inventory_reservation AS reservation
  ON reservation.reservation_id = orders.reservation_id
WHERE reservation.status <> 'CONFIRMED';
```

Đối soát là lớp phát hiện, không thay thế giao dịch an toàn. Cảnh báo phải dẫn
tới quy trình điều tra và sửa bằng thao tác có kiểm toán.

## 21. Dữ liệu cần giữ khi test thất bại

- trạng thái, `expires_at`, `confirmed_at`, `expired_at` và `version`;
- bộ đếm từng sản phẩm trước/sau;
- số đơn, outbox và checkout request;
- tác nhân nào nhận `0/1` dòng từ câu chuyển trạng thái;
- SQLSTATE, tên ràng buộc và nguyên nhân gốc;
- thời điểm `BEGIN`, giành trạng thái, khóa tồn kho, `COMMIT` hoặc `ROLLBACK`;
- số dòng bị `SKIP LOCKED` bỏ qua và số lần thử lại.

Không ghi khóa lũy đẳng hoặc dữ liệu khách hàng nguyên văn nếu chúng nhạy cảm.

## 22. Ma trận thực nghiệm tối thiểu

| Trường hợp | Điều phải chứng minh |
| --- | --- |
| Bản lỗi cùng vượt kiểm tra | Có đơn/lệnh thanh toán trong khi tồn kho đã hoàn |
| Khoản giữ đã quá hạn | Xác nhận bị từ chối, tác vụ hoàn đúng một lần |
| Khoản giữ còn hạn | Xác nhận thắng, tác vụ không hoàn kho |
| Hai xác nhận khác khóa | Một đơn và một lần tiêu thụ tồn kho |
| Hai tác vụ | Một `EXPIRED`, một bỏ qua hoặc lấy việc khác |
| Dòng đang khóa | `SKIP LOCKED` không chờ dòng đó |
| Đồng hồ JVM sai | Kết quả vẫn theo thời gian PostgreSQL |
| Lỗi sau chuyển trạng thái | Trạng thái và bộ đếm cùng hoàn tác |
| Lỗi tạo đơn/outbox | Xác nhận và quyền checkout cùng hoàn tác |
| Mất phản hồi | Cùng khóa phát lại cùng đơn |
| Từ chối do hết hạn | Cùng khóa phát lại cùng kết quả từ chối |
| Nhiều sản phẩm | Khóa đúng thứ tự và mọi dòng cùng thay đổi |
| Hết thời gian chờ/bế tắc | Được phân loại là lỗi kỹ thuật |
| Tác vụ đầu hoàn tác | Tác vụ sau xử lý được và chỉ hoàn một lần |
| Hai máy chủ | PostgreSQL vẫn phân xử đúng một nhánh |
| Đối soát | Không có bộ đếm lệch hoặc đơn trên khoản giữ không xác nhận |
