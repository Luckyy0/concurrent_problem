# Correct propagation choices và trade-offs

## Giải pháp 1: success state dùng REQUIRED cùng outer transaction

```java
@Service
public class PaymentAuditService {
    private final PaymentAuditRepository audits;

    @Transactional(propagation = Propagation.REQUIRED)
    public void recordPaymentCompleted(long orderId, UUID operationId) {
        audits.save(new PaymentAudit(operationId, orderId, "PAYMENT_COMPLETED"));
    }
}
```

Outer checkout gọi bean proxy trong Tx-O. Audit join cùng physical transaction;
outer failure rollback order và audit. Unique operation ID chống duplicate retry.

Nếu audit chỉ là internal database fact cùng business outcome, đây là lựa chọn
đơn giản và dễ chứng minh nhất.

> **Nói ngắn gọn:** dữ liệu cùng nói về một kết quả nên thường cùng nằm trong một
>commit decision.

## Giải pháp 2: publish success sau commit hoặc qua outbox

Nếu consumer chỉ cần chạy sau successful checkout, publish domain event trong
Tx-O và dùng after-commit listener cho local, non-durable handoff (SPR-002).

Nếu success event/audit không được mất khi crash, insert outbox row cùng Tx-O.
Relay publish sau commit, consumer idempotent. Không dùng `REQUIRES_NEW` để giả
lập atomicity giữa database và broker.

## Giải pháp 3: REQUIRES_NEW cho independent attempt audit

Independent commit hợp lệ khi record mô tả sự thật độc lập:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void recordAttempt(UUID operationId, long orderId, String stage) {
    attempts.insertIfAbsent(operationId, orderId, stage, Instant.now());
}
```

Tên/status là `ATTEMPT_STARTED` hoặc `OUTER_FAILED`, không phải
`PAYMENT_COMPLETED`. Table không phụ thuộc uncommitted outer data; method ngắn,
idempotent và không truy cập rows outer đang lock.

Trade-off: thêm connection, independent failure/timeout, có thể block pool. Audit
failure policy phải explicit: có làm outer fail hay chỉ metric/alert?

## Giải pháp 4: expected optional outcome không dùng rollback exception

```java
@Transactional(propagation = Propagation.REQUIRED)
public RiskOutcome evaluate(long orderId) {
    RiskDecision decision = calculate(orderId);
    return decision.approved()
            ? RiskOutcome.APPROVED
            : RiskOutcome.REJECTED;
}
```

Expected rejection là value; outer quyết định không mark paid hoặc ném domain
exception để rollback toàn transaction. Technical failure vẫn propagate và
rollback. Không catch runtime exception từ inner interceptor rồi giả commit được.

## Giải pháp 5: TransactionTemplate làm boundary explicit

Khi workflow thật sự cần nhiều physical transactions, dùng separate
`TransactionTemplate` với propagation rõ và đặt tên steps/outcomes. Ví dụ commit
attempt record độc lập, sau đó chạy business transaction. Code phải thừa nhận
partial outcome và có reconciliation; không gọi toàn workflow là atomic.

## NESTED/savepoint khi phù hợp

Savepoint có thể rollback optional inner database work mà giữ outer transaction,
nhưng chỉ khi transaction manager/driver hỗ trợ và business state sau rollback về
savepoint vẫn hợp lệ. Integration test PostgreSQL stack; không thay `REQUIRES_NEW`
bằng `NESTED` theo tên gọi mà không hiểu physical semantics.

## Những lựa chọn cần tránh

- `REQUIRES_NEW` cho mọi helper để tránh rollback lan truyền.
- Catch `UnexpectedRollbackException` và trả HTTP 200.
- `noRollbackFor = Exception.class` rộng.
- Inner method truy cập row outer đang lock.
- Remote I/O trong independent transaction dài.
- Retry không operation ID/unique constraint.

## So sánh

| Phương án | Physical commit | Outer rollback | Resource cost | Phù hợp |
| --- | --- | --- | --- | --- |
| `REQUIRED` | Một commit chung | Rollback tất cả | Một connection | Cùng business outcome |
| `REQUIRES_NEW` | Inner độc lập | Inner sống sót | Connection/lock thêm | Truthful independent record |
| `NESTED` | Cùng outer + savepoint | Rollback tất cả | Cùng connection thường lệ | Optional DB substep có support |
| After-commit local | Sau outer commit | Không dispatch | Executor, không durable | Best-effort local work |
| Outbox | Row cùng outer commit | Không có outbox row | Relay/storage | Durable event/work |

## Production checklist

Mỗi method documented propagation purpose; success facts cùng commit outcome;
REQUIRES_NEW tables semantically independent; rollback rules explicit; no swallowed
rollback-only; unique operation ID; pool sizing/timeout; integration tests cho
outer rollback, inner failure và UnexpectedRollback; observability liên kết
physical transaction/outcome.
