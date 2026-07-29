# Deterministic visibility experiments

## Mục tiêu

Tests phải chứng minh:

1. writer đã execute/flush UPDATE nhưng chưa commit;
2. RU-requested reader hoàn tất plain SELECT mà không block;
3. reader thấy old committed value `20`, không thấy `80`;
4. writer rollback giữ final `20`; writer commit làm later statement thấy `80`;
5. isolation label không được dùng thay behavior assertion;
6. committed heartbeat/state-machine alternatives visible đúng.

> **Nói ngắn gọn:** giữ writer transaction mở bằng gate, đọc từ connection khác,
> rồi quyết định commit hoặc rollback sau assertion.

## PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class DirtyReadExpectationIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Test method không annotated `@Transactional`; writer, reader và inspector là
Spring beans/proxies riêng.

Schema:

```sql
create table job_run (
    job_id uuid primary key,
    status varchar(32) not null,
    progress_percent integer not null
        check (progress_percent between 0 and 100),
    generation bigint not null,
    owner_token uuid,
    lease_until timestamptz
);

create table job_attempt_heartbeat (
    job_id uuid not null,
    generation bigint not null,
    owner_token uuid not null,
    progress_percent integer not null
        check (progress_percent between 0 and 100),
    last_seen_at timestamptz not null,
    primary key (job_id, generation)
);
```

Fixture setup commit:

```sql
insert into job_run(
    job_id, status, progress_percent, generation
)
values (:jobId, 'RUNNING', 20, 7);
```

## Writer gate

```java
final class WriterGate {
    private final CountDownLatch flushed = new CountDownLatch(1);
    private final CountDownLatch allowCompletion =
        new CountDownLatch(1);

    void afterFlush() {
        flushed.countDown();
        awaitOrFail(allowCompletion, Duration.ofSeconds(5));
    }

    void awaitFlushed() {
        awaitOrFail(flushed, Duration.ofSeconds(5));
    }

    void release() {
        allowCompletion.countDown();
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for writer gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

Writer calls `entityManager.flush()`/repository `flush()` before
`gate.afterFlush()`. `finally` luôn release gate.

## Reader observation

```java
public record ReadObservation(
    int progress,
    String reportedIsolation,
    int jdbcIsolation
) {}
```

```java
@Transactional(
    isolation = Isolation.READ_UNCOMMITTED,
    readOnly = true
)
public ReadObservation observe(UUID jobId) {
    int jdbcIsolation = jdbc.execute(
        (ConnectionCallback<Integer>)
            Connection::getTransactionIsolation
    );
    String reported = jdbc.queryForObject(
        "select current_setting('transaction_isolation')",
        String.class
    );
    int progress = jdbc.queryForObject(
        """
        select progress_percent
        from job_run
        where job_id = ?
        """,
        Integer.class,
        jobId
    );
    return new ReadObservation(
        progress,
        reported,
        jdbcIsolation
    );
}
```

Driver/server version có thể expose requested/alias label khác nhau. Test record cả
hai nhưng correctness assertion là `progress`.

## Experiment 1 — RU reader không thấy uncommitted UPDATE

```java
@Test
void readUncommittedRequestStillCannotSeeDirtyRow()
        throws Exception {
    WriterGate gate = writerProbe.install();
    ExecutorService actor = Executors.newSingleThreadExecutor();

    try {
        Future<Throwable> writerOutcome = actor.submit(() ->
            catchThrowable(() ->
                processor.processCurrentUnit(
                    JOB_ID,
                    true // throw after gate -> rollback
                )
            )
        );

        gate.awaitFlushed();

        ReadObservation observed = watchdog.observe(JOB_ID);

        assertThat(observed.progress()).isEqualTo(20);
        assertThat(observed.reportedIsolation())
            .isIn("read uncommitted", "read committed");
        assertThat(observed.jdbcIsolation()).isIn(
            Connection.TRANSACTION_READ_UNCOMMITTED,
            Connection.TRANSACTION_READ_COMMITTED
        );

        gate.release();
        Throwable failure =
            writerOutcome.get(5, TimeUnit.SECONDS);
        assertThat(failure)
            .isInstanceOf(ProcessingFailedException.class);

        assertThat(inspector.progress(JOB_ID)).isEqualTo(20);
    } finally {
        gate.release();
        writerProbe.clear();
        actor.shutdownNow();
        assertThat(actor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Watchdog call return trong khi writer gate vẫn closed, chứng minh plain SELECT
không chờ writer completion và đọc old committed version.

## Experiment 2 — Before commit old, after commit new

```java
@Test
void laterStatementSeesValueOnlyAfterWriterCommits()
        throws Exception {
    WriterGate gate = writerProbe.install();
    ExecutorService actor = Executors.newSingleThreadExecutor();

    try {
        Future<Void> writer = actor.submit(() -> {
            processor.processCurrentUnit(JOB_ID, false);
            return null;
        });

        gate.awaitFlushed();
        assertThat(watchdog.observe(JOB_ID).progress())
            .isEqualTo(20);

        gate.release();
        writer.get(5, TimeUnit.SECONDS);

        assertThat(watchdog.observe(JOB_ID).progress())
            .isEqualTo(80);
    } finally {
        gate.release();
        writerProbe.clear();
        actor.shutdownNow();
        assertThat(actor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Second watchdog invocation tạo transaction/statement snapshot mới sau commit. Đây
không phải dirty read; nó là normal committed visibility.

## Experiment 3 — Uncommitted INSERT cũng invisible

Writer transaction:

```sql
insert into job_run(
    job_id, status, progress_percent, generation
)
values (:newJob, 'RUNNING', 1, 1);
-- flush, no commit
```

RU reader:

```java
assertThat(watchdog.find(NEW_JOB_ID)).isEmpty();
```

Sau rollback vẫn empty; sau commit và new reader transaction thì present. Test
ngăn implementation chỉ đúng cho UPDATE example nhưng sai assumption với inserts.

## Experiment 4 — Locking reader waits, không dirty-read

Reader bean:

```java
@Transactional
public JobSnapshot lockAndRead(UUID jobId) {
    return jobs.findByIdForUpdate(jobId)
        .orElseThrow()
        .snapshot();
}
```

Giữ writer ở gate, start locking reader future. Dùng direct admin connection và
bounded Awaitility:

```sql
select count(*)
from pg_stat_activity
where datname = current_database()
  and wait_event_type = 'Lock';
```

Assert waiter count >= 1. Sau:

- writer commit -> locking reader returns progress 80;
- writer rollback -> locking reader returns progress 20.

Mọi future có timeout và holder release trong `finally`. Locking behavior không
được dùng cho dashboard polling; experiment chỉ phân biệt wait-for-outcome với
dirty visibility.

## Experiment 5 — Independent heartbeat visible, main Tx vẫn open

Processor main transaction update job row `80` và dừng ở gate. Một separate bean
publishes heartbeat ở bảng riêng:

```java
heartbeatPublisher.publish(new Heartbeat(
    JOB_ID,
    7,
    OWNER_TOKEN,
    80,
    TEST_NOW
));
```

Publisher dùng `REQUIRES_NEW`, commit trước return. Assertions khi main Tx còn open:

```java
assertThat(watchdog.observe(JOB_ID).progress()).isEqualTo(20);
assertThat(heartbeatReader.read(JOB_ID, 7).progressPercent())
    .isEqualTo(80);
```

Sau main rollback:

```java
assertThat(inspector.progress(JOB_ID)).isEqualTo(20);
assertThat(heartbeatReader.read(JOB_ID, 7).progressPercent())
    .isEqualTo(80);
```

Test chứng minh contract: heartbeat là committed attempt state, không phải final
job progress. Bảng riêng tránh inner transaction chờ row bị outer transaction lock.

## Experiment 6 — Same-row `REQUIRES_NEW` là anti-pattern

Outer Tx lock/update `job_run`, rồi synchronous inner NEW cố update same row. Dùng
bounded `lock_timeout` trong test và assert inner fails/rolls back thay vì treo.

```text
outer owns row lock
outer waits inner return
inner waits outer row lock
```

Không cần wait-for cycle giữa PostgreSQL sessions để application tự chặn theo call
dependency. Heartbeat resource/boundary phải được thiết kế tách biệt.

## Experiment 7 — Atomic recovery claim có đúng một winner

Expire committed lease, start hai watchdog actors qua gate, rồi cùng chạy:

```sql
update job_run
set generation = generation + 1,
    owner_token = :owner,
    lease_until = :leaseUntil
where job_id = :jobId
  and generation = 7
  and status = 'RUNNING'
  and lease_until < :databaseNow
returning generation, owner_token;
```

Test:

```java
assertThat(results.successCount()).isEqualTo(1);
assertThat(results.noOpCount()).isEqualTo(1);
assertThat(inspector.generation(JOB_ID)).isEqualTo(8);
assertThat(inspector.ownerToken(JOB_ID))
    .isEqualTo(results.winnerToken());
```

Dirty-read expectation không xuất hiện; atomic write chọn recovery owner.

## Experiment 8 — Label test không thay behavior test

Log/record:

```text
PostgreSQL server version
pgJDBC version
requested Spring isolation
Connection.getTransactionIsolation()
current_setting('transaction_isolation')
observed progress during uncommitted writer
```

Test suite chỉ fail correctness khi dirty value visible hoặc committed invariant
sai. Label thay đổi giữa driver/server profiles được review như compatibility
signal, không tự động kết luận visibility regression.

## Inspector

```java
@Service
class CommittedJobInspector {
    private final JdbcTemplate jdbc;

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    int progress(UUID jobId) {
        return jdbc.queryForObject(
            """
            select progress_percent
            from job_run
            where job_id = ?
            """,
            Integer.class,
            jobId
        );
    }
}
```

Inspector chạy sau writer future completion cho final state assertions.

## Coverage matrix

| Scenario | Reader timing | Visible progress |
| --- | --- | --- |
| RU plain SELECT, writer open | Before completion | 20 |
| RU plain SELECT, writer rollback | After rollback | 20 |
| RU plain SELECT, writer commit | New statement after commit | 80 |
| RU uncommitted INSERT | Before completion | No row |
| `FOR UPDATE`, writer commit | After wait | 80 |
| `FOR UPDATE`, writer rollback | After wait | 20 |
| Separate heartbeat commit | Main Tx open | Heartbeat 80, job 20 |

## Chống flaky

- Writer flush gate tạo order trước reader.
- Mọi latch, future, Awaitility wait và executor termination có timeout.
- `finally` release gate trước shutdown.
- Test class chạy `SAME_THREAD`; probes là single-scenario.
- Test methods không có outer transaction.
- Reader/writer/inspector là proxied beans với connections riêng.
- Không dùng H2 cho PostgreSQL visibility evidence.
- Assert row behavior, không chỉ isolation label.
- Lock tests có bounded timeout/admin diagnostics.

## Production verification

Theo dõi:

- long-running processor transaction age;
- committed checkpoint/heartbeat age;
- requested/reported isolation label theo database profile;
- recovery claims: winner/no-op;
- generation/fencing rejection;
- duplicate processors;
- main commit/rollback và independent heartbeat commits.

Không giải thích committed-old read là cache bug trước khi xác nhận transaction
boundary/MVCC timing. Đây là expected PostgreSQL behavior.
