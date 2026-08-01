# Các thử nghiệm tất định (Deterministic experiments) và kiểm chứng trên production

## Chiến lược kiểm thử

Không chạy một vòng lặp `while(true)` bị lỗi rồi hy vọng nó sẽ timeout. Cấu trúc kiểm thử (harness) sẽ giới hạn số attempt và dùng hai `CyclicBarrier`: barrier đầu tiên buộc mỗi actor phải giữ lock thứ nhất trước khi yêu cầu lock thứ hai; barrier thứ hai duy trì việc giữ lock thứ nhất cho tới khi cả hai actor đều đã thử lấy lock thứ hai, sau đó cả hai actor mới release để bắt đầu vòng lặp kế tiếp.

Mọi đối tượng barrier và future đều phải có timeout; không sử dụng `Thread.sleep`. Xem [Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Thử nghiệm 1: tái hiện N vòng lặp nhưng không có progress

```java
package com.example.channel;

import org.junit.jupiter.api.Test;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.locks.ReentrantLock;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;

class SymmetricRetryLivelockTest {

    @Test
    void symmetricWorkersRetryWithoutProgress() throws Exception {
        ReentrantLock lockA = new ReentrantLock();
        ReentrantLock lockB = new ReentrantLock();
        CyclicBarrier bothHoldFirst = new CyclicBarrier(2);
        CyclicBarrier bothAttemptedSecond = new CyclicBarrier(2);
        AtomicInteger attempts = new AtomicInteger();
        int attemptBudget = 20;

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            Future<Boolean> first = executor.submit(() -> runSymmetrically(
                    lockA, lockB, bothHoldFirst, bothAttemptedSecond,
                    attempts, attemptBudget));
            Future<Boolean> second = executor.submit(() -> runSymmetrically(
                    lockB, lockA, bothHoldFirst, bothAttemptedSecond,
                    attempts, attemptBudget));

            assertFalse(first.get(5, TimeUnit.SECONDS));
            assertFalse(second.get(5, TimeUnit.SECONDS));
        }

        assertEquals(attemptBudget * 2, attempts.get());
    }

    private static boolean runSymmetrically(
            ReentrantLock first,
            ReentrantLock second,
            CyclicBarrier bothHoldFirst,
            CyclicBarrier bothAttemptedSecond,
            AtomicInteger attempts,
            int attemptBudget
    ) {
        for (int attempt = 0; attempt < attemptBudget; attempt++) {
            first.lock();
            try {
                awaitBarrier(bothHoldFirst);
                boolean secondHeld = second.tryLock();
                try {
                    attempts.incrementAndGet();
                    awaitBarrier(bothAttemptedSecond);
                    if (secondHeld) {
                        return true;
                    }
                } finally {
                    if (secondHeld) {
                        second.unlock();
                    }
                }
            } finally {
                first.unlock();
            }
        }
        return false;
    }

    private static void awaitBarrier(CyclicBarrier barrier) {
        try {
            barrier.await(5, TimeUnit.SECONDS);
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("interrupted", exception);
        } catch (BrokenBarrierException | TimeoutException exception) {
            throw new IllegalStateException("barrier failed", exception);
        }
    }
}
```

Kết quả kiểm tra (assertion) trả về `false` cho cả hai actor với cùng số lần `attempts=40` chứng tỏ các thread vẫn đang hoạt động nhưng không có operation nào được hoàn tất. Ngân sách số lần thử (attempt budget) giúp giới hạn thời gian chạy của test.

> **Nói ngắn gọn:** bài test đo lường progress, chứ không chỉ kiểm tra xem thread còn sống hay không.

## Thử nghiệm 2: việc phân định thứ tự (ordering) cho phép hai operation hoàn tất

```java
@Test
void canonicalOrderCompletesOppositeDirectionCalls() throws Exception {
    Channel channelA = new Channel("A", "owner-a");
    Channel channelB = new Channel("B", "owner-b");
    OrderedChannelSwapService service = new OrderedChannelSwapService();
    CountDownLatch start = new CountDownLatch(1);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<?> first = executor.submit(() -> {
            awaitLatch(start);
            service.swap(channelA, channelB);
        });
        Future<?> second = executor.submit(() -> {
            awaitLatch(start);
            service.swap(channelB, channelA);
        });

        start.countDown();
        first.get(5, TimeUnit.SECONDS);
        second.get(5, TimeUnit.SECONDS);
    }

    assertEquals("owner-a", channelA.owner());
    assertEquals("owner-b", channelB.owner());
}

private static void awaitLatch(CountDownLatch latch) {
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

Hai lệnh swap tuần tự đưa ownership trở về state ban đầu. Future timeout đóng vai trò như một cơ chế giám sát (watchdog) trong trường hợp tính năng ordering bị lỗi (regression). Đoạn mã này cần import `CountDownLatch` và các static assertion đã dùng ở Thử nghiệm 1.

## Thử nghiệm 3: retry budget luôn tạo ra kết quả cuối cùng (terminal outcome)

Truyền vào (inject) một đối tượng `RandomGenerator` mang tính tất định (deterministic) cho giải pháp 2 và một harness luôn trả về trạng thái conflict. Các bước assert gồm:

- phương thức trả về kết quả `CONTENDED`, không chạy vô hạn;
- số lần attempt đếm được đúng bằng `maxAttempts`;
- tổng thời gian backoff không vượt quá operation deadline;
- không có lock nào còn bị giữ (held) sau khi hàm trả về;
- dữ liệu đầu vào không hợp lệ (`maxAttempts <= 0` hoặc timeout không phải số dương) sẽ bị từ chối (reject);
- nếu có interrupt trước hoặc trong quá trình backoff, hệ thống sẽ trả về `INTERRUPTED` và giữ nguyên cờ interrupt (flag).

Không assert rằng jitter "phải" bảo đảm trạng thái success; randomized backoff chỉ có tác dụng thay đổi xác suất thành công. Việc dừng vòng lặp (termination) phải đến từ deadline hoặc giới hạn attempt (attempt cap).

## Thử nghiệm 4: stress test với hot key

Sau các bài kiểm thử tất định (deterministic test), tiến hành chạy nhiều actor trên cùng một cặp channel và ghi lại seed. Cần assert rằng tổng số completed, contended và interrupted bằng số lượng submitted; không có operation nào vượt quá deadline; quyền sở hữu (ownership) luôn thuộc một tập owner hợp lệ; tỷ lệ attempt trên completion nằm trong giới hạn theo policy.

Stress test phải sử dụng random seed được inject, không log từng lần retry và không thể thay thế cho Thử nghiệm 1.

## Kiểm chứng trên môi trường production

- `swap_attempt_total`, `swap_completed_total`, `retry_exhausted_total`;
- số lần attempt cho mỗi operation và khoảng thời gian backoff;
- operation deadline hoặc tín hiệu interrupt;
- lock conflict dựa theo mã băm (hash) của cặp channel;
- mức sử dụng CPU, số lượng runnable thread và tiến trình chuyển đổi state version;
- throughput hoàn tất (completed throughput) so với tỷ lệ retry (retry rate).

Thiết lập cảnh báo (alert) khi retry tăng nhưng số completed hoặc state version không đổi. Một thread dump đơn lẻ có thể không bắt được tình trạng livelock; hãy dùng các metric chuỗi thời gian (time-series) và thu thập nhiều snapshot.

## Checklist chất lượng

- [ ] Tính đối xứng (symmetry) được tạo ra bằng barrier, không dùng hàm sleep.
- [ ] Harness bị lỗi sẽ vẫn có tính hữu hạn nhờ sử dụng attempt budget.
- [ ] Đảm bảo assert được việc cả hai actor đều retry nhưng không tạo ra progress.
- [ ] Test tính năng fixed ordering bằng cách chạy hai hướng (direction) cùng một lúc.
- [ ] Mọi đối tượng barrier, latch, hay future đều được cấu hình timeout.
- [ ] Cờ interrupt và quá trình dọn dẹp lock (lock cleanup) phải được test.
- [ ] Đối tượng `RandomGenerator` được inject để bảo đảm test có thể tái hiện được (reproducible).
- [ ] Các tín hiệu (signal) trên production phải đo lường được tỷ lệ completion cùng với retry, chứ không chỉ đo mức sử dụng CPU.
