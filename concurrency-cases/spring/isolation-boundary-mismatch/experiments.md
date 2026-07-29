# PostgreSQL integration experiments

## Mục tiêu

Bộ test phải chứng minh hai lớp:

1. **technical fact:** effective isolation của đúng reader transaction;
2. **business fact:** hai lần đọc trả `100/120` ở broken boundary và `100/100` ở
   fixed boundary.

Chỉ inspect annotation hoặc mock transaction manager không đủ. Test dùng
PostgreSQL thật qua Testcontainers vì snapshot semantics là database-specific.

> **Nói ngắn gọn:** điều phối writer commit đúng giữa hai SELECT, đo setting trên
> chính connection của reader, rồi assert outcome.

## Test topology

```text
test thread
  ├─ reader executor -> facade proxy -> Tx-R -> SELECT #1 -> gate -> SELECT #2
  └─ test thread     -> writer proxy -> Tx-W -> UPDATE -> COMMIT -> release gate
```

Test method không được annotated `@Transactional`. Nếu có, reader/writer có thể
join test-managed transaction hoặc cleanup rollback che mất commit thật.

## PostgreSQL container và schema

```java
@Testcontainers
@SpringBootTest
class IsolationBoundaryIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Schema tối thiểu:

```sql
create table product_price (
    product_id bigint primary key,
    price bigint not null
);
```

Mỗi test đưa database về committed state rõ ràng:

```sql
delete from product_price;
insert into product_price(product_id, price) values (1, 100);
```

Cleanup/setup chạy ngoài reader transaction. Nếu test suite chạy song song, dùng
ID riêng cho từng test hoặc disable parallel execution cho class này.

## Gate điều phối deterministic

Gate tạo partial order `first read < writer commit < second read`:

```java
@Component
final class SnapshotProbe {
    private final AtomicReference<Gate> current = new AtomicReference<>();
    private final AtomicReference<String> observedIsolation =
        new AtomicReference<>();

    Gate install() {
        Gate gate = new Gate(
            new CountDownLatch(1),
            new CountDownLatch(1)
        );
        if (!current.compareAndSet(null, gate)) {
            throw new IllegalStateException("A scenario is already active");
        }
        observedIsolation.set(null);
        return gate;
    }

    void afterFirstRead(JdbcTemplate jdbc) {
        Gate gate = requireGate();
        observedIsolation.set(jdbc.queryForObject(
            "select current_setting('transaction_isolation')",
            String.class
        ));
        gate.firstReadDone().countDown();
        awaitOrFail(gate.allowSecondRead(), Duration.ofSeconds(5));
    }

    String observedIsolation() {
        return Objects.requireNonNull(observedIsolation.get());
    }

    void clear(Gate expected) {
        if (!current.compareAndSet(expected, null)) {
            throw new IllegalStateException("Unexpected active scenario");
        }
    }

    private Gate requireGate() {
        return Objects.requireNonNull(
            current.get(),
            "No scenario installed"
        );
    }

    static void awaitOrFail(CountDownLatch latch, Duration timeout) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for scenario gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }

    record Gate(
        CountDownLatch firstReadDone,
        CountDownLatch allowSecondRead
    ) {
        void awaitFirstRead() {
            SnapshotProbe.awaitOrFail(firstReadDone, Duration.ofSeconds(5));
        }

        void releaseSecondRead() {
            allowSecondRead.countDown();
        }
    }
}
```

Mọi wait có timeout. `finally` luôn release gate để reader không bị treo khi writer
hoặc assertion ở test thread thất bại.

## Fixture services

Dùng JDBC query trực tiếp giúp hai SELECT thực sự chạm database, tránh việc JPA
first-level cache trả lại cùng managed entity và che anomaly:

```java
@Repository
class PriceQueries {
    private final JdbcTemplate jdbc;

    long getPrice(long productId) {
        return jdbc.queryForObject(
            "select price from product_price where product_id = ?",
            Long.class,
            productId
        );
    }

    long getCommittedPrice(long productId) {
        return getPrice(productId);
    }
}
```

Inner service mang declaration đang gây hiểu nhầm:

```java
@Service
class SnapshotQueryService {
    private final PriceQueries prices;
    private final SnapshotProbe probe;
    private final JdbcTemplate jdbc;

    @Transactional(isolation = Isolation.REPEATABLE_READ, readOnly = true)
    public PriceReport readTwice(long productId) {
        long first = prices.getPrice(productId);
        probe.afterFirstRead(jdbc);
        long second = prices.getPrice(productId);
        return new PriceReport(first, second);
    }
}
```

Hai facade chỉ khác creation-point isolation:

```java
@Service
class BrokenReportFacade {
    private final SnapshotQueryService snapshots;

    @Transactional
    public PriceReport generate(long productId) {
        return snapshots.readTwice(productId);
    }
}
```

```java
@Service
class FixedReportFacade {
    private final SnapshotQueryService snapshots;

    @Transactional(
        isolation = Isolation.REPEATABLE_READ,
        readOnly = true
    )
    public PriceReport generate(long productId) {
        return snapshots.readTwice(productId);
    }
}
```

Writer phải là proxied bean riêng để update và commit trước khi test release reader:

```java
@Service
class PriceWriter {
    private final JdbcTemplate jdbc;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void update(long productId, long price) {
        int changed = jdbc.update(
            "update product_price set price = ? where product_id = ?",
            price,
            productId
        );
        if (changed != 1) {
            throw new IllegalStateException("Expected exactly one product");
        }
    }
}
```

`REQUIRES_NEW` ở writer chỉ biểu đạt rằng Tx-W độc lập với test caller. Nó không
phải phương án sửa reader.

## Experiment 1 — Broken path thật sự là READ COMMITTED

```java
@Test
void innerRepeatableReadDoesNotUpgradeExistingDefaultTransaction()
        throws Exception {
    SnapshotProbe.Gate gate = probe.install();
    ExecutorService readerPool = Executors.newSingleThreadExecutor();

    try {
        Future<PriceReport> reader =
            readerPool.submit(() -> broken.generate(1L));

        gate.awaitFirstRead();
        writer.update(1L, 120L); // returns only after Tx-W commits
        gate.releaseSecondRead();

        PriceReport report = reader.get(5, TimeUnit.SECONDS);

        assertThat(probe.observedIsolation()).isEqualTo("read committed");
        assertThat(report.firstPrice()).isEqualTo(100L);
        assertThat(report.secondPrice()).isEqualTo(120L);
    } finally {
        gate.releaseSecondRead();
        probe.clear(gate);
        readerPool.shutdownNow();
        assertThat(readerPool.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Test này thất bại nếu implementation chỉ “tin” inner annotation. `100/120` là
business evidence rằng PostgreSQL tạo statement snapshot mới ở lần đọc thứ hai.

## Experiment 2 — Isolation trên outer boundary tạo stable snapshot

Reset giá về `100`, sau đó chạy cùng interleaving:

```java
@Test
void outerRepeatableReadKeepsBothReadsOnOneSnapshot() throws Exception {
    SnapshotProbe.Gate gate = probe.install();
    ExecutorService readerPool = Executors.newSingleThreadExecutor();

    try {
        Future<PriceReport> reader =
            readerPool.submit(() -> fixed.generate(1L));

        gate.awaitFirstRead();
        writer.update(1L, 120L);
        gate.releaseSecondRead();

        PriceReport report = reader.get(5, TimeUnit.SECONDS);

        assertThat(probe.observedIsolation()).isEqualTo("repeatable read");
        assertThat(report.firstPrice()).isEqualTo(100L);
        assertThat(report.secondPrice()).isEqualTo(100L);
        assertThat(prices.getCommittedPrice(1L)).isEqualTo(120L);
    } finally {
        gate.releaseSecondRead();
        probe.clear(gate);
        readerPool.shutdownNow();
        assertThat(readerPool.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Assertion cuối chạy sau khi reader hoàn tất, trong transaction/query độc lập. Nó
chứng minh `100/100` không phải vì writer chưa commit: committed state thật đã là
`120`, chỉ Tx-R tiếp tục thấy snapshot cũ.

## Experiment 3 — Validation biến mismatch thành fail-fast

Dùng một test application context riêng với manager bật validation:

```java
@TestConfiguration
class StrictTransactionTestConfiguration {

    @Bean
    @Primary
    JpaTransactionManager strictTransactionManager(
        EntityManagerFactory entityManagerFactory
    ) {
        JpaTransactionManager manager =
            new JpaTransactionManager(entityManagerFactory);
        manager.setValidateExistingTransaction(true);
        return manager;
    }
}
```

Test phải nằm ở class/context riêng để không mutate singleton transaction manager
trong lúc các test khác chạy:

```java
@Test
void incompatibleInnerIsolationIsRejectedBeforeQueryRuns() {
    assertThatThrownBy(() -> broken.generate(1L))
        .isInstanceOf(IllegalTransactionStateException.class)
        .hasMessageContaining("isolation");

    assertThat(queryCounter.value()).isZero();
}
```

`queryCounter` tăng ở đầu `readTwice()`. Nếu transaction interceptor từ chối join
trước method body, counter vẫn bằng zero. Tùy Spring version, message chi tiết có
thể thay đổi; type và việc body không chạy là assertions bền hơn.

## Experiment 4 — Self-invocation không tạo intended boundary

Một fixture riêng ghi nhận trạng thái transaction trong self-invoked helper:

```java
@Service
class SelfInvokingReport {

    PriceReport generate(long productId) {
        return this.readWithStableSnapshot(productId);
    }

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    PriceReport readWithStableSnapshot(long productId) {
        assertThat(TransactionSynchronizationManager
            .isActualTransactionActive()).isFalse();
        // probe two repository calls in the full test
        return null;
    }
}
```

Không đặt assertion production như ví dụ trên; trong fixture thật, trả trạng thái
qua probe rồi assert từ test. Experiment này bổ sung SPR-001 và không thay thế hai
PostgreSQL outcome tests.

## Experiment 5 — DEFAULT phải được đo theo environment

Tạo một public probe service:

```java
@Transactional
public String defaultIsolation() {
    return jdbc.queryForObject(
        "select current_setting('transaction_isolation')",
        String.class
    );
}
```

Assert container profile hiện tại trả `read committed`. Nếu production cấu hình
default khác, tạo profile/container config tương ứng và assert explicit. Không đổi
global/session default giữa các concurrent tests vì connection pool có thể làm
kết quả phụ thuộc connection được mượn.

## Coverage matrix

| Scenario | Expected effective isolation | Expected outcome |
| --- | --- | --- |
| Outer DEFAULT, inner REQUIRED RR | `read committed` | `100/120` |
| Outer RR, inner REQUIRED RR | `repeatable read` | `100/100` |
| Outer RC, inner RR, validation on | Không chạy query | Fail-fast |
| No outer Tx, proxied inner RR | `repeatable read` | `100/100` |
| No outer Tx, self-invoked inner RR | Không có intended service Tx | Boundary test fails |
| Outer RC, inner REQUIRES_NEW RR | Inner `repeatable read` | Independent `100/100` |

## Chống flaky

- Mọi latch, future và executor termination đều có timeout.
- `finally` release reader trước khi shutdown executor.
- Writer method chỉ return sau commit.
- Không dùng delay theo thời gian để đoán interleaving.
- Không chạy test method trong transaction.
- Không dùng H2 để đại diện PostgreSQL MVCC.
- Không reuse một gate giữa hai tests.
- Dùng committed fixture setup và product ID riêng khi chạy parallel.
- Nếu future timeout, log gate state và effective isolation đã quan sát.

## Assertions bắt buộc

Một test chỉ assert `read committed` có thể bỏ sót query/cache behavior. Một test
chỉ assert `100/120` có thể pass vì boundary khác tình cờ bị tách. Bộ test hoàn
chỉnh phải assert:

```text
broken: effective = read committed, values = 100/120
fixed:  effective = repeatable read, values = 100/100
after:  independently committed value = 120
strict: incompatible inner boundary rejected before method body
```

Đây là contract đủ mạnh để bắt regression khi annotation, propagation, datasource
default hoặc transaction manager configuration thay đổi.
