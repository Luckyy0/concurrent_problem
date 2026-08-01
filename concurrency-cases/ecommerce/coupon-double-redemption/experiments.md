# Thực nghiệm giới hạn coupon/voucher với PostgreSQL

## 1. Mục tiêu

Bộ thực nghiệm phải kiểm tra trực tiếp:

```text
redeemed_count không vượt global_limit
used_count không vượt per_user_limit
mỗi (promotion_id, checkout_id) có tối đa một lịch sử
bộ đếm toàn cục và bộ đếm người dùng khớp lịch sử APPLIED
```

Không chỉ khẳng định một ngoại lệ xảy ra. Một test có thể nhận đúng ngoại lệ
nhưng giao dịch vẫn chốt bộ đếm dở dang nếu ranh giới Spring bị đặt sai.

## 2. Cấu hình PostgreSQL Testcontainers

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("postgres-test")
class ConcurrentPromotionRedemptionTest {

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
    PromotionApplicationService promotions;

    @Autowired
    JdbcTemplate jdbc;
}
```

Test giải pháp dùng migration sản xuất. Test tái hiện lỗi dùng một migration cố
ý thiếu các lớp bảo vệ và phải được tách hồ sơ để không làm yếu bộ test chính.

## 3. Tiện ích điều phối hai tác nhân

```java
final class TwoActors implements AutoCloseable {

    private final ExecutorService executor =
        Executors.newFixedThreadPool(2);

    <T> List<T> runTogether(
        Callable<T> firstActor,
        Callable<T> secondActor
    ) throws Exception {
        CyclicBarrier start = new CyclicBarrier(2);

        Future<T> first = executor.submit(() -> {
            start.await(5, TimeUnit.SECONDS);
            return firstActor.call();
        });
        Future<T> second = executor.submit(() -> {
            start.await(5, TimeUnit.SECONDS);
            return secondActor.call();
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

Mọi lần `await` và `get` có thời gian tối đa. `Thread.sleep` không được dùng làm
cơ chế chính để đoán một giao dịch đã đọc hay đang chờ khóa.

## 4. Tái hiện lỗi đọc–kiểm tra–ghi

Biến thể dịch vụ bị lỗi nhận một cổng thử nghiệm và gọi nó sau khi đã đọc cả bộ
đếm toàn cục lẫn số lượt người dùng:

```java
@FunctionalInterface
interface AfterLimitChecks {
    void reached();
}
```

Trong test, hai lần gọi cùng chờ ở một `CyclicBarrier`. Nhờ đó cả hai đã quyết
định từ dữ liệu cũ trước khi bên nào ghi:

```java
@Test
void readCheckWriteAcceptsTwoRedemptions() throws Exception {
    PromotionFixture promotion = fixtures.promotion(
        100,
        99,
        1
    );
    UUID customerId = UUID.randomUUID();

    RedeemPromotionCommand first = fixtures.command(
        promotion.id(),
        customerId,
        UUID.randomUUID()
    );
    RedeemPromotionCommand second = fixtures.command(
        promotion.id(),
        customerId,
        UUID.randomUUID()
    );

    try (TwoActors actors = new TwoActors()) {
        actors.runTogether(
            () -> brokenService.apply(first),
            () -> brokenService.apply(second)
        );
    }

    assertThat(countApplied(promotion.id())).isEqualTo(2);
    assertThat(countAppliedByCustomer(promotion.id(), customerId))
        .isEqualTo(2);
    assertThat(readRedeemedCount(promotion.id())).isEqualTo(100);
}
```

Kết quả cho thấy chỉ kiểm tra `redeemed_count <= global_limit` sẽ bỏ sót lỗi:
bộ đếm là `100`, nhưng có hai lịch sử mới và một khách hàng dùng hai lần.

## 5. Hai khách hàng tranh lượt cuối

Hai lệnh dùng hai khách hàng và hai checkout khác nhau để chỉ kiểm tra giới hạn
toàn cục:

```java
record Attempt(RedemptionResult result, Throwable error) {

    static Attempt capture(Callable<RedemptionResult> action) {
        try {
            return new Attempt(action.call(), null);
        } catch (Throwable error) {
            return new Attempt(null, error);
        }
    }
}
```

```java
@Test
void onlyOneCustomerWinsTheLastGlobalSlot() throws Exception {
    PromotionFixture promotion = fixtures.promotion(
        100,
        99,
        1
    );
    RedeemPromotionCommand first = fixtures.commandForNewCustomer(
        promotion.id()
    );
    RedeemPromotionCommand second = fixtures.commandForNewCustomer(
        promotion.id()
    );

    List<Attempt> attempts;
    try (TwoActors actors = new TwoActors()) {
        attempts = actors.runTogether(
            () -> Attempt.capture(() -> promotions.redeem(first)),
            () -> Attempt.capture(() -> promotions.redeem(second))
        );
    }

    assertThat(attempts.stream().filter(a -> a.result() != null).count())
        .isEqualTo(1);
    assertThat(attempts.stream().map(Attempt::error).filter(Objects::nonNull))
        .singleElement()
        .isInstanceOf(PromotionUnavailableException.class);

    assertThat(readRedeemedCount(promotion.id())).isEqualTo(100);
    assertThat(countApplied(promotion.id())).isEqualTo(1);
    assertThat(sumUserUsage(promotion.id())).isEqualTo(1);
    assertReconciled(promotion.id());
}
```

Test không giả định khách hàng nào thắng. Nó chỉ yêu cầu đúng một lần áp dụng và
mọi bộ đếm khớp lịch sử.

## 6. Một khách hàng có hai checkout

Đặt giới hạn toàn cục còn nhiều chỗ và giới hạn người dùng bằng `1`:

```java
@Test
void onlyOneCheckoutWinsPerUserLimit() throws Exception {
    PromotionFixture promotion = fixtures.promotion(
        1_000,
        10,
        1
    );
    UUID customerId = UUID.randomUUID();
    RedeemPromotionCommand first = fixtures.command(
        promotion.id(),
        customerId,
        UUID.randomUUID()
    );
    RedeemPromotionCommand second = fixtures.command(
        promotion.id(),
        customerId,
        UUID.randomUUID()
    );

    List<Attempt> attempts;
    try (TwoActors actors = new TwoActors()) {
        attempts = actors.runTogether(
            () -> Attempt.capture(() -> promotions.redeem(first)),
            () -> Attempt.capture(() -> promotions.redeem(second))
        );
    }

    assertThat(attempts.stream().filter(a -> a.result() != null).count())
        .isEqualTo(1);
    assertThat(attempts.stream().map(Attempt::error).filter(Objects::nonNull))
        .singleElement()
        .isInstanceOf(PerUserLimitReachedException.class);

    assertThat(readRedeemedCount(promotion.id())).isEqualTo(11);
    assertThat(readUserUsage(promotion.id(), customerId)).isEqualTo(1);
    assertThat(countAppliedByCustomer(promotion.id(), customerId))
        .isEqualTo(1);
    assertReconciled(promotion.id());
}
```

Giao dịch thua đã tăng tạm bộ đếm toàn cục trước khi phát hiện giới hạn người
dùng. Khẳng định `redeemed_count = 11`, không phải `12`, chứng minh phép tăng đó
đã được hoàn tác.

## 7. Cùng checkout được phát lại

```java
@Test
void duplicateCheckoutReplaysWithoutConsumingAnotherSlot()
    throws Exception {

    PromotionFixture promotion = fixtures.promotion(100, 10, 3);
    RedeemPromotionCommand command = fixtures.commandForNewCustomer(
        promotion.id()
    );

    List<RedemptionResult> results;
    try (TwoActors actors = new TwoActors()) {
        results = actors.runTogether(
            () -> promotions.redeem(command),
            () -> promotions.redeem(command)
        );
    }

    assertThat(results)
        .extracting(RedemptionResult::redemptionId)
        .containsOnly(results.getFirst().redemptionId());
    assertThat(results)
        .extracting(RedemptionResult::replayed)
        .containsExactlyInAnyOrder(false, true);

    assertThat(readRedeemedCount(promotion.id())).isEqualTo(11);
    assertThat(readUserUsage(promotion.id(), command.customerId()))
        .isEqualTo(1);
    assertThat(countApplied(promotion.id())).isEqualTo(1);
}
```

Phép thử này kiểm tra ràng buộc duy nhất và phát lại. Hai phép thử trước mới kiểm
tra giới hạn giữa các lệnh khác nhau.

## 8. Cùng checkout nhưng khác nội dung

Sau khi áp dụng thành công, đổi `quoteId` hoặc `cartVersion` nhưng giữ nguyên
`checkoutId`:

```java
@Test
void reusedCheckoutWithDifferentFingerprintIsRejected() {
    PromotionFixture promotion = fixtures.promotion(100, 10, 3);
    RedeemPromotionCommand original = fixtures.commandForNewCustomer(
        promotion.id()
    );
    RedeemPromotionCommand changed = fixtures.withDifferentQuote(original);

    promotions.redeem(original);

    assertThatThrownBy(() -> promotions.redeem(changed))
        .isInstanceOf(RedemptionKeyReusedException.class);

    assertThat(readRedeemedCount(promotion.id())).isEqualTo(11);
    assertThat(countApplied(promotion.id())).isEqualTo(1);
}
```

Không cập nhật dấu vân tay cũ để khớp yêu cầu mới.

## 9. Hoàn tác sau khi tăng bộ đếm toàn cục

Đặt một điểm gây lỗi sau `incrementIfAvailable` nhưng trước phép tăng người dùng:

```java
@Test
void failureAfterGlobalIncrementRollsBackEveryChange() {
    PromotionFixture promotion = fixtures.promotion(100, 10, 3);
    RedeemPromotionCommand command = fixtures.commandForNewCustomer(
        promotion.id()
    );

    faults.failOnceAfterGlobalIncrement();

    assertThatThrownBy(() -> promotions.redeem(command))
        .isInstanceOf(SimulatedTechnicalFailure.class);

    assertThat(readRedeemedCount(promotion.id())).isEqualTo(10);
    assertThat(findUserUsage(promotion.id(), command.customerId()))
        .isEmpty();
    assertThat(countAllRedemptions(promotion.id())).isZero();

    RedemptionResult retry = promotions.redeem(command);

    assertThat(retry.replayed()).isFalse();
    assertThat(readRedeemedCount(promotion.id())).isEqualTo(11);
    assertThat(readUserUsage(promotion.id(), command.customerId()))
        .isEqualTo(1);
    assertThat(countApplied(promotion.id())).isEqualTo(1);
}
```

Điểm gây lỗi phải ném ngoại lệ ra khỏi phương thức `@Transactional`. Nếu mã bắt
lỗi và trả kết quả bình thường, test số dòng sẽ phát hiện thay đổi dở dang.

## 10. Bên giữ khóa hoàn tác

Test mức JDBC điều khiển chính xác hai kết nối:

1. Kết nối A mở giao dịch và chạy câu `UPDATE` có điều kiện từ `99` lên `100`.
2. Kết nối B chạy cùng câu lệnh trong một `Future` và bị chặn.
3. Kết nối quan sát dùng `pg_blocking_pids` để xác nhận B đang chờ A.
4. A gọi `ROLLBACK`.
5. B phải trả về một dòng với `redeemed_count = 100` rồi chốt.

```java
await()
    .atMost(Duration.ofSeconds(5))
    .untilAsserted(() ->
        assertThat(blockingObserver.blockersOf(waiterPid))
            .contains(holderPid)
    );

holder.rollback();
PromotionCapacity result = waitingUpdate.get(5, TimeUnit.SECONDS);
waiter.commit();

assertThat(result.redeemedCount()).isEqualTo(100);
```

Không dùng thời gian ngủ cố định để suy ra B đang chờ.

## 11. Bên giữ khóa chốt và điều kiện được kiểm tra lại

Lặp lại bố trí trên nhưng A gọi `COMMIT`. `Future` của B phải hoàn tất với kết
quả rỗng. Sau đó:

```text
redeemed_count = 100
không có giá trị 101
```

Đây là phép thử trực tiếp cho hành vi kiểm tra lại mệnh đề `WHERE` ở
`READ COMMITTED`.

## 12. Dòng người dùng chưa tồn tại

Hai kết nối cùng chạy upsert cho một `(promotion_id, customer_id)` chưa có, với
`per_user_limit = 1`:

```text
một kết nối nhận used_count = 1
kết nối còn lại nhận 0 dòng
bảng chỉ có một dòng used_count = 1
```

Thêm biến thể bên chèn đầu tiên hoàn tác. Bên chờ khi đó phải chèn thành công.
Test này chứng minh chỉ mục khóa chính bảo vệ “lần dùng đầu tiên” mà một câu
`SELECT count` không thể khóa.

## 13. Hết thời gian chờ khóa không phải hết lượt

Kết nối A giữ dòng `promotion`. Trong giao dịch B:

```sql
SET LOCAL lock_timeout = '100ms';
```

B chạy phép tăng và nhận SQLSTATE `55P03`. Ứng dụng phải ánh xạ thành lỗi kỹ
thuật, không phải `GLOBAL_LIMIT_REACHED`. Sau khi A chốt hoặc hoàn tác, lần thử
mới bằng cùng checkout sẽ đọc lại hoặc áp dụng theo trạng thái thật.

Giá trị `100ms` chỉ dùng để test hoàn tất nhanh và không phải khuyến nghị cho môi
trường thật.

## 14. Thời hạn được kiểm tra tại cơ sở dữ liệu

Tạo ba chương trình:

```text
chưa bắt đầu: starts_at > CURRENT_TIMESTAMP
đang hoạt động: starts_at <= CURRENT_TIMESTAMP < ends_at
đã hết hạn: ends_at <= CURRENT_TIMESTAMP
```

Chỉ chương trình giữa trả một dòng từ phép tăng. Test đúng các biên
`starts_at`/`ends_at` để tránh sai một đơn vị thời gian và không thay thời gian
cơ sở dữ liệu bằng đồng hồ của JVM.

## 15. Thay đổi giới hạn đồng thời

Chức năng quản trị phải khóa/cập nhật dòng `promotion` trước. Test hai tình huống:

- quản trị giảm `global_limit` không được chốt một giá trị nhỏ hơn
  `redeemed_count` vì `CHECK` từ chối;
- quản trị thay đổi `per_user_limit` và một lần đổi mã cùng tranh dòng
  `promotion`, nên lần đổi mã dùng một giá trị giới hạn nhất quán sau khi lấy
  được khóa.

Nếu quy định cho phép giảm giới hạn xuống dưới số đã dùng, lược đồ cần tách
“giới hạn mục tiêu mới” khỏi “giới hạn đang có hiệu lực”; không được bỏ `CHECK`
mà vẫn tuyên bố bộ đếm hợp lệ.

## 16. Kiểm tra ràng buộc trực tiếp

```java
@Test
void oneCheckoutCannotHaveTwoRedemptions() {
    PromotionFixture promotion = fixtures.promotion(100, 0, 3);
    UUID checkoutId = UUID.randomUUID();
    fixtures.insertAppliedRedemption(promotion.id(), checkoutId);

    assertThatThrownBy(
        () -> fixtures.insertAppliedRedemption(
            promotion.id(),
            checkoutId
        )
    ).satisfies(error -> assertUniqueViolation(
        error,
        "uk_redemption_promotion_checkout"
    ));
}
```

Trình phân loại phải kiểm tra SQLSTATE `23505` và tên ràng buộc. Một lỗi khóa
ngoại hoặc `CHECK` không được báo thành “checkout đã dùng mã”.

## 17. Truy vấn đối soát

### Bộ đếm toàn cục

```sql
SELECT p.promotion_id,
       p.redeemed_count,
       count(r.redemption_id) AS applied_count
FROM promotion p
LEFT JOIN promotion_redemption r
  ON r.promotion_id = p.promotion_id
 AND r.status = 'APPLIED'
GROUP BY p.promotion_id, p.redeemed_count
HAVING p.redeemed_count <> count(r.redemption_id);
```

### Bộ đếm mỗi khách hàng

```sql
SELECT u.promotion_id,
       u.customer_id,
       u.used_count,
       count(r.redemption_id) AS applied_count
FROM promotion_user_usage u
LEFT JOIN promotion_redemption r
  ON r.promotion_id = u.promotion_id
 AND r.customer_id = u.customer_id
 AND r.status = 'APPLIED'
GROUP BY u.promotion_id, u.customer_id, u.used_count
HAVING u.used_count <> count(r.redemption_id);
```

Cả hai truy vấn phải trả về rỗng. Cần thêm truy vấn tìm lịch sử `APPLIED` không
có dòng `promotion_user_usage`, vì phép nối bắt đầu từ bảng bộ đếm sẽ không thấy
trường hợp thiếu hoàn toàn.

## 18. Thử nghiệm nhiều tác nhân

Chạy nhiều lệnh có `checkout_id` khác nhau qua một bộ thực thi, dùng rào chắn để
bắt đầu cùng lúc. Không đặt một số lượng chuẩn chung; chọn dữ liệu test sao cho
giới hạn nhỏ hơn số tác nhân.

Khẳng định cuối:

```text
số APPLIED = số lượt còn lại ban đầu
redeemed_count = giá trị ban đầu + số APPLIED
mọi used_count <= per_user_limit
không có checkout trùng
không Future nào chờ vô hạn
```

Thu thập toàn bộ lỗi từ tác nhân trước khi khẳng định. Không để ngoại lệ trong
luồng nền bị mất và làm test thành công giả.

## 19. Nhiều máy chủ ứng dụng

Hai luồng với hai kết nối đã đi qua ranh giới có thẩm quyền. Để kiểm tra cấu hình
triển khai, có thể khởi động hai `ApplicationContext` cùng trỏ tới một container,
mỗi context gọi một lệnh và không chia sẻ bean khóa Java.

Kết quả phải giống test một ứng dụng: giới hạn được giữ, một checkout có một lịch
sử và các truy vấn đối soát trả rỗng.

## 20. Dữ liệu quan sát khi test tải

- tỷ lệ `APPLIED`, giới hạn toàn cục, giới hạn người dùng và phát lại;
- thời gian chờ dòng `promotion` và dòng `promotion_user_usage`;
- số lỗi `55P03`, `40P01`, `40001`;
- thời gian giữ giao dịch và số kết nối đang chờ;
- số lần thử lại trên mỗi lệnh;
- kết quả truy vấn đối soát;
- chương trình có hàng đợi khóa dài nhất.

Không gắn `customer_id` hoặc `checkout_id` thô vào nhãn metric. Có thể dùng log
có kiểm soát hoặc mẫu băm khi cần truy vết.

## 21. Danh sách kiểm tra bộ thực nghiệm

- [ ] Dùng PostgreSQL Testcontainers và migration thật.
- [ ] Mỗi tác nhân có giao dịch và kết nối riêng.
- [ ] Rào chắn/chốt đặt tại đúng điểm đọc, ghi, chốt hoặc hoàn tác.
- [ ] Không dùng `Thread.sleep` làm cơ chế đồng bộ chính.
- [ ] Mọi lần chờ có thời gian tối đa.
- [ ] Tái hiện được lỗi với cùng khách hàng và với hai khách hàng.
- [ ] Kiểm tra lượt toàn cục cuối cùng và giới hạn mỗi người.
- [ ] Kiểm tra cùng checkout, sai dấu vân tay và phát lại.
- [ ] Kiểm tra chốt, hoàn tác, hết thời gian chờ và lỗi sau phép tăng.
- [ ] Kiểm tra trực tiếp tên ràng buộc và SQLSTATE.
- [ ] Khẳng định bộ đếm khớp lịch sử, không chỉ khẳng định ngoại lệ.
- [ ] Có kiểm tra hoặc lập luận triển khai cho nhiều máy chủ.
