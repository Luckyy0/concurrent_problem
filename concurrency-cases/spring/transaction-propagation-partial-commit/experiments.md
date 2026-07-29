# PostgreSQL integration experiments và production verification

## Chiến lược kiểm thử

Dùng `@SpringBootTest` + PostgreSQL Testcontainers để quan sát commit thật,
rollback-only và `UnexpectedRollbackException`. Test method không annotated
`@Transactional`; reader dùng transaction độc lập. `CheckoutProbe` dừng outer
thread sau khi inner `REQUIRES_NEW` đã return/commit.

Không dùng `Thread.sleep`; mọi latch/future có timeout. Xem
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Infrastructure

```java
@Testcontainers
@SpringBootTest
class TransactionPropagationPartialCommitIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }

    @Autowired CheckoutService requiresNewCheckout;
    @Autowired AtomicCheckoutService requiredCheckout;
    @Autowired CheckoutWithCaughtFailure caughtFailureCheckout;
    @Autowired CheckoutProbe probe;
    @Autowired PlatformTransactionManager transactionManager;
    @Autowired JdbcTemplate jdbc;
}
```

Mỗi test setup committed order 42 status `PENDING`, xóa audit/risk rows. Audit
table dùng unique `(operation_id, event_type)`; test partial semantics không phụ
thuộc persistence-context cache.

## Bounded probe

```java
final class CheckoutProbe {
    private final AtomicReference<Gate> current = new AtomicReference<>();

    Gate arm() {
        Gate gate = new Gate();
        current.set(gate);
        return gate;
    }

    void afterAuditCommit() {
        Gate gate = current.get();
        if (gate == null) return;
        gate.innerReturned.countDown();
        awaitOrFail(gate.allowOuterFinish);
    }

    record Gate(
            CountDownLatch innerReturned,
            CountDownLatch allowOuterFinish
    ) {
        Gate() { this(new CountDownLatch(1), new CountDownLatch(1)); }
    }
}

private static void awaitOrFail(CountDownLatch latch) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new IllegalStateException("latch timed out");
        }
    } catch (InterruptedException exception) {
        Thread.currentThread().interrupt();
        throw new IllegalStateException("interrupted", exception);
    }
}
```

## Experiment 1: REQUIRES_NEW audit commit trước outer rollback

```java
@Test
void completedAuditIsVisibleBeforeOuterOutcomeAndSurvivesRollback()
        throws Exception {
    CheckoutProbe.Gate gate = probe.arm();

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> writer = executor.submit(() ->
                requiresNewCheckout.complete(42L, true));

        assertTrue(gate.innerReturned().await(5, TimeUnit.SECONDS));

        Snapshot duringOuter = readSnapshotInNewTransaction(42L);
        assertEquals("PENDING", duringOuter.orderStatus());
        assertEquals(List.of("PAYMENT_COMPLETED"), duringOuter.auditTypes());

        gate.allowOuterFinish().countDown();
        assertThrows(ExecutionException.class,
                () -> writer.get(5, TimeUnit.SECONDS));
    }

    Snapshot afterRollback = readSnapshotInNewTransaction(42L);
    assertEquals("PENDING", afterRollback.orderStatus());
    assertEquals(List.of("PAYMENT_COMPLETED"), afterRollback.auditTypes());
}
```

Probe nằm sau `recordPaymentCompleted` return, nên inner transaction đã commit.
Reader thấy semantic contradiction deterministic, không phụ thuộc race may mắn.

> **Nói ngắn gọn:** test chứng minh partial commit bằng committed business state,
>không chỉ bằng việc hai method báo transaction active.

## Experiment 2: REQUIRED success record rollback cùng outer

`AtomicCheckoutService` giống broken outer nhưng gọi audit `REQUIRED`. Sau audit,
probe giữ outer rồi test reader phải thấy committed state cũ và không audit. Cho
outer fail, final state vẫn PENDING/no audit:

```java
@Test
void requiredAuditRollsBackWithCheckout() throws Exception {
    CheckoutProbe.Gate gate = probe.arm();

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> writer = executor.submit(() ->
                requiredCheckout.complete(42L, true));

        assertTrue(gate.innerReturned().await(5, TimeUnit.SECONDS));
        Snapshot duringOuter = readSnapshotInNewTransaction(42L);
        assertEquals("PENDING", duringOuter.orderStatus());
        assertTrue(duringOuter.auditTypes().isEmpty());

        gate.allowOuterFinish().countDown();
        assertThrows(ExecutionException.class,
                () -> writer.get(5, TimeUnit.SECONDS));
    }

    Snapshot afterRollback = readSnapshotInNewTransaction(42L);
    assertEquals("PENDING", afterRollback.orderStatus());
    assertTrue(afterRollback.auditTypes().isEmpty());
}
```

Cùng latch `innerReturned` có semantics khác theo propagation: `REQUIRES_NEW`
return sau inner commit, còn `REQUIRED` return khi physical transaction vẫn mở.
Database assertion phân biệt hai trường hợp.

## Experiment 3: REQUIRED exception bị catch vẫn rollback-only

```java
@Test
void caughtInnerRequiredFailureCausesUnexpectedRollback() {
    UnexpectedRollbackException failure = assertThrows(
            UnexpectedRollbackException.class,
            () -> caughtFailureCheckout.complete(42L)
    );

    assertNotNull(failure);
    Snapshot snapshot = readSnapshotInNewTransaction(42L);
    assertEquals("PENDING", snapshot.orderStatus());
    assertTrue(snapshot.auditTypes().isEmpty());
}
```

Test phải gọi Spring proxy bean. Inner service cũng là bean khác/proxy để exception
đi qua transactional interceptor và mark rollback-only. Không mock transaction
manager trong test này.

## Experiment 4: truthful REQUIRES_NEW attempt record

Outer rollback nhưng independent row phải có type `ATTEMPT_STARTED`, không có
`PAYMENT_COMPLETED`. Retry cùng `operationId` hai lần rồi assert unique constraint/
`insertIfAbsent` chỉ tạo một attempt. Test này xác nhận intended partial commit
semantics và idempotency, không chỉ kỹ thuật propagation.

## Experiment 5: inner failure policy

Bổ sung matrix:

| Inner propagation/outcome | Outer xử lý | Expected |
| --- | --- | --- |
| REQUIRED runtime failure | propagate | toàn bộ rollback, original cause |
| REQUIRED runtime failure | catch | outer rollback + `UnexpectedRollbackException` |
| REQUIRED expected rejection value | branch explicit | commit/rollback theo outer decision |
| REQUIRES_NEW failure | catch | outer có thể commit; inner rollback |
| REQUIRES_NEW success, outer fail | propagate outer | inner commit sống sót |

Mỗi row cần integration assertion database, không chỉ Mockito verify.

## Reader transaction độc lập

```java
Snapshot readSnapshotInNewTransaction(long orderId) {
    TransactionTemplate reader = new TransactionTemplate(transactionManager);
    reader.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    reader.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    return reader.execute(status -> querySnapshot(jdbc, orderId));
}
```

## Pool/lock diagnostic experiment

Với pool test nhỏ, nhiều outer transaction đồng thời giữ connection rồi gọi
`REQUIRES_NEW` có thể chờ connection thứ hai. Test bounded future timeout và pool
metrics để minh họa resource amplification, nhưng không biến đây thành sizing
benchmark. Full connection-pool exhaustion thuộc `SPR-007`.

Inner transaction truy cập row outer đã update có thể block. Dùng PostgreSQL
`lock_timeout` nhỏ trong test diagnostic, assert timeout/rollback và thu blocking
PID; không dùng sleep.

## Production verification

- audit type so với committed order status;
- `UnexpectedRollbackException`, rollback-only và original inner cause;
- outer/inner transaction IDs, duration, suspend time;
- connection active/pending acquisition và pool timeout;
- inner lock timeout/deadlock;
- operation ID duplicate/conflict;
- after-commit/outbox outcome khi success publication được tách.

## Checklist chất lượng

- [ ] PostgreSQL Testcontainers được dùng.
- [ ] Test không có outer `@Transactional`.
- [ ] REQUIRES_NEW inner commit được quan sát khi outer còn mở.
- [ ] Outer rollback giữ independent audit nhưng rollback order.
- [ ] REQUIRED audit không visible và rollback cùng outer.
- [ ] Caught REQUIRED failure ném `UnexpectedRollbackException`.
- [ ] Reader dùng `REQUIRES_NEW` transaction độc lập.
- [ ] Mọi latch/future có timeout; không dùng sleep.
- [ ] Truthful attempt semantics/idempotency được test.
- [ ] Connection/lock amplification được phân biệt khỏi correctness.
