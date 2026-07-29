# Broken propagation code

## REQUIRES_NEW commit success record quá sớm

```java
@Service
public class CheckoutService {
    private final OrderRepository orders;
    private final PaymentAuditService audit;
    private final CheckoutProbe probe;

    @Transactional
    public void complete(long orderId, boolean failAfterAudit) {
        Order order = orders.findById(orderId).orElseThrow();
        order.markPaid();

        audit.recordPaymentCompleted(orderId);
        probe.afterAuditCommit();

        if (failAfterAudit) {
            throw new IllegalStateException("checkout failed after audit");
        }
    }
}
```

```java
@Service
public class PaymentAuditService {
    private final PaymentAuditRepository audits;

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void recordPaymentCompleted(long orderId) {
        audits.save(new PaymentAudit(orderId, "PAYMENT_COMPLETED"));
    }
}
```

Call phải đi qua bean khác nên `REQUIRES_NEW` thật sự có hiệu lực: Spring suspend
Tx-O, lấy connection/Tx-A mới, commit audit, rồi resume Tx-O. Outer rollback không
thể hoàn tác Tx-A đã commit.

> **Nói ngắn gọn:** inner commit không phải savepoint của outer transaction; nó
>là một database transaction đã kết thúc độc lập.

## REQUIRED exception bị catch nhưng transaction đã rollback-only

```java
@Service
public class CheckoutWithCaughtFailure {
    private final RequiredRiskService riskService;

    @Transactional
    public void complete(long orderId) {
        markOrderPaid(orderId);
        try {
            riskService.recordRisk(orderId);
        } catch (RuntimeException ignored) {
            // Developer nghĩ checkout vẫn có thể commit.
        }
        markWorkflowDone(orderId);
    }
}

@Service
public class RequiredRiskService {
    @Transactional(propagation = Propagation.REQUIRED)
    public void recordRisk(long orderId) {
        saveRiskRow(orderId);
        throw new IllegalStateException("risk write failed");
    }
}
```

Inner logical scope join Tx-O. Runtime exception vượt qua inner transactional
interceptor nên Tx-O bị mark rollback-only. Outer catch không xóa flag. Khi outer
interceptor cố commit, Spring rollback và ném `UnexpectedRollbackException`.

## Những cách sửa chưa đủ

- Dùng `REQUIRES_NEW` cho mọi method để “cô lập lỗi”: tạo partial commits.
- Catch `UnexpectedRollbackException` rồi trả success: database đã rollback.
- Đổi runtime exception thành checked chỉ để tránh rollback: che failure policy.
- Dùng `noRollbackFor` rộng: có thể commit state dở.
- Giả định inner `REQUIRED` là savepoint; nó dùng cùng physical transaction.
- Dùng `NESTED` mà không kiểm tra transaction manager/savepoint support.
- Gọi self-invocation: propagation annotation có thể không chạy (SPR-001).
- Publish remote event trong `REQUIRES_NEW` và gọi đó là atomic với message broker.

## Điều kiện tái hiện

Inner call đi qua proxy; outer transaction active; `REQUIRES_NEW` commit trước
outer terminal outcome, hoặc REQUIRED exception vượt interceptor rồi bị outer catch.
