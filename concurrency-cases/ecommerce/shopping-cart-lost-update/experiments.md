# Thực nghiệm thao tác giỏ hàng đồng thời với PostgreSQL

## 1. Mục tiêu

Mỗi test giải pháp phải kiểm tra trạng thái bền vững, không chỉ đếm ngoại lệ:

```text
mỗi sản phẩm xuất hiện tối đa một lần
quantity > 0 và không vượt giới hạn
chỉ một yêu cầu dùng cùng expectedVersion được chấp nhận
phiên bản tăng cùng thay đổi sản phẩm
bên xung đột không để lại thay đổi
nội dung cuối tương ứng đúng với bên thắng
```

Đối với biến thể cho phép thử lại lệnh cộng chênh lệch, còn phải kiểm tra mỗi
`commandId` chỉ tác động một lần.

## 2. Cấu hình Testcontainers

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("postgres-test")
class ShoppingCartConcurrencyTest {

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

    @Autowired CartMutationAttempt attempt;
    @Autowired BrokenCartService broken;
    @Autowired CartFixtures fixtures;
    @Autowired CartProbe probe;
}
```

Test phải chạy migration thật, gồm khóa chính ghép, kiểm tra số lượng, trạng thái
và phiên bản. H2 không thay thế hành vi MVCC, khóa dòng, `RETURNING` hay
`ON CONFLICT` của PostgreSQL.

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

Mọi lần chờ có thời gian tối đa. `Thread.sleep` không được dùng làm cách chính
để đoán rằng giao dịch đã đọc dữ liệu hoặc đang giữ khóa.

## 4. Điểm hẹn sau khi đọc

Để tái hiện lỗi cũ, thêm một cổng chỉ dành cho test ngay sau khi dịch vụ đã tải
giỏ nhưng trước khi thay danh sách:

```java
@FunctionalInterface
interface AfterCartLoad {
    void reached(UUID cartId, long observedVersion);
}
```

Trong test, cả hai giao dịch chờ tại cùng `CyclicBarrier`:

```java
CyclicBarrier bothLoaded = new CyclicBarrier(2);

AfterCartLoad rendezvous = (cartId, version) -> {
    try {
        assertThat(version).isEqualTo(7);
        bothLoaded.await(5, TimeUnit.SECONDS);
    } catch (Exception error) {
        throw new AssertionError(error);
    }
};
```

Không đưa cổng này vào quyết định nghiệp vụ. Nó chỉ làm lịch chạy xác định để
test luôn chứng minh cùng một khoảng hở.

## 5. Tái hiện mất cập nhật

```java
@Test
void replacingTwoStaleSnapshotsLosesAnAcceptedMutation() throws Exception {
    UUID cartId = fixtures.activeCart(7, item("SKU-A", 1));

    ReplaceCartRequest addB = request(
        item("SKU-A", 1),
        item("SKU-B", 1)
    );
    ReplaceCartRequest removeA = request();

    broken.setAfterCartLoad(rendezvousForVersion(7));

    List<Outcome<CartView>> outcomes;
    try (TwoActors actors = new TwoActors()) {
        outcomes = actors.runTogether(
            () -> broken.replace(cartId, addB),
            () -> broken.replace(cartId, removeA)
        );
    }

    assertThat(outcomes).allMatch(Outcome::isSuccess);

    CartSnapshot finalCart = probe.load(cartId);
    assertThat(finalCart.items())
        .isIn(List.of(), List.of(item("SKU-A", 1), item("SKU-B", 1)));
    assertThat(finalCart.items()).doesNotContainExactly(item("SKU-B", 1));
}
```

Tùy cách Hibernate sắp thứ tự xóa/chèn, biến thể cụ thể có thể kết thúc bằng lỗi
ràng buộc thay vì hai lần thành công. Khi đó, hãy giữ một bản triển khai lỗi tối
thiểu bằng SQL thay toàn bộ để chứng minh lost update, rồi viết test riêng cho
lỗi ánh xạ JPA. Không nới lỏng khẳng định đến mức test qua mà không chứng minh
thay đổi đã mất.

## 6. Cùng phiên bản chỉ một bên thắng

```java
@Test
void sameExpectedVersionAllowsExactlyOneMutation() throws Exception {
    UUID cartId = fixtures.activeCart(7, item("SKU-A", 1));

    AddItem addB = fixtures.add(cartId, "SKU-B", 1, 7);
    RemoveItem removeA = fixtures.remove(cartId, "SKU-A", 7);

    List<Outcome<CartView>> outcomes;
    try (TwoActors actors = new TwoActors()) {
        outcomes = actors.runTogether(
            () -> attempt.add(addB),
            () -> attempt.remove(removeA)
        );
    }

    assertThat(outcomes.stream().filter(Outcome::isSuccess)).hasSize(1);
    assertThat(outcomes.stream().filter(Outcome::isFailure)).singleElement()
        .satisfies(outcome -> assertThat(outcome.rootCause())
            .isInstanceOf(StaleCartVersionException.class));

    CartSnapshot finalCart = probe.load(cartId);
    assertThat(finalCart.version()).isEqualTo(8);

    if (outcomes.get(0).isSuccess()) {
        assertThat(finalCart.items()).containsExactlyInAnyOrder(
            item("SKU-A", 1), item("SKU-B", 1)
        );
    } else {
        assertThat(finalCart.items()).isEmpty();
    }

    probe.assertNoDuplicateProducts(cartId);
    probe.assertValidQuantities(cartId);
}
```

Không giả định tác nhân nào thắng. Trạng thái cuối phải đúng trọn vẹn với một
lệnh thắng, không phải hỗn hợp của hai lệnh.

## 7. Hai sản phẩm khác nhau vẫn xung đột ở cấp giỏ

```java
@Test
void changesToDifferentItemsStillShareAggregateVersion() throws Exception {
    UUID cartId = fixtures.activeCart(
        11,
        item("SKU-A", 1),
        item("SKU-B", 1)
    );

    SetQuantity changeA = fixtures.setQuantity(cartId, "SKU-A", 2, 11);
    SetQuantity changeB = fixtures.setQuantity(cartId, "SKU-B", 3, 11);

    List<Outcome<CartView>> outcomes;
    try (TwoActors actors = new TwoActors()) {
        outcomes = actors.runTogether(
            () -> attempt.setQuantity(changeA),
            () -> attempt.setQuantity(changeB)
        );
    }

    assertThat(outcomes.stream().filter(Outcome::isSuccess)).hasSize(1);
    assertThat(outcomes.stream().filter(Outcome::isFailure)).hasSize(1);
    assertThat(probe.version(cartId)).isEqualTo(12);

    Map<String, Integer> quantities = probe.quantities(cartId);
    assertThat(quantities).isIn(
        Map.of("SKU-A", 2, "SKU-B", 1),
        Map.of("SKU-A", 1, "SKU-B", 3)
    );
}
```

Test này ngăn một lần tối ưu hóa sai tách phiên bản theo dòng sản phẩm mà không
xem lại hợp đồng ảnh chụp toàn giỏ.

## 8. Lỗi sau khi giành phiên bản phải hoàn tác

```java
@Test
void itemFailureRollsBackClaimedVersion() {
    UUID cartId = fixtures.activeCart(4, item("SKU-A", 1));
    SetQuantity invalid = fixtures.setQuantity(cartId, "SKU-A", 1_000, 4);

    assertThatThrownBy(() -> attempt.setQuantity(invalid))
        .hasRootCauseInstanceOf(DataIntegrityViolationException.class);

    CartSnapshot finalCart = probe.load(cartId);
    assertThat(finalCart.version()).isEqualTo(4);
    assertThat(finalCart.items()).containsExactly(item("SKU-A", 1));
}
```

Nếu lớp Java chặn số lượng trước khi vào giao dịch, dùng một điểm gây lỗi có kiểm
soát ngay sau `claimNextVersion` hoặc gọi kho dữ liệu ở test tích hợp. Mục tiêu là
chứng minh rollback của PostgreSQL, không phải chỉ chứng minh kiểm tra đầu vào.

## 9. Giỏ không hoạt động không bị nhầm với phiên bản cũ

```java
@Test
void checkedOutCartIsReportedAsNotMutable() {
    UUID cartId = fixtures.cart(CartStatus.CHECKED_OUT, 9, item("SKU-A", 1));
    AddItem command = fixtures.add(cartId, "SKU-B", 1, 9);

    assertThatThrownBy(() -> attempt.add(command))
        .isInstanceOf(CartNotMutableException.class);

    assertThat(probe.version(cartId)).isEqualTo(9);
    assertThat(probe.items(cartId)).containsExactly(item("SKU-A", 1));
}
```

Việc câu `UPDATE` trả không dòng chưa đủ để trả mã lỗi. Test riêng cho không tồn
tại, sai trạng thái và sai phiên bản giữ hợp đồng phản hồi chính xác.

## 10. JPA `@Version` phát hiện xung đột

```java
@Test
void jpaAggregateVersionRollsBackLosingCollectionChange() throws Exception {
    UUID cartId = fixtures.activeCart(3, item("SKU-A", 1));
    JpaSetQuantity first = fixtures.jpaSet(cartId, "SKU-A", 2, 3);
    JpaSetQuantity second = fixtures.jpaSet(cartId, "SKU-A", 5, 3);

    jpaAttempt.setAfterLoad(rendezvousForVersion(3));

    List<Outcome<CartView>> outcomes;
    try (TwoActors actors = new TwoActors()) {
        outcomes = actors.runTogether(
            () -> jpaAttempt.setQuantity(first),
            () -> jpaAttempt.setQuantity(second)
        );
    }

    assertThat(outcomes.stream().filter(Outcome::isSuccess)).hasSize(1);
    assertThat(outcomes.stream().filter(Outcome::isFailure)).singleElement()
        .satisfies(outcome -> assertThat(outcome.rootCause())
            .isInstanceOfAny(
                ObjectOptimisticLockingFailureException.class,
                OptimisticLockException.class
            ));

    CartSnapshot finalCart = probe.load(cartId);
    assertThat(finalCart.version()).isEqualTo(4);
    assertThat(finalCart.quantity("SKU-A")).isIn(2, 5);
}
```

Bật ghi SQL trong test hoặc dùng bộ thu câu lệnh để xác nhận có điều kiện
`WHERE cart_id = ? AND version = ?`. Test cũng phải chứng minh thay đổi của bên
thua đã rollback, không chỉ thấy ngoại lệ.

## 11. Mọi đường sửa bảng con phải tăng phiên bản

```java
@Test
void everySupportedMutationAdvancesCartVersion() {
    UUID cartId = fixtures.activeCart(20, item("SKU-A", 1));

    CartView afterAdd = attempt.add(fixtures.add(cartId, "SKU-B", 1, 20));
    CartView afterSet = attempt.setQuantity(
        fixtures.setQuantity(cartId, "SKU-A", 2, afterAdd.version())
    );
    CartView afterRemove = attempt.remove(
        fixtures.remove(cartId, "SKU-B", afterSet.version())
    );

    assertThat(afterAdd.version()).isEqualTo(21);
    assertThat(afterSet.version()).isEqualTo(22);
    assertThat(afterRemove.version()).isEqualTo(23);
    assertThat(probe.version(cartId)).isEqualTo(23);
}
```

Nên có test kiến trúc hoặc rà soát kho dữ liệu để cấm `@Modifying` trực tiếp trên
`CartItem`. Một phép kiểm tra định kỳ có thể so sánh `updated_at` dòng con với
phiên bản gốc để phát hiện đường ghi bỏ qua.

## 12. Không tự thử lại lệnh đặt số lượng

```java
@Test
void staleSetQuantityIsReturnedToCallerWithoutSilentRetry() {
    UUID cartId = fixtures.activeCart(30, item("SKU-A", 1));

    attempt.setQuantity(fixtures.setQuantity(cartId, "SKU-A", 2, 30));

    assertThatThrownBy(
        () -> publicCartService.setQuantity(
            fixtures.setQuantity(cartId, "SKU-A", 5, 30)
        )
    ).isInstanceOf(StaleCartVersionException.class);

    assertThat(probe.version(cartId)).isEqualTo(31);
    assertThat(probe.quantity(cartId, "SKU-A")).isEqualTo(2);
    assertThat(probe.mutationAttemptCount()).isEqualTo(2);
}
```

Số lần gọi gồm một lệnh thành công và đúng một lần thử của lệnh cũ. Nếu thư viện
`@Retryable` âm thầm chạy lại, test này phải thất bại.

## 13. Lệnh cộng có thể thử lại nhưng không được nhân đôi

Sau khi bổ sung bảng kết quả theo `commandId`:

```java
@Test
void concurrentAddCommandsCanRebaseTheirDeltasOnce() throws Exception {
    UUID cartId = fixtures.activeCart(40, item("SKU-A", 1));
    AddItem addTwo = fixtures.add(cartId, "SKU-A", 2, 40, commandId("CMD-A"));
    AddItem addThree = fixtures.add(cartId, "SKU-A", 3, 40, commandId("CMD-B"));

    try (TwoActors actors = new TwoActors()) {
        List<Outcome<CartView>> outcomes = actors.runTogether(
            () -> coordinator.addWithBoundedRetry(addTwo),
            () -> coordinator.addWithBoundedRetry(addThree)
        );
        assertThat(outcomes).allMatch(Outcome::isSuccess);
    }

    assertThat(probe.quantity(cartId, "SKU-A")).isEqualTo(6);
    assertThat(probe.version(cartId)).isEqualTo(42);
    assertThat(probe.appliedCommandCount("CMD-A")).isEqualTo(1);
    assertThat(probe.appliedCommandCount("CMD-B")).isEqualTo(1);
}
```

Gửi lại `CMD-A` sau đó phải phát lại kết quả mà không đổi số lượng hay phiên bản.
Nếu chưa triển khai bảng mã lệnh, không được bật tự thử lại theo mẫu này.

## 14. Hết thời gian chờ khóa không phải xung đột phiên bản

Dùng hai kết nối điều khiển rõ ràng:

1. kết nối A bắt đầu giao dịch và cập nhật dòng `shopping_cart` nhưng chưa chốt;
2. `CountDownLatch` báo cho kết nối B bắt đầu;
3. B đặt `SET LOCAL lock_timeout = '100ms'` rồi chạy câu giành phiên bản;
4. xác nhận B nhận SQLSTATE `55P03` hoặc ngoại lệ khóa tương ứng;
5. hoàn tác giao dịch A trong khối `finally`;
6. xác nhận lớp HTTP không ánh xạ lỗi B thành `CART_VERSION_CONFLICT`.

Thời gian chờ tối đa là giới hạn kỹ thuật cho test, không phải số đo hiệu năng.
Không dùng `sleep` để giữ A; `CountDownLatch` và quyền chốt giao dịch tạo lịch
chạy xác định.

## 15. Nhiều phiên bản ứng dụng

```java
@Test
void databaseVersionCoordinatesTwoApplicationInstances() throws Exception {
    UUID cartId = fixtures.activeCart(50, item("SKU-A", 1));

    CartClient instanceA = appInstanceA.client();
    CartClient instanceB = appInstanceB.client();

    try (TwoActors actors = new TwoActors()) {
        List<Outcome<HttpResponse>> outcomes = actors.runTogether(
            () -> instanceA.add(cartId, "SKU-B", 1, 50),
            () -> instanceB.remove(cartId, "SKU-A", 50)
        );

        assertThat(outcomes).allMatch(Outcome::isSuccess);
        List<Integer> statuses = outcomes.stream()
            .map(Outcome::value)
            .map(HttpResponse::statusCode)
            .toList();

        assertThat(statuses).contains(200);
        assertThat(statuses.stream()
            .filter(status -> status == 409 || status == 412))
            .hasSize(1);
    }

    assertThat(probe.version(cartId)).isEqualTo(51);
    probe.assertMatchesSingleAcceptedMutation(cartId);
}
```

Hai ứng dụng phải dùng vùng kết nối và JVM riêng nhưng cùng PostgreSQL container.
Test này chứng minh không có giả định ẩn về khóa trong bộ nhớ.

## 16. Kiểm tra lặp và dấu vết thất bại

Chạy nhóm test đồng thời nhiều lần trong CI bằng tùy chọn của công cụ test, nhưng
mỗi lần vẫn phải có `CyclicBarrier` hoặc `CountDownLatch` cùng thời gian chờ tối
đa. Khi thất bại, lưu:

- mã lệnh và phiên bản dự kiến của từng tác nhân;
- phiên bản, trạng thái và sản phẩm cuối;
- kết quả `UPDATE ... RETURNING` của từng giao dịch;
- SQLSTATE và nguyên nhân gốc;
- thời điểm `BEGIN`, giành phiên bản, `flush`, `COMMIT` hoặc `ROLLBACK`;
- số lần thử lại.

Không ghi toàn bộ dữ liệu nhạy cảm của khách hàng. Một test qua một lần chỉ cho
thấy một lịch chạy; bộ điểm hẹn và các bất biến sau giao dịch mới làm bằng chứng
có thể lặp lại.

## 17. Ma trận thực nghiệm tối thiểu

| Trường hợp | Điều phải chứng minh |
| --- | --- |
| Hai ảnh chụp cũ thay toàn bộ | Tái hiện một thay đổi bị mất ở bản lỗi |
| Hai lệnh cùng phiên bản | Một thành công, một xung đột, phiên bản tăng một |
| Hai sản phẩm khác nhau | Vẫn tranh chấp cùng phiên bản toàn giỏ |
| Lỗi sau khi giành phiên bản | Dòng gốc và dòng con cùng hoàn tác |
| Giỏ đã checkout | Trả sai trạng thái, không nhầm phiên bản cũ |
| JPA `@Version` | Bên thua hoàn tác và câu SQL có điều kiện phiên bản |
| Lệnh đặt số lượng cũ | Không bị tự thử lại mù |
| Hai lệnh cộng an toàn | Cả hai chênh lệch xuất hiện đúng một lần |
| Mất phản hồi rồi gửi lại | `commandId` ngăn cộng lần hai |
| Hết thời gian chờ hoặc bế tắc | Được phân loại là lỗi kỹ thuật |
| Hai máy chủ | PostgreSQL vẫn chọn đúng một bên thắng |
| Mọi đường ghi | Nội dung đổi thì phiên bản giỏ cũng đổi |
