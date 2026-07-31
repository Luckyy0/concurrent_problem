# Giải pháp Code Chuẩn: Thử lại Có Chừng Mực (Bounded Retry) và Sạch Sẽ (Fresh Attempt)

## 1. Người Lính Đánh Thuê (One-attempt worker)

Nhiệm vụ của anh lính này chỉ đơn giản là: Vào trong, làm việc 1 lần duy nhất, thất bại thì báo cáo. Không tự ý thử lại!

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

    // BẮT BUỘC tạo Giao dịch mới tinh cho mỗi lần gọi
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public CreditResult creditOnce(CreditCommand command) {
        // Kiểm tra chống trùng trước tiên
        var existing = credits.findById(command.commandId());
        if (existing.isPresent()) {
            return CreditResult.replayed(existing.orElseThrow());
        }

        RewardWallet wallet = wallets.findById(command.walletId())
                .orElseThrow();
        wallet.requireActive(); // Bóp nghẹt mấy cái Ví đã bị khóa
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

`REQUIRES_NEW` cực kỳ phù hợp ở đây vì Class Chỉ huy (bên ngoài) hoàn toàn không có Giao dịch. Mặc dù tốn kém thêm chút kết nối (connection cost) cho mỗi Giao dịch mới, nhưng đó là cái giá phải trả để hệ thống không bị dính chùm. (Lưu ý: Phải có rào cản chặn mấy Class khác vô tình bọc cái Chỉ huy lại).

## 2. Luật Thử Lại (Retry policy)

```java
public record RetryBudget(
        int maxAttempts,
        Duration initialBackoff, // Thời gian ngủ lần đầu
        Duration maxBackoff      // Ngưỡng ngủ tối đa không được vượt qua
) {
    public RetryBudget {
        if (maxAttempts < 1 || initialBackoff.isNegative()
                || maxBackoff.compareTo(initialBackoff) < 0) {
            throw new IllegalArgumentException("Ngân sách thử lại sai quy tắc!");
        }
    }
}

@Component
public class OptimisticRetryPolicy {

    private final Clock clock;
    private final RetryBudget budget;

    // Phải thỏa mãn cả 2 điều kiện: Còn ngân sách số lần VÀ chưa hết giờ
    public boolean canRetry(int completedAttempts, Instant deadline) {
        return completedAttempts < budget.maxAttempts()
                && clock.instant().isBefore(deadline);
    }
}
```

> **Nhớ kỹ:** Giới hạn Số lần (attempts) là để cứu Database khỏi quá tải; còn Giới hạn Thời gian (deadline) là để API phản hồi cho User kịp lúc. Thiếu 1 trong 2 là toang!

## 3. Ngủ ngẫu nhiên (Backoff có jitter) và Dễ viết Test

Phải tách biệt "Thuật toán tính giờ ngủ" và "Hành động ngủ" ra làm 2 cái riêng:

```java
// Đồ xóc đĩa ngẫu nhiên
public interface JitterSource {
    long nextLong(long exclusiveUpperBound);
}

public final class BackoffPlan {

    private final RetryBudget budget;
    private final JitterSource jitter;

    public Duration delayAfter(int completedAttempt) {
        // Hàm số mũ: 1, 2, 4, 8... tối đa mũ 10
        long factor = 1L << Math.min(completedAttempt - 1, 10);
        long base = Math.min(
                budget.maxBackoff().toMillis(),
                Math.multiplyExact(
                        budget.initialBackoff().toMillis(),
                        factor
                )
        );
        // Trộn thêm "gia vị" thời gian ngẫu nhiên
        long extra = jitter.nextLong(Math.max(1, base / 2 + 1));
        return Duration.ofMillis(
                Math.min(budget.maxBackoff().toMillis(), base + extra)
        );
    }
}
```

Hành động Ngủ phải Tôn trọng người khác (lỡ bị gọi ngắt ngang interrupt):

```java
public interface RetryWaiter {
    void waitFor(Duration delay, Instant deadline);
}
```

Trên Production bạn xài Scheduler hay Parking tùy ý, nhưng khi viết Test thì dùng đồ giả (Recording waiter) để test chạy vèo vèo không phải chờ `sleep`.

## 4. Kẻ Chỉ Huy Không Giao Dịch (Non-transactional coordinator)

Đây mới là ngôi sao chính của chúng ta:

```java
@Service
public class RewardCreditCoordinator {

    private final RewardCreditAttempt attempts;
    private final OptimisticRetryPolicy policy;
    private final BackoffPlan backoff;
    private final RetryWaiter waiter;
    private final Clock clock;

    public CreditResult credit(CreditCommand command, Instant deadline) {
        // Đạp văng ra nếu thằng nào nhét Tui vào chung Transaction
        if (TransactionSynchronizationManager
                .isActualTransactionActive()) {
            throw new IllegalStateException(
                    "retry coordinator cannot join outer transaction"
            );
        }

        for (int attempt = 1; ; attempt++) {
            try {
                // Thả chó... à nhầm, thả Lính Đánh Thuê vào làm việc
                return attempts.creditOnce(command);
            } catch (ObjectOptimisticLockingFailureException conflict) {
                // Có Lỗi! Lấy Luật ra soi!
                if (!policy.canRetry(attempt, deadline)) {
                    // Cạn kiệt ngân sách hoặc hết giờ, văng cờ trắng đầu hàng
                    throw new WalletContentionException(
                            command.commandId(),
                            attempt,
                            conflict
                    );
                }
                
                // Xem xem thời gian tối đa còn lại là bao nhiêu
                Duration remaining = Duration.between(
                        clock.instant(),
                        deadline
                );
                // Chọn thời gian ngủ nhỏ nhất giữa Tính Toán và Thời Gian Còn Lại
                Duration delay = min(backoff.delayAfter(attempt), remaining);
                // Ôm gối đi ngủ ngoài phòng khách (Ngoài Transaction)
                waiter.waitFor(delay, deadline);
            }
        }
    }
}
```

Nhớ nhé, khối `catch` chạy lúc này là Proxy Giao Dịch đã bị tiêu hủy xong xuôi! Code Production xịn luôn bơm đồ vào bằng Constructor chứ không chơi `@Autowired` dơ dáy nha.

## 5. Cuộc đua Bấm Đúp (Duplicate command race)

Hai yêu cầu y hệt nhau lọt vào 2 luồng. Khóa chính `reward_credit_pkey` sẽ phán quyết thằng nào đi sau bị nổ (unique failure). 
Tuy nhiên, lỗi Unique này không được tính là lỗi Khóa Lạc Quan nên vòng lặp sẽ bị bẻ gãy. Class bên ngoài sẽ bắt được, mở thêm một Giao Dịch "Chỉ Đọc" cực nhanh để lôi kết quả cũ ra gửi trả về (replay):

```java
@Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
public CreditResult requireCommitted(UUID commandId) {
    return credits.findById(commandId)
            .map(CreditResult::replayed)
            .orElseThrow();
}
```

Lỗi Trùng Lặp Khóa (Duplicate) hoàn toàn khác với Lỗi Đụng Độ Dữ Liệu (Optimistic Retry), đừng gộp chung nhé!

## 6. Xài hàng có sẵn của Spring (`@Retryable`)

Bạn hoàn toàn có thể dùng `@Retryable` của Spring thay vì tự viết Vòng Lặp. NHƯNG hãy cẩn thận:

- Bắt buộc phải đặt `@Retryable` ở Class Chỉ Huy bên ngoài, còn thằng Lính bên trong vẫn mang `@Transactional(REQUIRES_NEW)`. Tuyệt đối không đặt hai Annotation này chung 1 chỗ!
- Phải lọc riêng Exception.
- Rất khó để kiểm soát cái `overall deadline` (hết hạn tổng). Lời khuyên là tách Bean riêng như cách trên dễ Audit và bảo trì hơn.

## 7. Giải pháp xài `TransactionTemplate` (Code chay)

Nếu lười chia thành 2 Class, bạn xài Code chay cũng được:

```java
var tx = new TransactionTemplate(transactionManager);
tx.setPropagationBehavior(
        TransactionDefinition.PROPAGATION_REQUIRES_NEW
);

for (int attempt = 1; ; attempt++) {
    try {
        // Cục execute này sẽ mở và đóng Giao dịch gọn gàng
        return tx.execute(status -> work.credit(command));
    } catch (ObjectOptimisticLockingFailureException conflict) {
        // Lúc nhảy xuống đây thì Giao dịch đã chết ngủ yên rồi
        // Áp dụng luật/deadline/backoff y như trên
    }
}
```

**Nhớ:** Không được nhét (cache) Entity ôi thiu từ bên ngoài vào trong `execute`.

## 8. Hợp đồng "Cạn Kiệt Sinh Lực" (Exhaustion contract)

Một khi đã báo lỗi Kiệt Sức, bạn phải trả thẳng về phía Client:

- API đồng bộ: Táng mã `409 Conflict` hoặc `503 Service Unavailable`. Có thể gửi kèm `Retry-After` (thử lại sau chừng đó giây).
- API bất đồng bộ: Nhét vào Hàng đợi (Queue) chờ cứu xét sau (workflow riêng).
- Tuyệt đối không xạo chó báo "Success 200" nếu bảng `reward_credit` chưa có dữ liệu.
- Cho phép người dùng hoặc App tự tra cứu trạng thái bằng Mã `command ID` nếu mọi chuyện không rõ ràng.

## 9. Các Phương Án Khác khi Đụng Độ Quá Gắt

### Phép màu Cộng Trừ Nguyên Tử (Atomic delta)

```sql
update reward_wallet
set points = points + :delta
where wallet_id = :id;
```

Chỉ cần update cộng/trừ thẳng vào DB mà khỏi cần quan tâm Version! Thường dùng cho các bộ đếm đơn giản. Nhưng nhớ là cái bảng Lưu lịch sử (idempotency) vẫn phải nằm chung một Transaction đó nha.

### Khóa Bi Quan (Pessimistic lock)

Khóa mõm luôn dòng dữ liệu. Giảm bớt số lần đập đầu thử lại vô ích, nhưng đổi lại làm đám luồng chờ chực tắc đường (wait/connection occupancy), dễ dính Tử Trận (Deadlock). Khó ăn lắm!

### Hàng đợi riêng biệt (Per-key queue/ownership)

Giảm được đua đòi chen lấn, nhưng phải thiết kế hệ thống xếp hàng rất phức tạp (đọc, lưu, sập nguồn...). Hãy đọc qua `LOCK-005` để xem cách chọn hệ thống.

## 10. Ma Trận Giải Quyết Thất Bại

| Hậu quả | Có thử lại không? | Tình trạng hiện tại |
| --- | --- | --- |
| Đụng độ Khóa Lạc Quan | **Có** (Nếu còn ngân sách/thời gian) | Lần thử trước đã bị Hủy bỏ |
| Bấm đúp trùng mã Lệnh | Làm phát Đọc mới (Fresh replay) | Trả về 1 lịch sử cũ |
| Ví bị cấm/Sai logic | **Không** | Báo lỗi Doanh Nghiệp (Domain rejection) |
| Cạn kiệt lần thử / Hết giờ | **Không** | Văng lỗi Quá tải/Tranh chấp |
| Khách tắt trình duyệt (Cancel) | **Không** | Truyền tiếp lệnh Cancel |
| Chốt xong nhưng mất mạng | Nhờ Client gửi lại `command ID` | Có khả năng DB đã được lưu (commit) |
| Bắn Event đi nơi khác | Phải dùng Outbox | Chỉ xuất ra SAU KHI chốt sổ |

## 11. Đánh Đổi Chọn Lựa (Trade-off)

| Chiến thuật | Tính Chính Xác | Tải hệ thống lúc Đụng độ | Độ Trễ (Latency) | Độ Phức Tạp |
| --- | --- | --- | --- | --- |
| Thử lại có chừng mực (Bài này) | Mạnh + Chống Trùng Lặp | Bão khuếch đại tải | Cao do rớt/ngủ chờ | Trung bình |
| Cộng Trừ Nguyên Tử (Atomic) | Mạnh cho cộng trừ | Rất ít Request rác | Thấp | Thấp |
| Khóa Bi Quan | Mạnh | Hàng dài kẹt cứng | Dễ bị quá giờ | Trung bình |
| Hàng đợi (Queue) | Tùy cơ chế vận chuyển | Chạy mượt mà tuần tự | Phải tốn thời gian xếp hàng | **Cao** |

## 12. Danh sách Check-list (Hãy tự soi lại Code của mình)

- [ ] Kẻ Chỉ huy không ôm Transaction; Tách riêng Lính Đánh Thuê bọc Proxy.
- [ ] Lần thử trước phải chết hẳn (Rollback hoàn tất) rồi mới được đi Ngủ.
- [ ] Lần thử sau phải bốc (load lại) 1 bản Entity sạch sẽ và kiểm tra lại luật từ đầu.
- [ ] Code chặn Cả số lần tối đa (attempt cap) lẫn Tổng Thời Gian (overall deadline).
- [ ] Đi ngủ tính giờ theo Hàm số Mũ có trộn Số Ngẫu Nhiên (jitter), biết tôn trọng lệnh Cancel.
- [ ] Giữ nguyên 1 Mã Lệnh (Command ID) từ đầu tới cuối, chốt cùng Transaction với cái Ví.
- [ ] Chỉ cho phép bế lỗi Khóa Lạc Quan (Optimistic conflict) đi thử lại.
- [ ] Kiệt sức là lỗi bận rộn, không được giả vờ thành công.
- [ ] Có Metric giám sát cẩn thận Tỷ lệ đâm (attempts/success), đụng độ, Hết Giờ và Áp lực Hồ Bơi (pool).
- [ ] Ví mà hot quá lâu thì phải đổi sang Queue/Bi quan chứ đừng có ngu ngục tăng `maxAttempts` lên nữa!
