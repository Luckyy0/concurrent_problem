# Thực nghiệm checkout đồng thời với PostgreSQL

## 1. Mục tiêu

Bộ thực nghiệm phải chứng minh quy tắc nghiệp vụ, không chỉ chứng minh một ngoại
lệ đã được ném:

```text
count(checkout_request theo customer_id/idempotency_key) = 1
count(purchase_order theo request_id) <= 1
count(payment outbox theo order_id) <= 1
cùng khóa + cùng nội dung → cùng phản hồi
cùng khóa + khác nội dung → một bên bị từ chối
```

Các phép thử dùng PostgreSQL thật qua Testcontainers. H2 không mô phỏng đầy đủ
ảnh chụp MVCC, tranh chấp chỉ mục duy nhất, mã SQLSTATE và hành vi
`ON CONFLICT` của PostgreSQL.

## 2. Cấu hình Testcontainers

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("postgres-test")
class ConcurrentCheckoutTest {

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

    @Autowired
    CheckoutApplicationService checkout;

    @Autowired
    JdbcTemplate jdbc;
}
```

Migration trong phép thử phải giống migration sản xuất, bao gồm đúng tên ràng
buộc. Không để Hibernate tự sinh bảng rồi vô tình bỏ qua khác biệt lược đồ.

Mọi `Future.get`, `await` và truy vấn quan sát đều có thời gian tối đa. Không dùng
`Thread.sleep` để đoán hai luồng đã đi tới đâu.

## 3. Tiện ích chạy hai tác nhân

```java
final class TwoActors implements AutoCloseable {

    private final ExecutorService executor =
        Executors.newFixedThreadPool(2);

    <T> List<T> runTogether(
        Callable<T> actorA,
        Callable<T> actorB
    ) throws Exception {
        CyclicBarrier start = new CyclicBarrier(2);

        Future<T> first = executor.submit(() -> {
            start.await(5, TimeUnit.SECONDS);
            return actorA.call();
        });
        Future<T> second = executor.submit(() -> {
            start.await(5, TimeUnit.SECONDS);
            return actorB.call();
        });

        return List.of(
            first.get(10, TimeUnit.SECONDS),
            second.get(10, TimeUnit.SECONDS)
        );
    }

    @Override
    public void close() {
        executor.shutdownNow();
    }
}
```

Rào chắn chỉ đồng bộ thời điểm bắt đầu. Những test cần ép đúng vị trí
`SELECT`, `INSERT`, `COMMIT` hoặc `ROLLBACK` sẽ dùng chốt riêng tại vị trí đó.

## 4. Tái hiện lỗi `check → insert`

Dùng một cổng thử nghiệm đặt ngay sau câu kiểm tra. Mã nghiệp vụ thật không cần
biết chi tiết `CyclicBarrier`; nó chỉ gọi một giao diện mặc định không làm gì:

```java
@FunctionalInterface
interface AfterDuplicateCheck {
    void reached();

    static AfterDuplicateCheck noOp() {
        return () -> { };
    }
}
```

Trong hồ sơ kiểm thử, cài đặt giao diện bằng một rào chắn:

```java
@TestConfiguration
class BrokenRaceConfiguration {

    @Bean
    AfterDuplicateCheck afterDuplicateCheck() {
        CyclicBarrier bothChecked = new CyclicBarrier(2);
        return () -> {
            try {
                bothChecked.await(5, TimeUnit.SECONDS);
            } catch (InterruptedException error) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException(error);
            } catch (BrokenBarrierException | TimeoutException error) {
                throw new IllegalStateException(error);
            }
        };
    }
}
```

Biến thể dịch vụ bị lỗi gọi `afterDuplicateCheck.reached()` sau khi
`existsBy...` trả `false` và trước `save()`. Khi lược đồ cố ý bỏ ràng buộc nghiệp
vụ, phép thử sau luôn mở đúng khoảng hở:

```java
@Test
void checkThenInsertCreatesTwoOrders() throws Exception {
    UUID customerId = UUID.randomUUID();
    String key = "checkout-race-1";
    CheckoutCommand command = fixtures.validCommand();

    try (TwoActors actors = new TwoActors()) {
        actors.runTogether(
            () -> brokenCheckout.checkout(customerId, key, command),
            () -> brokenCheckout.checkout(customerId, key, command)
        );
    }

    Integer orders = jdbc.queryForObject(
        """
        SELECT count(*)
        FROM purchase_order
        WHERE customer_id = ? AND checkout_key = ?
        """,
        Integer.class,
        customerId,
        key
    );

    assertThat(orders).isEqualTo(2);
}
```

Test này chỉ tồn tại trong bộ tái hiện lỗi. Migration an toàn phải được dùng cho
các test giải pháp.

## 5. Hai yêu cầu giống nhau chỉ tạo một đơn

```java
@Test
void sameKeyAndPayloadReturnOneStoredResult() throws Exception {
    UUID customerId = UUID.randomUUID();
    IdempotencyKey key = IdempotencyKey.parse("checkout-safe-1");
    CheckoutCommand command = fixtures.validCommand();

    List<CheckoutResult> results;
    try (TwoActors actors = new TwoActors()) {
        results = actors.runTogether(
            () -> checkout.checkout(customerId, key, command),
            () -> checkout.checkout(customerId, key, command)
        );
    }

    Set<String> orderIds = results.stream()
        .map(CheckoutResult::responseBody)
        .map(body -> body.path("orderId").asText())
        .collect(Collectors.toSet());

    assertThat(orderIds).hasSize(1);
    assertThat(results)
        .extracting(CheckoutResult::httpStatus)
        .containsOnly(201);
    assertThat(results)
        .extracting(CheckoutResult::replayed)
        .containsExactlyInAnyOrder(false, true);

    assertThat(countCheckoutRequests(customerId, key.value())).isEqualTo(1);
    assertThat(countOrders(customerId, key.value())).isEqualTo(1);
    assertThat(countPaymentEvents(customerId, key.value())).isEqualTo(1);
}
```

Không nên chỉ khẳng định “một bên thành công, một bên ném lỗi”. Hợp đồng của case
này yêu cầu cả hai nhận cùng kết quả bền vững.

## 6. Cùng khóa nhưng khác nội dung

Để bắt được ngoại lệ mà vẫn thu thập kết quả của cả hai tác nhân, dùng một kiểu
kết quả thử nghiệm:

```java
record Attempt(CheckoutResult result, Throwable error) {

    static Attempt capture(Callable<CheckoutResult> call) {
        try {
            return new Attempt(call.call(), null);
        } catch (Throwable error) {
            return new Attempt(null, error);
        }
    }
}
```

```java
@Test
void sameKeyWithDifferentPayloadIsRejected() throws Exception {
    UUID customerId = UUID.randomUUID();
    IdempotencyKey key = IdempotencyKey.parse("checkout-mismatch-1");
    CheckoutCommand first = fixtures.validCommand();
    CheckoutCommand changed = fixtures.withDifferentQuote(first);

    List<Attempt> attempts;
    try (TwoActors actors = new TwoActors()) {
        attempts = actors.runTogether(
            () -> Attempt.capture(
                () -> checkout.checkout(customerId, key, first)
            ),
            () -> Attempt.capture(
                () -> checkout.checkout(customerId, key, changed)
            )
        );
    }

    assertThat(attempts.stream().filter(a -> a.result() != null).count())
        .isEqualTo(1);
    assertThat(attempts.stream().map(Attempt::error).filter(Objects::nonNull))
        .singleElement()
        .isInstanceOf(IdempotencyKeyReusedException.class);

    assertThat(countCheckoutRequests(customerId, key.value())).isEqualTo(1);
    assertThat(countOrders(customerId, key.value())).isEqualTo(1);
    assertThat(countPaymentEvents(customerId, key.value())).isEqualTo(1);
}
```

Test không giả định tác nhân nào thắng; lịch lập lịch của hệ điều hành không phải
một phần hợp đồng. Nó chỉ khẳng định một nội dung trở thành nội dung chuẩn và nội
dung còn lại không tạo tác dụng phụ.

## 7. Quan sát bên thua chờ chỉ mục

Phép thử mức JDBC giúp nhìn rõ hành vi PostgreSQL mà không phụ thuộc Hibernate:

1. Kết nối A tắt tự động chốt và chèn một `checkout_request` nhưng chưa chốt.
2. Kết nối B đặt `application_name`, rồi chạy câu chèn cùng khóa trong một
   `Future`.
3. Kết nối quan sát chờ tới khi `pg_blocking_pids(pid)` của B chứa tiến trình A.
4. Chốt A.
5. Khẳng định B nhận `0` dòng từ `RETURNING`, rồi đọc được bản ghi hoàn tất.

Ví dụ phần quan sát có giới hạn:

```java
await()
    .atMost(Duration.ofSeconds(5))
    .untilAsserted(() -> {
        Integer blockers = observer.queryForObject(
            """
            SELECT cardinality(pg_blocking_pids(pid))
            FROM pg_stat_activity
            WHERE application_name = ?
            """,
            Integer.class,
            waiterApplicationName
        );
        assertThat(blockers).isGreaterThan(0);
    });
```

Sau `connectionA.commit()`, `futureB.get(5, SECONDS)` phải hoàn tất. Không dùng
một khoảng ngủ cố định để kết luận B đang chờ.

`wait_event_type` và `wait_event` hữu ích để chẩn đoán, nhưng tên chi tiết có thể
thay đổi theo phiên bản PostgreSQL. Quy tắc nghiệp vụ và `pg_blocking_pids` mới
là khẳng định chính của test.

## 8. Bên thắng hoàn tác, bên chờ trở thành bên thắng

Lặp lại cách bố trí ở phần trước nhưng gọi `connectionA.rollback()`:

```java
@Test
void waiterCanClaimAfterWinnerRollsBack() throws Exception {
    try (
        Connection first = dataSource.getConnection();
        Connection second = dataSource.getConnection()
    ) {
        first.setAutoCommit(false);
        second.setAutoCommit(false);

        UUID firstRequest = insertClaim(first, CUSTOMER, KEY, FINGERPRINT);
        assertThat(firstRequest).isNotNull();

        Future<UUID> waitingInsert = executor.submit(
            () -> insertClaim(second, CUSTOMER, KEY, FINGERPRINT)
        );

        blockingObserver.awaitBlocked(second);
        first.rollback();

        UUID secondRequest = waitingInsert.get(5, TimeUnit.SECONDS);
        assertThat(secondRequest).isNotNull();
        second.commit();
    }

    assertThat(countCheckoutRequests(CUSTOMER, KEY)).isEqualTo(1);
}
```

`insertClaim` phải dùng chính câu `ON CONFLICT DO NOTHING RETURNING` của sản
phẩm. `blockingObserver` nhận PID bằng `SELECT pg_backend_pid()` trên kết nối B
và dùng một kết nối thứ ba để quan sát.

## 9. Lỗi sau khi chiếm quyền phải hoàn tác tất cả

Đặt một điểm gây lỗi thử nghiệm sau khi tạo đơn nhưng trước khi hoàn tất phản
hồi:

```java
@Test
void technicalFailureRollsBackClaimOrderAndOutbox() {
    UUID customerId = UUID.randomUUID();
    IdempotencyKey key = IdempotencyKey.parse("checkout-rollback-1");
    CheckoutCommand command = fixtures.validCommand();

    faults.failOnceAfterOrderInsert();

    assertThatThrownBy(
        () -> checkout.checkout(customerId, key, command)
    ).isInstanceOf(SimulatedTechnicalFailure.class);

    assertThat(countCheckoutRequests(customerId, key.value())).isZero();
    assertThat(countOrders(customerId, key.value())).isZero();
    assertThat(countPaymentEvents(customerId, key.value())).isZero();

    CheckoutResult retry = checkout.checkout(customerId, key, command);

    assertThat(retry.httpStatus()).isEqualTo(201);
    assertThat(countCheckoutRequests(customerId, key.value())).isEqualTo(1);
    assertThat(countOrders(customerId, key.value())).isEqualTo(1);
    assertThat(countPaymentEvents(customerId, key.value())).isEqualTo(1);
}
```

Điểm gây lỗi phải ném một ngoại lệ không bị bắt trong giao dịch. Nếu mã sản phẩm
bắt ngoại lệ rồi trả kết quả bình thường, Spring có thể chốt dữ liệu dở dang.

## 10. Chốt thành công nhưng mất phản hồi

Không cần phá kết nối thật để kiểm tra quy tắc quan trọng. Gọi dịch vụ lần đầu,
bỏ kết quả như thể phản hồi bị mất, rồi gửi lại cùng khóa:

```java
@Test
void retryAfterLostResponseReplaysCommittedResult() {
    UUID customerId = UUID.randomUUID();
    IdempotencyKey key = IdempotencyKey.parse("checkout-lost-response-1");
    CheckoutCommand command = fixtures.validCommand();

    CheckoutResult notDelivered = checkout.checkout(
        customerId,
        key,
        command
    );
    CheckoutResult replay = checkout.checkout(customerId, key, command);

    assertThat(replay.replayed()).isTrue();
    assertThat(replay.httpStatus()).isEqualTo(notDelivered.httpStatus());
    assertThat(replay.responseBody()).isEqualTo(notDelivered.responseBody());
    assertThat(countOrders(customerId, key.value())).isEqualTo(1);
    assertThat(countPaymentEvents(customerId, key.value())).isEqualTo(1);
}
```

Một test mức kết nối có thể dùng proxy mạng để cắt phản hồi sau khi PostgreSQL
nhận `COMMIT`, nhưng test trên đã chứng minh hợp đồng phát lại mà không phụ thuộc
thời điểm mạng khó lặp lại.

## 11. Kết quả nghiệp vụ bị từ chối cũng được phát lại

Nếu chính sách chọn lưu kết quả giỏ hết hạn hoặc báo giá không hợp lệ, cần test:

```text
lần đầu: 422 QUOTE_EXPIRED, checkout_request = COMPLETED/REJECTED
lần hai cùng khóa và dấu vân tay: đúng 422 và đúng nội dung cũ
purchase_order = 0
outbox_event = 0
```

Sau lần đầu, thay đổi báo giá trong cơ sở dữ liệu không được làm lần phát lại đổi
thành thành công. Nếu sản phẩm chọn đánh giá lại thay vì lưu từ chối, hợp đồng và
test phải nói rõ điều ngược lại.

## 12. Ràng buộc phòng thủ cho đơn và outbox

Hai test trực tiếp xác minh migration, không chỉ mã dịch vụ:

```java
@Test
void oneRequestCannotOwnTwoOrders() {
    UUID requestId = fixtures.completedOrProcessingRequest();
    fixtures.insertOrder(requestId);

    assertThatThrownBy(() -> fixtures.insertOrder(requestId))
        .satisfies(error -> assertUniqueViolation(
            error,
            "uk_order_checkout_request"
        ));
}

@Test
void oneOrderHasOneLogicalPaymentEvent() {
    UUID orderId = fixtures.order();
    fixtures.insertPaymentRequested(orderId);

    assertThatThrownBy(() -> fixtures.insertPaymentRequested(orderId))
        .satisfies(error -> assertUniqueViolation(
            error,
            "uk_outbox_aggregate_event"
        ));
}
```

`assertUniqueViolation` duyệt chuỗi nguyên nhân, kiểm tra SQLSTATE `23505` và tên
ràng buộc từ trình điều khiển PostgreSQL. Không coi mọi lỗi toàn vẹn dữ liệu là
lỗi trùng.

## 13. Nhiều máy chủ ứng dụng

Hai luồng với hai giao dịch đã kiểm tra ranh giới có thẩm quyền là PostgreSQL.
Để kiểm tra cấu hình triển khai, có thể khởi động hai `ApplicationContext` cùng
trỏ tới một container và gọi mỗi bean một lần:

```text
App A ─┐
       ├── PostgreSQL Testcontainer
App B ─┘
```

Khẳng định cuối vẫn là một `checkout_request`, một đơn, một outbox và cùng phản
hồi. Không thêm khóa tĩnh trong test, vì nó sẽ che giấu lỗi mà môi trường nhiều
máy chủ cần phát hiện.

## 14. Thử nghiệm hết thời gian chờ

Đặt `lock_timeout` ngắn chỉ trong giao dịch B, giữ A chưa chốt rồi cho B chèn cùng
khóa. B phải nhận lỗi chờ khóa, không được nhận `IDEMPOTENCY_KEY_REUSED`,
`OUT_OF_STOCK` hoặc một phản hồi thành công giả.

Sau khi A chốt:

```text
B gửi lại cùng khóa + dấu vân tay
→ đọc checkout_request đã hoàn tất
→ nhận phản hồi đã lưu
→ số đơn và outbox vẫn bằng 1
```

Sau khi A hoàn tác:

```text
B gửi lại cùng khóa + dấu vân tay
→ chiếm quyền thành công
→ tạo đúng một đơn và một outbox
```

## 15. Truy vấn xác minh bất biến

Các truy vấn sau phù hợp cho test và đối soát:

```sql
SELECT customer_id, idempotency_key, count(*)
FROM checkout_request
GROUP BY customer_id, idempotency_key
HAVING count(*) > 1;

SELECT checkout_request_id, count(*)
FROM purchase_order
GROUP BY checkout_request_id
HAVING count(*) > 1;

SELECT aggregate_id, event_type, count(*)
FROM outbox_event
WHERE aggregate_type = 'ORDER'
GROUP BY aggregate_id, event_type
HAVING count(*) > 1;
```

Cả ba truy vấn phải trả về rỗng. Cũng cần đối soát chiều ngược lại: mọi
`checkout_request` thành công phải có đúng một đơn, và mọi đơn
`PENDING_PAYMENT` phải có một lệnh `PAYMENT_REQUESTED`.

## 16. Dữ liệu quan sát trong test tải

Không đặt một con số chuẩn chung. Khi chạy thử trên hạ tầng gần sản xuất, thu
thập:

- tỷ lệ yêu cầu chiếm quyền mới, yêu cầu trùng và phát lại;
- thời gian chờ tại câu `INSERT` theo phân vị;
- số lần sai dấu vân tay;
- số giao dịch hoàn tác, bế tắc và hết thời gian chờ;
- số kết nối đang chờ khóa giao dịch;
- độ tuổi của dòng outbox chưa phát;
- số lần thông điệp thanh toán được gửi lại;
- kết quả ba truy vấn bất biến ở phần trên.

Nhiều lần phát lại không tự động là lỗi; chúng có thể phản ánh mạng không ổn định
hoặc chính sách thử lại. Vi phạm là xuất hiện nhiều đơn hoặc nhiều lệnh logic cho
cùng quyền xử lý.

## 17. Danh sách kiểm tra bộ thực nghiệm

- [ ] PostgreSQL Testcontainers và migration thật được sử dụng.
- [ ] Mỗi tác nhân có giao dịch và kết nối riêng.
- [ ] Rào chắn hoặc chốt đặt tại đúng điểm cần điều khiển.
- [ ] Không dùng `Thread.sleep` làm cơ chế đồng bộ chính.
- [ ] Mọi lần chờ đều có thời gian tối đa.
- [ ] Lỗi `check → insert` được tái hiện trước khi xác nhận bản sửa.
- [ ] Hai yêu cầu giống nhau nhận cùng `order_id`.
- [ ] Khác dấu vân tay bị từ chối mà không có tác dụng phụ thứ hai.
- [ ] `COMMIT`, `ROLLBACK`, mất phản hồi và hết thời gian chờ đều được kiểm tra.
- [ ] Ràng buộc trên mã yêu cầu, đơn hàng và outbox được kiểm tra trực tiếp.
- [ ] Khẳng định số dòng nghiệp vụ, không chỉ loại ngoại lệ.
- [ ] Có phép thử hoặc lập luận triển khai cho hai máy chủ ứng dụng.
