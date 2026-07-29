# Deterministic livelock experiments và production verification

## Chiến lược kiểm thử

Không chạy broken `while(true)` rồi hy vọng timeout. Harness giới hạn số attempt
và dùng hai `CyclicBarrier`: barrier đầu buộc mỗi actor giữ first lock trước khi
xin second; barrier thứ hai giữ first lock cho tới khi cả hai đã thử second lock,
rồi hai actor mới release để bắt đầu vòng kế tiếp.

Mọi barrier/future có timeout; không dùng `Thread.sleep`. Xem
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Experiment 1: tái hiện N vòng không progress

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

Assertion `false` cho cả hai actor cùng `attempts=40` chứng minh thread vẫn hoạt
động nhưng không operation nào hoàn tất. Attempt budget giữ test hữu hạn.

> **Nói ngắn gọn:** test đo progress, không chỉ đo thread còn sống hay không.

## Experiment 2: ordering cho phép hai operation hoàn tất

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

Hai swap tuần tự đưa ownership về state ban đầu. Future timeout là watchdog nếu
ordering bị regression. Snippet cần imports `CountDownLatch` và static assertions
đã dùng ở Experiment 1.

## Experiment 3: retry budget luôn tạo terminal outcome

Inject `RandomGenerator` deterministic vào solution 2 và một harness luôn trả
conflict. Assert:

- method trả `CONTENDED`, không chạy vô hạn;
- attempt count đúng `maxAttempts`;
- tổng backoff không vượt operation deadline;
- không lock nào còn held sau return;
- input invalid (`maxAttempts <= 0`, timeout không dương) bị reject;
- interrupt trước/trong backoff trả `INTERRUPTED` và giữ interrupt flag.

Không assert jitter “phải” bảo đảm success; randomized backoff chỉ thay đổi xác
suất. Termination đến từ deadline/attempt cap.

## Experiment 4: stress với hot key

Sau deterministic tests, chạy nhiều actor trên cùng channel pair và ghi seed.
Assert completed + contended + interrupted bằng submitted; không operation vượt
deadline; ownership luôn thuộc tập owner hợp lệ; attempt/completion ratio có giới
hạn theo policy.

Stress test dùng injected random seed, không log từng retry, không thay thế
Experiment 1.

## Production verification

- `swap_attempt_total`, `swap_completed_total`, `retry_exhausted_total`;
- attempts per operation và backoff duration;
- operation deadline/interrupt;
- lock conflict theo channel pair hash;
- CPU/runnable thread count và state-version progress;
- completed throughput so với retry rate.

Alert khi retry tăng nhưng completed/state version không đổi. Một thread dump đơn
lẻ có thể không bắt được livelock; dùng metric/time-series và nhiều snapshot.

## Checklist chất lượng

- [ ] Symmetry được tạo bằng barrier, không bằng sleep.
- [ ] Broken harness hữu hạn nhờ attempt budget.
- [ ] Cả hai actor retry nhưng không progress được assert.
- [ ] Fixed ordering test chạy hai direction cùng lúc.
- [ ] Mọi barrier/latch/future có timeout.
- [ ] Interrupt flag và lock cleanup được test.
- [ ] RandomGenerator được inject để test reproducible.
- [ ] Production signal đo completion cùng retry, không chỉ CPU.
