# Kiểm thử deadlock và xác minh trong production

## Chiến lược

Đừng xài `Thread.sleep` để test đa luồng, dùng CountDownLatch đi! Trong bài này, ta ép hai thread chạy song song, đợi tụi nó tạo vòng lặp rồi dùng `ThreadMXBean` để bắt quả tang. Nhớ dùng hàm `lockInterruptibly` để lỡ test bị kẹt thì vẫn huỷ (cancel) được, tránh để lại rác trong JVM.

## Thử nghiệm 1: tạo và phát hiện chu trình tất định

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

Đoạn test này vừa chứng minh được có vòng lặp, vừa phá được nó nhờ `cancel(true)` (vì dùng `lockInterruptibly`). 

> **Nói ngắn gọn:** Test này cố tình dàn cảnh để bắt lỗi, xong tự dọn dẹp luôn để không làm treo máy.

## Thử nghiệm 2: kiểm thử thoái lui (regression test) cho định hướng tất định

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
Test này để đảm bảo code mới vừa không bị treo, vừa tính toán chuẩn số dư tài khoản.

## Thử nghiệm 3: giới hạn thời gian acquire và release lock đầu tiên

Thử xin lock với thời gian ngắn. Nếu không xin được lock thứ 2, bắt buộc phải nhả cái thứ nhất ra. Các bài test cần check xem:
- Có văng lỗi nghiệp vụ không.
- Thread có bị huỷ không.
- Có nhả sạch lock không.
- Số dư giữ nguyên chưa thay đổi gì không.

## Kiểm thử tải (Stress test) và kiểm thử thuộc tính

Bắn cả rổ request cùng lúc để test chịu tải:
- Đảm bảo xong đúng thời gian.
- Tiền không tự nhiên sinh ra hay mất đi.
- Không có lỗi âm tiền.
- Chả có thằng nào bị kẹt deadlock.
- Số lần retry nằm trong mức cho phép.

## Xác minh trên production

- Khi thấy CPU hay số lượng thread tăng bất thường, chụp ngay thread dump.
- Cài tool theo dõi và lưu log khi phát hiện.
- Đo đạt metric: thời gian chờ lock, số lần xin lại...
- Log đàng hoàng, rõ ràng để dễ troubleshoot. Đừng lỡ tay log số dư tài khoản nhé.

## Checklist

- [ ] Bài test cũ chạy lỗi vì sinh chu trình vòng tròn (dùng Latch).
- [ ] Dùng đúng `findDeadlockedThreads` cho `ReentrantLock`.
- [ ] Worker trong test huỷ được gọn cảng.
- [ ] Test code mới chạy trơn tru hai hướng chuyển.
- [ ] Data luôn đảm bảo chính xác.
- [ ] Khi ngắt lệnh phải nhả lock đàng hoàng.
- [ ] Cài đặt giới hạn thời gian/số lần retry chuẩn.
- [ ] Log rành mạch lỗi trên production.
