# PostgreSQL integration experiments và production verification

## Chiến lược kiểm thử

Unit test thuần Java không kiểm tra Spring proxy, repository transaction commit
hay PostgreSQL MVCC. Dùng `@SpringBootTest` + PostgreSQL Testcontainers, gọi bean
từ application context và không annotate test method bằng `@Transactional`.

Một latch probe dừng writer sau khi repository debit method trả về. Với broken
service, debit transaction đã commit. Với transactional harness, debit SQL nằm
trong outer transaction và chưa commit.

Xem [Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Test infrastructure

```java
@Testcontainers
@SpringBootTest
class TransactionalSelfInvocationIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }

    @Autowired BrokenTransferService broken;
    @Autowired TransactionalTransferHarness fixed;
    @Autowired LatchingTransferStepProbe probe;
    @Autowired JdbcTemplate jdbc;
    @Autowired PlatformTransactionManager transactionManager;
}
```

Schema/migration tạo `account(id bigint primary key, balance bigint not null
check (balance >= 0))`. Trước mỗi test insert A=100, B=100 trong committed setup
transaction. Executor dùng hai platform thread và luôn được shutdown/await bounded.

## Latch probe

```java
final class LatchingTransferStepProbe implements TransferStepProbe {
    private final AtomicReference<Gate> current = new AtomicReference<>();

    Gate arm() {
        Gate gate = new Gate();
        current.set(gate);
        return gate;
    }

    @Override
    public void afterDebit() {
        Gate gate = current.get();
        if (gate == null) return;
        gate.debitReached.countDown();
        awaitOrFail(gate.allowCredit);
    }

    record Gate(
            CountDownLatch debitReached,
            CountDownLatch allowCredit
    ) {
        Gate() { this(new CountDownLatch(1), new CountDownLatch(1)); }
    }
}
```

```java
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

Production bean của `TransferStepProbe` là no-op; test configuration thay bằng
`@Primary` latching probe.

## Experiment 1: broken self-invocation lộ partial commit

```java
@Test
void readerSeesCommittedDebitBeforeCredit() throws Exception {
    LatchingTransferStepProbe.Gate gate = probe.arm();

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> writer = executor.submit(() -> broken.transfer(
                new TransferCommand(1L, 2L, 10L)
        ));

        assertTrue(gate.debitReached().await(5, TimeUnit.SECONDS));
        assertEquals(190L, readTotalInNewTransaction());

        gate.allowCredit().countDown();
        writer.get(5, TimeUnit.SECONDS);
    }

    assertEquals(200L, readTotalInNewTransaction());
}
```

Business assertion `total=190` chứng minh partial state externally visible; chỉ
assert transaction-active flag là chưa đủ.

> **Nói ngắn gọn:** test đọc bằng transaction khác, vì cùng persistence context
>hoặc outer test transaction có thể che commit boundary thật.

## Experiment 2: credit failure không rollback broken debit

Dùng destination ID không tồn tại để `credit` trả affected rows bằng 0:

```java
@Test
void brokenCreditFailureLeavesDebitCommitted() {
    assertThrows(ExecutionException.class, () -> {
        try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
            Future<?> writer = executor.submit(() -> broken.transfer(
                    new TransferCommand(1L, 999L, 10L)
            ));
            writer.get(5, TimeUnit.SECONDS);
        }
    });

    assertEquals(90L, balanceOf(1L));
    assertEquals(190L, readTotalInNewTransaction());
}
```

Trong code test thực tế, unwrap `ExecutionException.getCause()` và assert domain
exception; executor cleanup đặt trong `finally` nếu không dùng Java 21
try-with-resources.

## Experiment 3: transactional proxy giữ state invisible tới commit

Test harness là Spring bean riêng, public method annotated `@Transactional` và có
cùng debit → probe → credit sequence. Nó tồn tại chỉ trong `@TestConfiguration`
để đặt deterministic barrier trong transaction.

```java
@Test
void outerTransactionHidesDebitUntilBothUpdatesCommit() throws Exception {
    assertTrue(AopUtils.isAopProxy(fixed));
    LatchingTransferStepProbe.Gate gate = probe.arm();

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> writer = executor.submit(() -> fixed.transferWithPause(
                new TransferCommand(1L, 2L, 10L)
        ));

        assertTrue(gate.debitReached().await(5, TimeUnit.SECONDS));
        assertEquals(200L, readTotalInNewTransaction());

        gate.allowCredit().countDown();
        writer.get(5, TimeUnit.SECONDS);
    }

    assertEquals(90L, balanceOf(1L));
    assertEquals(110L, balanceOf(2L));
    assertEquals(200L, readTotalInNewTransaction());
}
```

Probe có thể ghi lại
`TransactionSynchronizationManager.isActualTransactionActive()` để assert `true`
ở fixed harness và `false` sau broken repository debit, nhưng đây là structural
assertion bổ sung, không thay invariant total.

## Experiment 4: failure rollback cả transfer đúng

Fixed harness transfer tới destination không tồn tại phải ném exception. Sau
future completion, assert A=100, B=100, total=200. Bổ sung variant checked
exception nếu production dùng `rollbackFor`; test phải chứng minh policy thực,
không chỉ đọc annotation.

## Reader transaction độc lập

```java
long readTotalInNewTransaction() {
    TransactionTemplate reader = new TransactionTemplate(transactionManager);
    reader.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    reader.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    return reader.execute(status -> jdbc.queryForObject(
            "select sum(balance) from account", Long.class));
}
```

Không reuse entity đã load trước bulk update; đọc SQL scalar trong transaction mới
tránh persistence-context cache.

## Kiểm tra proxy và call path

- Assert bean lấy từ context là AOP proxy.
- Gọi public entry qua injected bean, không `new` và không gọi target unwrap.
- Dùng log/test probe xác nhận transaction active đúng boundary.
- Test self-invocation broken và external-bean fixed như hai case riêng.
- Nếu dùng `TransactionTemplate`, assert active bên trong callback và inactive
  sau return.

## Production verification

- total-balance/conservation invariant qua reconciliation;
- partial transfer/affected-row mismatch;
- transaction commit/rollback/duration và rollback-only failure;
- datasource connection use, statement/lock timeout;
- correlation/transfer ID xuyên debit-credit;
- event published before/after commit;
- startup diagnostic xác nhận expected beans được proxied khi phù hợp.

## Checklist chất lượng

- [ ] PostgreSQL Testcontainers được dùng.
- [ ] Test method không có outer `@Transactional` che commit.
- [ ] Reader dùng independent `REQUIRES_NEW` transaction.
- [ ] Latch dừng đúng sau debit, không dùng sleep.
- [ ] Mọi latch/future có timeout.
- [ ] Broken test thấy total 190 trước credit.
- [ ] Broken failure giữ debit committed.
- [ ] Fixed reader chỉ thấy total 200 và failure rollback cả hai update.
- [ ] Proxy/transaction-active assertion bổ sung business invariant.
- [ ] Executor và container lifecycle được cleanup.
