# Giải pháp ordering, bounded retry và chính sách bảo đảm progress

## Giải pháp 1: deterministic lock ordering

```java
package com.example.channel;

public final class OrderedChannelSwapService {

    public void swap(Channel left, Channel right) {
        if (left == right) {
            return;
        }
        int comparison = left.channelId().compareTo(right.channelId());
        if (comparison == 0) {
            throw new IllegalArgumentException("channelId must be unique");
        }

        Channel first = comparison < 0 ? left : right;
        Channel second = first == left ? right : left;

        first.lock().lock();
        try {
            second.lock().lock();
            try {
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

Cả hai hướng gọi (call direction) sử dụng chung một thứ tự (order), do đó không còn xảy ra symmetric collision. Đây là lựa chọn mặc định khi resource có khoá định danh duy nhất ổn định (stable unique key).

> **Nói ngắn gọn:** việc phân định thứ tự (ordering) khiến một actor phải chờ ở cửa đầu tiên, nhường cho actor thắng cuộc lấy cửa thứ hai và hoàn tất công việc.

## Giải pháp 2: bounded randomized backoff

Khi conflict không thể bị loại bỏ thông qua total order, cơ chế retry phải được cấu hình một terminal budget:

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
    long deadline = System.nanoTime() + timeout.toNanos();

    for (int attempt = 1; attempt <= maxAttempts; attempt++) {
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
        if (remaining <= 0 || attempt == maxAttempts) {
            return SwapOutcome.CONTENDED;
        }
        long cap = Math.min(
                remaining,
                Math.max(1, backoffCapNanos(attempt))
        );
        long delay = random.nextLong(cap);
        LockSupport.parkNanos(delay);
    }
    return SwapOutcome.CONTENDED;
}
```

`backoffCapNanos` sử dụng thuật toán tính toán theo cấp số mũ có giới hạn (saturating/capped exponential calculation), không sử dụng phép dịch bit (shift) để tránh gây ra lỗi overflow. Đối tượng `RandomGenerator` được inject vào để phục vụ cho các bài kiểm tra mang tính tất định. Nếu một interrupt xảy ra bên trong `parkNanos`, vòng lặp tiếp theo sẽ trả về `INTERRUPTED` và không thực hiện xoá cờ interrupt (clear interrupt flag).

Random backoff giúp giảm xác suất va chạm, trong khi deadline và giới hạn attempt (attempt cap) sẽ bảo đảm việc dừng vòng lặp (termination). Nó không đảm bảo fairness cho từng actor.

## Giải pháp 3: single owner hoặc queue

Một bộ điều phối (coordinator) sẽ sở hữu cả hai channel và xử lý các lệnh hoán đổi (swap command) một cách tuần tự. Bằng cách này, hệ thống không còn các vòng lặp multi-lock retry, nhưng bù lại sẽ cần quản lý queueing, tính sẵn sàng của owner (owner availability) và các chính sách backpressure. Phương pháp này phù hợp khi các yếu tố như fairness hoặc việc kiểm toán (audit) quan trọng hơn so với parallel mutation.

## So sánh đánh đổi

| Phương án | Progress | Latency | Fairness | Complexity |
| --- | --- | --- | --- | --- |
| Total lock order | Loại bỏ symmetric cycle | Chờ khi có contention | Theo lock scheduler | Thấp |
| Bounded jitter retry | Có terminal deadline, tỷ lệ success mang tính xác suất | Biến động | Không bảo đảm | Vừa |
| Fair/coarse lock | Không xảy ra retry collision | Chờ trong queue | Có thể tốt hơn | Thấp-vừa |
| Single owner queue | Tuần tự, có tính quan sát (observable) | Xếp hàng trong queue | Policy rõ ràng | Cao hơn |

## Chính sách retry và vận hành trên production

- Ưu tiên các biện pháp ngăn chặn mang tính cấu trúc (structural prevention) trước khi thực hiện điều chỉnh retry (retry tuning).
- Chỉ thực hiện retry đối với các trường hợp conflict đã được phân loại; không retry với các lỗi validation hoặc business failure.
- Sử dụng chung một deadline, có bounded attempt, jitter và admission control.
- Không giữ lock trong thời gian backoff; không thực hiện side effect trước khi quá trình acquire hoàn tất toàn bộ (full acquisition).
- Các số liệu (metric) cần theo dõi: tỷ lệ attempt trên success, exhausted, interrupted, delay, lock conflict và completed state version.
- Thực hiện load test đối với các symmetric hot key; phát cảnh báo (alert) khi tỷ lệ attempt trên completion tăng lên trong khi throughput không thay đổi.
