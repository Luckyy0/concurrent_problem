# Giao dịch kéo dài bị lỗi (Broken long transaction)

## Khối dữ liệu thanh toán (Payment aggregate)

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

## Kho lưu trữ bi quan (Pessimistic repository)

```java
public interface PaymentOrderRepository
        extends JpaRepository<PaymentOrder, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select p from PaymentOrder p where p.id = :id")
    Optional<PaymentOrder> findByIdForUpdate(UUID id);
}
```

Hibernate sẽ tự động "dịch" ra câu lệnh SQL tương đương:

```sql
select id, status, amount, customer_id, version
from payment_order
where id = ?
for update;
```

Khóa (Lock) sẽ bị giam lỏng cho đến chừng nào giao dịch kết thúc bằng lệnh commit/rollback.

## Gọi hệ thống từ xa bị lỗi ngay giữa lòng giao dịch (Broken remote call bên trong transaction)

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

Trong ví dụ này, `riskClient.assess()` là một quyết định HTTP chỉ-đọc (read-only decision). Dù cho phương thức hoàn toàn không gọi (gõ) bất kỳ dòng lệnh JDBC nào trong suốt quãng thời gian dài cổ đứng chờ (wait for response), nhưng luồng hiện tại (thread) vẫn nắm trong tay một EntityManager, một kết nối mạng (connection) và cả khóa dòng (row lock) liên kết chặt chẽ với giao dịch đó.

> **Nói ngắn gọn:** Trạng thái “đang đứng mòn mỏi chờ mạng tải” tuyệt nhiên không có nghĩa là “đã chịu trả kết nối”.

## Cấu hình pool chỉ kìm chân được số lượng lỗi, chứ chẳng chữa được gốc rễ ranh giới

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 8
      connection-timeout: 750ms
```

Xin lưu ý các con số này chỉ dùng để phác họa cho một môi trường thử nghiệm (test/profile) thu nhỏ, hoàn toàn không phải là lời khuyên cho cấu hình trên hệ thống thực (production recommendation). Khi có 8 giao dịch đang cùng lúc nghiễm nhiên ẵm trọn các kết nối, cái yêu cầu thứ 9 sẽ phải nai lưng chờ cho tới khi vượt mức ngưỡng tối đa của việc lấy kết nối từ pool (pool acquisition timeout), rồi lủi thủi báo lỗi (fail).

## Trên cùng một đơn hàng: một lần chờ từ xa cộng dồn với hàng tá lần chờ giành khóa

```text
Yêu cầu A (Request A):
  giành được kết nối 1 -> giật được khóa FOR UPDATE P-42 -> đứng ngây ra chờ từ xa (waits remote)

Yêu cầu B (Request B):
  giành được kết nối 2 -> vấp lệnh FOR UPDATE P-42 bèn đứng chờ thằng A

Yêu cầu C (Request C):
  giành được kết nối 3 -> vấp lệnh FOR UPDATE P-42 lại tò tò đứng chờ theo sau hàng đợi của A/B
```

Rõ ràng chỉ có mình thằng A là đang kết nối từ xa (gọi remote), nhưng đám theo đuôi (waiters) vẫn hồn nhiên chiếm dụng các kết nối quý giá của hệ thống trong quá trình chết trân chờ khóa (lock wait) của PostgreSQL. Hồ chứa có thể sạch bách dù cho thông lượng xử lý đồng thời (remote concurrency) chỉ lèo tèo bằng một.

## Với các đơn hàng khác nhau: mạnh ai nấy chờ hệ thống từ xa (Khác orders: mọi request cùng chờ remote)

```text
Đơn hàng P-01 -> chiếm kết nối 1 -> chiếm khóa dòng riêng (own row lock) -> đứng ngây ra chờ từ xa
Đơn hàng P-02 -> chiếm kết nối 2 -> chiếm khóa dòng riêng -> đứng ngây ra chờ từ xa
...
Đơn hàng P-08 -> chiếm kết nối 8 -> chiếm khóa dòng riêng -> đứng ngây ra chờ từ xa
```

Chẳng có một xích mích khóa dòng (row conflict) hay một vòng lặp đứng đợi (wait-for cycle) nào cả. Mà cái hồ chứa kết nối (Pool) vẫn cứ đầy ứ ự (full).

## Dùng Executor để đứng chờ ngay bên trong giao dịch (Executor wait bên trong transaction)

Lời gọi dịch vụ từ xa (Remote call) có thể đã được gói ghém (bọc) cẩn thận trong một executor nhưng ranh giới giao dịch thì vẫn sai bét:

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

Luồng (Caller thread) vẫn bám riết lấy giao dịch/kết nối trong lúc đứng chực chờ lời hứa trả về từ tương lai (future). Thử tưởng tượng nếu luồng rủi ro (risk executor) cũng lâm vào tình trạng ngập ngụa (saturated), khi đó những tài nguyên vô giá của cơ sở dữ liệu sẽ vô tình bị trói nghiến lấy cùng chung số phận với cái hàng đợi executor đó.

## Cái bùa `@Async` không tự động giải thoát giao dịch của người gọi

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

Tuy rằng công việc bất đồng bộ (Async task) không hề tham gia (join) vào giao dịch của phương thức gốc (caller), nhưng phương thức gốc lại ngang nhiên chặn đứng (block) luồng xử lý trước khi giao dịch kịp khép lại. Hơn nữa, bạn tuyệt đối không được bốc một thực thể đang bị quản lý (managed entity) hay trạng thái lười biếng (lazy state) đem ném rầm sang cho bên luồng bất đồng bộ (async thread) tự bơi.

## Trồng chéo thời gian chờ (Timeout nesting sai)

Giả sử ngân sách thời gian tổng (overall request budget) là hữu hạn nhưng bạn lại thiết kế:

```text
thời gian mượn từ hồ (pool acquire timeout) + thời gian chờ khóa (lock timeout) + thời gian gọi mạng (remote timeout) + thời gian thử lại (retries)
> thời hạn cam kết (caller deadline)
```

Điều này dẫn tới việc khối công việc vẫn tiếp tục cắn xén tài nguyên một cách phung phí kể cả khi khách hàng thượng nguồn (upstream) đã hết kiên nhẫn và bỏ đi. Mỗi lớp lang áp dụng thời gian chờ (timeout riêng) chẳng thể nào tự nó móc nối lại tạo thành một thời hạn mạch lạc nhất quán (coherent deadline).

## Tiền đề làm phát sinh lỗi (Preconditions tái hiện)

- Giao dịch thò tay mượn luôn kết nối (acquire connection) trước cả khi diễn ra quá trình chờ máy chủ từ xa/luồng thực thi.
- Cánh cổng từ xa hay độ trễ rớt mạng (Remote gate/latency) dai dẳng hơn hẳn so với những thao tác dữ liệu thông thường.
- Cùng lúc đó, số lượng giao dịch đang chắp cánh (concurrent in-flight) nhảy vọt chạm trần sức chứa `maximumPoolSize`.
- Hồ chứa bốc hơi nhẵn các kết nối rảnh (idle connection) đến nỗi một lệnh truy vấn bâng quơ khác cũng trắng tay.
- Đội ngũ đang chực lấy khóa (Lock waiters) thiếu vắng hẳn đi chốt chặn thất bại tức thì `lock_timeout` hữu dụng.
- Dòng thử lại của đối tác phía trên (Upstream retry) hoặc công cụ ngắt mạch (circuit breaker) chưa phát huy vai trò khoanh vùng ngăn tải (contain load).

## Những liều thuốc chữa trị nửa vời (chưa đủ)

- Chỉ biết đắp đổi thêm số lượng vào `maximumPoolSize`.
- Chỉ lò dò chỉnh nới ranh giới chịu đựng thời gian xin kết nối (connection acquisition timeout).
- Chuyển giao nhiệm vụ gọi kết nối sang cho một bộ thực thi (executor) nhưng lại trơ mắt đứng gọi hàm hợp nhất (join future) ngay giữa lồng ngực của giao dịch đó.
- Cắm vội nhãn `@Async` rồi mạnh tay nhét hàm chặn đứng `join()`.
- Lấp liếm bằng cách biến cờ `FOR UPDATE` thành cái khóa của máy ảo (JVM lock) trong khi đang triển khai ở môi trường hàng tá máy chủ (multi-instance).
- Đi cài đặt hì hục độc mỗi `lock_timeout`; rồi mặc xác tình trạng chờ chực liên mạng bóp chết cạn kiệt hồ chứa ở những giao dịch khác nhau.
- Thấy thông báo mượn kết nối trễ hạn (acquisition timeout) là lại cho gọi nhồi thử lại (retry) một cách mù quáng.
- Chờ phép màu hiện ra từ hệ thống phát hiện bế tắc (deadlock detector) để xử lý gọn nhẹ đám cạn kiệt kết nối mặc dù chẳng tồn tại bất kì mắc xích chết chóc (wait-for cycle) nào.
