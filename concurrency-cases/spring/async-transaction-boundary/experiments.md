# PostgreSQL integration experiments và production verification

## Chiến lược kiểm thử

Dùng `@SpringBootTest`, PostgreSQL Testcontainers và executor thật. Test method
không annotated `@Transactional`. Một test harness Spring bean mở outer transaction,
`saveAndFlush` order rồi dừng bằng latch. Test thread gọi async proxy và giữ
returned future để quan sát outcome trước khi cho outer transaction commit/rollback.

Không dùng `Thread.sleep`; mọi latch/future có timeout. Kiến thức nền:
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Infrastructure

```java
@Testcontainers
@SpringBootTest
class AsyncTransactionBoundaryIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }

    @Autowired TransactionalOrderHarness harness;
    @Autowired AsyncOrderProcessor brokenProcessor;
    @Autowired OrderPlacementService fixedPlacement;
    @Autowired AsyncProcessingProbe processingProbe;
    @Autowired ThreadPoolTaskExecutor orderExecutor;
    @Autowired OrderRepository orders;
    @Autowired DispatchAttemptRepository attempts;
}
```

Schema dùng sequence/identity cho order ID. `dispatch_attempt` của orphan test cố
ý không có FK để minh họa side effect độc lập; production schema có FK có thể
block/fail thay vì orphan nhưng vẫn không có correct ordering.

## Transactional harness

```java
@Service
public class TransactionalOrderHarness {

    @Transactional
    public long insertAndPause(
            PlaceOrderCommand command,
            Gate gate,
            boolean rollbackAfterPause
    ) {
        Order order = orders.saveAndFlush(Order.pending(command.customerId()));
        gate.orderId().set(order.getId());
        gate.flushed().countDown();
        awaitOrFail(gate.allowCompletion());
        if (rollbackAfterPause) {
            throw new IllegalStateException("rollback requested");
        }
        return order.getId();
    }
}

record Gate(
        AtomicLong orderId,
        CountDownLatch flushed,
        CountDownLatch allowCompletion
) {
    Gate() {
        this(new AtomicLong(), new CountDownLatch(1), new CountDownLatch(1));
    }
}
```

`awaitOrFail`:

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

## Experiment 1: async reader không thấy uncommitted order

```java
@Test
void asyncTransactionCannotReadUncommittedOuterInsert() throws Exception {
    Gate gate = new Gate();

    try (ExecutorService caller = Executors.newSingleThreadExecutor()) {
        Future<Long> writer = caller.submit(() -> harness.insertAndPause(
                new PlaceOrderCommand(7L, false), gate, false));

        assertTrue(gate.flushed().await(5, TimeUnit.SECONDS));
        long orderId = gate.orderId().get();

        CompletableFuture<Void> async = brokenProcessor.process(orderId);
        ExecutionException failure = assertThrows(
                ExecutionException.class,
                () -> async.get(5, TimeUnit.SECONDS)
        );
        assertInstanceOf(IllegalStateException.class, failure.getCause());
        assertFalse(orders.existsById(orderId));

        gate.allowCompletion().countDown();
        assertEquals(orderId, writer.get(5, TimeUnit.SECONDS));
        assertTrue(orders.existsById(orderId));
    }
}
```

Async future được giữ để exception không biến mất. Outer writer vẫn bị gate giữ,
nên missing read không phụ thuộc scheduler.

> **Nói ngắn gọn:** `saveAndFlush` đã cấp ID và gửi INSERT, nhưng future trên
>thread khác chỉ đọc committed database state.

## Experiment 2: async side effect commit trước outer rollback

Test-specific async writer nhận immutable order ID, insert `dispatch_attempt` trong
worker transaction mà không query/FK tới order:

```java
@Test
void asyncSideEffectSurvivesOuterRollback() throws Exception {
    Gate gate = new Gate();

    try (ExecutorService caller = Executors.newSingleThreadExecutor()) {
        Future<Long> writer = caller.submit(() -> harness.insertAndPause(
                new PlaceOrderCommand(7L, false), gate, true));

        assertTrue(gate.flushed().await(5, TimeUnit.SECONDS));
        long orderId = gate.orderId().get();
        orphanAttemptWriter.write(orderId).get(5, TimeUnit.SECONDS);
        assertEquals(1, attempts.countByOrderId(orderId));

        gate.allowCompletion().countDown();
        assertThrows(ExecutionException.class,
                () -> writer.get(5, TimeUnit.SECONDS));

        assertFalse(orders.existsById(orderId));
        assertEquals(1, attempts.countByOrderId(orderId));
    }
}
```

Business assertion là orphan attempt tồn tại khi order không tồn tại, không chỉ
assert hai thread có transaction ID khác nhau.

## Experiment 3: after-commit listener không chạy trước commit

Test harness khác save order, publish `OrderPlacedEvent`, signal `eventPublished`
rồi chờ `allowCommit`, tất cả trong public `@Transactional` method. Listener và
processor lấy từ solution.

```java
@Test
void afterCommitDispatchWaitsForSuccessfulCommit() throws Exception {
    CommitGate gate = new CommitGate();
    processingProbe.reset();

    try (ExecutorService caller = Executors.newSingleThreadExecutor()) {
        Future<Long> writer = caller.submit(() ->
                afterCommitHarness.placeAndPause(gate, false));

        assertTrue(gate.eventPublished().await(5, TimeUnit.SECONDS));
        assertEquals(0, processingProbe.startedCount());

        gate.allowCommit().countDown();
        long orderId = writer.get(5, TimeUnit.SECONDS);
        assertTrue(processingProbe.started().await(5, TimeUnit.SECONDS));
        assertTrue(processingProbe.succeeded().await(5, TimeUnit.SECONDS));
        assertEquals(orderId, processingProbe.lastOrderId());
        assertTrue(orders.existsById(orderId));
    }
}
```

`startedCount=0` được đọc trong lúc transaction chắc chắn còn bị gate giữ; không
cần negative sleep assertion.

## Experiment 4: rollback không dispatch listener

Harness publish event rồi rollback. Sau writer future failure, submit một sentinel
vào cùng executor và chờ nó để drain mọi task đã enqueue trước sentinel:

```java
@Test
void rollbackDoesNotScheduleAfterCommitProcessor() throws Exception {
    processingProbe.reset();
    assertThrows(ExecutionException.class, () -> {
        try (ExecutorService caller = Executors.newSingleThreadExecutor()) {
            Future<Long> writer = caller.submit(() ->
                    afterCommitHarness.placeAndRollback());
            writer.get(5, TimeUnit.SECONDS);
        }
    });

    Future<?> sentinel = orderExecutor.getThreadPoolExecutor().submit(() -> {});
    sentinel.get(5, TimeUnit.SECONDS);

    assertEquals(0, processingProbe.startedCount());
    assertEquals(0, orders.count());
}
```

Sentinel tránh “đợi 300 ms rồi hy vọng listener không chạy”. Nếu event bị schedule
trước sentinel trên single-worker test executor, probe sẽ tăng trước assertion.

## Experiment 5: executor rejection sau commit

Cấu hình test executor một worker, queue một slot và `AbortPolicy`; giữ worker và
queue bằng latch, sau đó commit order/event thứ ba. Assert order vẫn committed dù
after-commit dispatch ném/reported `TaskRejectedException`, đồng thời rejection
metric tăng. Đây là bằng chứng local after-commit không durable/atomic với handoff.

Không để rejection làm test thread giữ transaction mở vô hạn; release blocker và
await executor termination trong `finally`.

## Context và transaction assertions

Probe worker ghi:

```text
callerThread != workerThread
TransactionSynchronizationManager.isActualTransactionActive() == true
```

`true` là transaction riêng của async processor. Bổ sung TaskDecorator test cho
MDC/trace cleanup, nhưng không copy transaction resource. Không truyền managed
`Order` vào listener; event chỉ có ID.

## Production verification

- committed orders không có processing outcome quá SLA;
- missing-order/orphan-attempt count;
- after-commit scheduled/started/succeeded/failed/rejected;
- executor active, queue age/capacity, rejection và shutdown loss;
- worker transaction commit/rollback/timeout;
- late completion sau cancel;
- outbox backlog/redelivery/idempotency khi dùng durable solution.

## Checklist chất lượng

- [ ] PostgreSQL Testcontainers được dùng.
- [ ] Test không có outer `@Transactional`.
- [ ] Outer insert được flush nhưng giữ uncommitted bằng latch.
- [ ] Async future có bounded `get` và exception được quan sát.
- [ ] Missing read và orphan side effect đều được assert.
- [ ] After-commit listener chưa start trước commit và start sau commit.
- [ ] Rollback test dùng executor sentinel, không dùng sleep.
- [ ] Rejection-after-commit gap được kiểm tra.
- [ ] Executor/caller threads được cleanup bounded.
- [ ] Transaction context không được propagate qua thread.
