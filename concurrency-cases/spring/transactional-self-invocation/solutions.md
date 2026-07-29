# Solutions và transaction-boundary trade-offs

## Giải pháp 1: đặt transaction trên public entry method

```java
@Service
public class TransferService {
    private final AccountRepository accounts;

    @Transactional
    public void transfer(TransferCommand command) {
        validate(command);
        executeTransfer(command);
    }

    private void executeTransfer(TransferCommand command) {
        int debited = accounts.debit(command.sourceId(), command.amount());
        if (debited != 1) throw new IllegalStateException("debit failed");

        int credited = accounts.credit(command.destinationId(), command.amount());
        if (credited != 1) throw new IllegalStateException("credit failed");
    }
}
```

Controller/scheduler/bean khác phải gọi `transfer` trên Spring-managed bean. Cả
repository method `REQUIRED` join transaction hiện tại. Helper private không giả
vờ có annotation proxy không thể intercept.

> **Nói ngắn gọn:** đặt transaction tại application-service entry point làm
>atomicity boundary nhìn thấy ngay từ call graph.

## Giải pháp 2: tách transactional executor sang bean khác

Phù hợp khi orchestration ngoài transaction nhưng một đoạn cần atomic:

```java
@Service
public class TransferOrchestrator {
    private final TransactionalTransferExecutor executor;

    public void transfer(TransferCommand command) {
        validateRequestShape(command);
        executor.execute(command);
    }
}

@Service
public class TransactionalTransferExecutor {
    private final AccountRepository accounts;

    @Transactional
    public void execute(TransferCommand command) {
        requireDebit(accounts.debit(command.sourceId(), command.amount()));
        requireCredit(accounts.credit(command.destinationId(), command.amount()));
    }
}
```

Call đi qua bean reference/proxy. Tên bean làm boundary rõ và dễ integration test;
đổi lại thêm một collaborator có purpose thật, không chỉ để né proxy ngẫu nhiên.

## Giải pháp 3: TransactionTemplate explicit

```java
@Service
public class ProgrammaticTransferService {
    private final TransactionTemplate transactions;
    private final AccountRepository accounts;

    public void transfer(TransferCommand command) {
        transactions.executeWithoutResult(status -> {
            requireDebit(accounts.debit(command.sourceId(), command.amount()));
            requireCredit(accounts.credit(command.destinationId(), command.amount()));
        });
    }
}
```

Không phụ thuộc self-invocation interception và phù hợp boundary động. Trade-off:
transaction mechanics xuất hiện trong application code; rollback checked/error
handling phải được review rõ.

## AspectJ weaving

AspectJ mode có thể advise self-invocation vì weaving không dựa riêng vào proxy,
nhưng tăng build/runtime complexity. Chỉ chọn khi project đã dùng weaving có chủ
đích; không bật để chữa một service có thể refactor đơn giản.

## Các phương án không khuyến nghị mặc định

- Self-injection/lazy self reference: coupling implementation proxy, circular
  dependency và dễ gọi nhầm `this` lần sau.
- `AopContext.currentProxy()`: yêu cầu expose proxy, làm domain code biết AOP.
- `REQUIRES_NEW` cho debit/credit: cố ý tách commit, ngược invariant transfer.
- `saveAndFlush`: không thay commit boundary.
- `synchronized`: không cung cấp rollback/durability/multi-node atomicity.

## So sánh

| Phương án | Boundary clarity | Testability | Complexity | Self-invocation safe |
| --- | --- | --- | --- | --- |
| Transactional public entry | Cao | Cao | Thấp | Có, helper không cần advice |
| Separate transactional bean | Rất cao | Cao | Vừa | Có, call qua proxy |
| `TransactionTemplate` | Explicit trong code | Cao | Vừa | Có |
| AspectJ | Advice cả internal call | Vừa | Cao | Có |
| Self-injection/AopContext | Thấp | Thấp | Vừa | Kỹ thuật có, thiết kế mong manh |

## Production policy

- Boundary bao trọn business invariant nhưng tránh remote I/O dài trong transaction.
- Rollback policy cho checked/unchecked exception explicit.
- Event cần atomic publication dùng outbox/after-commit phù hợp.
- Transaction/statement/lock timeout và connection-pool budget nhất quán.
- Integration test gọi proxy bean, không chỉ new service.
- Không annotate test transaction nếu cần quan sát commit từ transaction khác.
- Boundary đúng trước; isolation/locking/retry được chọn ở database cases.
