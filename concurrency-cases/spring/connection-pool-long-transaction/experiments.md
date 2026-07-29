# Bounded saturation experiments

## Mục tiêu

Bộ test phải chứng minh:

1. remote wait bên trong transaction giữ Hikari connections;
2. PostgreSQL lock waiter cũng giữ connection;
3. unrelated short query fail ở pool acquisition, không phải SQL execution;
4. split boundary giải phóng pool trong remote wait;
5. commit phase revalidate stale snapshot;
6. mọi wait/test cleanup đều bounded.

> **Nói ngắn gọn:** ép finite pool full bằng gates, đo active/pending connections,
> rồi assert committed business state sau cleanup.

## Test topology

```text
pool max = 2

broken:
  actor A -> Tx/conn-1 -> row lock -> remote gate
  actor B -> Tx/conn-2 -> row lock -> remote gate hoặc lock wait
  actor U -> waits for pool -> acquisition timeout

fixed:
  actor A/B -> snapshot Tx ends -> remote gate (no connection)
  actor U   -> short query succeeds
  release A/B -> short apply transactions
```

## PostgreSQL Testcontainers và Hikari test profile

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest(properties = {
    "spring.datasource.hikari.maximum-pool-size=2",
    "spring.datasource.hikari.minimum-idle=0",
    "spring.datasource.hikari.connection-timeout=400ms"
})
class LongTransactionPoolIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

`400ms` chỉ làm test fail-fast và deterministic; không phải production setting.
Không annotate test method bằng `@Transactional`.

Schema:

```sql
create table payment_order (
    id uuid primary key,
    customer_id uuid not null,
    amount bigint not null check (amount > 0),
    status varchar(32) not null,
    risk_reason varchar(80),
    version bigint not null
);
```

Mỗi test insert committed fixtures P-42/P-99 ở `RISK_PENDING`.

## Controlled remote gate

```java
final class RemoteScenario {
    private final CountDownLatch entered;
    private final CountDownLatch release = new CountDownLatch(1);

    RemoteScenario(int expectedCalls) {
        this.entered = new CountDownLatch(expectedCalls);
    }

    void enterAndWait() {
        entered.countDown();
        awaitOrFail(release, Duration.ofSeconds(5));
    }

    void awaitAllEntered() {
        awaitOrFail(entered, Duration.ofSeconds(5));
    }

    void releaseAll() {
        release.countDown();
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for remote gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

```java
@Component
@Primary
class ControlledRiskClient implements RiskClient {
    private final AtomicReference<RemoteScenario> current =
        new AtomicReference<>();
    private final AtomicInteger calls = new AtomicInteger();

    @Override
    public RiskDecision assess(
        RiskSubject subject,
        Duration timeout
    ) {
        calls.incrementAndGet();
        RemoteScenario scenario = current.get();
        if (scenario != null) {
            scenario.enterAndWait();
        }
        return RiskDecision.approved(subject);
    }
}
```

`finally` luôn release scenario trước khi shutdown actor executor.

## Pool probe

```java
@Component
final class PoolProbe {
    private final HikariDataSource hikari;

    PoolProbe(DataSource dataSource) throws SQLException {
        this.hikari = dataSource.unwrap(HikariDataSource.class);
    }

    int active() {
        return hikari.getHikariPoolMXBean().getActiveConnections();
    }

    int idle() {
        return hikari.getHikariPoolMXBean().getIdleConnections();
    }

    int pending() {
        return hikari.getHikariPoolMXBean()
            .getThreadsAwaitingConnection();
    }
}
```

Pool metrics là eventual observations. Dùng bounded Awaitility:

```java
await()
    .atMost(Duration.ofSeconds(2))
    .untilAsserted(() -> {
        assertThat(pool.active()).isEqualTo(2);
        assertThat(pool.idle()).isZero();
    });
```

Awaitility polling có deadline; gate vẫn là primary interleaving mechanism.

## Experiment 1 — Hai remote waits làm cạn pool

Dùng hai payment IDs khác nhau để loại row contention:

```java
@Test
void remoteWaitInsideTransactionsStarvesUnrelatedQuery()
        throws Exception {
    RemoteScenario scenario = riskClient.install(2);
    ExecutorService actors = Executors.newFixedThreadPool(3);

    try {
        Future<ApprovalResult> a = actors.submit(() ->
            broken.assessAndApprove(PAYMENT_42)
        );
        Future<ApprovalResult> b = actors.submit(() ->
            broken.assessAndApprove(PAYMENT_99)
        );

        scenario.awaitAllEntered();
        await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
            assertThat(pool.active()).isEqualTo(2);
            assertThat(pool.idle()).isZero();
        });

        Future<Throwable> unrelated = actors.submit(() ->
            catchThrowable(shortQuery::selectOne)
        );

        await().atMost(Duration.ofSeconds(2)).untilAsserted(() ->
            assertThat(pool.pending()).isEqualTo(1)
        );

        Throwable acquisitionFailure =
            unrelated.get(2, TimeUnit.SECONDS);

        assertThat(acquisitionFailure)
            .isInstanceOf(CannotGetJdbcConnectionException.class)
            .hasRootCauseInstanceOf(SQLTransientConnectionException.class);

        scenario.releaseAll();

        assertThat(a.get(5, TimeUnit.SECONDS).isApproved()).isTrue();
        assertThat(b.get(5, TimeUnit.SECONDS).isApproved()).isTrue();

        await().atMost(Duration.ofSeconds(2)).untilAsserted(() ->
            assertThat(pool.active()).isZero()
        );
    } finally {
        scenario.releaseAll();
        riskClient.clear();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

`shortQuery.selectOne()` không chạy SQL vì không mượn được connection. Assertion
root cause phân biệt pool acquisition failure với PostgreSQL statement timeout.

## Experiment 2 — Lock waiter chiếm connection

Cả A và B dùng P-42. A acquire lock rồi vào remote gate; B borrow connection thứ
hai và block ở `FOR UPDATE`.

Dùng một direct admin connection ngoài application pool chỉ cho diagnostics:

```java
long lockWaiterCount() {
    try (
        Connection connection = DriverManager.getConnection(
            POSTGRES.getJdbcUrl(),
            POSTGRES.getUsername(),
            POSTGRES.getPassword()
        );
        PreparedStatement statement = connection.prepareStatement(
            """
            select count(*)
            from pg_stat_activity
            where datname = current_database()
              and wait_event_type = 'Lock'
            """
        );
        ResultSet rows = statement.executeQuery()
    ) {
        rows.next();
        return rows.getLong(1);
    }
}
```

Test sequence:

```java
@Test
void rowLockWaiterConsumesSecondPoolConnection() throws Exception {
    RemoteScenario firstCall = riskClient.install(1);
    ExecutorService actors = Executors.newFixedThreadPool(3);

    try {
        Future<ApprovalResult> holder = actors.submit(() ->
            broken.assessAndApprove(PAYMENT_42)
        );
        firstCall.awaitAllEntered(); // conn-1 owns row lock

        Future<Throwable> waiter = actors.submit(() ->
            catchThrowable(() ->
                broken.assessAndApprove(PAYMENT_42)
            )
        );

        await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
            assertThat(pool.active()).isEqualTo(2);
            assertThat(lockWaiterCount()).isGreaterThanOrEqualTo(1);
            assertThat(riskClient.callCount()).isEqualTo(1);
        });

        Future<Throwable> unrelated = actors.submit(() ->
            catchThrowable(shortQuery::selectOne)
        );
        assertThat(unrelated.get(2, TimeUnit.SECONDS))
            .isInstanceOf(CannotGetJdbcConnectionException.class);

        firstCall.releaseAll();
        assertThat(holder.get(5, TimeUnit.SECONDS).isApproved()).isTrue();

        Throwable duplicateOutcome =
            waiter.get(5, TimeUnit.SECONDS);
        assertThat(duplicateOutcome)
            .isInstanceOf(IllegalStateException.class);
    } finally {
        firstCall.releaseAll();
        riskClient.clear();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Controlled client gates first call only; duplicate proceeds after holder commit,
then broken code performs unnecessary risk call and fails state validation. Main
assertion là one remote holder + one DB lock waiter đã full pool.

Admin connection không được dùng như production workaround; nó chỉ quan sát
application pool từ bên ngoài.

## Experiment 3 — Fixed remote waits không giữ connection

```java
@Test
void splitBoundaryLeavesPoolAvailableDuringRemoteWait()
        throws Exception {
    RemoteScenario scenario = riskClient.install(2);
    ExecutorService actors = Executors.newFixedThreadPool(3);

    try {
        Future<ApprovalResult> a = actors.submit(() ->
            fixed.assessAndApprove(
                PAYMENT_42,
                testDeadline()
            )
        );
        Future<ApprovalResult> b = actors.submit(() ->
            fixed.assessAndApprove(
                PAYMENT_99,
                testDeadline()
            )
        );

        scenario.awaitAllEntered();

        await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
            assertThat(pool.active()).isZero();
            assertThat(pool.pending()).isZero();
        });

        assertThat(shortQuery.selectOne()).isEqualTo(1);

        scenario.releaseAll();
        assertThat(a.get(5, TimeUnit.SECONDS).isApproved()).isTrue();
        assertThat(b.get(5, TimeUnit.SECONDS).isApproved()).isTrue();

        assertThat(inspector.status(PAYMENT_42))
            .isEqualTo(PaymentStatus.APPROVED);
        assertThat(inspector.status(PAYMENT_99))
            .isEqualTo(PaymentStatus.APPROVED);
    } finally {
        scenario.releaseAll();
        riskClient.clear();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Snapshot transactions đã end trước actors enter remote gate. Pool active `0` là
technical evidence; unrelated query success là business/service evidence.

## Experiment 4 — Revalidation từ chối stale decision

```java
@Test
void concurrentChangeMakesRemoteDecisionStale() throws Exception {
    RemoteScenario scenario = riskClient.install(1);
    ExecutorService actor = Executors.newSingleThreadExecutor();

    try {
        Future<ApprovalResult> assessment = actor.submit(() ->
            fixed.assessAndApprove(PAYMENT_42, testDeadline())
        );

        scenario.awaitAllEntered(); // snapshot version 12, no Tx open
        canceller.cancel(PAYMENT_42); // short independent Tx -> version 13
        scenario.releaseAll();

        ApprovalResult result =
            assessment.get(5, TimeUnit.SECONDS);

        assertThat(result.isStaleDecision()).isTrue();
        assertThat(inspector.status(PAYMENT_42))
            .isEqualTo(PaymentStatus.CANCELLED);
        assertThat(inspector.version(PAYMENT_42)).isEqualTo(13L);
    } finally {
        scenario.releaseAll();
        riskClient.clear();
        actor.shutdownNow();
        assertThat(actor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Test này bắt regression “move remote out of Tx nhưng quên revalidate”.

## Experiment 5 — Remote timeout xảy ra khi không có DB transaction

Test client ném controlled `RiskDependencyTimeoutException` sau bounded gate:

```java
@Test
void remoteTimeoutDoesNotOpenCommitTransaction() {
    assertThatThrownBy(() ->
        fixed.assessAndApprove(PAYMENT_42, expiredRemoteDeadline())
    ).isInstanceOf(RiskDependencyTimeoutException.class);

    assertThat(pool.active()).isZero();
    assertThat(inspector.status(PAYMENT_42))
        .isEqualTo(PaymentStatus.RISK_PENDING);
    assertThat(decisionWriter.invocationCount()).isZero();
}
```

Snapshot read transaction đã commit trước remote call; apply transaction không bắt
đầu khi dependency fails.

## Experiment 6 — Bounded lock timeout cleanup

Trong test profile, apply transaction có trusted test guardrail:

```sql
set local lock_timeout = '300ms';
```

Giữ P-42 lock bằng một independent transaction, gọi fixed writer và assert:

- exception/cause có PostgreSQL lock-timeout SQLSTATE phù hợp;
- apply transaction rollback;
- pool active trở về baseline;
- status/version không đổi;
- retry không chạy bên trong failed transaction.

Giá trị timeout là test fixture. Production classification phải dựa trên driver
cause/SQLSTATE và remaining deadline.

## Experiment 7 — Pool size không sửa long duration

Chạy parameterized test với pool capacities nhỏ khác nhau, luôn tạo đúng số remote
waiters bằng capacity. Với mỗi capacity:

```text
active == maximumPoolSize
unrelated borrower times out
transaction duration tracks remote gate duration
```

Không dùng test để tuyên bố một pool size tối ưu. Mục tiêu là chứng minh tăng finite
capacity chỉ tăng số long transactions trước saturation.

## Coverage matrix

| Scenario | Active connections trong remote wait | Lock waiter | Unrelated query |
| --- | --- | --- | --- |
| Broken, different rows | 2/2 | Không | Acquisition timeout |
| Broken, same row | 2/2 | Có | Acquisition timeout |
| Fixed split boundary | 0/2 | Không | Success |
| Fixed stale snapshot | 0 trong remote; short apply | Short | Stale/no-op |
| Remote timeout outside Tx | 0 sau snapshot | Không | Unaffected |
| Apply lock timeout | Bounded short Tx | Có | Pool recovers |

## Chống flaky

- Remote latches chứng minh actors đã qua DB query trước khi block.
- Mọi latch, future, Awaitility wait và executor termination có timeout.
- `finally` release remote gate trước executor shutdown.
- Test class chạy `SAME_THREAD`; controlled client là single-scenario.
- Test method không có outer transaction.
- Pool capacity/config chỉ áp dụng test context.
- Direct admin connection luôn đóng bằng try-with-resources.
- Fixtures dùng committed setup và IDs riêng.
- Assert cả pool state, exception layer và committed payment state.

## Production verification

Dashboard correlate cùng time window:

```text
Hikari active/max, pending, acquisition timeout
PostgreSQL xact age, idle-in-tx, lock waits/blockers
remote latency/timeouts, bulkhead active/rejected
request/executor active, queue, rejection
instance count and total configured connections
```

Diagnostic SQL:

```sql
select a.pid,
       a.application_name,
       a.state,
       a.wait_event_type,
       a.wait_event,
       now() - a.xact_start as transaction_age,
       pg_blocking_pids(a.pid) as blockers
from pg_stat_activity a
where a.datname = current_database()
  and a.xact_start is not null
order by a.xact_start;
```

Production alert ưu tiên rising transaction/connection usage duration và pending
borrowers. Pool timeout là late signal của cascade.
