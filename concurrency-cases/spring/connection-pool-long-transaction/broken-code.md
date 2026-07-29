# Broken long transaction

## Payment aggregate

```java
@Entity
@Table(name = "payment_order")
public class PaymentOrder {
    @Id
    private UUID id;

    @Enumerated(EnumType.STRING)
    private PaymentStatus status;

    private long amount;
    private UUID customerId;

    @Version
    private long version;

    protected PaymentOrder() {
    }

    public RiskSubject riskSubject() {
        return new RiskSubject(id, customerId, amount, version);
    }

    public void approve(RiskDecision decision) {
        if (status != PaymentStatus.RISK_PENDING) {
            throw new IllegalStateException("Payment is not RISK_PENDING");
        }
        if (!decision.subjectId().equals(id)) {
            throw new IllegalArgumentException("Decision subject mismatch");
        }
        status = PaymentStatus.APPROVED;
    }
}
```

## Pessimistic repository

```java
public interface PaymentOrderRepository
        extends JpaRepository<PaymentOrder, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select p from PaymentOrder p where p.id = :id")
    Optional<PaymentOrder> findByIdForUpdate(UUID id);
}
```

Hibernate tạo SQL tương đương:

```sql
select id, status, amount, customer_id, version
from payment_order
where id = ?
for update;
```

Lock được giữ tới transaction commit/rollback.

## Broken remote call bên trong transaction

```java
@Service
public class PaymentRiskService {
    private final PaymentOrderRepository payments;
    private final RiskClient riskClient;

    public PaymentRiskService(
        PaymentOrderRepository payments,
        RiskClient riskClient
    ) {
        this.payments = payments;
        this.riskClient = riskClient;
    }

    @Transactional
    public ApprovalResult assessAndApprove(UUID paymentId) {
        PaymentOrder payment = payments
            .findByIdForUpdate(paymentId)
            .orElseThrow();

        RiskDecision decision =
            riskClient.assess(payment.riskSubject());

        if (decision.isRejected()) {
            return ApprovalResult.rejected(paymentId);
        }

        payment.approve(decision);
        return ApprovalResult.approved(paymentId);
    }
}
```

`riskClient.assess()` là HTTP read-only decision trong case này. Dù method không
gọi JDBC trong lúc chờ response, thread vẫn có transaction-bound EntityManager,
connection và row lock.

> **Nói ngắn gọn:** “đang chờ network” không đồng nghĩa “đã trả connection”.

## Pool configuration làm failure hữu hạn nhưng không sửa boundary

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 8
      connection-timeout: 750ms
```

Các giá trị chỉ minh họa một test/profile nhỏ, không phải production recommendation.
Khi 8 transactions đang giữ connections, request thứ 9 chờ tối đa pool acquisition
timeout rồi fail.

## Cùng order: một remote wait cộng nhiều lock waits

```text
Request A:
  connection 1 -> FOR UPDATE P-42 acquired -> waits remote

Request B:
  connection 2 -> FOR UPDATE P-42 waits for A

Request C:
  connection 3 -> FOR UPDATE P-42 waits for A/B queue
```

Chỉ A gọi remote, nhưng waiters vẫn chiếm connections trong PostgreSQL lock wait.
Pool có thể cạn dù remote concurrency chỉ bằng một.

## Khác orders: mọi request cùng chờ remote

```text
P-01 -> connection 1 -> own row lock -> remote wait
P-02 -> connection 2 -> own row lock -> remote wait
...
P-08 -> connection 8 -> own row lock -> remote wait
```

Không có row conflict hoặc wait-for cycle. Pool vẫn full.

## Executor wait bên trong transaction

Remote call có thể bị bọc trong executor nhưng boundary vẫn sai:

```java
@Transactional
public ApprovalResult assessAndApprove(UUID paymentId) {
    PaymentOrder payment =
        payments.findByIdForUpdate(paymentId).orElseThrow();

    Future<RiskDecision> future = riskExecutor.submit(
        () -> riskClient.assess(payment.riskSubject())
    );

    RiskDecision decision;
    try {
        decision = future.get(500, TimeUnit.MILLISECONDS);
    } catch (TimeoutException timeout) {
        future.cancel(true);
        throw new RiskDependencyTimeoutException(timeout);
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new RiskDependencyInterruptedException(interrupted);
    } catch (ExecutionException failed) {
        throw translate(failed.getCause());
    }

    payment.approve(decision);
    return ApprovalResult.approved(paymentId);
}
```

Caller thread giữ transaction/connection trong lúc chờ future. Nếu risk executor
cũng saturated, database resources bị ghép với executor queue.

## `@Async` không tự giải phóng caller transaction

```java
@Transactional
public ApprovalResult assessAndApprove(UUID paymentId) {
    PaymentOrder payment =
        payments.findByIdForUpdate(paymentId).orElseThrow();

    RiskDecision decision = asyncRiskClient
        .assess(payment.riskSubject())
        .orTimeout(500, TimeUnit.MILLISECONDS)
        .join();

    payment.approve(decision);
    return ApprovalResult.approved(paymentId);
}
```

Async task không join caller transaction, nhưng caller vẫn block trước transaction
end. Ngoài ra không truyền managed entity/lazy state sang async thread.

## Timeout nesting sai

Nếu overall request budget là hữu hạn nhưng:

```text
pool acquire timeout + lock timeout + remote timeout + retries
> caller deadline
```

work tiếp tục tiêu thụ resources sau khi upstream đã bỏ cuộc. Mỗi layer timeout
riêng không tự tạo coherent deadline.

## Preconditions tái hiện

- Transaction acquire connection trước remote/executor wait.
- Remote gate/latency dài hơn normal database work.
- Concurrent in-flight transactions đạt `maximumPoolSize`.
- Pool không còn idle connection cho unrelated query.
- Lock waiters không có fail-fast `lock_timeout` phù hợp.
- Upstream retry/circuit breaker chưa contain load.

## Những cách sửa chưa đủ

- Chỉ tăng `maximumPoolSize`.
- Chỉ tăng connection acquisition timeout.
- Đưa remote call sang executor nhưng join future trong transaction.
- Thêm `@Async` rồi block bằng `join()`.
- Đổi `FOR UPDATE` thành JVM lock trong multi-instance deployment.
- Chỉ cấu hình `lock_timeout`; different-row remote waits vẫn cạn pool.
- Retry acquisition timeout ngay lập tức.
- Mong deadlock detector xử lý starvation không có cycle.
