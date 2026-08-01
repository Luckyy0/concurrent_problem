# Giải pháp và sự đánh đổi ranh giới transaction (transaction-boundary trade-offs)

## Giải pháp 1: đặt transaction trên phương thức public đầu vào

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
repository method `REQUIRED` sẽ tham gia vào transaction hiện tại. Hàm helper private không giả
vờ có annotation mà proxy không thể intercept.

> **Nói ngắn gọn:** đặt transaction tại điểm đầu vào application-service làm
>ranh giới atomicity nhìn thấy ngay từ đồ thị lời gọi.

## Giải pháp 2: tách executor có transaction sang bean khác

Phù hợp khi orchestration ngoài transaction nhưng một đoạn cần đảm bảo tính atomic:

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

Lời gọi đi qua bean reference/proxy. Tên bean làm ranh giới rõ ràng và dễ integration test;
đổi lại có thêm một collaborator có mục đích thật sự, không chỉ để né proxy một cách ngẫu nhiên.

## Giải pháp 3: TransactionTemplate tường minh

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

Không phụ thuộc self-invocation interception và phù hợp với ranh giới động. Trade-off:
cơ chế transaction xuất hiện trong application code; rollback các ngoại lệ checked/lỗi
phải được đánh giá (review) rõ ràng.

## AspectJ weaving

AspectJ mode có thể advise self-invocation vì weaving không dựa riêng vào proxy,
nhưng tăng độ phức tạp khi build/runtime. Chỉ chọn khi project đã dùng weaving có chủ
đích; không bật để chữa một service có thể refactor đơn giản.

## Các phương án không khuyến nghị mặc định

- Self-injection/tham chiếu bản thân lazy (lazy self reference): dính chặt (coupling) vào implementation proxy, phụ thuộc vòng
  và dễ gọi nhầm `this` trong lần sau.
- `AopContext.currentProxy()`: yêu cầu lộ proxy, làm mã nguồn nghiệp vụ biết đến AOP.
- `REQUIRES_NEW` cho debit/credit: cố ý tách commit, đi ngược lại bất biến của transfer.
- `saveAndFlush`: không thay đổi ranh giới commit.
- `synchronized`: không cung cấp khả năng rollback/bền vững/atomic đa node (multi-node atomicity).

## So sánh

| Phương án | Mức độ rõ ràng ranh giới | Khả năng test | Độ phức tạp | An toàn với self-invocation |
| --- | --- | --- | --- | --- |
| Transactional public entry | Cao | Cao | Thấp | Có, hàm helper không cần advice |
| Separate transactional bean | Rất cao | Cao | Vừa | Có, gọi qua proxy |
| `TransactionTemplate` | Tường minh trong code | Cao | Vừa | Có |
| AspectJ | Advice cả lời gọi nội bộ | Vừa | Cao | Có |
| Self-injection/AopContext | Thấp | Thấp | Vừa | Kỹ thuật có, thiết kế mong manh |

## Chính sách trên production

- Ranh giới bao trọn bất biến nghiệp vụ nhưng tránh I/O từ xa kéo dài trong transaction.
- Chính sách rollback cho checked/unchecked exception được ghi rõ tường minh.
- Sự kiện cần phát hành atomic dùng outbox/after-commit phù hợp.
- Transaction/statement/lock timeout và định mức connection-pool cần nhất quán.
- Integration test gọi proxy bean, không chỉ khởi tạo (new) service.
- Không annotate test transaction nếu cần quan sát commit từ transaction khác.
- Ranh giới đúng trước; isolation/locking/retry được chọn ở các trường hợp xử lý cơ sở dữ liệu.
