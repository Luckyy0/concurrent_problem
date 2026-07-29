# Các thí nghiệm optimistic version conflict

## Mục tiêu

1. Chứng minh entity không version làm cả hai edits success và lost update.
2. Ép hai transactions load cùng version rồi cho một actor commit trước.
3. Assert loser affected rows `0`/optimistic exception và rollback.
4. Kiểm tra final price/version, không chỉ exception type.
5. Xác minh generated SQL có version predicate.
6. Kiểm tra stale HTTP/client expected version.

> **Nói ngắn gọn:** deterministic gate khóa đúng stale window; load test ngẫu
> nhiên không thay thế được proof về version predicate.

## PostgreSQL Testcontainers

```java
@Testcontainers
@SpringBootTest
class OptimisticOfferIT {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    private OfferAttemptService attempts;

    @Autowired
    private JdbcTemplate jdbc;

    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    @BeforeEach
    void seed() {
        jdbc.update("delete from product_offer");
        jdbc.update("""
                insert into product_offer(offer_id, price, title, version)
                values (42, 100.00, 'Launch offer', 7)
                """);
    }

    @AfterEach
    void shutdown() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

JUnit method không dùng `@Transactional`; mỗi actor phải commit/rollback thật.

## Gate sau load

```java
@TestConfiguration
class GateConfiguration {

    @Bean
    @Primary
    OfferEditGate offerEditGate() {
        return new TwoEditorGate();
    }
}

final class TwoEditorGate implements OfferEditGate {

    private final CountDownLatch bothLoaded = new CountDownLatch(2);
    private final CountDownLatch allowA = new CountDownLatch(1);
    private final CountDownLatch aCommitted = new CountDownLatch(1);

    @Override
    public void afterLoad(String actor, long version) {
        bothLoaded.countDown();
        await(bothLoaded, "both editors loaded v" + version);
        if ("A".equals(actor)) {
            await(allowA, "allow editor A");
        } else {
            await(aCommitted, "A committed before B flush");
        }
    }

    void allowA() {
        allowA.countDown();
    }

    void aCommitted() {
        aCommitted.countDown();
    }
}

static void await(CountDownLatch latch, String description) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Timed out: " + description);
        }
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Interrupted: " + description, interrupted);
    }
}
```

Production gate là no-op. Test gọi `aCommitted()` chỉ sau Future A return, nghĩa
transaction proxy đã commit.

## Attempt service qua Spring proxy

```java
@Service
public class OfferAttemptService {

    private final ProductOfferRepository offers;
    private final OfferEditGate gate;

    public OfferAttemptService(
            ProductOfferRepository offers,
            OfferEditGate gate
    ) {
        this.offers = offers;
        this.gate = gate;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public long edit(String actor, BigDecimal price) {
        ProductOffer offer = offers.findById(42L).orElseThrow();
        gate.afterLoad(actor, offer.version());
        offer.changePrice(price);
        offers.flush();
        return offer.version();
    }
}
```

## Thí nghiệm 1 — Baseline không version bị overwrite

Dùng mapping/schema variant không có version:

```java
Future<Void> a = executor.submit(() -> {
    brokenAttempts.edit("A", new BigDecimal("90.00"));
    return null;
});
Future<Void> b = executor.submit(() -> {
    brokenAttempts.edit("B", new BigDecimal("80.00"));
    return null;
});

gate.allowA();
a.get(5, TimeUnit.SECONDS);
gate.aCommitted();
b.get(5, TimeUnit.SECONDS);

assertThat(price()).isEqualByComparingTo("80.00");
assertThat(successfulCalls()).isEqualTo(2);
```

Final value alone không cho biết A bị mất; test còn assert cả hai calls báo
success.

## Thí nghiệm 2 — `@Version` tạo visible conflict

```java
@Test
void secondEditorCannotOverwriteCommittedVersion() throws Exception {
    Future<Long> a = executor.submit(() ->
            attempts.edit("A", new BigDecimal("90.00"))
    );
    Future<Throwable> b = executor.submit(() -> {
        try {
            attempts.edit("B", new BigDecimal("80.00"));
            return null;
        } catch (Throwable failure) {
            return failure;
        }
    });

    gate.allowA();
    assertThat(a.get(5, TimeUnit.SECONDS)).isEqualTo(8);
    gate.aCommitted();

    Throwable loser = b.get(5, TimeUnit.SECONDS);
    assertThat(loser)
            .isInstanceOf(ObjectOptimisticLockingFailureException.class);
    assertThat(price()).isEqualByComparingTo("90.00");
    assertThat(version()).isEqualTo(8);
}
```

Nếu Spring wrapper khác theo configuration, assert cause chain chứa
`OptimisticLockException` và SQL inspector affected-row outcome; không match
message text.

## Thí nghiệm 3 — Failed transaction không được reuse

Test probe ghi transaction ID/persistence-context identity. Sau loser conflict,
gọi query trong same attempt catch sẽ thấy rollback-only/failure; solution không
catch ở đó.

Một separate call sau rollback load `v8` thành công:

```java
OfferView reloaded = readService.get(42);
assertThat(reloaded.price()).isEqualByComparingTo("90.00");
assertThat(reloaded.version()).isEqualTo(8);
assertThat(attemptProbe.transactionIds()).doesNotHaveDuplicates();
```

Case này không auto-apply B edit; nó chỉ chứng minh recovery read có boundary mới.

## Thí nghiệm 4 — Stale client version bị reject trước mutation

```java
ChangeOfferPrice stale = new ChangeOfferPrice(
        42,
        7,
        new BigDecimal("80.00"),
        UUID.randomUUID()
);

assertThatThrownBy(() -> editor.changePrice(stale))
        .isInstanceOf(StaleOfferEditException.class);
assertThat(price()).isEqualByComparingTo("90.00");
assertThat(version()).isEqualTo(8);
```

Test controller/HTTP adapter riêng assert `If-Match: "7"` map thành `412` và
response không báo success.

## Thí nghiệm 5 — SQL inspection

Dùng Hibernate `StatementInspector` chỉ trong test để capture normalized SQL:

```java
assertThat(sqlCapture.updatesFor("product_offer"))
        .anySatisfy(sql -> {
            assertThat(sql).contains("version=?");
            assertThat(sql).contains("where offer_id=?");
            assertThat(sql).contains("and version=?");
        });
```

Đừng assert toàn chuỗi whitespace/alias/order cứng vì Hibernate version có thể đổi
format. Mục tiêu là version increment và expected-version predicate.

## Thí nghiệm 6 — Native/bulk writer contract

Chạy native CAS:

```sql
update product_offer
set price = 95.00,
    version = version + 1
where offer_id = 42
  and version = 7;
```

Assert affected `1`, version `8`; lần lặp expected `7` affected `0`. Sau native
update, clear persistence context trước read assertion để không lấy cached entity.

Một negative test chạy unsafe bulk UPDATE không tăng version, rồi chứng minh JPA
writer không thấy revision thay đổi. Test này bảo vệ migration/batch review.

## Thí nghiệm 7 — Multi-instance equivalent

Hai transactions/connections đủ mô phỏng hai pods vì conflict boundary là
PostgreSQL. Không dùng shared Java gate làm lock; gate chỉ điều phối test. Chạy
service qua hai application contexts là smoke test bổ sung, không thay SQL
predicate assertion.

## Thí nghiệm 8 — Delete cạnh tranh

A load v7; B delete/commit; A flush affected `0`. Assert optimistic/not-found
outcome theo mapping và final row absent. Không retry edit để recreate deleted
offer.

## Ma trận bao phủ

| Thí nghiệm | Winner | Loser | Final |
| --- | --- | --- | --- |
| No version | A và B success | Không có signal | `80`, A lost |
| `@Version` | A affected `1` | B affected `0`/rollback | `90/v8` |
| Stale request | Current v8 | expected v7 rejected | `90/v8` |
| Native CAS | expected v7 → `1` | stale v7 → `0` | version tăng một lần |
| Delete race | Delete commit | edit conflict | Row absent |

## Chống flaky

- Barrier sau cả hai loads; B chỉ flush sau A Future return.
- Mọi latch/Future có timeout; không dùng fixed sleep.
- Executor shutdown và interrupt status được xử lý.
- PostgreSQL Testcontainers, không H2.
- Reset data ngoài actor transactions.
- Khi timeout, thu thread dump, `pg_stat_activity` và SQL capture.

## Xác minh trong production

Theo dõi optimistic conflict theo use case, HTTP `409/412`, current/expected
version trong sampled logs, transaction duration và hot aggregates. Alert khi
unsafe native/bulk writer không tăng version, conflict rate tăng đột biến hoặc
controller trả success trước commit.
