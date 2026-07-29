# Short transaction and backpressure solutions

## Mục tiêu thiết kế

```text
admit bounded remote work
  -> short DB snapshot, connection released
  -> remote decision outside transaction
  -> short DB transaction
  -> lock/reload/revalidate
  -> commit, stale/no-op, or bounded retry decision
```

> **Nói ngắn gọn:** transaction không bao quanh latency domain mà database không
> kiểm soát.

## Solution 1 — Snapshot, remote call, revalidate

### Immutable snapshot reader

```java
public record PaymentRiskSnapshot(
    UUID paymentId,
    UUID customerId,
    long amount,
    long version,
    PaymentStatus status
) {
    RiskSubject toRiskSubject() {
        return new RiskSubject(
            paymentId,
            customerId,
            amount,
            version
        );
    }
}
```

```java
@Service
public class PaymentSnapshotReader {
    private final PaymentOrderRepository payments;

    public PaymentSnapshotReader(PaymentOrderRepository payments) {
        this.payments = payments;
    }

    @Transactional(readOnly = true)
    public PaymentRiskSnapshot read(UUID paymentId) {
        PaymentOrder payment = payments.findById(paymentId)
            .orElseThrow();
        return payment.riskSnapshot();
    }
}
```

Method return DTO trước transaction end; không trả managed entity/lazy association
ra remote phase.

### Coordinator không transactional

```java
@Service
public class PaymentRiskCoordinator {
    private final PaymentSnapshotReader snapshots;
    private final BoundedRiskGateway riskGateway;
    private final PaymentDecisionWriter decisions;

    public PaymentRiskCoordinator(
        PaymentSnapshotReader snapshots,
        BoundedRiskGateway riskGateway,
        PaymentDecisionWriter decisions
    ) {
        this.snapshots = snapshots;
        this.riskGateway = riskGateway;
        this.decisions = decisions;
    }

    public ApprovalResult assessAndApprove(
        UUID paymentId,
        OperationDeadline deadline
    ) {
        PaymentRiskSnapshot snapshot = snapshots.read(paymentId);

        if (snapshot.status() != PaymentStatus.RISK_PENDING) {
            return ApprovalResult.alreadyResolved(
                paymentId,
                snapshot.status()
            );
        }

        RiskDecision decision = riskGateway.assess(
            snapshot.toRiskSubject(),
            deadline
        );

        return decisions.apply(snapshot, decision, deadline);
    }
}
```

Không thêm `@Transactional` lên coordinator. Snapshot transaction kết thúc trước
`riskGateway.assess()`.

### Short commit transaction

```java
@Service
public class PaymentDecisionWriter {
    private final PaymentOrderRepository payments;

    public PaymentDecisionWriter(PaymentOrderRepository payments) {
        this.payments = payments;
    }

    @Transactional
    public ApprovalResult apply(
        PaymentRiskSnapshot assessed,
        RiskDecision decision,
        OperationDeadline deadline
    ) {
        deadline.requireCommitBudget();

        PaymentOrder current = payments
            .findByIdForUpdate(assessed.paymentId())
            .orElseThrow();

        if (!current.matchesAssessedSubject(assessed)) {
            return ApprovalResult.staleDecision(
                assessed.paymentId()
            );
        }

        if (decision.isRejected()) {
            current.reject(decision.reasonCode());
            return ApprovalResult.rejected(current.getId());
        }

        current.approve(decision);
        return ApprovalResult.approved(current.getId());
    }
}
```

`matchesAssessedSubject()` kiểm tra version, status, amount và customer/decision
subject. Lock chỉ được giữ trong reload/check/update/commit.

### Vì sao invariant được bảo vệ?

- Remote wait không giữ JDBC connection.
- Concurrent change làm snapshot mismatch, không bị overwrite.
- Same-order contenders chỉ xếp hàng trong short commit phase.
- Loser nhận `staleDecision`/stored terminal outcome, không apply decision cũ.
- PostgreSQL vẫn là authoritative state boundary.

Coordinator có thể lấy snapshot/assess lại nếu decision stale và overall deadline
còn đủ, nhưng retry phải bounded và remote call read-only/idempotent theo case.

## Solution 2 — Atomic conditional apply

Nếu apply decision chỉ là state transition đơn giản, dùng affected-row count:

```java
public interface PaymentOrderRepository
        extends JpaRepository<PaymentOrder, UUID> {

    @Modifying
    @Query(
        value = """
            update payment_order
               set status = :newStatus,
                   version = version + 1,
                   risk_reason = :reason
             where id = :paymentId
               and status = 'RISK_PENDING'
               and version = :assessedVersion
               and amount = :assessedAmount
               and customer_id = :assessedCustomer
            """,
        nativeQuery = true
    )
    int applyDecision(
        UUID paymentId,
        long assessedVersion,
        long assessedAmount,
        UUID assessedCustomer,
        String newStatus,
        String reason
    );
}
```

```java
@Transactional
public ApprovalResult apply(
    PaymentRiskSnapshot assessed,
    RiskDecision decision
) {
    int changed = payments.applyDecision(
        assessed.paymentId(),
        assessed.version(),
        assessed.amount(),
        assessed.customerId(),
        decision.isApproved() ? "APPROVED" : "REJECTED",
        decision.reasonCode()
    );

    return changed == 1
        ? ApprovalResult.applied(assessed.paymentId())
        : ApprovalResult.staleDecision(assessed.paymentId());
}
```

UPDATE acquire row lock chỉ trong statement/commit window. PostgreSQL re-evaluate
predicate after waiting; affected rows `0` maps to stale/no-op. Cách này giảm
round trips nhưng domain transition/audit logic phải vẫn đầy đủ.

## Solution 3 — Bounded remote bulkhead trước DB commit

Một local bulkhead minh họa:

```java
@Component
public class BoundedRiskGateway {
    private final Semaphore permits;
    private final RiskClient client;

    public BoundedRiskGateway(
        RiskClient client,
        RiskCapacityProperties capacity
    ) {
        this.client = client;
        this.permits = new Semaphore(
            capacity.maxConcurrentCalls(),
            true
        );
    }

    public RiskDecision assess(
        RiskSubject subject,
        OperationDeadline deadline
    ) {
        boolean acquired;
        try {
            acquired = permits.tryAcquire(
                deadline.remoteAdmissionNanos(),
                TimeUnit.NANOSECONDS
            );
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new RiskDependencyInterruptedException(interrupted);
        }

        if (!acquired) {
            throw new RiskBulkheadFullException();
        }

        try {
            return client.assess(
                subject,
                deadline.remoteTimeout()
            );
        } finally {
            permits.release();
        }
    }
}
```

Bulkhead wait có timeout và nằm ngoài transaction. Per-instance limit phải được
tính cùng số instances và remote capacity. Một library bulkhead/circuit breaker có
thể thay implementation, nhưng placement/capacity contract vẫn vậy.

Executor cho remote calls phải bounded queue và explicit rejection policy. Không
dùng unbounded queue để đổi pool starvation thành memory/latency growth.

## Solution 4 — Durable asynchronous state machine

Phù hợp khi remote latency lớn, cần durable retry hoặc synchronous request không
nên giữ thread.

Tx-1:

```sql
update payment_order
set status = 'RISK_REQUESTED',
    version = version + 1
where id = :id
  and status = 'RISK_PENDING';

insert into outbox_event(
    id, aggregate_id, event_type, payload, created_at
)
values (
    :eventId, :paymentId, 'RISK_ASSESSMENT_REQUESTED',
    :payload, now()
);
```

Worker:

```text
claim outbox -> commit claim
call risk API outside DB transaction
open Tx-2 -> conditional apply by payment/version/request ID -> commit
```

Decision table:

| Current state | Matching request/version? | Worker action |
| --- | --- | --- |
| `RISK_REQUESTED` | Có | Apply decision |
| Terminal | Bất kỳ | Replay/no-op |
| Version/state changed | Không | Mark decision stale |
| Dependency transient failure | N/A | Durable bounded retry |

Workflow cần outbox uniqueness, idempotent consumer, stuck-state recovery và
observability. Đổi lại, không giữ request/DB resources qua remote latency.

## Solution 5 — Timeout budget và database guardrails

Timeouts contain failure:

```text
overall request deadline
  ├─ remote admission + remote call budget
  ├─ pool acquisition budget
  ├─ lock/statement budget
  └─ rollback/response margin
```

PostgreSQL short transaction có thể dùng local guardrails:

```sql
set local lock_timeout = '250ms';
set local statement_timeout = '500ms';
```

Các literals chỉ minh họa test/configuration. Production values phải derive từ
remaining deadline và được set bằng trusted configuration, không nối raw user
input vào SQL.

Spring transaction timeout là thêm một guardrail cho database unit, không thay
remote client timeout. Pool `connectionTimeout` giới hạn pending borrower wait,
không phải query/transaction timeout.

Sau timeout/error, rollback transaction trước retry. Không retry acquisition/lock
timeout ngay lập tức nếu pool/database vẫn saturated.

## Solution 6 — Backpressure ở ingress

Khi in-flight risk work đạt budget:

- reject sớm bằng overload response phù hợp;
- queue bounded với deadline-aware admission;
- shed low-priority work;
- stop server-side retries;
- open circuit khi dependency unhealthy;
- propagate `Retry-After` chỉ khi client retry policy an toàn.

Admission diễn ra trước DB transaction. Backpressure biến unbounded resource wait
thành bounded, observable outcome.

## Pool sizing

Pool size là capacity control, không phải root fix. Xem xét:

- database safe connection budget;
- instance count/failover/autoscaling;
- normal short transaction concurrency;
- reserved operational/migration capacity;
- query CPU/I/O, not only request threads;
- connection usage duration distribution;
- pool pending/acquisition latency.

Tổng configured pool capacity không được âm thầm vượt PostgreSQL budget. Benchmark
bằng workload đại diện; không có universal “CPU × constant” cho mọi database.

## Same-order duplicate optimization

Sau khi acquire lock, check terminal state trước remote work. Với split boundary,
snapshot reader có thể trả stored terminal result ngay. Commit phase cũng recheck
để xử lý race.

Idempotency key/command result giúp duplicate request replay thay vì đánh giá risk
lại. Nó không thay bulkhead/pool protection cho distinct requests.

## Failure behavior

| Failure | Transaction đang mở khi wait? | Outcome |
| --- | --- | --- |
| Bulkhead full | Không | Fail-fast/no DB mutation |
| Remote timeout | Không | No commit phase |
| State changed during remote | Chỉ short apply Tx | Stale/no-op |
| Lock timeout in apply | Có, short | Rollback; bounded policy |
| Pool acquisition timeout | Chưa có connection | Overload/fail |
| Worker redelivery | Không qua remote Tx | Idempotent apply/replay |

## Trade-off comparison

| Lựa chọn | Connection/lock duration | Latency model | Complexity | Best fit |
| --- | --- | --- | --- | --- |
| Remote inside Tx | Bằng remote latency | Synchronous | Thấp bề mặt | Không khuyến nghị |
| Snapshot + revalidate | Short DB phases | Synchronous | Vừa | Read-only remote decision |
| Atomic conditional apply | Rất ngắn | Synchronous | Vừa | Simple state transition |
| Async state machine/outbox | Short DB phases | Eventual | Cao | Long/unreliable dependency |
| Bigger pool only | Không đổi | Failure bị dời | Thấp | Capacity tune sau boundary fix |

## Recommendation cho case này

1. Dùng Solution 1 với immutable `PaymentRiskSnapshot`.
2. Đặt bulkhead/remote timeout trước commit transaction.
3. Revalidate version/status/subject dưới short row lock hoặc conditional UPDATE.
4. Return stale/no-op rõ ràng; bounded re-assessment chỉ khi deadline cho phép.
5. Dùng async outbox nếu remote SLA không phù hợp synchronous request.
6. Tune pool/timeouts sau khi transaction duration đã được rút ngắn.

## Production checklist

### Boundary

- [ ] Không remote/executor wait trong transaction.
- [ ] Managed entity không đi qua async/remote phase.
- [ ] Commit transaction reload/revalidate current state.
- [ ] Row lock chỉ tồn tại trong short mutation phase.
- [ ] Cancellation luôn dẫn tới bounded cleanup.

### Capacity và timeout

- [ ] Remote bulkhead/queue bounded trước DB work.
- [ ] Overall deadline phân bổ cho mọi phase.
- [ ] Pool/lock/statement/remote timeouts không cộng vượt deadline.
- [ ] Pool capacity tính theo toàn bộ instances.
- [ ] Retries có backoff và không khuếch đại overload.

### Data và vận hành

- [ ] Stale decision không được apply.
- [ ] Duplicate request replay bằng durable command/result khi cần.
- [ ] Async workflow có outbox/idempotent recovery.
- [ ] Pool metrics correlate với remote/lock/transaction duration.
- [ ] Admin/migration capacity nằm trong PostgreSQL budget.
