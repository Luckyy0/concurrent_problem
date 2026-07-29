# Broken code

## Service tự gọi transactional method

```java
@Service
public class BrokenTransferService {
    private final AccountRepository accounts;
    private final TransferStepProbe probe;

    public void transfer(TransferCommand command) {
        validate(command);
        this.executeTransfer(command);
    }

    @Transactional
    public void executeTransfer(TransferCommand command) {
        int debited = accounts.debit(command.sourceId(), command.amount());
        if (debited != 1) throw new IllegalStateException("debit failed");

        probe.afterDebit();

        int credited = accounts.credit(command.destinationId(), command.amount());
        if (credited != 1) throw new IllegalStateException("credit failed");
    }
}
```

```java
public interface AccountRepository extends JpaRepository<Account, Long> {
    @Modifying
    @Transactional
    @Query("update Account a set a.balance = a.balance - :amount " +
           "where a.id = :id and a.balance >= :amount")
    int debit(long id, long amount);

    @Modifying
    @Transactional
    @Query("update Account a set a.balance = a.balance + :amount where a.id = :id")
    int credit(long id, long amount);
}
```

Repository methods đi qua repository proxy nên mỗi method `REQUIRED` tự tạo và
commit transaction khi service không có transaction. `probe` là no-op production
hook; integration test dùng latch để dừng đúng sau debit commit.

## Interception thực tế

Controller → service proxy → `transfer()` không annotated: proxy không mở
transaction. Bên trong target, `this.executeTransfer()` không quay lại proxy, nên
annotation trên method đó bị bỏ qua.

> **Nói ngắn gọn:** runtime call graph, không phải vị trí annotation trong source,
>quyết định advice có chạy hay không.

## Những cách sửa chưa đủ

- Đổi method thành `private @Transactional`; proxy mode càng không intercept.
- Gọi `saveAndFlush`; flush không tạo atomic boundary cho hai commit.
- Catch credit exception và tiếp tục; debit đã commit.
- Dùng `AopContext.currentProxy()` hoặc self-injection; coupling proxy và dễ tạo
  circular dependency, khó review.
- Unit test gọi `new BrokenTransferService(...)`; hoàn toàn không kiểm tra proxy.
- Annotate test bằng `@Transactional`; outer test transaction có thể che lỗi.

## Điều kiện tái hiện

Bean dùng proxy mode mặc định; entry không có transaction; repository methods có
transaction riêng; reader/exception xuất hiện giữa debit và credit.
