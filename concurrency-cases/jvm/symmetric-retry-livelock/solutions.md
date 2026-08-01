# Giải pháp ordering, bounded retry và chính sách bảo đảm progress

## Giải pháp 1: Sắp xếp thứ tự lấy khóa (Deterministic lock ordering)

Đây là chiêu "ăn tiền" nhất. Thay vì mạnh ai nấy lấy, mình bắt buộc tụi nó phải xếp hàng. 

```java
package com.example.channel;

public final class OrderedChannelSwapService {

    public void swap(Channel left, Channel right) {
        if (left == right) {
            return;
        }
        // So sánh ID để xác định ai lớn ai nhỏ
        int comparison = left.channelId().compareTo(right.channelId());
        if (comparison == 0) {
            throw new IllegalArgumentException("channelId must be unique");
        }

        // Đứa nào nhỏ hơn thì ưu tiên lấy trước
        Channel first = comparison < 0 ? left : right;
        Channel second = first == left ? right : left;

        first.lock().lock();
        try {
            second.lock().lock();
            try {
                // Đủ 2 khóa rồi mới bắt đầu đổi
                String owner = left.owner();
                left.changeOwner(right.owner());
                right.changeOwner(owner);
            } finally {
                second.lock().unlock();
            }
        } finally {
            first.lock().unlock();
        }
    }
}
```

Bất kể T1 hay T2 gọi swap kiểu gì, mã code này cũng ép cả hai phải tranh nhau cái khóa "đứng trước" (dựa theo thứ tự từ điển của ID). Không bao giờ có chuyện đụng nhau ngớ ngẩn (symmetric collision) nữa. Chỉ xài được chiêu này khi dữ liệu của bạn có ID cố định và duy nhất nhé.

> **Nói ngắn gọn:** Sắp xếp thế này khiến một anh phải đợi anh kia làm xong mới được vô cửa, trị dứt điểm cái bệnh "nhường nhau" vớ vẩn.

## Giải pháp 2: Giới hạn số lần thử và chờ ngẫu nhiên (Bounded randomized backoff)

Lỡ xui mà không thể sắp xếp thứ tự thì sao? Thì xài chiêu "Retry có ý thức" - thử lại nhưng có giới hạn, không chày cối.

```java
public SwapOutcome trySwap(
        Channel firstChoice,
        Channel secondChoice,
        Duration timeout,
        int maxAttempts,
        RandomGenerator random
) {
    if (timeout.isZero() || timeout.isNegative() || maxAttempts <= 0) {
        throw new IllegalArgumentException("invalid retry budget");
    }
    // Chốt deadline
    long deadline = System.nanoTime() + timeout.toNanos();

    for (int attempt = 1; attempt <= maxAttempts; attempt++) {
        // Có ai gọi giật ngược (interrupt) không?
        if (Thread.currentThread().isInterrupted()) {
            return SwapOutcome.INTERRUPTED;
        }

        boolean firstHeld = firstChoice.lock().tryLock();
        boolean secondHeld = false;
        try {
            if (firstHeld) {
                secondHeld = secondChoice.lock().tryLock();
                if (secondHeld) {
                    swapOwnership(firstChoice, secondChoice);
                    return SwapOutcome.SWAPPED;
                }
            }
        } finally {
            if (secondHeld) secondChoice.lock().unlock();
            if (firstHeld) firstChoice.lock().unlock();
        }

        long remaining = deadline - System.nanoTime();
        // Hết giờ hoặc hết lượt thì nghỉ
        if (remaining <= 0 || attempt == maxAttempts) {
            return SwapOutcome.CONTENDED;
        }
        
        // Chờ lâu dần đều, nhưng có mức trần
        long cap = Math.min(
                remaining,
                Math.max(1, backoffCapNanos(attempt))
        );
        // Random để trật nhịp với thằng kia
        long delay = random.nextLong(cap);
        LockSupport.parkNanos(delay);
    }
    return SwapOutcome.CONTENDED;
}
```

Hàm `backoffCapNanos` sẽ tính toán thời gian chờ tăng dần (mũ), nhưng được gài trần để không bị tràn số (overflow). Truyền vào cục `RandomGenerator` cũng tiện lợi nếu bạn muốn viết test mô phỏng cứng. Khác với cách 1, cách này chỉ mang tính xui rủi (giảm thiểu khả năng va chạm) chứ không chắc 100% là chia đều phần (fairness) cho các thread đâu nhé.

## Giải pháp 3: Chỉ định một Owner duy nhất hoặc xài Queue

Cách này là bạn gom mọi request ném vào một hàng đợi (Queue), sau đó có một anh Coordinator đứng ra lần lượt lấy từng cặp xử lý. Thế là dẹp luôn nỗi lo kẹt khóa! Bù lại, hệ thống của bạn sẽ chạy chậm đi một chút vì phải xếp hàng, và bạn phải tốn công quản lý thêm mớ lùng bùng của Queue. Phù hợp cho những hệ thống cần tính tuần tự, dễ theo dõi (audit) hơn là hì hục chạy song song.

## So sánh đánh đổi

| Phương án | Khả năng tiến triển (Progress) | Độ trễ (Latency) | Độ công bằng (Fairness) | Độ phức tạp |
| --- | --- | --- | --- | --- |
| Lock order theo thứ tự | Hết cửa kẹt luôn | Phải chờ nếu đông | Hên xui tùy hệ điều hành | Dễ òm |
| Retry có deadline + Jitter | Dừng khi hết hạn, hên xui mới thành công | Giật cục, biến động | Không có | Vừa vừa |
| Single owner / Queue | Xử lý lần lượt, dễ theo dõi | Phải xếp hàng nên hơi trễ | Rất rõ ràng | Phức tạp hơn |

## Chính sách retry và vận hành trên production

Để ăn ngon ngủ yên khi đẩy code lên Prod:
- Nếu sửa được tận gốc bằng cấu trúc (như cách 1) thì múc ngay, đừng đâm đầu vào tinh chỉnh hàm retry.
- Phân biệt rõ loại lỗi nào mới cần retry. Đừng có lỗi rành rành ra đó (kiểu sai nghiệp vụ) mà cũng đem đi retry thì toang.
- Bắt buộc phải có: Deadline chung, giới hạn số lần thử, chờ ngẫu nhiên (jitter).
- Tuyệt đối không ôm khóa trong lúc ngồi chờ (backoff); và không đụng chạm dữ liệu khi chưa nắm đủ bộ khóa.
- Theo dõi sát sao: Lượng retry/success, có đụng khóa nhiều không, CPU có thở nổi không.
- Nhớ stress-test với các khóa nóng (hot key); cài báo động (alert) nếu thấy tỉ lệ retry tăng vụt mà tỉ lệ thành công vẫn giậm chân tại chỗ.
