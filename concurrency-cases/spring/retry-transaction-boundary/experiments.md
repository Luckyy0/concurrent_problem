# Deterministic optimistic-conflict experiments

## Mục tiêu

Bộ test phải chứng minh:

1. hai actors load cùng version;
2. loser nhận real optimistic conflict từ PostgreSQL/Hibernate;
3. failed attempt rollback trước retry;
4. retry dùng transaction và persistence context mới;
5. retry reload version mới và revalidate business rule;
6. final committed inventory/reservation invariant đúng.

Chỉ assert `attemptCount == 2` không đủ; hai iterations có thể cùng nằm trong một
doomed transaction.

> **Nói ngắn gọn:** quan sát `version + transaction ID + completion status +
> business state`, không chỉ đếm loop.

## Test topology

```text
test thread
  ├─ actor A -> retry coordinator -> Tx-A1 -> load v7 -> commit v8
  └─ actor B -> retry coordinator -> Tx-B1 -> load v7 -> conflict/rollback
                                      Tx-B2 -> load v8 -> commit v9
```

Test method không annotated `@Transactional`. Mỗi actor gọi Spring-managed proxy
trên executor thread riêng.

## PostgreSQL Testcontainers và schema

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class RetryTransactionBoundaryIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Schema:

```sql
create table inventory_item (
    sku varchar(80) primary key,
    available integer not null check (available >= 0),
    version bigint not null
);

create table reservation_record (
    id uuid primary key,
    command_id uuid not null,
    sku varchar(80) not null references inventory_item(sku),
    quantity integer not null check (quantity > 0),
    constraint uk_reservation_command unique (command_id)
);
```

Fixture setup commit:

```sql
insert into inventory_item(sku, available, version)
values ('BOOK-42', 2, 7);
```

## Race gate

Gate ép cả hai first attempts load version 7, sau đó chỉ cho winner flush:

```java
final class OptimisticRaceGate {
    private final UUID winnerCommand;
    private final UUID loserCommand;
    private final CountDownLatch bothLoaded = new CountDownLatch(2);
    private final CountDownLatch allowWinner = new CountDownLatch(1);
    private final CountDownLatch winnerCommitted = new CountDownLatch(1);

    OptimisticRaceGate(UUID winnerCommand, UUID loserCommand) {
        this.winnerCommand = winnerCommand;
        this.loserCommand = loserCommand;
    }

    void afterLoad(UUID commandId, int attempt, long version) {
        if (attempt != 1) {
            return;
        }
        if (version != 7L) {
            throw new AssertionError(
                "First attempt must load version 7, got " + version
            );
        }

        bothLoaded.countDown();
        awaitOrFail(bothLoaded, Duration.ofSeconds(5));

        if (commandId.equals(winnerCommand)) {
            awaitOrFail(allowWinner, Duration.ofSeconds(5));
        } else if (commandId.equals(loserCommand)) {
            awaitOrFail(winnerCommitted, Duration.ofSeconds(5));
        } else {
            throw new AssertionError("Unexpected command");
        }
    }

    void awaitBothLoaded() {
        awaitOrFail(bothLoaded, Duration.ofSeconds(5));
    }

    void releaseWinner() {
        allowWinner.countDown();
    }

    void signalWinnerCommitted() {
        winnerCommitted.countDown();
    }

    void releaseAll() {
        allowWinner.countDown();
        winnerCommitted.countDown();
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for race gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

Gate không dùng timing guess. B first flush chỉ được phép sau A future return, tức
Tx-A commit đã hoàn tất.

## Attempt observation

Test-only probe ghi transaction ID, actual Hibernate session identity, loaded
version và completion:

```java
public record AttemptObservation(
    UUID commandId,
    int attempt,
    String transactionId,
    int sessionIdentity,
    long loadedVersion,
    AtomicInteger completionStatus
) {}
```

```java
@Component
final class AttemptProbe {
    private final JdbcTemplate jdbc;
    private final EntityManager entityManager;
    private final ConcurrentMap<UUID, AtomicInteger> counters =
        new ConcurrentHashMap<>();
    private final ConcurrentMap<UUID, List<AttemptObservation>> observations =
        new ConcurrentHashMap<>();
    private final AtomicReference<OptimisticRaceGate> gate =
        new AtomicReference<>();

    AttemptObservation afterLoad(UUID commandId, long loadedVersion) {
        int attempt = counters
            .computeIfAbsent(commandId, ignored -> new AtomicInteger())
            .incrementAndGet();

        String transactionId = jdbc.queryForObject(
            "select pg_current_xact_id()::text",
            String.class
        );
        int sessionIdentity = System.identityHashCode(
            entityManager.unwrap(org.hibernate.Session.class)
        );
        AtomicInteger completion =
            new AtomicInteger(Integer.MIN_VALUE);

        AttemptObservation observation = new AttemptObservation(
            commandId,
            attempt,
            transactionId,
            sessionIdentity,
            loadedVersion,
            completion
        );
        observations
            .computeIfAbsent(
                commandId,
                ignored -> new CopyOnWriteArrayList<>()
            )
            .add(observation);

        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void afterCompletion(int status) {
                    completion.set(status);
                }
            }
        );

        OptimisticRaceGate activeGate = gate.get();
        if (activeGate != null) {
            activeGate.afterLoad(commandId, attempt, loadedVersion);
        }
        return observation;
    }
}
```

Production code không cần probe. Test fixture worker gọi nó sau repository load và
trước mutation.

## Experiment 1 — Fixed pipeline tạo clean retry

```java
@Test
void optimisticLoserRetriesInNewTransactionAndReloads()
        throws Exception {
    UUID commandA = UUID.randomUUID();
    UUID commandB = UUID.randomUUID();
    OptimisticRaceGate gate =
        probe.installGate(commandA, commandB);
    ExecutorService actors = Executors.newFixedThreadPool(2);

    try {
        Future<ReservationResult> winner = actors.submit(() ->
            coordinator.reserve(commandA, "BOOK-42", 1)
        );
        Future<ReservationResult> loser = actors.submit(() ->
            coordinator.reserve(commandB, "BOOK-42", 1)
        );

        gate.awaitBothLoaded();
        gate.releaseWinner();

        ReservationResult winnerResult =
            winner.get(5, TimeUnit.SECONDS);
        gate.signalWinnerCommitted();

        ReservationResult loserResult =
            loser.get(5, TimeUnit.SECONDS);

        assertThat(winnerResult.isAccepted()).isTrue();
        assertThat(loserResult.isAccepted()).isTrue();

        List<AttemptObservation> b =
            probe.observations(commandB);
        assertThat(b).hasSize(2);
        assertThat(b.get(0).loadedVersion()).isEqualTo(7L);
        assertThat(b.get(0).completionStatus().get())
            .isEqualTo(TransactionSynchronization.STATUS_ROLLED_BACK);
        assertThat(b.get(1).loadedVersion()).isEqualTo(8L);
        assertThat(b.get(1).completionStatus().get())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);
        assertThat(b.get(1).transactionId())
            .isNotEqualTo(b.get(0).transactionId());
        assertThat(b.get(1).sessionIdentity())
            .isNotEqualTo(b.get(0).sessionIdentity());

        InventorySnapshot finalState =
            inspector.read("BOOK-42");
        assertThat(finalState.available()).isZero();
        assertThat(finalState.version()).isEqualTo(9L);
        assertThat(finalState.reservationCount()).isEqualTo(2L);
        assertThat(inspector.commandCount(commandA)).isEqualTo(1L);
        assertThat(inspector.commandCount(commandB)).isEqualTo(1L);
    } finally {
        gate.releaseAll();
        probe.reset();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Assertion completion của B attempt 1 được đọc sau toàn operation. Một probe mạnh
hơn có thể record event sequence và assert `ROLLBACK(B1) < BEGIN(B2)`.

## Vì sao A future phải hoàn tất trước signal?

Nếu signal ngay sau A flush nhưng trước commit, B UPDATE có thể block trên row lock.
Test vẫn có thể đúng nhưng khó phân biệt lock wait với retry boundary. Chờ
`winner.get(timeout)` xác nhận transaction interceptor đã commit trước khi B flush.

## Experiment 2 — Broken loop reuse transaction

Chạy A bằng fixed worker và B bằng broken service có instrument ở đầu mỗi loop:

```java
@Test
void localRetryLoopCannotRecoverFailedTransaction() throws Exception {
    // same gate: both load v7, A commits, then B flushes stale update
    BrokenOutcome outcome = runWinnerAndBrokenLoser();

    assertThat(outcome.firstConflictObserved()).isTrue();
    assertThat(outcome.rollbackOnlyInsideCatch()).isTrue();
    assertThat(outcome.returnedCommittedSuccess()).isFalse();

    InventorySnapshot finalState = inspector.read("BOOK-42");
    assertThat(finalState.available()).isEqualTo(1);
    assertThat(finalState.version()).isEqualTo(8L);
    assertThat(inspector.commandCount(COMMAND_A)).isEqualTo(1L);
    assertThat(inspector.commandCount(COMMAND_B)).isZero();
}
```

Nếu provider cho phép diagnostic iteration 2 tiến tới query, record phải cho thấy
transaction ID không đổi. Không nên khóa test vào exact follow-up exception:
`UnexpectedRollbackException`, optimistic/persistence failure hoặc aborted-state
error có thể surface khác nhau theo flush path/provider version.

Portable business assertion là B không commit và local loop không tạo clean
transaction.

## Experiment 3 — Conflict ở commit nằm ngoài local catch

Fixture không gọi explicit `flush()`:

```java
@Transactional
public ReservationResult reserveWithoutFlush(...) {
    try {
        InventoryItem item = inventory.findById(sku).orElseThrow();
        item.reserve(1);
        return ReservationResult.accepted(...);
    } catch (ObjectOptimisticLockingFailureException conflict) {
        localCatchCounter.incrementAndGet();
        throw conflict;
    }
}
```

Ép same interleaving, rồi assert:

```java
assertThatThrownBy(() -> reserveWithoutFlush(...))
    .isInstanceOf(ObjectOptimisticLockingFailureException.class);
assertThat(localCatchCounter.get()).isZero();
```

Exception được tạo trong transaction interceptor flush/commit, sau target return.
Retry coordinator outside proxy vẫn bắt được; method-local catch không.

## Experiment 4 — Fresh retry phải revalidate stock

Initial stock đổi thành `1`. A và B cùng load `1/version 7`; A thắng. B retry
reload `0/version 8` và phải trả insufficient stock, không retry validation error:

```java
@Test
void retryReloadsAndTurnsStaleAcceptanceIntoDomainRejection()
        throws Exception {
    ConcurrentResults results =
        runTwoCommandsAgainstInitialStock(1);

    assertThat(results.acceptedCount()).isEqualTo(1);
    assertThat(results.insufficientStockCount()).isEqualTo(1);
    assertThat(probe.observations(COMMAND_B)).hasSize(2);
    assertThat(probe.observations(COMMAND_B).get(1).loadedVersion())
        .isEqualTo(8L);

    InventorySnapshot finalState = inspector.read("BOOK-42");
    assertThat(finalState.available()).isZero();
    assertThat(finalState.version()).isEqualTo(8L);
    assertThat(finalState.reservationCount()).isEqualTo(1L);
}
```

`InsufficientStockException` không nằm trong retry allowlist.

## Experiment 5 — Advisor ordering regression

Nếu project giữ `@Retryable` và `@Transactional` trên cùng method, behavior test
phải chứng minh runtime chain:

```java
assertThat(attempts).hasSize(2);
assertThat(attempts.get(0).transactionId())
    .isNotEqualTo(attempts.get(1).transactionId());
assertThat(attempts.get(0).completionStatus().get())
    .isEqualTo(TransactionSynchronization.STATUS_ROLLED_BACK);
assertThat(attempts.get(1).loadedVersion()).isEqualTo(8L);
```

Có thể inspect `Advised.getAdvisors()` như diagnostic supplement, nhưng transaction
identity/outcome là contract test mạnh hơn tên/order của advisor classes.

## Experiment 6 — Serialization/deadlock retry boundary

Dùng `TransactionTemplate` per attempt và một test fixture gây SQLSTATE `40001`
hoặc `40P01`. Assert:

```text
attempt 1: database abort -> Spring rollback -> connection returned
attempt 2: new transaction ID -> queries succeed
```

Không gửi statement tiếp theo trên aborted attempt để “test retry”. PostgreSQL sẽ
reject nó cho tới rollback; correct code phải exit attempt immediately.

Deadlock test cần deterministic opposite row-lock order và bounded futures; luôn
cleanup victim/survivor transactions. Chi tiết detector thuộc DB-009/C-DEADLOCK.

## Committed-state inspector

```java
@Service
class InventoryInspector {
    private final JdbcTemplate jdbc;

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    InventorySnapshot read(String sku) {
        return jdbc.queryForObject(
            """
            select i.available,
                   i.version,
                   count(r.id)
            from inventory_item i
            left join reservation_record r on r.sku = i.sku
            where i.sku = ?
            group by i.available, i.version
            """,
            (rs, rowNum) -> new InventorySnapshot(
                rs.getInt(1),
                rs.getLong(2),
                rs.getLong(3)
            ),
            sku
        );
    }
}
```

Inspector chạy sau actor futures và dùng transaction/context mới.

## Coverage matrix

| Scenario | B attempts | Tx identities | Final result |
| --- | --- | --- | --- |
| Same-Tx loop | 1+ code iterations | Cùng Tx hoặc provider stops | B không commit |
| Separate worker/retry | 2 | Khác nhau | B commit trên fresh state |
| Stock hết sau winner | 2 | Khác nhau | B domain rejection |
| Conflict at commit | Local catch 0 | Attempt Tx rollback | Outer retry catches |
| Retry budget exhausted | Bounded | Một Tx mỗi attempt | Explicit failure |
| Serialization/deadlock | 2+ bounded | Khác nhau | New Tx or exhaustion |

## Chống flaky

- Latches tạo order `both load < A commit < B stale flush`.
- Mọi latch, future và executor termination có timeout.
- `finally` release gates trước executor shutdown.
- Test class chạy `SAME_THREAD`; fixtures là single-scenario.
- Test method không có outer transaction.
- Probe keyed theo command ID và reset sau test.
- Backoff được thay bằng deterministic/no-wait test policy; gate điều phối race.
- Assert completion status và committed business state.
- Không dùng H2 cho version/lock behavior.

## Production verification

Metrics/logs cần tách:

- operation count và attempt count;
- conflict type/SQLSTATE;
- rollback completion trước retry;
- loaded version theo attempt;
- final accepted/rejected/exhausted outcome;
- retry time ngoài transaction;
- hot-key conflict rate và pool utilization;
- duplicate command replay/result.

Một useful invariant query:

```sql
select i.sku,
       i.available,
       i.version,
       count(r.id) as reservations
from inventory_item i
left join reservation_record r on r.sku = i.sku
where i.sku = 'BOOK-42'
group by i.sku, i.available, i.version;
```

Test fixture kỳ vọng `version - initial_version` bằng số successful mutations và
reservation count bằng số distinct accepted commands. Production có thể có các
mutation types khác, nên reconciliation phải dùng domain ledger/event semantics
thay vì áp dụng công thức đơn giản này một cách mù quáng.
