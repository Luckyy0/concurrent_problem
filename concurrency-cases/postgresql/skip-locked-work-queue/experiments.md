# Các thí nghiệm `SKIP LOCKED` với PostgreSQL thật

## Mục tiêu

- Plain SELECT cho hai workers cùng IDs.
- Blocking `FOR UPDATE` tạo one-way wait/`55P03`.
- `SKIP LOCKED` đi qua locked J1 và lấy J2.
- Concurrent batch claims không giao nhau.
- Rollback/crash release claim locks.
- Expired lease có thể reclaim; stale token complete affected-row `0`.
- Business assertion gồm ownership, status, attempt và idempotent effect.

> **Nói ngắn gọn:** test phải chứng minh disjoint ownership và recovery, không chỉ
> chứng minh query có chứa chuỗi `SKIP LOCKED`.

## PostgreSQL Testcontainers fixture

```java
@Testcontainers
class SkipLockedQueueIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("queue_cases")
                    .withUsername("cases")
                    .withPassword("cases");

    private final ExecutorService executor = Executors.newFixedThreadPool(4);

    @BeforeAll
    static void schema() throws SQLException {
        try (Connection connection = open();
                Statement sql = connection.createStatement()) {
            sql.execute("""
                    create table work_job (
                        job_id uuid primary key,
                        payload jsonb not null,
                        status varchar(16) not null,
                        priority integer not null,
                        available_at timestamptz not null,
                        claim_token uuid,
                        claimed_by varchar(128),
                        lease_until timestamptz,
                        attempt_count integer not null default 0,
                        completed_at timestamptz
                    )
                    """);
            sql.execute("""
                    create index ix_work_job_claim
                    on work_job(priority desc, available_at, job_id)
                    where status = 'READY'
                    """);
        }
    }

    @BeforeEach
    void seed() throws SQLException {
        try (Connection connection = open();
                Statement sql = connection.createStatement()) {
            sql.execute("truncate work_job");
            for (int i = 1; i <= 8; i++) {
                try (PreparedStatement insert = connection.prepareStatement("""
                        insert into work_job(
                          job_id, payload, status, priority, available_at
                        ) values (?, '{}'::jsonb, 'READY', 0, ?)
                        """)) {
                    insert.setObject(1, jobId(i));
                    insert.setTimestamp(
                            2,
                            Timestamp.from(Instant.parse(
                                    "2026-01-01T00:00:00Z"
                            ).plusSeconds(i))
                    );
                    insert.executeUpdate();
                }
            }
        }
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }

    static Connection open() throws SQLException {
        return DriverManager.getConnection(
                POSTGRES.getJdbcUrl(),
                POSTGRES.getUsername(),
                POSTGRES.getPassword()
        );
    }
}
```

## Helper điều phối

```java
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

Mọi session đặt `statement_timeout`, test dùng `Future.get(timeout)` làm outer
watchdog.

## Thí nghiệm 1 — Plain SELECT trả duplicate IDs

```java
@Test
void plainSelectDoesNotClaim() throws Exception {
    var bothSelected = new CountDownLatch(2);
    var returnRows = new CountDownLatch(1);

    Callable<List<UUID>> select = () -> {
        try (Connection connection = open()) {
            List<UUID> ids = selectReady(connection, 2, "");
            bothSelected.countDown();
            await(returnRows, "return duplicate selections");
            return ids;
        }
    };

    Future<List<UUID>> a = executor.submit(select);
    Future<List<UUID>> b = executor.submit(select);
    await(bothSelected, "both plain selects");
    returnRows.countDown();

    assertThat(a.get(5, TimeUnit.SECONDS))
            .containsExactlyElementsOf(b.get(5, TimeUnit.SECONDS));
    assertThat(readyCount()).isEqualTo(8);
}
```

## Thí nghiệm 2 — Blocking `FOR UPDATE` tạo convoy

Holder lock J1. Waiter chạy ordered `LIMIT 1 FOR UPDATE` với
`lock_timeout='200ms'`:

```java
assertThat(waiter.get(5, TimeUnit.SECONDS)).isEqualTo("55P03");
assertThat(readyCount()).isEqualTo(8);
releaseHolder.countDown();
holder.get(5, TimeUnit.SECONDS);
```

Wait graph chỉ có waiter → holder, không có cycle. Test không gọi outcome này là
deadlock.

## Thí nghiệm 3 — `SKIP LOCKED` lấy row kế tiếp

```java
@Test
void skipLockedPassesLockedFirstJob() throws Exception {
    var firstLocked = new CountDownLatch(1);
    var release = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> {
        try (Connection connection = open()) {
            connection.setAutoCommit(false);
            lockJob(connection, jobId(1));
            firstLocked.countDown();
            await(release, "release J1");
            connection.rollback();
        }
        return null;
    });

    await(firstLocked, "J1 locked");
    try (Connection worker = open()) {
        worker.setAutoCommit(false);
        List<UUID> ids = selectReady(
                worker,
                1,
                "for update skip locked"
        );
        assertThat(ids).containsExactly(jobId(2));
        worker.rollback();
    } finally {
        release.countDown();
    }
    holder.get(5, TimeUnit.SECONDS);
}
```

## Thí nghiệm 4 — Hai atomic claims tạo disjoint batches

```java
@Test
void concurrentClaimBatchesNeverOverlap() throws Exception {
    var start = new CountDownLatch(1);

    Future<List<ClaimedJob>> a = executor.submit(() -> {
        await(start, "common claim start");
        return claimService.claimBatch("A", 3, Duration.ofMinutes(1));
    });
    Future<List<ClaimedJob>> b = executor.submit(() -> {
        await(start, "common claim start");
        return claimService.claimBatch("B", 3, Duration.ofMinutes(1));
    });

    start.countDown();
    List<ClaimedJob> first = a.get(8, TimeUnit.SECONDS);
    List<ClaimedJob> second = b.get(8, TimeUnit.SECONDS);

    assertThat(first).hasSize(3);
    assertThat(second).hasSize(3);
    assertThat(first).extracting(ClaimedJob::jobId)
            .doesNotContainAnyElementsOf(
                    second.stream().map(ClaimedJob::jobId).toList()
            );
    assertThat(Stream.concat(first.stream(), second.stream())
            .map(ClaimedJob::jobId)
            .distinct()).hasSize(6);
    assertThat(processingCount()).isEqualTo(6);
    assertThat(readyCount()).isEqualTo(2);
    assertThat(allClaimTokens()).doesNotContainNull().doesNotHaveDuplicates();
}
```

Service calls phải đi qua Spring proxy để returned batches chỉ được dùng sau
commit.

## Atomic claim helper SQL test

Raw JDBC version map `UPDATE RETURNING`:

```java
static List<ClaimedJob> claim(
        Connection connection,
        String worker,
        int limit
) throws SQLException {
    connection.setAutoCommit(false);
    try (PreparedStatement sql = connection.prepareStatement("""
            with candidates as (
              select job_id
              from work_job
              where status = 'READY'
                and available_at <= clock_timestamp()
              order by priority desc, available_at, job_id
              for update skip locked
              limit ?
            )
            update work_job j
            set status='PROCESSING',
                claim_token=gen_random_uuid(),
                claimed_by=?,
                lease_until=clock_timestamp() + interval '1 minute',
                attempt_count=attempt_count+1
            from candidates c
            where j.job_id=c.job_id
            returning j.job_id, j.claim_token, j.lease_until,
                      j.attempt_count
            """)) {
        sql.setInt(1, limit);
        sql.setString(2, worker);
        List<ClaimedJob> result = mapRows(sql.executeQuery());
        connection.commit();
        return result;
    } catch (SQLException failure) {
        connection.rollback();
        throw failure;
    }
}
```

## Thí nghiệm 5 — Rollback làm job claimable lại

Một connection chạy claim SQL nhưng rollback thay commit. Sau rollback, second
connection claim:

```java
assertThat(firstUncommittedIds).containsExactly(jobId(1));
assertThat(secondCommittedIds).containsExactly(jobId(1));
assertThat(job(jobId(1)).status()).isEqualTo("PROCESSING");
assertThat(job(jobId(1)).attemptCount()).isEqualTo(1);
```

Attempt increment của rolled-back claim không tồn tại. Connection close không
commit cũng cho cùng cleanup outcome.

## Thí nghiệm 6 — Stale completion bị từ chối

```text
A claim J1/token-A
test set lease_until vào quá khứ
sweeper requeue J1
B claim J1/token-B
A complete(token-A) → 0
B complete(token-B) → 1
```

Assertions:

```java
assertThat(complete(jobId(1), tokenA)).isZero();
assertThat(complete(jobId(1), tokenB)).isEqualTo(1);
assertThat(job(jobId(1)).status()).isEqualTo("DONE");
assertThat(job(jobId(1)).attemptCount()).isEqualTo(2);
```

Không dùng system sleep để chờ lease; test update `lease_until` có kiểm soát hoặc
inject database clock abstraction.

## Thí nghiệm 7 — External effect idempotent sau crash

Fake sink lưu unique `effect_key = job_id`:

```text
B claim J1 → sink apply(J1) → simulate crash trước complete
expire/reclaim → C process J1 → sink apply(J1) replay
C complete current token
```

Assert:

```java
assertThat(sink.deliveryAttempts(jobId(1))).isEqualTo(2);
assertThat(sink.committedEffects(jobId(1))).isEqualTo(1);
assertThat(job(jobId(1)).status()).isEqualTo("DONE");
```

Fake sink phải model atomic unique claim; một in-memory list không đồng bộ không
phải bằng chứng idempotency.

## Thí nghiệm 8 — Table lock vẫn có thể chặn

Session DDL:

```sql
lock table work_job in access exclusive mode;
```

Poller `FOR UPDATE SKIP LOCKED` vẫn cần relation `ROW SHARE`, nên nhận `55P03`
theo bounded `lock_timeout`. Test này khóa nuance: skip chỉ áp dụng row-level
locks.

## Thí nghiệm 9 — Fairness và starvation

Hai tests riêng:

1. Không contention: claim từng batch size 1 và assert order theo
   `priority DESC, available_at, job_id`.
2. Holder giữ oldest row: N bounded polls claim later rows; oldest không xuất hiện
   khi locked. Sau release/rollback, next poll phải claim oldest.

Không assert strict FIFO dưới contention. Stress test hữu hạn thêm priority mix,
ghi oldest-ready age và bảo đảm mọi job cuối cùng DONE/DEAD trong deadline.

## Core helpers

```java
static List<UUID> selectReady(
        Connection connection,
        int limit,
        String lockingClause
) throws SQLException {
    String sqlText = """
            select job_id
            from work_job
            where status='READY'
              and available_at <= clock_timestamp()
            order by priority desc, available_at, job_id
            limit ?
            """ + lockingClause;
    try (PreparedStatement sql = connection.prepareStatement(sqlText)) {
        sql.setInt(1, limit);
        try (ResultSet rows = sql.executeQuery()) {
            var ids = new ArrayList<UUID>();
            while (rows.next()) {
                ids.add(rows.getObject(1, UUID.class));
            }
            return List.copyOf(ids);
        }
    }
}
```

Production code không build SQL từ untrusted input; helper chỉ chọn constant test
clauses.

## Ma trận bao phủ

| Thí nghiệm | Cơ chế | Business assertion |
| --- | --- | --- |
| 1 | Plain SELECT barrier | Duplicate IDs, state vẫn READY |
| 2 | Blocking row lock | `55P03`, convoy không phải deadlock |
| 3 | `SKIP LOCKED` | J1 locked, worker lấy J2 |
| 4 | Atomic concurrent claim | Hai disjoint batches, unique tokens |
| 5 | Rollback/connection close | Same job claimable, attempt không tăng giả |
| 6 | Lease + token | Stale complete `0`, current owner `1` |
| 7 | Idempotent sink | Hai deliveries, một effect |
| 8 | `ACCESS EXCLUSIVE` | Skip không bỏ qua table lock |
| 9 | Order/starvation | Stable preference, eventual claim sau release |

## Chống flaky và xác minh production

- Barrier/latches đặt sau exact lock/read point.
- `lock_timeout < statement_timeout < Future` watchdog.
- Không dùng fixed sleep; mọi wait bounded.
- Rollback/close connections và shutdown executor trong cleanup.
- Khi timeout, thu `pg_stat_activity`, `pg_locks`, `pg_blocking_pids`.
- Theo dõi claim latency, empty poll, oldest READY age, expired lease, reclaim,
  stale completion, DEAD và downstream dedupe.
- Query plan/index được kiểm tra với volume gần production; không tạo benchmark
  numbers từ Testcontainers laptop.
