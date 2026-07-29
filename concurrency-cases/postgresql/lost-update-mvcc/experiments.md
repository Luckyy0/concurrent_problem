# Deterministic PostgreSQL experiments

## Mục tiêu

Tests phải chứng minh:

1. A và B dùng independent transactions;
2. cả hai SELECT cùng thấy `10`;
3. A commit `13` trước B commit stale `14`;
4. cả calls return success nhưng final invariant `17` bị phá;
5. atomic delta compose thành `17`;
6. alternative detectors tạo block/conflict thay vì silent overwrite.

> **Nói ngắn gọn:** ép cùng-read/different-write order bằng latches, rồi đọc final
> state bằng transaction mới.

## PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class LostUpdateIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Test method không annotated `@Transactional`. H2 không phải evidence cho PostgreSQL
MVCC/lock behavior.

Schema broken variant:

```sql
create table job_progress (
    job_id uuid primary key,
    completed_units integer not null,
    total_units integer not null,
    constraint ck_job_progress_range check (
        total_units >= 0
        and completed_units >= 0
        and completed_units <= total_units
    )
);
```

Fixture setup commit:

```sql
insert into job_progress(job_id, completed_units, total_units)
values (:jobId, 10, 100);
```

## Gate ép both reads trước first commit

```java
final class LostUpdateGate {
    private final UUID actorA;
    private final UUID actorB;
    private final CountDownLatch bothLoaded = new CountDownLatch(2);
    private final CountDownLatch allowA = new CountDownLatch(1);
    private final CountDownLatch allowB = new CountDownLatch(1);
    private final ConcurrentMap<UUID, Integer> observed =
        new ConcurrentHashMap<>();

    LostUpdateGate(UUID actorA, UUID actorB) {
        this.actorA = actorA;
        this.actorB = actorB;
    }

    void afterLoad(UUID actorId, int value) {
        observed.put(actorId, value);
        bothLoaded.countDown();
        awaitOrFail(bothLoaded, Duration.ofSeconds(5));

        if (actorId.equals(actorA)) {
            awaitOrFail(allowA, Duration.ofSeconds(5));
        } else if (actorId.equals(actorB)) {
            awaitOrFail(allowB, Duration.ofSeconds(5));
        } else {
            throw new AssertionError("Unexpected actor");
        }
    }

    void awaitBothLoaded() {
        awaitOrFail(bothLoaded, Duration.ofSeconds(5));
    }

    void releaseA() {
        allowA.countDown();
    }

    void releaseB() {
        allowB.countDown();
    }

    void releaseAll() {
        allowA.countDown();
        allowB.countDown();
    }

    int observedBy(UUID actorId) {
        return Objects.requireNonNull(observed.get(actorId));
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for test gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

Test fixture service gọi `gate.afterLoad(actorId, completedUnits)` sau repository
SELECT và trước entity mutation. Gate là test-only component.

## Transaction observation

Trong đúng service transaction, probe lấy:

```sql
select pg_current_xact_id()::text,
       current_setting('transaction_isolation');
```

Record:

```java
public record TransactionObservation(
    UUID actorId,
    String transactionId,
    String isolation,
    AtomicInteger completionStatus
) {}
```

Đăng ký `TransactionSynchronization.afterCompletion` để assert cả A/B
`STATUS_COMMITTED`.

## Experiment 1 — Broken dirty checking mất A delta

```java
@Test
void staleAbsoluteWriteSilentlyOverwritesCommittedDelta()
        throws Exception {
    UUID actorA = UUID.randomUUID();
    UUID actorB = UUID.randomUUID();
    LostUpdateGate gate = probe.install(actorA, actorB);
    ExecutorService actors = Executors.newFixedThreadPool(2);

    try {
        Future<ProgressResult> a = actors.submit(() ->
            broken.addCompletedUnits(actorA, JOB_ID, 3)
        );
        Future<ProgressResult> b = actors.submit(() ->
            broken.addCompletedUnits(actorB, JOB_ID, 4)
        );

        gate.awaitBothLoaded();
        assertThat(gate.observedBy(actorA)).isEqualTo(10);
        assertThat(gate.observedBy(actorB)).isEqualTo(10);

        gate.releaseA();
        ProgressResult resultA = a.get(5, TimeUnit.SECONDS);
        assertThat(resultA.after()).isEqualTo(13);

        gate.releaseB();
        ProgressResult resultB = b.get(5, TimeUnit.SECONDS);
        assertThat(resultB.after()).isEqualTo(14);

        TransactionObservation txA = probe.transaction(actorA);
        TransactionObservation txB = probe.transaction(actorB);
        assertThat(txA.transactionId())
            .isNotEqualTo(txB.transactionId());
        assertThat(txA.isolation()).isEqualTo("read committed");
        assertThat(txB.isolation()).isEqualTo("read committed");
        assertThat(txA.completionStatus().get())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);
        assertThat(txB.completionStatus().get())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);

        JobProgressSnapshot finalState = inspector.read(JOB_ID);
        assertThat(finalState.completedUnits()).isEqualTo(14);
        assertThat(finalState.completedUnits())
            .isNotEqualTo(10 + 3 + 4);
    } finally {
        gate.releaseAll();
        probe.reset();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

`a.get(timeout)` chỉ return sau A transaction interceptor commit. Vì thế B release
sau đó đảm bảo B stale write là last commit.

## Experiment 2 — Atomic delta compose cả hai updates

Gate đơn giản cho cả actors ready trước UPDATE:

```java
final class StartGate {
    private final CountDownLatch ready = new CountDownLatch(2);
    private final CountDownLatch start = new CountDownLatch(1);

    void readyAndAwaitStart() {
        ready.countDown();
        awaitOrFail(start, Duration.ofSeconds(5));
    }

    void awaitReady() {
        awaitOrFail(ready, Duration.ofSeconds(5));
    }

    void release() {
        start.countDown();
    }
}
```

Test:

```java
@Test
void atomicDeltaPreservesBothConcurrentCommands() throws Exception {
    StartGate gate = atomicProbe.install(2);
    ExecutorService actors = Executors.newFixedThreadPool(2);

    try {
        Future<ProgressApplyResult> a = actors.submit(() -> {
            gate.readyAndAwaitStart();
            return atomic.addCompletedUnits(JOB_ID, 3);
        });
        Future<ProgressApplyResult> b = actors.submit(() -> {
            gate.readyAndAwaitStart();
            return atomic.addCompletedUnits(JOB_ID, 4);
        });

        gate.awaitReady();
        gate.release();

        assertThat(a.get(5, TimeUnit.SECONDS).isApplied()).isTrue();
        assertThat(b.get(5, TimeUnit.SECONDS).isApplied()).isTrue();

        JobProgressSnapshot finalState = inspector.read(JOB_ID);
        assertThat(finalState.completedUnits()).isEqualTo(17);
    } finally {
        gate.release();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Test không cần quyết định actor nào acquire row lock trước; phép cộng compose theo
cả hai commit orders.

## Experiment 3 — Conditional cap chỉ accept valid winner

Initial `completed=95`, `total=100`; concurrent deltas `3` và `4`:

```java
@Test
void atomicPredicatePreventsAcceptedTotalFromCrossingCap()
        throws Exception {
    ConcurrentApplyResults results =
        runAtomicDeltasFromSameStart(95, 100, 3, 4);

    assertThat(results.appliedCount()).isEqualTo(1);
    assertThat(results.notAppliedCount()).isEqualTo(1);

    int finalValue = inspector.read(JOB_ID).completedUnits();
    assertThat(finalValue).isIn(98, 99);
    assertThat(finalValue).isLessThanOrEqualTo(100);
}
```

PostgreSQL re-evaluate predicate trên current row sau lock wait. Affected row count
map thành applied/not-applied; không actor nào được báo applied khi delta biến mất.

## Experiment 4 — Optimistic version tạo visible conflict

Schema thêm:

```sql
alter table job_progress
add column version bigint not null default 0;
```

Dùng same `LostUpdateGate` để A/B load version `7`. A commit first. Khi B flush:

```java
Throwable conflict = catchThrowable(() ->
    b.get(5, TimeUnit.SECONDS)
);

assertThat(rootCause(conflict))
    .isInstanceOfAny(
        OptimisticLockException.class,
        StaleObjectStateException.class
    );
```

Spring-facing assertion nên chấp nhận/ưu tiên
`ObjectOptimisticLockingFailureException` tại service proxy. Sau B rollback:

```text
completed=13, version=8
```

Retry B qua separate coordinator/transaction reloads `13/version 8`, commits:

```text
completed=17, version=9
```

Probe assert B retry transaction ID và Hibernate session identity khác failed
attempt. Không retry trong rollback-only transaction.

## Experiment 5 — Pessimistic read serializes before calculation

A gọi `findByIdForUpdate`, load `10`, rồi dừng ở gate. B call bắt đầu nhưng chưa
thể return từ locked repository query.

Dùng direct admin connection/Awaitility để assert B có
`wait_event_type = 'Lock'`. Sau release A:

```text
A commits 13
B lock query returns current 13
B calculates/commits 17
```

Assertions:

```java
assertThat(probe.loadedValue(actorA)).isEqualTo(10);
assertThat(probe.loadedValue(actorB)).isEqualTo(13);
assertThat(inspector.read(JOB_ID).completedUnits()).isEqualTo(17);
```

Mọi futures/lock waits có timeout; lock holder luôn release trong `finally`.

## Experiment 6 — PostgreSQL `REPEATABLE READ` đổi silent loss thành abort

Hai transactions dùng:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

Ép cùng snapshot và A commit trước B UPDATE. Expected:

```text
one commit
one rollback with serialization failure
SQLSTATE = 40001
no silent two-success outcome
```

Sau abort, retry toàn B unit trong transaction mới mới có thể đạt final `17`.
Exact Hibernate/Spring wrapper có thể đổi; SQLSTATE/root cause và committed
business state là assertions bền hơn message text.

## Experiment 7 — Multi-instance equivalent

Test executor threads đã dùng independent transactions, nhưng có thể chạy cùng
scenario qua hai application contexts/containers trỏ cùng PostgreSQL. Kết quả
broken vẫn là `14`; JVM-local `synchronized` trên mỗi context không coordinate.

Database solutions cho cùng outcome ở single hoặc multi-instance deployment.

## Inspector

```java
@Service
class JobProgressInspector {
    private final JdbcTemplate jdbc;

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    JobProgressSnapshot read(UUID jobId) {
        return jdbc.queryForObject(
            """
            select completed_units, total_units
            from job_progress
            where job_id = ?
            """,
            (rs, rowNum) -> new JobProgressSnapshot(
                rs.getInt(1),
                rs.getInt(2)
            ),
            jobId
        );
    }
}
```

Inspector chạy sau futures và dùng transaction/persistence context mới.

## Coverage matrix

| Scenario | Reads | Conflict behavior | Final |
| --- | --- | --- | --- |
| Broken JPA | A=10, B=10 | None; both commit | 14 |
| Atomic delta | DB current row | Wait then compose | 17 |
| Conditional cap | DB predicate | One affected row 0 | 98 or 99 |
| `@Version` no retry | A/B v7 | B rollback | 13/v8 |
| `@Version` fresh retry | B reload v8 | Retry commit | 17/v9 |
| `FOR UPDATE` | A=10, B=13 | B blocks before read | 17 |
| Repeatable read | Same snapshot | One SQLSTATE 40001 | No silent loss |

## Chống flaky

- Gate tạo order `both read 10 < A commit < B stale commit`.
- Mọi latch, future, Awaitility wait và executor termination có timeout.
- `finally` release gates trước shutdown.
- Test class chạy `SAME_THREAD`; probes là single-scenario.
- Test method không có outer transaction.
- Fixture setup và inspector dùng committed independent transactions.
- Không assert exact winner cho commutative atomic test.
- Không dùng H2 để đại diện PostgreSQL MVCC.
- Assert final invariant và completion, không chỉ exceptions.

## Production verification

Reconciliation nếu có durable commands:

```sql
select p.job_id,
       p.completed_units,
       coalesce(sum(c.delta), 0) as accepted_delta
from job_progress p
left join progress_command c on c.job_id = p.job_id
where p.job_id = :jobId
group by p.job_id, p.completed_units;
```

So sánh projection với known initial/base plus accepted distinct deltas theo domain
model. Theo dõi affected-row zero, optimistic/serialization conflicts, hot keys và
effective isolation. Silent lost update không tạo exception metric riêng.
