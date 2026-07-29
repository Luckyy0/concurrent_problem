# Committed coordination and progress solutions

## Mục tiêu thiết kế

Một solution đúng phải chọn rõ:

```text
final business state:
  visible only after its transaction commits

attempt progress/heartbeat:
  independently committed with owner/generation semantics

recovery:
  atomic claim, never "read stale -> blindly start"
```

> **Nói ngắn gọn:** publish một committed signal có tên/semantics đúng, không dùng
> uncommitted row làm message bus.

## Solution 1 — Chấp nhận committed-only observation

Nếu watchdog/dashboard không cần intermediate progress:

```java
@Service
public class JobStatusReader {
    private final JobRunRepository jobs;

    public JobStatusReader(JobRunRepository jobs) {
        this.jobs = jobs;
    }

    @Transactional(readOnly = true)
    public JobSnapshot read(UUID jobId) {
        return jobs.findById(jobId)
            .orElseThrow()
            .snapshot();
    }
}
```

Dùng default/explicit `READ_COMMITTED`; document rằng value chỉ đổi sau writer
commit. UI hiển thị `lastCommittedAt`, không ngụ ý entity state đang xử lý bên
trong transaction.

Phù hợp khi operation ngắn và stale-until-commit nằm trong SLO.

## Solution 2 — Chia processing thành committed checkpoints

Long workflow được chia thành chunks. Mỗi chunk:

```text
BEGIN
  apply chunk output idempotently
  update committed progress/checkpoint
COMMIT
```

Repository claim dùng một atomic insert:

```sql
insert into chunk_result(chunk_id, job_id, status)
values (:chunkId, :jobId, 'APPLIED')
on conflict (chunk_id) do nothing;
```

```java
@Service
public class JobChunkService {
    private final JobRunRepository jobs;
    private final ChunkResultRepository results;

    public JobChunkService(
        JobRunRepository jobs,
        ChunkResultRepository results
    ) {
        this.jobs = jobs;
        this.results = results;
    }

    @Transactional
    public ChunkOutcome applyChunk(
        UUID jobId,
        UUID chunkId,
        int checkpoint
    ) {
        int claimed = results.claim(chunkId, jobId);
        if (claimed == 0) {
            ChunkResult existing =
                results.findByChunkId(chunkId).orElseThrow();
            return ChunkOutcome.replayed(existing);
        }

        JobRun job = jobs.findByIdForUpdate(jobId)
            .orElseThrow();
        job.advanceCommittedCheckpoint(checkpoint);
        ChunkResult result =
            results.findByChunkId(chunkId).orElseThrow();
        return ChunkOutcome.applied(result);
    }
}
```

Unique `chunk_id` cùng `ON CONFLICT DO NOTHING` biến duplicate arrival thành one
atomic claim. Watchdog đọc checkpoint đã commit; rollback một chunk cũng rollback
claim và không publish progress của chunk đó.


Trade-off:

- không còn all-or-nothing cho toàn workflow;
- cần resumable/idempotent chunks;
- schema/state machine phức tạp hơn;
- transaction/lock/connection duration ngắn hơn.

## Solution 3 — Independent heartbeat/progress record

Khi main business unit không thể chia nhưng cần liveness signal, dùng bảng riêng:

```sql
create table job_attempt_heartbeat (
    job_id uuid not null,
    generation bigint not null,
    owner_token uuid not null,
    progress_percent integer not null,
    last_seen_at timestamptz not null,
    primary key (job_id, generation),
    check (progress_percent between 0 and 100)
);
```

Publisher là bean riêng để proxy tạo independent transaction:

```java
@Service
public class JobHeartbeatPublisher {
    private final JobHeartbeatRepository heartbeats;

    public JobHeartbeatPublisher(
        JobHeartbeatRepository heartbeats
    ) {
        this.heartbeats = heartbeats;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void publish(Heartbeat heartbeat) {
        int changed = heartbeats.upsertIfOwnerMatches(
            heartbeat.jobId(),
            heartbeat.generation(),
            heartbeat.ownerToken(),
            heartbeat.progressPercent(),
            heartbeat.observedAt()
        );
        if (changed != 1) {
            throw new StaleJobOwnerException(
                heartbeat.jobId(),
                heartbeat.generation()
            );
        }
    }
}
```

SQL upsert:

```sql
insert into job_attempt_heartbeat(
    job_id,
    generation,
    owner_token,
    progress_percent,
    last_seen_at
)
values (
    :jobId,
    :generation,
    :ownerToken,
    :progress,
    :observedAt
)
on conflict (job_id, generation)
do update
set progress_percent = excluded.progress_percent,
    last_seen_at = excluded.last_seen_at
where job_attempt_heartbeat.owner_token = excluded.owner_token
  and job_attempt_heartbeat.progress_percent
      <= excluded.progress_percent;
```

Heartbeat commit survives main transaction rollback, nên tên của nó phải là
`attempt progress`, không phải final success.

### Tránh self-deadlock

Không để outer transaction lock row X rồi gọi `REQUIRES_NEW` update cùng X:

```text
outer Tx holds X
inner NEW waits X
outer waits inner return
```

Dùng heartbeat table/resource riêng hoặc publish ngoài outer lock scope. Pool phải
có capacity cho nested `REQUIRES_NEW`; independent commit là intentional contract.

## Solution 4 — Atomic recovery claim

Watchdog không start recovery chỉ từ read. Nó attempt one conditional write:

```sql
update job_run
set generation = generation + 1,
    owner_token = :newOwnerToken,
    lease_until = :newLeaseUntil,
    status = 'RUNNING'
where job_id = :jobId
  and generation = :observedGeneration
  and status = 'RUNNING'
  and lease_until < :databaseNow
returning generation, owner_token, lease_until;
```

Outcome:

| Returned rows | Ý nghĩa |
| --- | --- |
| 1 | Watchdog thắng claim; start generation mới |
| 0 | State/lease changed; reload/no-op |

Database time/lease/fencing design phải nhất quán. Mọi side effect của recovery
mang generation/fencing token để stale processor không commit sau khi mất quyền.

Atomic claim bảo vệ multiple watchdogs/multi-instance; `READ_UNCOMMITTED` không.

## Solution 5 — Wait for writer outcome bằng locking read

Nếu use case thật sự cần chờ current writer complete:

```sql
select job_id, status, progress_percent, generation
from job_run
where job_id = :jobId
for update;
```

Reader blocks/timeout tới writer commit/rollback, rồi đọc outcome phù hợp. Nó không
thấy uncommitted state.

Phù hợp cho short critical transaction, không phù hợp dashboard polling/long
processor vì:

- giữ connection khi wait;
- tăng lock queue/tail latency;
- cần timeout/deadlock handling;
- không cung cấp liveness trong open transaction.

## Solution 6 — Durable event/outbox

Khi consumers cần react sau commit:

```text
business Tx
  -> mutate job state
  -> insert outbox event
COMMIT

publisher
  -> deliver committed event
```

Outbox event không visible/delivered như durable committed record trước transaction
commit. Consumers xử lý redelivery bằng inbox/idempotency.

PostgreSQL `NOTIFY` có commit-time delivery semantics nhưng không phải durable
queue; dùng nó chỉ khi loss/reconnect contract phù hợp.

## Portability contract

Application-supported databases phải pass behavior tests:

```text
uncommitted row is never used for coordination
committed checkpoint visibility works
atomic recovery claim has one winner
rollback does not publish final success
```

Không branch logic theo `"read uncommitted"` string. Nếu cần database-specific
feature, isolate adapter và document guarantees.

## Failure behavior

| Failure | Durable observation |
| --- | --- |
| Main Tx rollback | No final state; prior checkpoints remain |
| Heartbeat Tx commit, main Tx rollback | Attempt heartbeat remains, final success absent |
| Heartbeat publisher fails | Watchdog may expire lease; processor must fence |
| Multiple watchdogs | Conditional claim chooses one |
| Outbox delivery duplicate | Inbox/idempotency replays |
| Locking reader timeout | No dirty data; explicit timeout outcome |

## Timeout và recovery budget

- heartbeat interval nhỏ hơn lease duration với margin cho scheduling/network;
- watchdog dùng database time hoặc controlled clock semantics;
- recovery claim có bounded transaction/lock timeout;
- no immediate retry storm on zero-row claim;
- processor checks ownership before checkpoint/final commit;
- cancellation/crash cleanup không dựa vào in-memory flag.

Không có universal timing value; tune từ pause distribution, SLO và failure
detector tolerance.

## Trade-off comparison

| Lựa chọn | Visibility | Atomicity | Complexity | Use case |
| --- | --- | --- | --- | --- |
| Committed-only final state | Sau main commit | Full unit | Thấp | Short operation |
| Chunk checkpoints | Sau mỗi chunk commit | Per chunk | Vừa | Resumable workflow |
| Independent heartbeat | Near-real-time committed attempt state | Tách final outcome | Vừa-cao | Long-running work |
| Atomic lease claim | One recovery owner | Claim only | Vừa | Watchdog coordination |
| `FOR UPDATE` wait | Sau writer completion | Same row lock | Vừa | Short synchronous wait |
| Outbox events | After commit, async | Atomic DB+event record | Cao | Durable consumers |

## Recommendation cho case này

1. Bỏ `READ_UNCOMMITTED` dependency.
2. Với job dài, dùng committed chunk checkpoints hoặc heartbeat table riêng.
3. Phân biệt attempt progress với final outcome.
4. Watchdog recovery phải atomic claim generation/lease.
5. Fencing stale processor trước mọi durable mutation.
6. Test commit và rollback visibility trên PostgreSQL thật.

## Production checklist

### Semantics

- [ ] Không decision nào phụ thuộc dirty read.
- [ ] Progress, heartbeat và final state có tên/contract riêng.
- [ ] Rollback không publish final success.
- [ ] Reported isolation label không được dùng làm behavior proof.
- [ ] Cross-database guarantees được integration test.

### Coordination

- [ ] Recovery dùng atomic claim, không check-then-act.
- [ ] Owner generation/token được validate khi write.
- [ ] Chunk/command delivery có idempotency key.
- [ ] `REQUIRES_NEW` independent commit là intentional.
- [ ] Không inner NEW update row bị outer Tx lock.

### Operations

- [ ] Heartbeat/lease timing có failure margin.
- [ ] Long transaction age được theo dõi.
- [ ] Duplicate recovery và stale-owner rejection có metrics.
- [ ] Lock waits/timeouts bounded nếu dùng locking reader.
- [ ] Reconciliation phân biệt attempt và terminal outcomes.
