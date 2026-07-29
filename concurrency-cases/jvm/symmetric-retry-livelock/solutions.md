# Giải pháp ordering, bounded retry và progress policy

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

Cả hai call direction dùng cùng order, nên không còn symmetric collision. Đây là
lựa chọn mặc định khi resource có stable unique key.

> **Nói ngắn gọn:** ordering làm một actor chờ ở cửa đầu tiên, để actor thắng có
>thể lấy cửa thứ hai và hoàn tất.

## Giải pháp 2: bounded randomized backoff

Khi conflict không thể loại bằng total order, retry phải có terminal budget:

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

`backoffCapNanos` dùng saturating/capped exponential calculation, không shift gây
overflow. `RandomGenerator` được inject để test deterministic. Nếu interrupt xảy
ra trong `parkNanos`, vòng sau trả `INTERRUPTED` và không clear interrupt flag.

Random backoff giảm xác suất va chạm, còn deadline/attempt cap bảo đảm termination.
Nó không bảo đảm fairness cho từng actor.

## Giải pháp 3: single owner hoặc queue

Một coordinator sở hữu cả hai channel và xử lý swap command tuần tự. Không có
multi-lock retry, nhưng có queueing, owner availability và backpressure policy.
Phù hợp khi fairness/audit quan trọng hơn parallel mutation.

## So sánh đánh đổi

| Phương án | Progress | Latency | Fairness | Complexity |
| --- | --- | --- | --- | --- |
| Total lock order | Loại symmetric cycle | Chờ contention | Theo lock scheduler | Thấp |
| Bounded jitter retry | Terminal deadline, probabilistic success | Biến động | Không bảo đảm | Vừa |
| Fair/coarse lock | Không retry collision | Queue wait | Có thể tốt hơn | Thấp-vừa |
| Single owner queue | Tuần tự, observable | Queueing | Policy explicit | Cao hơn |

## Retry và production policy

- Ưu tiên structural prevention trước retry tuning.
- Retry chỉ conflict được phân loại; không retry validation/business failure.
- Deadline chung, bounded attempt, jitter và admission control.
- Không giữ lock trong backoff; không side effect trước full acquisition.
- Metric: attempts/success, exhausted, interrupted, delay, lock conflict và
  completed state version.
- Load test symmetric hot keys; alert khi attempt/completion ratio tăng mà
  throughput không tăng.
