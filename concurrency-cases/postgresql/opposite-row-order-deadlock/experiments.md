# Các thí nghiệm PostgreSQL deadlock có điều phối

## Mục tiêu

Experiment suite phải chứng minh cả safety và progress:

1. ép được wait-for cycle bằng hai PostgreSQL transactions thật;
2. đúng một victim nhận `40P01`, transaction đó ở trạng thái `25P02` trước
   rollback;
3. partial debit của victim bị rollback và tổng balance được bảo toàn;
4. canonical order cho phép hai opposing transfers hoàn tất;
5. retry mở transaction mới, không reuse failed attempt;
6. phân biệt deadlock với `55P03` lock timeout;
7. mọi latch/future/database wait đều có timeout.

Không dùng H2 để suy luận deadlock detector. Không dùng timing đoán actor đã tới
đâu; barrier nằm ngay sau row lock thứ nhất.

> **Nói ngắn gọn:** test tốt không chỉ chờ exception; nó dựng cycle, kiểm tra
> rollback và xác nhận business state sau khi mọi actor kết thúc.

## Môi trường PostgreSQL Testcontainers

```java
package example.transfer;

import static org.assertj.core.api.Assertions.assertThat;

import java.math.BigDecimal;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.time.Duration;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.function.BooleanSupplier;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
class PostgreSqlDeadlockIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("deadlock_cases")
                    .withUsername("cases")
                    .withPassword("cases");

    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    @BeforeAll
    static void createSchema() throws SQLException {
        try (Connection connection = open();
                Statement statement = connection.createStatement()) {
            statement.execute("""
                    create table account (
                        id bigint primary key,
                        balance numeric(19, 2) not null
                            check (balance >= 0)
                    )
                    """);
        }
    }

    @BeforeEach
    void resetRows() throws SQLException {
        try (Connection connection = open();
                Statement statement = connection.createStatement()) {
            statement.execute("truncate table account");
            statement.execute("""
                    insert into account(id, balance)
                    values (101, 1000.00), (202, 1000.00)
                    """);
        }
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }

    private static Connection open() throws SQLException {
        return DriverManager.getConnection(
                POSTGRES.getJdbcUrl(),
                POSTGRES.getUsername(),
                POSTGRES.getPassword()
        );
    }
}
```

Testcontainers user của fixture có quyền hạ `deadlock_timeout` cho test. Production
không nên copy cấu hình này mà không đánh giá workload.

## Helper điều phối

```java
private static void await(
        CountDownLatch latch,
        String description
) {
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

Mọi `Future` bên dưới dùng `get(timeout)`. Nếu implementation regression thành
wait vô hạn, test fail có chẩn đoán thay vì treo CI.

## Thí nghiệm 1 — Ép cycle và chứng minh victim rollback

Mỗi task debit source sau lock thứ nhất, rồi dừng ở barrier. Khi cả hai đã giữ
source row, test cho phép chúng xin destination row. Một actor trở thành victim;
partial debit của actor đó phải rollback.

```java
record AttemptOutcome(
        long fromId,
        String sqlState,
        String stateBeforeRollback
) {
    static AttemptOutcome committed(long fromId) {
        return new AttemptOutcome(fromId, "00000", "00000");
    }

    static AttemptOutcome failed(
            long fromId,
            String sqlState,
            String stateBeforeRollback
    ) {
        return new AttemptOutcome(fromId, sqlState, stateBeforeRollback);
    }
}

@Test
void oppositeOrderCreatesOneVictimAndRollsBackItsDebit() throws Exception {
    var firstRowsLocked = new CountDownLatch(2);
    var requestSecondRow = new CountDownLatch(1);

    Future<AttemptOutcome> t1 = executor.submit(() ->
            brokenTransferAttempt(
                    101, 202, new BigDecimal("100.00"),
                    firstRowsLocked, requestSecondRow
            )
    );
    Future<AttemptOutcome> t2 = executor.submit(() ->
            brokenTransferAttempt(
                    202, 101, new BigDecimal("70.00"),
                    firstRowsLocked, requestSecondRow
            )
    );

    await(firstRowsLocked, "both source rows locked");
    requestSecondRow.countDown();

    List<AttemptOutcome> outcomes = List.of(
            t1.get(8, TimeUnit.SECONDS),
            t2.get(8, TimeUnit.SECONDS)
    );

    assertThat(outcomes)
            .extracting(AttemptOutcome::sqlState)
            .containsExactlyInAnyOrder("00000", "40P01");

    AttemptOutcome victim = outcomes.stream()
            .filter(outcome -> outcome.sqlState().equals("40P01"))
            .findFirst()
            .orElseThrow();
    assertThat(victim.stateBeforeRollback()).isEqualTo("25P02");

    assertThat(totalBalance()).isEqualByComparingTo("2000.00");
    assertThat(readBalances()).satisfiesAnyOf(
            balances -> assertThat(balances)
                    .containsExactly("101=900.00", "202=1100.00"),
            balances -> assertThat(balances)
                    .containsExactly("101=1070.00", "202=930.00")
    );
}
```

Task implementation:

```java
private static AttemptOutcome brokenTransferAttempt(
        long fromId,
        long toId,
        BigDecimal amount,
        CountDownLatch firstRowsLocked,
        CountDownLatch requestSecondRow
) throws SQLException {
    try (Connection connection = open()) {
        connection.setAutoCommit(false);
        configureDeadlockTestSession(connection);

        try {
            lockRow(connection, fromId);
            debit(connection, fromId, amount);

            firstRowsLocked.countDown();
            await(requestSecondRow, "permission to lock destination");

            lockRow(connection, toId);
            credit(connection, toId, amount);
            connection.commit();
            return AttemptOutcome.committed(fromId);
        } catch (SQLException failure) {
            String failedTransactionState =
                    executeAndReturnSqlState(connection, "select 1");
            connection.rollback();
            return AttemptOutcome.failed(
                    fromId,
                    failure.getSQLState(),
                    failedTransactionState
            );
        }
    }
}

private static void configureDeadlockTestSession(
        Connection connection
) throws SQLException {
    try (Statement statement = connection.createStatement()) {
        statement.execute("set local deadlock_timeout = '100ms'");
        statement.execute("set local lock_timeout = '2s'");
        statement.execute("set local statement_timeout = '4s'");
    }
}

private static void lockRow(
        Connection connection,
        long accountId
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
            select id
            from account
            where id = ?
            for update
            """)) {
        statement.setLong(1, accountId);
        try (ResultSet rows = statement.executeQuery()) {
            if (!rows.next()) {
                throw new SQLException("Missing account " + accountId);
            }
        }
    }
}

private static void debit(
        Connection connection,
        long accountId,
        BigDecimal amount
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
            update account
            set balance = balance - ?
            where id = ?
              and balance >= ?
            """)) {
        statement.setBigDecimal(1, amount);
        statement.setLong(2, accountId);
        statement.setBigDecimal(3, amount);
        if (statement.executeUpdate() != 1) {
            throw new SQLException("Insufficient funds: " + accountId);
        }
    }
}

private static void credit(
        Connection connection,
        long accountId,
        BigDecimal amount
) throws SQLException {
    try (PreparedStatement statement = connection.prepareStatement("""
            update account
            set balance = balance + ?
            where id = ?
            """)) {
        statement.setBigDecimal(1, amount);
        statement.setLong(2, accountId);
        if (statement.executeUpdate() != 1) {
            throw new SQLException("Missing account " + accountId);
        }
    }
}

private static String executeAndReturnSqlState(
        Connection connection,
        String sql
) {
    try (Statement statement = connection.createStatement()) {
        statement.execute(sql);
        return "00000";
    } catch (SQLException failure) {
        return failure.getSQLState();
    }
}
```

`lock_timeout` lớn hơn `deadlock_timeout`, nên detector có cơ hội trả `40P01`
trước timeout. Outer `Future.get(8, TimeUnit.SECONDS)` vẫn là watchdog độc lập.

## Thí nghiệm 2 — Canonical order cho hai transfer cùng commit

Chạy solution thật qua Spring proxy, không gọi worker bằng `this`:

```java
@SpringBootTest
@Testcontainers
class OrderedTransferIT {

    @Autowired
    private OrderedTransferAttempt attempts;

    @Autowired
    private JdbcTemplate jdbc;

    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    @Test
    void opposingTransfersUseSameOrderAndPreserveTotal() throws Exception {
        var start = new CountDownLatch(1);

        TransferCommand aToB = command(
                101, 202, new BigDecimal("100.00")
        );
        TransferCommand bToA = command(
                202, 101, new BigDecimal("70.00")
        );

        Future<TransferReceipt> first = executor.submit(() -> {
            await(start, "common start");
            return attempts.execute(aToB);
        });
        Future<TransferReceipt> second = executor.submit(() -> {
            await(start, "common start");
            return attempts.execute(bToA);
        });

        start.countDown();
        first.get(8, TimeUnit.SECONDS);
        second.get(8, TimeUnit.SECONDS);

        assertThat(balance(101)).isEqualByComparingTo("970.00");
        assertThat(balance(202)).isEqualByComparingTo("1030.00");
        assertThat(totalBalance()).isEqualByComparingTo("2000.00");
    }
}
```

Đặt transaction-local `lock_timeout`/`statement_timeout` trong test profile để
regression fail bounded. Test này kiểm tra implementation gần production; proof
order còn được khóa bằng unit test tiếp theo.

## Thí nghiệm 3 — Contract test cho canonical comparator

Tách calculation thành value object nhỏ:

```java
record AccountLockOrder(long firstId, long secondId) {

    static AccountLockOrder of(long leftId, long rightId) {
        if (leftId == rightId) {
            throw new IllegalArgumentException("duplicate account");
        }
        return leftId < rightId
                ? new AccountLockOrder(leftId, rightId)
                : new AccountLockOrder(rightId, leftId);
    }
}

@Test
void directionDoesNotChangeLockOrder() {
    assertThat(AccountLockOrder.of(101, 202))
            .isEqualTo(new AccountLockOrder(101, 202));
    assertThat(AccountLockOrder.of(202, 101))
            .isEqualTo(new AccountLockOrder(101, 202));
}
```

Service phải dùng chính value object này, không tự lặp lại `min/max` ở nhiều
nơi. Với N resources, property test thêm permutations và assert cùng sorted
distinct sequence.

## Thí nghiệm 4 — Deadlock retry dùng transaction mới

Integration test dùng test-only hook sau first lock cho hai **first attempts**,
giống Experiment 1. Hook tắt sau khi cycle hình thành để victim retry có thể
tiến tiếp.

Một probe thread-safe ghi `txid_current()` ngay khi attempt bắt đầu:

```java
long transactionId = jdbc.queryForObject(
        "select txid_current()",
        Long.class
);
attemptProbe.record(command.commandId(), attemptNumber, transactionId);
```

Hai task gọi `TransferCoordinator.transfer(...)` qua Spring proxy. Assertions:

```java
start.countDown();
TransferReceipt firstReceipt = first.get(10, TimeUnit.SECONDS);
TransferReceipt secondReceipt = second.get(10, TimeUnit.SECONDS);

assertThat(firstReceipt).isNotNull();
assertThat(secondReceipt).isNotNull();
assertThat(attemptProbe.totalAttempts()).isEqualTo(3);
assertThat(attemptProbe.deadlockVictims()).isEqualTo(1);
assertThat(attemptProbe.transactionIds())
        .hasSize(3)
        .doesNotHaveDuplicates();
assertThat(totalBalance()).isEqualByComparingTo("2000.00");
assertThat(balance(101)).isEqualByComparingTo("970.00");
assertThat(balance(202)).isEqualByComparingTo("1030.00");
```

Ba transaction IDs tương ứng winner first attempt, victim first attempt và fresh
retry. Không assert command cụ thể là victim. Probe chỉ dành cho test và ghi
in-memory; production code không chờ barrier trong transaction.

Nếu test `@Retryable` dùng backoff thật, đặt values nhỏ nhưng khác zero trong test
profile và giữ outer watchdog. Có thể thay sleeper bằng recording test
implementation để assert số lần backoff mà không làm suite chậm.

## Thí nghiệm 5 — Lock timeout không phải deadlock

Một holder khóa A; waiter chỉ chờ A và không giữ resource mà holder cần. Graph
không có cycle. Waiter phải nhận `55P03` khi `lock_timeout` hết hạn:

```java
@Test
void oneWayWaitReturnsLockTimeoutNotDeadlock() throws Exception {
    var holderLocked = new CountDownLatch(1);
    var releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = open()) {
            connection.setAutoCommit(false);
            lockRow(connection, 101);
            holderLocked.countDown();
            await(releaseHolder, "release lock holder");
            connection.commit();
        }
        return null;
    });

    Future<String> waiter = executor.submit(() -> {
        await(holderLocked, "holder locked row 101");
        try (Connection connection = open()) {
            connection.setAutoCommit(false);
            try (Statement statement = connection.createStatement()) {
                statement.execute("set local lock_timeout = '150ms'");
                statement.execute("set local statement_timeout = '2s'");
            }
            try {
                lockRow(connection, 101);
                connection.commit();
                return "00000";
            } catch (SQLException failure) {
                connection.rollback();
                return failure.getSQLState();
            }
        }
    });

    assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
    releaseHolder.countDown();
    holder.get(5, TimeUnit.SECONDS);
}
```

Retry classifier không được gọi outcome này là `40P01`. Policy có thể retry lock
timeout riêng, nhưng phải dựa trên idempotency/deadline và điều tra long holder.

## Thí nghiệm 6 — Đóng connection giải phóng lock

Test mở holder connection, `BEGIN` và khóa A. Một waiter chuẩn bị xin A. Sau khi
đã phát tín hiệu waiter bắt đầu, test đóng holder connection thay vì commit:

```text
holder connection close
→ PostgreSQL abort holder transaction
→ row A lock được release
→ waiter acquire A và commit trong bounded deadline
```

Assertions:

```java
assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo("ACQUIRED");
assertThat(totalBalance()).isEqualByComparingTo("2000.00");
assertThat(countOpenTransactionsForTestUsers()).isZero();
```

Test này xác nhận database cleanup, không giải quyết ambiguous client outcome nếu
connection mất đúng lúc commit. Idempotency/status lookup cần test ở case domain.

## Thí nghiệm 7 — Quan sát blocking graph

Trong một one-way wait được giữ mở có kiểm soát, lấy waiter PID bằng
`select pg_backend_pid()` rồi query từ observer connection:

```sql
select a.pid,
       a.wait_event_type,
       a.wait_event,
       pg_blocking_pids(a.pid) as blocking_pids,
       a.xact_start,
       a.query_start
from pg_stat_activity a
where a.pid = :waiter_pid;
```

Dùng bounded polling helper với deadline, không dùng fixed sleep:

```java
static void awaitCondition(
        Duration timeout,
        BooleanSupplier condition
) {
    long deadline = System.nanoTime() + timeout.toNanos();
    while (System.nanoTime() < deadline) {
        if (condition.getAsBoolean()) {
            return;
        }
        java.util.concurrent.locks.LockSupport.parkNanos(
                Duration.ofMillis(10).toNanos()
        );
        if (Thread.currentThread().isInterrupted()) {
            throw new AssertionError("Interrupted while polling");
        }
    }
    throw new AssertionError("Condition not reached before " + timeout);
}
```

Assert `wait_event_type = 'Lock'` và `blocking_pids` chứa holder PID. Sau cleanup,
assert PID không còn chờ. `pg_blocking_pids` snapshot có thể đổi nhanh; nó là
diagnostic evidence, không phải business synchronization primitive.

## Helper đọc trạng thái

```java
private static BigDecimal totalBalance() throws SQLException {
    try (Connection connection = open();
            Statement statement = connection.createStatement();
            ResultSet row = statement.executeQuery(
                    "select sum(balance) from account"
            )) {
        row.next();
        return row.getBigDecimal(1);
    }
}

private static List<String> readBalances() throws SQLException {
    try (Connection connection = open();
            Statement statement = connection.createStatement();
            ResultSet rows = statement.executeQuery("""
                    select id, balance
                    from account
                    order by id
                    """)) {
        var result = new java.util.ArrayList<String>();
        while (rows.next()) {
            result.add(
                    rows.getLong("id")
                            + "="
                            + rows.getBigDecimal("balance").toPlainString()
            );
        }
        return List.copyOf(result);
    }
}
```

## Ma trận bao phủ

| Experiment | Cơ chế | Assertion chính |
| --- | --- | --- |
| 1 | Opposite row order + barrier | `40P01`, `25P02`, đúng một commit, total=`2000` |
| 2 | Ordered Spring worker | Hai commits, A=`970`, B=`1030`, total=`2000` |
| 3 | Pure canonical comparator | Hai hướng tạo cùng order `[101, 202]` |
| 4 | Retry coordinator + new Tx probe | Ba unique txids, một victim, cả hai commands hoàn tất |
| 5 | One-way row wait | `55P03`, không gắn nhãn deadlock |
| 6 | Holder connection close | Waiter tiến tiếp, không partial state/open transaction |
| 7 | Live database inspection | Waiter/holder PID nối được bằng `pg_blocking_pids` |

## Chống flaky

- Barrier đặt sau first row lock, không đặt trước transaction.
- `deadlock_timeout < lock_timeout < statement_timeout < Future watchdog`.
- Không assert actor cụ thể là victim.
- Reset rows ở mỗi test và không chạy suite này song song trên cùng schema.
- Mọi error path rollback/close connection bằng try-with-resources.
- Executor shutdown trong `finally`/lifecycle và có bounded termination.
- Khi watchdog fail, dump task stacks, `pg_stat_activity`, `pg_locks` và
  `pg_blocking_pids` trước cleanup.
- Stress test có thể lặp hữu hạn nhiều opposing commands, nhưng không thay
  deterministic cycle test.

## Xác minh trong production

Theo dõi:

```sql
select datname, deadlocks
from pg_stat_database
where datname = current_database();
```

Kết hợp với:

- application metric theo operation: deadlock, retry, exhaustion, latency;
- PostgreSQL logs chứa deadlock graph/query context;
- pool active/pending/acquisition timeout;
- transaction age và `idle in transaction`;
- deploy/version để phát hiện code path chưa dùng canonical order.

Alert nên dựa trên rate/baseline và user impact, không dựa vào một account ID có
high cardinality. Sau rollout, mục tiêu là cycle đã biết biến mất và mọi rare
victim còn lại được rollback/retry bounded mà không phá invariant.
