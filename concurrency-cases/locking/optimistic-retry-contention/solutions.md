# Giải pháp kiến trúc: Thử lại có giới hạn và tái cập nhật tự động

## 1. Phương thức xử lý một lần (One-attempt worker)

Nhiệm vụ của phương thức xử lý (worker) là tiến hành thao tác trên database duy nhất một lần trong ngữ cảnh transaction độc lập. Mọi tác vụ thử lại không thuộc trách nhiệm của lớp này.

```java
@Service
public class RewardCreditAttempt {

    private final RewardWalletRepository wallets;
    private final RewardCreditRepository credits;
    private final OutboxRepository outbox;

    public RewardCreditAttempt(
            RewardWalletRepository wallets,
            RewardCreditRepository credits,
            OutboxRepository outbox
    ) {
        this.wallets = wallets;
        this.credits = credits;
        this.outbox = outbox;
    }

    // Bắt buộc yêu cầu mở transaction mới cho từng nỗ lực thử
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public CreditResult creditOnce(CreditCommand command) {
        // Kiểm tra tính lũy đẳng dựa trên định danh (Command ID)
        var existing = credits.findById(command.commandId());
        if (existing.isPresent()) {
            return CreditResult.replayed(existing.orElseThrow());
        }

        RewardWallet wallet = wallets.findById(command.walletId())
                .orElseThrow();
        wallet.requireActive(); // Xác thực các quy tắc nghiệp vụ
        wallet.credit(command.points());

        credits.save(RewardCredit.from(command));
        outbox.save(RewardCreditedEvent.from(command));
        wallets.flush();
        credits.flush();
        outbox.flush();
        return CreditResult.applied(command.commandId(), wallet.points());
    }
}
```

Việc sử dụng mức lan truyền `REQUIRES_NEW` giúp đóng gói mọi rủi ro xử lý. Phương pháp này tiêu tốn nhiều chi phí kết nối hơn do mỗi lần khởi tạo yêu cầu cung cấp connection mới, nhưng đảm bảo tính cô lập transaction giữa các nỗ lực xử lý, ngăn chặn việc ảnh hưởng tới tiến trình bên ngoài.

## 2. Quy tắc thử lại

```java
public record RetryBudget(
        int maxAttempts,
        Duration initialBackoff, // Thời lượng chờ khởi điểm
        Duration maxBackoff      // Ngưỡng chờ tối đa cho mỗi chu kỳ
) {
    public RetryBudget {
        if (maxAttempts < 1 || initialBackoff.isNegative()
                || maxBackoff.compareTo(initialBackoff) < 0) {
            throw new IllegalArgumentException("Thông số cấp phát cấu hình không hợp lệ");
        }
    }
}

@Component
public class OptimisticRetryPolicy {

    private final Clock clock;
    private final RetryBudget budget;

    // Chính sách cho phép thực thi dựa trên nỗ lực và thời gian thực
    public boolean canRetry(int completedAttempts, Instant deadline) {
        return completedAttempts < budget.maxAttempts()
                && clock.instant().isBefore(deadline);
    }
}
```

> **Nguyên tắc kỹ thuật:** Giới hạn số lần đóng vai trò ngăn chặn hệ thống bị kéo sập do quá tải database; trong khi giới hạn thời gian bảo đảm tiêu chuẩn của phía gọi. Việc thiếu bất kỳ rào cản nào đều dễ dẫn tới suy giảm hiệu năng.

## 3. Quản trị độ trễ và phân tách logic

Kiến trúc yêu cầu phân tách thuật toán "tính toán độ trễ" và tác vụ "ngưng hoạt động" thành hai phân hệ độc lập để dễ dàng triển khai unit test:

```java
// Giao diện cung cấp độ lệch hướng
public interface JitterSource {
    long nextLong(long exclusiveUpperBound);
}

public final class BackoffPlan {

    private final RetryBudget budget;
    private final JitterSource jitter;

    public Duration delayAfter(int completedAttempt) {
        // Công thức lũy thừa cấp số (2^N)
        long factor = 1L << Math.min(completedAttempt - 1, 10);
        long base = Math.min(
                budget.maxBackoff().toMillis(),
                Math.multiplyExact(
                        budget.initialBackoff().toMillis(),
                        factor
                )
        );
        // Tích hợp hệ số phân bổ ngẫu nhiên
        long extra = jitter.nextLong(Math.max(1, base / 2 + 1));
        return Duration.ofMillis(
                Math.min(budget.maxBackoff().toMillis(), base + extra)
        );
    }
}
```

Bộ điều phối ngưng hoạt động yêu cầu bảo toàn lệnh đóng từ hệ thống cha:

```java
public interface RetryWaiter {
    void waitFor(Duration delay, Instant deadline);
}
```

Trong thực tế, bạn có thể triển khai `RetryWaiter` thông qua hệ thống lập lịch hoặc cơ chế khóa thread (`LockSupport.park`), nhưng đối với khả năng kiểm thử, thành phần này được thay thế bằng đối tượng giả lập.

## 4. Lớp điều phối ngoại trừ transaction

Thành phần trọng yếu trong cơ chế quản lý xung đột:

```java
@Service
public class RewardCreditCoordinator {

    private final RewardCreditAttempt attempts;
    private final OptimisticRetryPolicy policy;
    private final BackoffPlan backoff;
    private final RetryWaiter waiter;
    private final Clock clock;

    public CreditResult credit(CreditCommand command, Instant deadline) {
        // Ngăn cản lớp gọi bên ngoài thiết lập sai phân tách transaction
        if (TransactionSynchronizationManager
                .isActualTransactionActive()) {
            throw new IllegalStateException(
                    "retry coordinator cannot join outer transaction"
            );
        }

        for (int attempt = 1; ; attempt++) {
            try {
                // Triển khai xử lý thông qua proxy
                return attempts.creditOnce(command);
            } catch (ObjectOptimisticLockingFailureException conflict) {
                // Xác thực bộ quy tắc xử lý khi xảy ra ngoại lệ khóa lạc quan
                if (!policy.canRetry(attempt, deadline)) {
                    // Trạng thái cạn kiệt năng lực giải quyết
                    throw new WalletContentionException(
                        command.commandId(),
                        attempt,
                        conflict
                    );
                }
                
                // Cập nhật phân bổ thời gian tối đa còn lại
                Duration remaining = Duration.between(
                        clock.instant(),
                        deadline
                );
                // Giới hạn giá trị khoảng chờ không vượt quá quỹ thời gian khả dụng
                Duration delay = min(backoff.delayAfter(attempt), remaining);
                
                // Thực thi tạm ngừng quá trình ngoài phạm vi transaction
                waiter.waitFor(delay, deadline);
            }
        }
    }
}
```

Khi khối lệnh `catch` bắt được ngoại lệ, transaction của proxy thực thi đã hoàn thành khâu tiêu hủy bộ đệm (rollback). Thiết kế này tuân thủ tiêu chuẩn tiêm phụ thuộc với thông số truyền vào bằng hàm khởi tạo.

## 5. Xử lý tương tranh khi phát sinh trùng định danh

Khi cấu hình chống trùng bằng khóa chính, hệ thống sẽ bảo vệ database: hai tác vụ đồng bộ thực thi cùng lệnh sẽ kết thúc với lỗi vi phạm ràng buộc duy nhất. Ngoại lệ này độc lập với việc đụng độ cấu hình phiên bản.
Hệ thống xử lý ngoại lệ này thông qua việc phân luồng và truy hồi lại bằng tiến trình chỉ đọc để tăng tốc độ phản hồi:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
public CreditResult requireCommitted(UUID commandId) {
    return credits.findById(commandId)
            .map(CreditResult::replayed)
            .orElseThrow();
}
```

> Cần phân định rõ lỗi phát sinh do đụng độ khóa lạc quan và lỗi sinh ra do trùng thông số định danh.

## 6. Tiềm năng tích hợp `@Retryable` trong Spring

Thay thế vòng lặp truyền thống bằng Spring Retry Framework có ưu điểm nhất định. Các nguyên tắc bắt buộc khi sử dụng:

- Annotation `@Retryable` CHỈ được phép khai báo tại lớp điều phối (bên ngoài ngữ cảnh transaction), còn lớp xử lý phải giữ nguyên `@Transactional(REQUIRES_NEW)`.
- Thiết lập cụ thể tham số `include` để tránh bắt nhầm các lỗi từ chối nghiệp vụ.
- Việc quản lý thời gian tổng thể của Spring Retry có mức độ tích hợp phức tạp, đòi hỏi khả năng bảo trì hệ thống chặt chẽ so với việc cấu trúc bean thủ công.

## 7. Giải pháp áp dụng chức năng bậc thấp (`TransactionTemplate`)

Bạn có thể tích hợp ranh giới thực thi mà không phụ thuộc vào proxy annotation của Spring:

```java
var tx = new TransactionTemplate(transactionManager);
tx.setPropagationBehavior(
        TransactionDefinition.PROPAGATION_REQUIRES_NEW
);

for (int attempt = 1; ; attempt++) {
    try {
        // Mọi quy trình mở - đóng transaction được xử lý tự động trong khối lệnh lambda
        return tx.execute(status -> work.credit(command));
    } catch (ObjectOptimisticLockingFailureException conflict) {
        // Bộ nhớ và database connection đã được giải phóng
        // Áp dụng độ trễ/chính sách kiểm duyệt tương tự như mô hình lớp tách biệt
    }
}
```

Khuyến cáo quan trọng: Không bảo lưu dữ liệu thực thể cũ đưa vào tham số của `TransactionTemplate`.

## 8. Chuẩn phản hồi khi cạn kiệt nỗ lực

Khi xảy ra lỗi do hệ thống hết khả năng chịu đựng:

- **Đồng bộ hóa API:** Cung cấp mã lỗi HTTP phù hợp như `409 Conflict` hoặc `503 Service Unavailable`, có thể đi kèm tiêu đề `Retry-After`.
- **Bất đồng bộ:** Đẩy thông tin đối tượng không thành công vào hàng đợi ngoại lệ cục bộ phục vụ quá trình dò quét.
- **Tuyệt đối không:** Chuyển đổi mã lỗi hệ thống kỹ thuật để gửi mã thành công HTTP `200 Success`. Phía gọi phải nắm bắt chính xác về trạng thái để hỗ trợ đồng bộ theo phương thức lũy đẳng.

## 9. Các phương án xử lý cạnh tranh hiệu quả khác

### Cập nhật delta nguyên tử

```sql
update reward_wallet
set points = points + :delta
where wallet_id = :id;
```

Kiến trúc thay đổi trên cấp độ database phù hợp cho những bài toán đếm định lượng đơn lẻ. Database sử dụng trạng thái mới nhất cho mỗi lần tác động mà không đòi hỏi cập nhật dựa vào phiên bản. Chi tiết ứng dụng được tham chiếu ở bài `LOCK-004`. Bảng lịch sử thao tác vẫn phải nằm trong phạm vi của một transaction.

### Khóa bi quan (`FOR UPDATE`)

Tiến hành cấp phát cơ chế ngắt tạm thời trong xử lý song song, hỗ trợ chặn số lượng lặp thử lại không cần thiết nhưng gây ra việc nghẽn kết nối và nguy cơ deadlock. Chỉ nên ưu tiên khi quy trình thay đổi yêu cầu lấy dữ liệu kết hợp qua nhiều bảng cùng lúc. (Chi tiết tham khảo `LOCK-003`).

### Cơ cấu hàng đợi thông điệp theo khóa

Chuyển mô hình tải đa thread thành mô hình cập nhật chuỗi đơn theo khóa. Cải thiện năng lực xử lý bằng việc không cho phép xung đột cục bộ xảy ra, nhưng thiết kế đòi hỏi hệ thống định tuyến phức tạp và tính năng lưu vết hàng đợi linh động. (Tham khảo phân tích `LOCK-005`).

## 10. Bảng đánh giá quyết định thử lại

| Kịch bản lỗi hệ thống | Áp dụng thử lại? | Tình trạng phân giải tài nguyên |
| --- | --- | --- |
| Xung đột khóa lạc quan | **Có** (Dựa vào hạn mức/thời gian) | Transaction trước đã được rollback an toàn |
| Yêu cầu trùng lặp định danh | Cập nhật truy hồi | Trả kết quả của sự kiện lịch sử thành công trước |
| Quy tắc nghiệp vụ cấm thao tác | **Không** | Trả về lỗi từ chối nghiệp vụ |
| Cạn kiệt nỗ lực | **Không** | Trả về lỗi khả dụng hệ thống |
| Hủy từ phía gọi | **Không** | Đồng bộ hủy toàn bộ quá trình liên đới |
| Commit thành công nhưng rớt phản hồi | Áp dụng phản hồi qua payload | Dữ liệu được xác nhận đã commit trên database |
| Điều hướng thông điệp ngoại vi | Mô hình hộp thư | CHỈ gửi thông điệp khi commit được xác nhận |

## 11. Bảng cân nhắc so sánh kiến trúc

| Chiến lược | Bảo toàn dữ liệu | Hệ quả tải trọng | Tác động độ trễ | Độ khó triển khai |
| --- | --- | --- | --- | --- |
| Thử lại có giới hạn | Cao, hỗ trợ chống lặp lại | Tăng tải trọng cao | Cao | Trung bình |
| Cập nhật delta nguyên tử | Phù hợp với tính tương đối | Giảm thiểu cạnh tranh | Thấp | Thấp |
| Khóa bi quan | Tuyệt đối | Cản trở việc truy xuất | Gia tăng lỗi quá tải | Trung bình |
| Kiến trúc hàng đợi thông điệp theo khóa | Dựa vào hệ sinh thái broker | Xử lý đa thread mượt mà | Phụ thuộc tốc độ luân chuyển | **Đòi hỏi cao** |

## 12. Danh sách kiểm tra triển khai

- [ ] Thành phần bộ điều phối đảm bảo độc lập với database transaction (`@Transactional`). Lớp xử lý nội hàm áp dụng bọc xử lý riêng.
- [ ] Transaction phải hoàn tất tiến trình hủy (rollback) giải phóng trước khi cơ chế chờ kích hoạt.
- [ ] Trong transaction mới, ứng dụng phải truy xuất lại thực thể để lấy dữ liệu mới nhất.
- [ ] Hệ thống thiết lập đầy đủ ranh giới tối đa giới hạn về lần thử và thời gian giải quyết đóng gói.
- [ ] Hàm trì hoãn bảo đảm áp dụng tính chất độ lệch ngẫu nhiên nhằm phân tán quá trình đánh thức thread. Hỗ trợ ngắt từ tiến trình máy chủ.
- [ ] Đơn hàng gắn kết cùng mã lệnh từ phía gọi làm nền tảng định danh lũy đẳng.
- [ ] Chỉ những mã lỗi có tính chất tranh chấp tài nguyên (khóa lạc quan) mới thỏa mãn lệnh tạo vòng lặp thử lại.
- [ ] Việc cạn kiệt được ghi nhận là một lỗi kỹ thuật (nghiêm cấm trả phản hồi mô phỏng thành công 200).
- [ ] Trạng thái thực tế triển khai các biểu đồ đo metric: Số lần thao tác, khối lượng chờ, xung đột hệ thống, timeout và mức độ sử dụng pool.
- [ ] Khi biểu đồ tranh chấp có dấu hiệu chuyển tiếp liên tục quá giới hạn xử lý: Đề xuất chuyển cấu trúc thành cập nhật nguyên tử hoặc hàng đợi thông điệp thay vì cố chấp nới lỏng thông số số lần thử.
