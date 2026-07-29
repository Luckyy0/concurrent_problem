# Kiểm thử deadlock và xác minh trong production

## Chiến lược

Dùng latch để buộc hai worker giữ lock đầu tiên trước khi cùng xin lock thứ hai.
`ThreadMXBean.findDeadlockedThreads()` xác nhận wait-for cycle trên
`ReentrantLock`. Harness dùng `lockInterruptibly` để test có thể cancel và cleanup;
không tạo intrinsic-lock deadlock làm rò platform thread trong test JVM.

Không dùng `Thread.sleep`; mọi latch/future có timeout. Xem
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Experiment 1: tạo và phát hiện cycle deterministic

```java
package com.example.transfer;

import org.junit.jupiter.api.Test;

import java.lang.management.ManagementFactory;
import java.lang.management.ThreadMXBean;
import java.time.Duration;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertTrue;

class OppositeLockOrderDeadlockTest {

    @Test
    void detectsOppositeOrderDeadlockAndCleansItUp() throws Exception {
        ReentrantLock lockA = new ReentrantLock();
        ReentrantLock lockB = new ReentrantLock();
        CountDownLatch bothHoldFirst = new CountDownLatch(2);
        CountDownLatch bothAttemptSecond = new CountDownLatch(2);

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            Future<?> first = executor.submit(() -> acquireOpposite(
                    lockA, lockB, bothHoldFirst, bothAttemptSecond));
            Future<?> second = executor.submit(() -> acquireOpposite(
                    lockB, lockA, bothHoldFirst, bothAttemptSecond));

            assertTrue(bothAttemptSecond.await(5, TimeUnit.SECONDS));
            long[] deadlocked = awaitDeadlock(Duration.ofSeconds(5));
            assertNotNull(deadlocked);
            assertTrue(deadlocked.length >= 2);

            first.cancel(true);
            second.cancel(true);
        }
    }

    private static void acquireOpposite(
            ReentrantLock first,
            ReentrantLock second,
            CountDownLatch bothHoldFirst,
            CountDownLatch bothAttemptSecond
    ) {
        boolean firstHeld = false;
        boolean secondHeld = false;
        try {
            first.lockInterruptibly();
            firstHeld = true;
            bothHoldFirst.countDown();
            awaitOrFail(bothHoldFirst);

            bothAttemptSecond.countDown();
            second.lockInterruptibly();
            secondHeld = true;
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
        } finally {
            if (secondHeld) second.unlock();
            if (firstHeld) first.unlock();
        }
    }

    private static long[] awaitDeadlock(Duration timeout) {
        ThreadMXBean bean = ManagementFactory.getThreadMXBean();
        long deadline = System.nanoTime() + timeout.toNanos();
        long[] ids;
        while ((ids = bean.findDeadlockedThreads()) == null
                && System.nanoTime() < deadline) {
            Thread.onSpinWait();
        }
        return ids;
    }

    private static void awaitOrFail(CountDownLatch latch)
            throws InterruptedException {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new IllegalStateException("latch timed out");
        }
    }
}
```

`findDeadlockedThreads` nhận cả ownable synchronizers; `findMonitorDeadlockedThreads`
chỉ tìm intrinsic monitor cycle. `cancel(true)` hoạt động vì harness chờ bằng
`lockInterruptibly`; production broken code dùng `lock()` có thể không thoát.

> **Nói ngắn gọn:** test vừa chứng minh cycle tồn tại, vừa chủ động phá cycle để
> test suite không giữ thread chết sau khi assertion kết thúc.

## Experiment 2: regression test cho deterministic ordering

```java
@Test
void oppositeTransfersCompleteWithCanonicalOrder() throws Exception {
    LocalAccount accountA = new LocalAccount("A", 100);
    LocalAccount accountB = new LocalAccount("B", 100);
    OrderedLocalTransferService service = new OrderedLocalTransferService();
    CountDownLatch start = new CountDownLatch(1);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<?> first = executor.submit(() -> {
            awaitUnchecked(start);
            service.transfer(accountA, accountB, 10);
        });
        Future<?> second = executor.submit(() -> {
            awaitUnchecked(start);
            service.transfer(accountB, accountA, 20);
        });

        start.countDown();
        first.get(5, TimeUnit.SECONDS);
        second.get(5, TimeUnit.SECONDS);
    }

    assertEquals(110, accountA.balance());
    assertEquals(90, accountB.balance());
    assertEquals(200, accountA.balance() + accountB.balance());
}

private static void awaitUnchecked(CountDownLatch latch) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new IllegalStateException("start timed out");
        }
    } catch (InterruptedException exception) {
        Thread.currentThread().interrupt();
        throw new IllegalStateException(exception);
    }
}
```

Snippet dùng imports trước và static `assertEquals`. Bounded future timeout là
outer watchdog: nếu ordering regression tái tạo deadlock, test fail thay vì treo.
Assertion conservation bổ sung cho progress assertion.

## Experiment 3: timed acquisition release lock đầu

Giữ second lock ở một helper thread, gọi `transferWithin` với deadline ngắn rồi
assert method trả `false`. Sau đó main/helper khác phải acquire được first lock,
chứng minh failure path đã release. Cũng cần test interrupt trong lúc chờ:

- method ném `TransferInterruptedException`;
- interrupt flag của worker được giữ;
- cả first/second lock đều không còn thuộc worker;
- không balance nào thay đổi.

Không dùng millisecond duration như bằng chứng duy nhất; latch xác nhận blocker đã
giữ lock trước khi bắt đầu attempt, future timeout làm watchdog.

## Stress và property checks

Sinh nhiều transfer hai chiều trên nhiều account, luôn assert:

- mọi future hoàn tất trong operation deadline;
- tổng balance được bảo toàn;
- balance không âm nếu domain cấm overdraft;
- `ThreadMXBean.findDeadlockedThreads()` không chứa worker của test;
- retry count bị giới hạn.

Stress tăng độ phủ nhưng không thay deterministic cycle/regression tests.

## Xác minh production

- Chụp thread dump khi request latency/active thread tăng.
- Chạy detector định kỳ với tần suất thấp, alert và lưu `ThreadInfo`/stack cho IDs.
- Metric: lock acquisition duration/timeout, critical-section duration, retry và
  operation deadline exceeded.
- Log canonical first/second account ID và correlation ID, không log balance nhạy
  cảm nếu policy cấm.
- Phân biệt JVM detector với PostgreSQL deadlock SQLSTATE/log.

## Checklist

- [ ] Broken test tạo cycle bằng latch, không bằng sleep.
- [ ] Detector phù hợp `ReentrantLock` được dùng.
- [ ] Deadlocked workers được cancel/cleanup.
- [ ] Fixed test chạy hai hướng transfer và có bounded future wait.
- [ ] Progress và balance conservation đều được assert.
- [ ] Timeout/interrupt path release lock đã giữ.
- [ ] Retry có deadline/attempt cap.
- [ ] Production diagnostics lưu owner/waiter stack.
