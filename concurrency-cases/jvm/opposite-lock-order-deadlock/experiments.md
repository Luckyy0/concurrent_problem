# Kiểm thử deadlock và xác minh trong production

## Chiến lược

Dùng latch để buộc hai tiến trình xử lý (worker) giữ lock đầu tiên trước khi cùng xin lock thứ hai. Gọi `ThreadMXBean.findDeadlockedThreads()` để xác nhận wait-for cycle trên `ReentrantLock`. Trình kiểm thử (harness) dùng `lockInterruptibly` để kiểm tra khả năng hủy và dọn dẹp; không tạo intrinsic-lock deadlock làm rò rỉ platform thread trong JVM kiểm thử.

Không dùng `Thread.sleep`; mọi latch và future đều có timeout. Xem [Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

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

`findDeadlockedThreads` nhận cả ownable synchronizers; `findMonitorDeadlockedThreads` chỉ tìm chu trình monitor intrinsic. Lệnh `cancel(true)` hoạt động vì trình kiểm thử chờ bằng `lockInterruptibly`; code bị lỗi trên production dùng `lock()` có thể không thoát.

> **Nói ngắn gọn:** kiểm thử vừa chứng minh chu trình tồn tại, vừa chủ động phá chu trình để bộ kiểm thử (test suite) không giữ thread chết sau khi quá trình xác nhận kết quả (assertion) kết thúc.

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

Đoạn mã dùng import trước và static `assertEquals`. Bounded future timeout đóng vai trò bộ canh gác (watchdog) bên ngoài: nếu thoái lui tái tạo deadlock, bài kiểm thử sẽ thất bại thay vì bị treo. Việc xác nhận bảo toàn dữ liệu (assertion conservation) bổ sung cho việc kiểm tra tính tiến triển.

## Thử nghiệm 3: giới hạn thời gian acquire và release lock đầu tiên

Giữ lock thứ hai ở một thread hỗ trợ (helper thread), gọi `transferWithin` với thời hạn ngắn rồi kiểm tra xem hàm có trả về `false` không. Sau đó, thread chính hoặc một helper khác phải acquire được lock thứ nhất, chứng minh luồng lỗi đã release lock. Cũng cần kiểm tra ngắt (interrupt) trong lúc chờ:

- Hàm ném ra `TransferInterruptedException`;
- Cờ ngắt (interrupt flag) của worker được giữ;
- Cả lock thứ nhất và thứ hai đều không còn thuộc worker;
- Không balance nào bị thay đổi.

Không dùng khoảng thời gian tính bằng mili giây làm bằng chứng duy nhất; latch xác nhận yếu tố chặn (blocker) đã giữ lock trước khi bắt đầu lượt thử, future timeout đóng vai trò watchdog.

## Kiểm thử tải (Stress test) và kiểm thử thuộc tính

Sinh nhiều thao tác chuyển hai chiều trên nhiều account, luôn kiểm tra:

- Mọi future hoàn tất trong thời hạn thao tác;
- Tổng balance được bảo toàn;
- Balance không âm nếu quy tắc nghiệp vụ cấm thấu chi (overdraft);
- `ThreadMXBean.findDeadlockedThreads()` không chứa worker của bộ kiểm thử;
- Số lượt thử lại (retry count) bị giới hạn.

Kiểm thử tải làm tăng độ phủ nhưng không thay thế được các kiểm thử thoái lui và chu trình tất định.

## Xác minh trên production

- Chụp thread dump khi độ trễ yêu cầu hoặc số lượng active thread tăng.
- Chạy detector định kỳ với tần suất thấp, cảnh báo và lưu `ThreadInfo` cũng như stack trace cho các ID.
- Số liệu (Metric): thời gian acquire lock, timeout, thời gian chạy critical-section, số lượt thử lại và số yêu cầu vượt thời hạn.
- Ghi log account ID thứ nhất và thứ hai theo thứ tự chuẩn cùng correlation ID, không ghi log balance nhạy cảm nếu chính sách cấm.
- Phân biệt bộ phát hiện (detector) của JVM với deadlock SQLSTATE và log của PostgreSQL.

## Checklist

- [ ] Kiểm thử lỗi (Broken test) tạo chu trình bằng latch, không bằng lệnh sleep.
- [ ] Dùng đúng detector phù hợp với `ReentrantLock`.
- [ ] Các worker bị deadlocked được hủy và dọn dẹp.
- [ ] Kiểm thử sửa lỗi (Fixed test) chạy hai hướng chuyển và giới hạn thời gian chờ future.
- [ ] Tiến trình và bảo toàn balance đều được xác nhận kết quả (assert).
- [ ] Luồng ngắt hoặc timeout thực hiện release lock đã giữ.
- [ ] Thử lại có thời hạn và giới hạn số lượt (attempt cap).
- [ ] Chẩn đoán trên production lưu trữ owner và waiter stack.
