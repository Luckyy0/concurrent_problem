# Mã nguồn propagation bị lỗi

## REQUIRES_NEW commit bản ghi thành công quá sớm

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

Lời gọi (call) phải đi qua bean khác nên `REQUIRES_NEW` thật sự có hiệu lực: Spring tạm dừng
Tx-O, lấy connection/Tx-A mới, commit bản ghi audit, rồi tiếp tục Tx-O. Lệnh rollback ở khối ngoài không
thể hoàn tác Tx-A đã commit.

> **Nói ngắn gọn:** lần commit bên trong không phải là savepoint của transaction ngoài; nó
> là một database transaction đã kết thúc độc lập.

## REQUIRED ném ngoại lệ bị bắt (catch) nhưng transaction đã rollback-only

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
            // Lập trình viên nghĩ checkout vẫn có thể commit.
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

Phạm vi logic bên trong tham gia vào Tx-O. Ngoại lệ runtime vượt qua transactional
interceptor bên trong nên Tx-O bị đánh dấu là rollback-only. Việc bắt ngoại lệ (catch) ở khối ngoài không xóa cờ này. Khi interceptor
khối ngoài cố gắng commit, Spring sẽ thực hiện rollback và ném `UnexpectedRollbackException`.

## Những cách sửa chưa đủ

- Dùng `REQUIRES_NEW` cho mọi phương thức để “cô lập lỗi”: tạo ra các partial commit.
- Bắt lỗi `UnexpectedRollbackException` rồi trả về kết quả thành công: database thực tế đã rollback.
- Đổi ngoại lệ runtime thành checked exception chỉ để tránh rollback: che đậy chính sách xử lý lỗi.
- Dùng `noRollbackFor` quá rộng: có thể commit trạng thái dang dở.
- Giả định `REQUIRED` bên trong là savepoint; thực tế nó dùng chung transaction vật lý.
- Dùng `NESTED` mà không kiểm tra sự hỗ trợ savepoint/transaction manager.
- Tự gọi phương thức trong cùng class (self-invocation): annotation propagation có thể không hoạt động (SPR-001).
- Xuất bản sự kiện từ xa (remote event) trong `REQUIRES_NEW` và coi đó là thao tác nguyên tử (atomic) với message broker.

## Điều kiện tái hiện

Lời gọi bên trong đi qua proxy; transaction ngoài đang hoạt động; `REQUIRES_NEW` commit trước
khi có kết quả cuối cùng của khối ngoài, hoặc ngoại lệ `REQUIRED` vượt qua interceptor rồi bị khối ngoài bắt (catch).
