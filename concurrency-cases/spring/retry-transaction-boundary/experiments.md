# Các thử nghiệm optimistic-conflict tất định

## Mục tiêu

Bộ test phải chứng minh:

1. hai tiến trình (actors) cùng tải một version;
2. tiến trình thua nhận một xung đột lạc quan (optimistic conflict) thực sự từ PostgreSQL/Hibernate;
3. lần thử bị lỗi (failed attempt) được rollback trước khi retry;
4. retry sử dụng transaction và persistence context mới;
5. retry tải lại version mới và kiểm tra lại quy tắc nghiệp vụ;
6. dữ liệu tồn kho (inventory) / đặt trước (reservation) được commit cuối cùng là chính xác.

Chỉ kiểm tra `attemptCount == 2` là không đủ; hai lần lặp có thể cùng nằm trong một doomed transaction.

> **Nói ngắn gọn:** quan sát `version + transaction ID + trạng thái hoàn thành +
> trạng thái nghiệp vụ`, không chỉ đếm số vòng lặp.

## Cấu trúc (topology) của test

```text
test thread
  ├─ actor A -> retry coordinator -> Tx-A1 -> đọc v7 -> commit v8
  └─ actor B -> retry coordinator -> Tx-B1 -> đọc v7 -> xung đột/rollback
                                      Tx-B2 -> đọc v8 -> commit v9
```

Phương thức test không sử dụng annotation `@Transactional`. Mỗi tiến trình gọi một proxy do Spring quản lý trên luồng thực thi (executor thread) riêng biệt.

## PostgreSQL Testcontainers và lược đồ (schema)

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

Lược đồ (Schema):

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

Dữ liệu mồi (Fixture setup) commit:

```sql
insert into inventory_item(sku, available, version)
values ('BOOK-42', 2, 7);
```

## Cổng tương tranh (Race gate)

Gate này ép cả hai lần thử đầu tiên đọc version 7, sau đó chỉ cho phép tiến trình thắng tiến hành flush:

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
                "Lần thử đầu tiên phải đọc version 7, nhưng nhận được " + version
            );
        }

        bothLoaded.countDown();
        awaitOrFail(bothLoaded, Duration.ofSeconds(5));

        if (commandId.equals(winnerCommand)) {
            awaitOrFail(allowWinner, Duration.ofSeconds(5));
        } else if (commandId.equals(loserCommand)) {
            awaitOrFail(winnerCommitted, Duration.ofSeconds(5));
        } else {
            throw new AssertionError("Command không mong đợi");
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
                throw new AssertionError("Hết thời gian chờ đợi race gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Bị ngắt ngang khi đang chờ", interrupted);
        }
    }
}
```

Gate này không sử dụng dự đoán về mặt thời gian (timing guess). Việc flush đầu tiên của B chỉ được phép sau khi future của A trả về kết quả, tức là quá trình commit của Tx-A đã hoàn tất.

## Quan sát lần thử (Attempt observation)

Một probe dành riêng cho quá trình test sẽ ghi lại transaction ID, định danh thực sự của Hibernate session, version đã tải và trạng thái hoàn thành:

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

Mã nguồn production không cần probe này. Worker của test fixture sẽ gọi nó sau khi repository đọc (load) dữ liệu và trước khi thực hiện các thay đổi (mutation).

## Thử nghiệm 1 — Luồng thực thi cố định (Fixed pipeline) tạo ra retry sạch

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

Trạng thái hoàn thành (completion) của B trong lần thử 1 được đọc sau khi toàn bộ operation kết thúc. Một probe mạnh hơn có thể ghi lại chuỗi sự kiện (event sequence) và khẳng định `ROLLBACK(B1) < BEGIN(B2)`.

## Vì sao future của A phải hoàn tất trước tín hiệu (signal)?

Nếu ra tín hiệu ngay sau khi A thực hiện flush nhưng trước quá trình commit, thao tác UPDATE của B có thể bị chặn chờ khóa dòng (row lock).
Bài test vẫn có thể đúng nhưng sẽ khó để phân biệt giữa trạng thái chờ khóa (lock wait) với ranh giới retry. Việc chờ `winner.get(timeout)` xác nhận rằng transaction interceptor đã thực sự hoàn thành commit trước khi B thực hiện flush.

## Thử nghiệm 2 — Vòng lặp lỗi dùng lại transaction

Chạy A bằng worker cố định và B bằng một service bị lỗi, có cài đặt công cụ đo (instrument) ở đầu mỗi vòng lặp:

```java
@Test
void localRetryLoopCannotRecoverFailedTransaction() throws Exception {
    // cùng một gate: cả hai đều tải v7, A commit, sau đó B thực hiện flush dữ liệu cũ
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

Nếu provider cho phép lần lặp chẩn đoán thứ 2 tiếp tục thực thi câu truy vấn, bản ghi đo được phải cho thấy transaction ID không đổi. Không nên bó buộc test vào một ngoại lệ cụ thể (exact follow-up exception): các lỗi `UnexpectedRollbackException`, optimistic/persistence failure hoặc aborted-state error có thể bộc lộ khác nhau tùy theo luồng flush hoặc phiên bản của provider.

Khẳng định nghiệp vụ có thể di chuyển (portable business assertion) là B không được commit và vòng lặp cục bộ không thể tạo ra một transaction mới, sạch.

## Thử nghiệm 3 — Xung đột tại thời điểm commit nằm ngoài khối catch cục bộ

Fixture không gọi lệnh `flush()` tường minh:

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

Ép thực thi cùng một trình tự lồng ghép (interleaving), rồi assert:

```java
assertThatThrownBy(() -> reserveWithoutFlush(...))
    .isInstanceOf(ObjectOptimisticLockingFailureException.class);
assertThat(localCatchCounter.get()).isZero();
```

Ngoại lệ được tạo ra bên trong transaction interceptor trong pha flush/commit, sau khi phương thức đích đã trả về kết quả.
Trình điều phối (Retry coordinator) nằm bên ngoài proxy vẫn bắt được; nhưng khối catch cục bộ thì không.

## Thử nghiệm 4 — Một lượt retry sạch phải kiểm tra lại stock

Stock ban đầu đổi thành `1`. A và B cùng tải giá trị `1/version 7`; A thắng. Lượt retry của B sẽ tải lại `0/version 8` và phải trả về lỗi không đủ stock (insufficient stock), không retry trên lỗi xác thực (validation error):

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

Ngoại lệ `InsufficientStockException` không nằm trong danh sách được phép (allowlist) retry.

## Thử nghiệm 5 — Chống suy thoái đối với thứ tự advisor (Advisor ordering regression)

Nếu dự án duy trì các annotation `@Retryable` và `@Transactional` trên cùng một phương thức, các bài test về hành vi phải chứng minh được chuỗi thực thi ở runtime:

```java
assertThat(attempts).hasSize(2);
assertThat(attempts.get(0).transactionId())
    .isNotEqualTo(attempts.get(1).transactionId());
assertThat(attempts.get(0).completionStatus().get())
    .isEqualTo(TransactionSynchronization.STATUS_ROLLED_BACK);
assertThat(attempts.get(1).loadedVersion()).isEqualTo(8L);
```

Có thể kiểm tra (inspect) `Advised.getAdvisors()` như một chẩn đoán phụ trợ, nhưng khẳng định transaction identity/outcome mới là hợp đồng kiểm thử (contract test) mạnh mẽ hơn nhiều so với việc chỉ kiểm tra tên hay thứ tự của các advisor class.

## Thử nghiệm 6 — Ranh giới retry đối với Serialization/deadlock

Dùng `TransactionTemplate` cho mỗi lần thử và một test fixture gây ra lỗi SQLSTATE `40001` hoặc `40P01`. Assert:

```text
attempt 1: database abort -> Spring rollback -> connection được trả về
attempt 2: transaction ID mới -> truy vấn thành công
```

Không được gửi thêm câu lệnh tiếp theo lên một lần thử đã bị hủy (aborted attempt) để "kiểm thử retry". PostgreSQL sẽ từ chối nó cho đến khi rollback diễn ra; mã code đúng phải thoát khỏi lần thử ngay lập tức.

Test deadlock cần xác định chính xác thứ tự khóa dòng bị ngược (deterministic opposite row-lock order) và các future có timeout giới hạn (bounded); luôn dọn dẹp các transaction nạn nhân/sống sót. Chi tiết bộ dò deadlock (detector) thuộc bài DB-009/C-DEADLOCK.

## Inspector trạng thái đã commit

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

Inspector chạy sau các future của tiến trình và sử dụng một transaction/context mới.

## Ma trận độ bao phủ (Coverage matrix)

| Kịch bản | Lần thử của B | Nhận dạng (Tx identities) | Kết quả cuối |
| --- | --- | --- | --- |
| Lặp trên cùng Tx | 1+ lần lặp mã code | Cùng Tx hoặc bị provider dừng | B không commit |
| Tách biệt worker/retry | 2 | Khác nhau | B commit trên trạng thái mới |
| Hết stock sau người thắng | 2 | Khác nhau | B bị từ chối nghiệp vụ (domain rejection) |
| Xung đột tại thời điểm commit| Bắt cục bộ (Local catch) 0 lần | Lần thử Tx bị rollback | Retry ở vòng ngoài bắt được lỗi |
| Hết lượt retry | Hữu hạn | Một Tx mỗi lần thử | Báo lỗi rõ ràng |
| Serialization/deadlock | 2+ hữu hạn | Khác nhau | Tạo Tx mới hoặc cạn kiệt lượt retry |

## Chống flaky

- Các latch điều khiển thứ tự `cả hai cùng đọc < A commit < B flush dữ liệu cũ`.
- Mọi latch, future và quá trình dừng executor đều có thiết lập hết hạn (timeout).
- Khối `finally` đảm bảo giải phóng (release) gate trước khi tắt (shutdown) executor.
- Test class chạy chế độ `SAME_THREAD`; các fixture đều là đơn kịch bản (single-scenario).
- Phương thức test không có transaction bên ngoài (outer transaction).
- Probe được định danh theo command ID và sẽ reset sau test.
- Thời gian chờ (backoff) được thay bằng chính sách test không chờ/tất định; gate đóng vai trò điều phối race.
- Assert trạng thái hoàn thành và trạng thái nghiệp vụ đã được commit.
- Không sử dụng H2 cho hành vi version/khóa.

## Xác minh ở production

Số liệu (Metrics) / bản ghi (logs) cần bóc tách:

- số lượng thao tác (operation) và số lượt thực thi (attempt);
- loại xung đột/SQLSTATE;
- xác nhận quá trình rollback hoàn tất trước khi retry;
- version tải được theo từng lần thử;
- kết quả cuối cùng (được chấp nhận/từ chối/hết lượt thử);
- thời gian retry nằm ngoài transaction;
- tỷ lệ xung đột của hot-key và mức độ sử dụng pool;
- quá trình phát lại command trùng lặp / kết quả.

Một truy vấn bất biến hữu ích:

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

Test fixture luôn kỳ vọng rằng giá trị `version - initial_version` sẽ bằng với số lượt thay đổi (mutations) thành công và số bản ghi reservation bằng số các command riêng biệt được chấp nhận. Trên hệ thống production có thể xuất hiện các loại thao tác thay đổi khác, do đó quá trình đối chiếu (reconciliation) phải dùng ngữ nghĩa sổ cái/event (domain ledger) thay vì áp dụng công thức đơn giản này một cách mù quáng.
