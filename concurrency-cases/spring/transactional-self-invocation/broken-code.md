# Mã lỗi (Broken code)

## Service tự gọi method có transaction

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

Các phương thức repository đi qua repository proxy nên mỗi phương thức `REQUIRED` tự tạo và
commit transaction khi service không có transaction. `probe` là hook no-op trên production;
integration test dùng latch để dừng đúng sau khi debit commit.

## Interception thực tế

Controller → service proxy → `transfer()` không được annotate: proxy không mở
transaction. Bên trong target, `this.executeTransfer()` không quay lại proxy, nên
annotation trên method đó bị bỏ qua.

> **Nói ngắn gọn:** đồ thị lời gọi lúc runtime (runtime call graph), không phải vị trí annotation trong mã nguồn,
>quyết định advice có chạy hay không.

## Những cách sửa chưa đủ

- Đổi method thành `private @Transactional`; proxy mode càng không intercept.
- Gọi `saveAndFlush`; flush không tạo atomic boundary cho hai commit.
- Bắt ngoại lệ của credit và tiếp tục; debit đã commit.
- Dùng `AopContext.currentProxy()` hoặc self-injection; làm dính chặt (coupling) proxy và dễ tạo
  phụ thuộc vòng (circular dependency), khó đánh giá (review).
- Unit test gọi `new BrokenTransferService(...)`; hoàn toàn không kiểm tra proxy.
- Annotate test bằng `@Transactional`; outer test transaction có thể che giấu lỗi.

## Điều kiện tái hiện

Bean dùng proxy mode mặc định; method đầu vào (entry) không có transaction; các phương thức repository có
transaction riêng; luồng đọc/ngoại lệ xuất hiện giữa debit và credit.
