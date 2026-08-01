# Mã nguồn hoàn tiền theo kiểu đọc–kiểm tra–ghi

## 1. Mô hình dữ liệu dễ gây hiểu nhầm

Ví dụ sau chỉ lưu số tiền đã hoàn và không tách phần đang chờ nhà cung cấp xử lý:

```java
@Entity
@Table(name = "payment_charge")
public class PaymentCharge {
    @Id
    private UUID chargeId;

    private UUID merchantId;
    private long capturedAmount;
    private long refundedAmount;
    private String currency;

    public long refundableAmount() {
        return capturedAmount - refundedAmount;
    }

    public void addRefund(long amount) {
        refundedAmount = refundedAmount + amount;
    }
}
```

Đoạn mã nhìn có vẻ hợp lý vì mọi phép tính đều nằm trong thực thể. Tuy nhiên,
giá trị `refundedAmount` chỉ là ảnh chụp tại lúc thực thể được tải. Nó không giữ
quyền sử dụng phần tiền còn lại.

## 2. Kiểm tra trùng trước khi chèn

```java
public interface RefundRepository extends JpaRepository<Refund, UUID> {
    boolean existsByMerchantIdAndExternalReference(
            UUID merchantId,
            String externalReference
    );
}
```

```java
@Entity
@Table(name = "refund")
public class Refund {
    @Id
    private UUID refundId;

    private UUID merchantId;
    private UUID chargeId;
    private String externalReference;
    private long amount;
    private String currency;

    @Enumerated(EnumType.STRING)
    private RefundStatus status;
}
```

Hai giao dịch có thể cùng chạy `exists... = false` trước khi bất kỳ bên nào chèn
dữ liệu. Câu kiểm tra không phải là quyền sở hữu khóa lũy đẳng.

## 3. Dịch vụ bị lỗi

```java
@Service
@RequiredArgsConstructor
public class BrokenRefundService {
    private final PaymentChargeRepository charges;
    private final RefundRepository refunds;
    private final PaymentProviderClient provider;

    @Transactional
    public RefundResponse refund(RefundCommand command) {
        if (refunds.existsByMerchantIdAndExternalReference(
                command.merchantId(),
                command.externalReference()
        )) {
            return RefundResponse.alreadyProcessed();
        }

        PaymentCharge charge = charges.findById(command.chargeId())
                .orElseThrow(ChargeNotFoundException::new);

        if (!charge.getMerchantId().equals(command.merchantId())) {
            throw new ChargeNotFoundException();
        }
        if (!charge.getCurrency().equals(command.currency())) {
            throw new CurrencyMismatchException();
        }
        if (charge.refundableAmount() < command.amount()) {
            return RefundResponse.limitExceeded();
        }

        Refund refund = Refund.pending(
                UUID.randomUUID(),
                command.merchantId(),
                command.chargeId(),
                command.externalReference(),
                command.amount(),
                command.currency()
        );
        refunds.save(refund);

        charge.addRefund(command.amount());

        ProviderRefund result = provider.refund(
                charge.getChargeId(),
                command.amount(),
                command.currency()
        );
        refund.markSucceeded(result.providerRefundId());

        return RefundResponse.accepted(refund.getRefundId());
    }
}
```

`@Transactional` không biến chuỗi đọc–kiểm tra–ghi thành một thao tác nguyên tử.
Nó cũng không làm lời gọi mạng trở thành một phần có thể hoàn tác của PostgreSQL.

## 4. Dòng thời gian vượt số tiền đã thu

Trạng thái ban đầu:

```text
captured_amount = 1.000.000
refunded_amount = 0
```

Hai khóa khác nhau đại diện cho hai yêu cầu thật sự khác nhau:

| Bước | Yêu cầu A — `700.000` | Yêu cầu B — `600.000` |
| --- | --- | --- |
| 1 | Kiểm tra tham chiếu, chưa tồn tại | Kiểm tra tham chiếu, chưa tồn tại |
| 2 | Đọc `refunded = 0` | Đọc `refunded = 0` |
| 3 | Tính còn `1.000.000`, chấp nhận | Tính còn `1.000.000`, chấp nhận |
| 4 | Tạo refund A | Tạo refund B |
| 5 | Đặt bản sao trong Java thành `700.000` | Đặt bản sao trong Java thành `600.000` |
| 6 | Gọi nhà cung cấp hoàn `700.000` | Gọi nhà cung cấp hoàn `600.000` |
| 7 | Chốt | Chốt |

Hai refund và hai tác dụng bên ngoài có tổng `1.300.000`. Trong khi đó, giá trị
cuối ở `payment_charge.refunded_amount` có thể chỉ là `600.000` hoặc `700.000`
do lần ghi sau đè lần ghi trước. Bảng tổng hợp vừa mất cập nhật, vừa che giấu số
tiền đã thực sự yêu cầu hoàn.

## 5. Dòng thời gian hoàn trùng cùng một yêu cầu

| Bước | Lần gọi A | Lần gửi lại B |
| --- | --- | --- |
| 1 | `exists(reference) = false` | `exists(reference) = false` |
| 2 | Tạo `refund_id = R1` | Tạo `refund_id = R2` |
| 3 | Gọi nhà cung cấp với R1 | Gọi nhà cung cấp với R2 |
| 4 | Chốt | Chốt hoặc lỗi do ràng buộc muộn |

Ngay cả khi bảng `refund` có ràng buộc duy nhất, tác dụng ở nhà cung cấp đã có
thể xảy ra trước lúc `flush` phát hiện lỗi. Ràng buộc duy nhất là cần thiết nhưng
phải được dùng như bước chiếm quyền trước mọi tác dụng nghiệp vụ.

## 6. Vì sao `READ COMMITTED` không tự sửa lỗi

Mỗi câu `SELECT` thường thấy dữ liệu đã chốt tại đầu câu lệnh. Hai giao dịch có
thể cùng đọc `refunded_amount = 0`. Không có khóa nào được giữ chỉ vì Java đã
đọc một giá trị và dự định dùng nó sau đó.

Khi Hibernate ghi giá trị tuyệt đối, PostgreSQL chỉ tuần tự hóa hai câu
`UPDATE`; nó không biết phép kiểm tra hạn mức trước đó là một phần của quyết
định. Bên ghi sau có thể ghi đè bằng kết quả được tính từ ảnh chụp cũ.

## 7. Chỉ thêm `@Version` nhưng thử lại mù quáng

```java
@Version
private long version;
```

`@Version` giúp phát hiện lần ghi thứ hai dựa trên dữ liệu cũ, nhưng chưa hoàn
thiện nghiệp vụ:

- phải thử lại toàn bộ quyết định bằng một giao dịch mới;
- không được gọi nhà cung cấp trước khi biết giao dịch đã chốt;
- vẫn cần khóa lũy đẳng và phát lại phản hồi;
- không được đổi khóa khi thử lại;
- cần giới hạn số lần thử và xử lý xung đột kéo dài.

Nếu bắt `OptimisticLockException` rồi coi là thành công, hoặc chỉ thử lại phần
`save`, bất biến vẫn không được bảo vệ.

## 8. Chỉ dùng `synchronized`

```java
public synchronized RefundResponse refund(RefundCommand command) {
    // cùng đoạn mã đọc–kiểm tra–ghi
}
```

Cách này chỉ tuần tự hóa các luồng đi qua cùng một đối tượng trong một JVM. Máy
chủ khác, tác vụ nền khác hoặc tiến trình sau khi khởi động lại không dùng cùng
khóa. Nó cũng làm nghẽn mọi giao dịch dù chúng thuộc các `charge_id` khác nhau.

## 9. Cộng bằng SQL nhưng thiếu điều kiện giới hạn

```sql
UPDATE payment_charge
SET refunded_amount = refunded_amount + :amount
WHERE charge_id = :chargeId;
```

Cập nhật chênh lệch tránh mất cập nhật, nhưng cả `700.000` và `600.000` vẫn được
cộng. Kết quả chính xác về số học là `1.300.000`, nhưng vẫn sai nghiệp vụ. Điều
kiện `allocated_refund_amount + :amount <= captured_amount` phải nằm trong cùng
câu lệnh.

## 10. Chỉ tính tổng bảng refund

```sql
SELECT COALESCE(SUM(amount), 0)
FROM refund
WHERE charge_id = :chargeId
  AND status IN ('PENDING', 'SUCCEEDED');
```

Hai giao dịch vẫn có thể cùng thấy một tổng cũ rồi cùng chèn. Phép tổng không tự
biến thành quyền giữ hạn mức. Hệ thống cần một dòng tổng hợp có thể được cập nhật
nguyên tử và đối soát với sổ lịch sử.

## 11. Gọi nhà cung cấp trong giao dịch cơ sở dữ liệu

Lời gọi mạng trong ví dụ giữ giao dịch và có thể giữ khóa trong suốt thời gian
chờ. Nếu nhà cung cấp thành công nhưng PostgreSQL hoàn tác, hệ thống bên ngoài đã
hoàn tiền còn dữ liệu nội bộ lại không ghi nhận. Nếu PostgreSQL chốt nhưng phản
hồi mạng bị mất, phía gọi cũng không biết nên làm lại hay tra cứu.

Outbox cùng một mã `refund_id` ổn định giúp tách hai ranh giới: giao dịch cơ sở
dữ liệu quyết định và ghi lệnh; tiến trình gửi gọi nhà cung cấp sau đó bằng mã có
tính lũy đẳng.

## 12. Xóa refund thất bại để trả hạn mức

```java
refundRepository.delete(failedRefund);
charge.setRefundedAmount(charge.getRefundedAmount() - failedRefund.getAmount());
```

Xóa lịch sử làm mất dấu vết kiểm toán. Hai thông báo lỗi trùng có thể cùng trừ
và giải phóng hai lần. Cách đúng là chuyển trạng thái có điều kiện, cập nhật
chênh lệch và thêm bút toán `REFUND_RELEASED` trong cùng giao dịch.

## 13. Dấu hiệu thường gặp

- Tổng các refund lớn hơn `captured_amount`.
- `payment_charge` không bằng tổng các refund đang chờ và đã thành công.
- Một tham chiếu bên ngoài xuất hiện với nhiều `refund_id`.
- Có refund nhưng thiếu outbox, hoặc có outbox nhưng thiếu bút toán.
- Nhà cung cấp có khoản hoàn mà cơ sở dữ liệu không có bản ghi tương ứng.
- Bản ghi thất bại bị xóa nên không thể giải thích lần giải phóng hạn mức.
- Hết thời gian chờ khóa bị trả về như một lỗi nghiệp vụ về số tiền.
- Chỉ một máy chủ chạy đúng; thêm máy chủ làm lỗi xuất hiện thường xuyên hơn.
