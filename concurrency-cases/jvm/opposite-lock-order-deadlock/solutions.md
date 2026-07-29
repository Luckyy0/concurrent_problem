# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: deterministic total order

```java
package com.example.transfer;

public final class OrderedLocalTransferService {

    public void transfer(LocalAccount source, LocalAccount destination, long amount) {
        if (source == destination) {
            return;
        }
        if (amount <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }

        LocalAccount first = source.accountId().compareTo(destination.accountId()) < 0
                ? source : destination;
        LocalAccount second = first == source ? destination : source;

        first.lock().lock();
        try {
            second.lock().lock();
            try {
                if (source.balance() < amount) {
                    throw new IllegalStateException("insufficient balance");
                }
                source.debit(amount);
                destination.credit(amount);
            } finally {
                second.lock().unlock();
            }
        } finally {
            first.lock().unlock();
        }
    }
}
```

`accountId` phải unique, immutable và cùng comparator ở mọi call site. Acquire
order không còn phụ thuộc transfer direction. Balance mutation chỉ chạy sau khi
có cả hai lock; release theo thứ tự ngược.

> **Nói ngắn gọn:** canonical order biến hai hướng transfer thành cùng một đường
> đi qua lock graph, nên không thể tạo cạnh chờ ngược.

## Giải pháp 2: ordering kết hợp bounded interruptible acquisition

```java
public boolean transferWithin(
        LocalAccount source,
        LocalAccount destination,
        long amount,
        Duration timeout
) {
    if (timeout.isZero() || timeout.isNegative()) {
        throw new IllegalArgumentException("timeout must be positive");
    }
    LocalAccount first = minByAccountId(source, destination);
    LocalAccount second = first == source ? destination : source;
    long deadline = System.nanoTime() + timeout.toNanos();
    boolean firstHeld = false;
    boolean secondHeld = false;

    try {
        firstHeld = first.lock().tryLock(timeout.toNanos(), TimeUnit.NANOSECONDS);
        if (!firstHeld) return false;

        long remaining = deadline - System.nanoTime();
        if (remaining <= 0) return false;
        secondHeld = second.lock().tryLock(remaining, TimeUnit.NANOSECONDS);
        if (!secondHeld) return false;

        moveAfterValidation(source, destination, amount);
        return true;
    } catch (InterruptedException exception) {
        Thread.currentThread().interrupt();
        throw new TransferInterruptedException(exception);
    } finally {
        if (secondHeld) second.lock().unlock();
        if (firstHeld) first.lock().unlock();
    }
}
```

Các helper chọn ID order và mutate giống solution 1; exception là
`RuntimeException` domain mỏng. Deadline chung ngăn tổng wait nhân đôi. Timeout
loser release lock đã giữ và caller quyết định retry; không retry trong lock.

## Phương án 3: một coarse lock

Một private final monitor cho toàn registry loại bỏ multi-lock cycle và dễ chứng
minh, nhưng serialize transfer trên account không liên quan. Phù hợp state nhỏ,
throughput thấp; không mở rộng qua JVM.

## Phương án 4: không giữ nhiều lock

Partition account theo single owner, gửi command qua queue/actor hoặc đổi state
model để một coordinator thực hiện mutation. Cách này loại hold-and-wait nhưng
thay đổi architecture, latency và failure semantics.

## So sánh đánh đổi

| Phương án | Deadlock prevention | Latency | Throughput | Complexity | Multi-node |
| --- | --- | --- | --- | --- | --- |
| Total order + `lock()` | Phá circular wait | Có thể chờ contention | Account độc lập song song | Thấp | Không |
| Order + timed `tryLock` | Prevention + bounded wait | Có deadline/rejection | Có retry overhead | Vừa | Không |
| Coarse lock | Chỉ một lock | Head-of-line blocking | Thấp hơn | Rất thấp | Không |
| Actor/single owner | Không giữ nhiều lock | Queueing | Theo partition | Cao | Cần protocol riêng |

## Retry và production policy

- Retry chỉ sau cleanup, dùng operation deadline và bounded attempts.
- Backoff có jitter để tránh hai actor tái va chạm.
- Không retry insufficient balance/validation failure.
- External side effect cần idempotency key.
- Metric: lock wait, timeout, retry, detector count và critical-section duration.
- Thread dump phải lưu owner/waiter stack khi incident.
- Không dùng solution local cho database transfer; xem `DB-008` và banking cases.
