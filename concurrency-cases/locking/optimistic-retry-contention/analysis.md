# Phân tích retry amplification và progress

## Trạng thái ban đầu

Wallet `77`: points `100`, version `10`. Tám unique commands cùng cộng `10`.

## Interleaving

| Pha | Winner | Bảy losers |
| --- | --- | --- |
| Load | đọc `100/v10` | cùng đọc `100/v10` |
| Flush | affected `1`, v11 | affected `0` sau winner |
| Retry ngay | hoàn tất | cùng reload v11 |
| Flush tiếp | một loser thắng v12 | sáu losers lại conflict |

Nếu mọi loser retry đồng nhịp, số attempts có thể lớn hơn nhiều số commands.
Correctness vẫn giữ nhờ version predicate, nhưng throughput/tail latency suy giảm.

> **Nói ngắn gọn:** optimistic locking có thể giữ dữ liệu đúng trong khi hệ thống
> progress rất tệ; retry policy là liveness/capacity contract.

## Safety và liveness

- Safety: points không mất/không duplicate; version tăng theo committed credits.
- Liveness: command kết thúc success/replay/exhaustion trong deadline.
- Fairness: không guarantee command nào sẽ thắng; một actor có thể starve.

Final success không đủ; test/metrics phải đo attempts và exhaustion.

## Vì sao transaction mới bắt buộc?

Sau optimistic conflict:

- transaction cũ rollback-only;
- persistence context chứa entity/version stale;
- command record insert của attempt rollback;
- retry cần snapshot mới thấy winner.

Coordinator catch sau transactional proxy rollback. Backoff diễn ra ngoài
transaction. Attempt mới kiểm tra `reward_credit(command_id)` trước, rồi load
wallet current version và revalidate.

## Amplification model định tính

Với `R` requests và tối đa `A` attempts, database work có upper bound theo policy
gần `R × A`, chưa kể reads/flushes. Không dùng công thức này làm benchmark; nó chỉ
cho thấy tăng attempt cap trực tiếp mở rộng worst-case load.

Immediate retry làm collision window trùng nhau. Exponential backoff tách dần
actors; jitter tránh cùng wake-up. Backoff không sửa hot-key capacity nếu arrival
rate dài hạn vượt service rate.

## Deadline và attempt cap

Cần cả hai:

- attempt cap chặn một command tạo vô hạn work;
- overall deadline chặn slow connection/query/backoff vượt latency budget;
- cancellation/interrupt dừng work khi caller không còn chờ;
- cleanup time phải nằm trong budget.

Policy kiểm tra deadline trước attempt mới và sau conflict trước backoff. Delay bị
clamp theo remaining time.

## Idempotency

`reward_credit.command_id` primary key nằm cùng transaction với wallet update.

- conflict rollback: cả delta và record biến mất;
- retry same ID: apply một lần;
- response loss after commit: next call replay record;
- concurrent duplicate ID: unique conflict rồi fresh read committed result.

Idempotency không ngăn different commands cạnh tranh cùng wallet.

## Business revalidation

Fresh attempt không chỉ reload version. Nó kiểm tra wallet active, command chưa
commit, delta/rule còn hợp lệ. Nếu winner làm wallet suspended/cap reached,
loser trả domain rejection, không tiếp tục retry.

## Exception classification

Allowlist `ObjectOptimisticLockingFailureException` cho aggregate/mutation dự kiến.
Không retry:

- invalid/suspended;
- unique failure không phải same command;
- timeout/outage nếu policy chưa phê duyệt;
- cancellation;
- programming/mapping error;
- external outcome không rõ.

Exception có thể lộ ở flush/commit; catch bao lời gọi proxy đầy đủ.

## Backoff và resource lifetime

Attempt rollback release PostgreSQL row locks/connection trước delay. Backoff giữ
application task/thread hoặc scheduled continuation, nhưng không giữ transaction.
Virtual threads không làm database capacity vô hạn.

## Crash và ambiguous commit

Crash trước commit rollback attempt. Crash sau commit trước response để command
outcome ambiguous; same command ID replay `reward_credit`. Attempt number không
được dùng làm identity nghiệp vụ.

External events dùng outbox trong successful transaction. Retry không gọi remote
service trước commit.

## Multi-instance

Version/unique constraints tại PostgreSQL coordinate mọi instances. Backoff jitter
phải độc lập giữa processes; local semaphore có thể admission-control một node
nhưng không phải correctness boundary.

Scale-out tăng contenders trên hot wallet. Nếu conflict/exhaustion bền vững, chọn
atomic delta, partition ownership, queue hoặc pessimistic strategy ở `LOCK-005`.

## Root cause theo layer

| Layer | Vai trò |
| --- | --- |
| Application | Retry scope, idempotency, deadline và business revalidation |
| Spring | Proxy/advisor/propagation tạo hoặc phá fresh boundary |
| Hibernate | Versioned flush và rollback-only persistence context |
| PostgreSQL | Row lock, affected-row/version/unique constraints |

## Failure outcomes

| Outcome | Hành vi |
| --- | --- |
| First-attempt win | Commit, no backoff |
| Conflict rồi win | Rollback, backoff, fresh commit |
| Duplicate command | Replay durable credit |
| Business rejection | Commit/reject theo domain, không retry |
| Deadline/attempt exhausted | Stop, explicit busy/contention failure |
| Interrupted | Restore/carry cancellation, stop |

## Starvation và retry storm

Jitter giảm synchronization nhưng không guarantee fairness. Theo dõi command age,
attempt distribution và repeated exhaustion. Admission control, per-key queue hoặc
strategy change cần thiết khi một wallet liên tục hot.

## Observability

Metrics:

- commands vs attempts;
- optimistic conflicts;
- success by attempt number;
- exhaustion/deadline/cancellation;
- backoff duration;
- duplicate replay;
- transaction/query duration và pool pending;
- hot-key evidence trong sampled trace, không metric labels.

Một tỷ lệ success cao vẫn có thể che amplification nếu attempts/success tăng.
