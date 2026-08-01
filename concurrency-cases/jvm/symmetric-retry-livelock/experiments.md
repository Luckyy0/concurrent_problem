# Các thử nghiệm tất định (Deterministic experiments) và kiểm chứng trên production

## Chiến lược kiểm thử

Khi test concurrency, đừng dại gì mà viết một vòng lặp `while(true)` lỗi rồi ngồi chắp tay cầu nguyện cho nó timeout. Để test đàng hoàng, ta nên dùng mưu hèn kế bẩn một chút: ép số lần thử tối đa và xài `CyclicBarrier`. 
Đại loại là tạo 2 cái rào chắn (barrier):
- Rào 1: Ép 2 anh worker phải lấy xong cái khóa đầu tiên rồi mới được đi tiếp.
- Rào 2: Ép 2 anh thử lấy cái khóa thứ hai (tất nhiên là sẽ tạch). Chờ cả 2 tạch xong xuôi rồi mới thả cho chạy vòng lặp mới.

Nhớ là mọi loại barrier hay future đều phải set timeout nhé, tuyệt đối không xài `Thread.sleep` bừa bãi. Chi tiết xem tại [Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Thử nghiệm 1: Lặp N vòng nhưng chả được tích sự gì

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
            // T1 chạy A -> B
            Future<Boolean> first = executor.submit(() -> runSymmetrically(
                    lockA, lockB, bothHoldFirst, bothAttemptedSecond,
                    attempts, attemptBudget));
            // T2 chạy B -> A
            Future<Boolean> second = executor.submit(() -> runSymmetrically(
                    lockB, lockA, bothHoldFirst, bothAttemptedSecond,
                    attempts, attemptBudget));

            // Kiểm tra xem có anh nào xong việc không (hy vọng là không)
            assertFalse(first.get(5, TimeUnit.SECONDS));
            assertFalse(second.get(5, TimeUnit.SECONDS));
        }

        // Chắc chắn là cả hai anh đã thử mút chỉ 40 lần (mỗi anh 20 lần)
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
                awaitBarrier(bothHoldFirst); // Đợi nhau cầm khóa đầu
                boolean secondHeld = second.tryLock();
                try {
                    attempts.incrementAndGet();
                    awaitBarrier(bothAttemptedSecond); // Đợi nhau tạch khóa thứ hai
                    if (secondHeld) {
                        return true;
                    }
                } finally {
                    if (secondHeld) {
                        second.unlock();
                    }
                }
            } finally {
                first.unlock(); // Nhả khóa đầu rồi làm lại từ đầu
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

Nhìn code là thấy: 2 anh worker loay hoay 40 lần nhưng cuối cùng vẫn xôi hỏng bỏng không (trả về `false`). May mà có giới hạn `attemptBudget` nên cái test mới chịu dừng lại.

> **Nói ngắn gọn:** Chúng ta đang test xem chương trình có "tiến triển" không, chứ không phải test xem cái thread nó có đang thở không.

## Thử nghiệm 2: Sắp xếp thứ tự khóa để tụi nó làm xong việc

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

        start.countDown(); // Đếm ngược 3, 2, 1, CHẠY!
        first.get(5, TimeUnit.SECONDS);
        second.get(5, TimeUnit.SECONDS);
    }

    // Đổi qua đổi lại thì mèo lại hoàn mèo thôi
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

Ở đây ta áp dụng giải pháp 1 (sắp thứ tự). Vì T1 và T2 đều phải đi qua cánh cửa A trước, nên sẽ không bao giờ có chuyện đụng nhau. Future timeout ở đây đóng vai giám thị, lỡ code bị lỗi lùi (regression) thì nó sẽ báo ngay.

## Thử nghiệm 3: Dùng giới hạn retry (budget) thì kiểu gì cũng phải dừng

Bài test này dùng cho giải pháp số 2 (chờ ngẫu nhiên). Ta truyền vào một cục random có thể đoán trước (deterministic), và gài cho chúng nó kẹt cứng luôn.
Điều cần kiểm tra (assert) là:
- Hàm phải kết thúc bằng lỗi `CONTENDED` chứ không được chạy hoài.
- Đếm số lần thử phải khớp đúng với biến `maxAttempts`.
- Dù có chờ ngẫu nhiên cỡ nào thì tổng thời gian cũng không được lố deadline.
- Lúc về đích phải nhả hết mọi ổ khóa ra.
- Nếu nhập tham số bậy (`maxAttempts <= 0` chẳng hạn) thì phải báo lỗi ngay.
- Đang chờ mà có người giật cùi chỏ (interrupt) thì phải dừng cuộc chơi và trả về `INTERRUPTED`.

Lưu ý nhé: đừng có assert rằng cái hàm này "chắc chắn sẽ thành công". Thời gian chờ ngẫu nhiên (jitter) chỉ là tăng xác suất qua ải thôi, chứ điểm mấu chốt là nó phải BIẾT DỪNG LẠI.

## Thử nghiệm 4: Stress test để tra tấn ổ khóa

Sau khi test các case cứng ngắc ở trên, hãy nã hàng đống request vô một cặp channel và chạy.
Nhớ kiểm tra xem: tổng số ca thành công + ca thất bại + ca bị cắt ngang có bằng tổng số ném vào ban đầu không? Có bị lố giờ không? Dữ liệu có bị rác không?
Đừng rảnh mà đi log mấy cái retry này nhé, nát ổ cứng đó. Và nhớ, bài test này chỉ để dập thêm cho an tâm, không thay thế được bài Test 1 đâu.

## Kiểm chứng trên môi trường production

Đem code lên Prod rồi thì phải gắn đồ chơi (metric) vô để theo dõi:
- Số lần thử swap, số ca trót lọt, số ca kiệt sức rớt đài.
- Tần suất retry, thời gian chờ là bao nhiêu.
- Có ai bị đụng khóa nhiều không (dựa vô mã băm của channel).
- Chú ý cái CPU: nếu CPU hú còi inh ỏi, tỉ lệ retry nhảy dựng lên mà chả làm được mấy việc, thì đích thị là mùi Livelock rồi.

Một tấm ảnh thread dump thì chẳng nói lên được gì, hãy chụp nhiều tấm liên tục (time-series) thì mới chẩn bệnh được.

## Checklist chất lượng

- [ ] Livelock được tạo ra bài bản nhờ barrier, nói KHÔNG với hàm sleep.
- [ ] Lỡ code test có bị điên thì vòng lặp cũng tự dừng nhờ budget đàng hoàng.
- [ ] Chắc chắn rằng đã assert vụ "cắm đầu chạy nhưng chả ra ngô ra khoai".
- [ ] Test giải pháp "chọn thứ tự khóa" bằng cách gọi chéo 2 chiều cùng lúc.
- [ ] Cái gì có đợi (barrier, latch, future) là phải nhét timeout vô.
- [ ] Test kỹ vụ dọn dẹp khóa (unlock) và xử lý interrupt.
- [ ] Mấy cục Random phải được tiêm (inject) vào để dễ test lại.
- [ ] Trên Prod phải đo được tỷ lệ xong việc so với tỷ lệ thử lại, chứ đừng mỗi coi % CPU.
