# Các thí nghiệm bounded optimistic retry

## Mục tiêu

- Ép nhiều actors load cùng version.
- Assert conflicts rollback và retries dùng transaction IDs mới.
- Final points bằng tổng unique commands; credit records không duplicate.
- Attempt cap/deadline dừng retry.
- Backoff/jitter được kiểm tra không dùng fixed sleep.
- Đo amplification: commands, attempts, conflicts, success-attempt.

> **Nói ngắn gọn:** final balance đúng chưa đủ; retry có thể đúng dữ liệu nhưng
> vẫn phá capacity nếu attempts không bounded.

## PostgreSQL Testcontainers

```java
@Testcontainers
@SpringBootTest
class OptimisticRetryIT {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    RewardCreditCoordinator coordinator;

    @Autowired
    JdbcTemplate jdbc;

    private final ExecutorService executor = Executors.newFixedThreadPool(8);

    @BeforeEach
    void seed() {
        jdbc.update("delete from reward_credit");
        jdbc.update("delete from reward_wallet");
        jdbc.update("""
                insert into reward_wallet(wallet_id, points, version, active)
                values (77, 100, 10, true)
                """);
    }

    @AfterEach
    void shutdown() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Test method không `@Transactional`.

## Deterministic first-attempt gate

Test-only gate trong attempt, ngay sau wallet load:

```java
final class FirstAttemptGate {
    private final CountDownLatch allLoaded;
    private final CountDownLatch continueFlush = new CountDownLatch(1);
    private final ConcurrentMap<UUID, AtomicInteger> attempts =
            new ConcurrentHashMap<>();

    FirstAttemptGate(int actors) {
        this.allLoaded = new CountDownLatch(actors);
    }

    void afterLoad(UUID commandId) {
        int number = attempts.computeIfAbsent(
                commandId,
                ignored -> new AtomicInteger()
        ).incrementAndGet();
        if (number == 1) {
            allLoaded.countDown();
            await(continueFlush, "release first-attempt flushes");
        }
    }

    void awaitAllAndRelease() {
        await(allLoaded, "all first attempts loaded same version");
        continueFlush.countDown();
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

Production gate no-op. Probe còn ghi `txid_current()`, version và outcome.

## Thí nghiệm 1 — Tám commands cùng commit qua fresh retries

```java
@Test
void uniqueCommandsEventuallyCommitWithoutLostOrDuplicatePoints()
        throws Exception {
    int actors = 8;
    List<CreditCommand> commands = IntStream.range(0, actors)
            .mapToObj(index -> new CreditCommand(
                    UUID.randomUUID(), 77, 10
            ))
            .toList();

    List<Future<CreditResult>> futures = commands.stream()
            .map(command -> executor.submit(() ->
                    coordinator.credit(
                            command,
                            Instant.now().plusSeconds(5)
                    )
            ))
            .toList();

    gate.awaitAllAndRelease();
    for (Future<CreditResult> future : futures) {
        assertThat(future.get(10, TimeUnit.SECONDS).committed()).isTrue();
    }

    assertThat(points()).isEqualTo(180);
    assertThat(version()).isEqualTo(18);
    assertThat(creditCount()).isEqualTo(8);
    assertThat(distinctCommandCount()).isEqualTo(8);
    assertThat(probe.totalAttempts()).isGreaterThanOrEqualTo(8);
    assertThat(probe.optimisticConflicts()).isGreaterThan(0);
    assertThat(probe.transactionIds()).doesNotHaveDuplicates();
}
```

Test config đủ retry budget để tất cả hoàn tất; không assert exact attempt count vì
scheduler quyết định interleaving sau first wave.

## Thí nghiệm 2 — Retry reloads winner state

Hai actors gate v10. A commit +10/v11. B first attempt conflict; second attempt
probe phải thấy points `110`, version `11`, rồi commit `130/v12`.

```java
assertThat(probe.loads(commandB))
        .containsExactly(
                new LoadObservation(100, 10),
                new LoadObservation(110, 11)
        );
assertThat(probe.transactionIds(commandB))
        .hasSize(2)
        .doesNotHaveDuplicates();
assertThat(recordingWaiter.invocations(commandB)).isEqualTo(1);
```

Đây là proof transaction/persistence context mới, không chỉ exception catch.

## Thí nghiệm 3 — Attempt cap tạo exhaustion

Test double của attempt worker luôn ném
`ObjectOptimisticLockingFailureException`. Budget `maxAttempts=3`, deadline còn
dài:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline))
        .isInstanceOf(WalletContentionException.class);
assertThat(attempts.invocations()).isEqualTo(3);
assertThat(recordingWaiter.invocations()).isEqualTo(2);
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isFalse();
```

Không có backoff sau final attempt.

## Thí nghiệm 4 — Deadline dừng trước attempt cap

Inject mutable/fake `Clock`. Recording waiter advance clock tới deadline sau first
conflict:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline))
        .isInstanceOf(WalletContentionException.class);
assertThat(attempts.invocations()).isEqualTo(1);
assertThat(recordingWaiter.totalRequestedDelay())
        .isLessThanOrEqualTo(Duration.between(start, deadline));
```

Test hoàn toàn deterministic, không chờ wall clock.

## Thí nghiệm 5 — Backoff và jitter bounds

Inject deterministic `JitterSource` trả min/max:

```java
assertThat(planWithZeroJitter.delayAfter(1))
        .isEqualTo(Duration.ofMillis(20));
assertThat(planWithMaxJitter.delayAfter(1))
        .isBetween(
                Duration.ofMillis(20),
                Duration.ofMillis(30)
        );
assertThat(plan.delayAfter(20))
        .isLessThanOrEqualTo(Duration.ofMillis(200));
```

Property test hữu hạn assert delay không âm, nondecreasing base và luôn <= max.
Không dùng fixed sleep để test backoff calculator.

## Thí nghiệm 6 — Same command replay

```java
CreditResult first = coordinator.credit(command, deadline());
long pointsAfterFirst = points();
long versionAfterFirst = version();

CreditResult replay = coordinator.credit(command, deadline());

assertThat(first.applied()).isTrue();
assertThat(replay.replayed()).isTrue();
assertThat(points()).isEqualTo(pointsAfterFirst);
assertThat(version()).isEqualTo(versionAfterFirst);
assertThat(creditCountFor(command.commandId())).isEqualTo(1);
```

Concurrent same-ID test còn assert unique loser rollback rồi fresh replay, không
tạo second wallet delta.

## Thí nghiệm 7 — Business rejection không retry

Set wallet `active=false`. Attempt trả/throw domain rejection:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline()))
        .isInstanceOf(WalletSuspendedException.class);
assertThat(attempts.invocations()).isEqualTo(1);
assertThat(recordingWaiter.invocations()).isZero();
assertThat(points()).isEqualTo(100);
```

## Thí nghiệm 8 — Backoff không giữ transaction

Recording waiter assert tại mỗi invocation:

```java
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isFalse();
assertThat(probe.previousOutcome()).isEqualTo("ROLLED_BACK");
```

Pool metric/test probe xác nhận connection returned trước waiter. Không suy luận
chỉ từ method annotations.

## Thí nghiệm 9 — Outer transaction guard

Transactional test bean gọi coordinator; guard phải fail trước attempt:

```java
assertThatThrownBy(() -> transactionalCaller.call(command))
        .isInstanceOf(IllegalStateException.class);
assertThat(attempts.invocations()).isZero();
```

## Ma trận bao phủ

| Test | Safety | Liveness/capacity |
| --- | --- | --- |
| 8 unique commands | Points/credits/version đúng | Attempts/conflicts bounded |
| Two-actor reload | Fresh state/version | One backoff |
| Attempt cap | Không partial write | Exact 3 attempts, exhaustion |
| Deadline | Không attempt dở dang | Stop early |
| Jitter | N/A | Delay bounds |
| Replay | One delta/record | Fast stable result |
| Rejection | State unchanged | No retry |
| Waiter boundary | Rollback complete | No connection held |

## Chống flaky và production verification

- Gate chỉ first attempts; later retries chạy implementation thật.
- Latch/Future đều bounded; không fixed sleep.
- Recording clock/jitter/waiter cho policy tests.
- PostgreSQL Testcontainers cho version/unique/transaction semantics.
- Không assert exact winner hay exact attempts sau barrier.
- Thu SQL/txid/probe state khi watchdog fail.
- Production theo dõi commands, attempts, conflicts, success-attempt histogram,
  exhaustion, backoff, latency, duplicate replay và pool pending.
- Khi attempts/success tăng bền vững, đánh giá atomic delta/pessimistic/queue ở
  `LOCK-005`, không tăng cap theo phản xạ.
