# Kiểm thử đồng thời và xác minh trong môi trường thực tế

## Chiến lược kiểm thử

Với cái bài toán này, mình có tận hai chỗ hổng (race window) để các luồng (thread) chen lấn nhau. Nên mình phải thiết kế hai ca test chắc ăn, chắc chắn ra lỗi (deterministic test):

1. Phục kích bắt hai anh thanh niên dừng lại ngay lúc vừa kiểm tra xong capacity nhưng chưa kịp tăng biến đếm.
2. Phục kích bắt quá trình transition ngay lúc vừa giảm `pending` xong nhưng chưa kịp chêm `active` lên.

Khi test các giải pháp chuẩn, phải soi thật kỹ: tổng có ngon không, có bao nhiêu anh húp được slot cuối cùng, có nhả vé đúng luật không, và có trò âm biến đếm hay nhả 2 lần không. Xài `CountDownLatch` hay Future có timeout nhé, dẹp ngay cái trò xài `Thread.sleep`!

Muốn nắm rõ ngọn ngành thì đọc thêm ở bài [Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Thí nghiệm 1: Hai anh em cùng chen giành slot cuối

Tạo một môi trường (test harness) giả lập giống y chang cái code bị lỗi với hai cái `AtomicInteger`. Rồi gài thêm chốt chặn (latch) vào ngay chỗ tranh chấp. Cài đặt lúc đầu là `active=9`, `pending=0`, `limit=10`.

```java
package com.example.connection;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class BrokenConnectionBudgetTest {

    @Test
    void twoActorsCanBothReserveTheLastSlot() throws Exception {
        // Chuẩn bị hàng họ: nhà chứa 10, đang có 9, pending 0
        HookedBrokenBudget budget = new HookedBrokenBudget(10, 9, 0);
        CountDownLatch bothPassedCheck = new CountDownLatch(2);
        CountDownLatch allowIncrement = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            // Đẩy 2 thanh niên nhào vô giành giật
            Future<Boolean> first = executor.submit(() ->
                    budget.tryReserve(bothPassedCheck, allowIncrement));
            Future<Boolean> second = executor.submit(() ->
                    budget.tryReserve(bothPassedCheck, allowIncrement));

            // Đợi 2 ông nhõi kia báo đã pass vòng giữ xe
            assertTrue(bothPassedCheck.await(5, TimeUnit.SECONDS));
            
            // Xong, mở chốt cho 2 ông cùng lướt vào tăng biến
            allowIncrement.countDown();

            // Phải xác nhận là cả 2 cùng tưởng mình ngon
            assertTrue(first.get(5, TimeUnit.SECONDS));
            assertTrue(second.get(5, TimeUnit.SECONDS));
        }

        // Kiểm tra hậu quả: nhà có 10 chỗ mà giờ thành 11 mạng!
        assertEquals(9, budget.active());
        assertEquals(2, budget.pending());
        assertEquals(11, budget.used());
    }

    private static final class HookedBrokenBudget {
        private final int limit;
        private final AtomicInteger active;
        private final AtomicInteger pending;

        private HookedBrokenBudget(
                int limit,
                int initialActive,
                int initialPending
        ) {
            this.limit = limit;
            this.active = new AtomicInteger(initialActive);
            this.pending = new AtomicInteger(initialPending);
        }

        boolean tryReserve(
                CountDownLatch passedCheck,
                CountDownLatch allowIncrement
        ) {
            // Bước 1: Check xem nhà còn chỗ không
            if (active.get() + pending.get() >= limit) {
                return false;
            }
            // Khựng lại báo tin: "anh đã check xong"
            passedCheck.countDown();
            // Đợi lệnh chốt sổ mới chạy tiếp
            awaitOrFail(allowIncrement);
            // Cắm đầu vào tăng
            pending.incrementAndGet();
            return true;
        }

        int active() {
            return active.get();
        }

        int pending() {
            return pending.get();
        }

        int used() {
            return active() + pending();
        }
    }

    private static void awaitOrFail(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("latch timed out");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("interrupted while waiting", exception);
        }
    }
}
```

Cái chốt quan trọng nhất không phải là xem `AtomicInteger` có chửi bới ném lỗi hay không; cái quan trọng là bắt quả tang cái `used=11` dù cái cổng limit chỉ có 10.

> **Nói ngắn gọn:** Dùng cái chốt (latch) để dụ 2 thằng cùng chốt kết quả dựa trên cái thông tin cũ rích, rồi xua 2 thằng cùng một lúc ập vô đôn số lên.

## Thí nghiệm 2: Sơ hở mở ra khe hở dung lượng ảo

Giờ mình gài chốt ở chỗ khác. Lúc một luồng đang hoàn tất công việc, vừa giảm `pending--` xong thì đứng lại. Có đứa khác bay vào là lọt khe ngay trước khi thanh niên kia kịp `active++`.

```java
@Test
void pendingToActiveTransitionCanExposeFalseCapacity() throws Exception {
    // Nhà lúc này đang đầy cứng: active 9, pending 1, tính ra là 10
    TransitionGapBudget budget = new TransitionGapBudget(10, 9, 1);
    CountDownLatch gapOpened = new CountDownLatch(1);
    CountDownLatch reservationFinished = new CountDownLatch(1);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        // Cho một thanh niên làm thủ tục thăng cấp
        Future<?> completion = executor.submit(() -> budget.creationSucceeded(
                gapOpened,
                reservationFinished
        ));
        
        // Quăng thanh niên thứ hai vào để luồn lách khe hở
        Future<Boolean> reservation = executor.submit(() -> {
            awaitOrFail(gapOpened); // Chờ khe hở ma thuật mở ra
            boolean accepted = budget.tryReserve(); // Dzo!
            reservationFinished.countDown(); // Xong, báo cáo
            return accepted;
        });

        // Thanh niên chui khe chắc chắn sẽ thành công
        assertTrue(reservation.get(5, TimeUnit.SECONDS));
        completion.get(5, TimeUnit.SECONDS);
    }

    // Kết quả mĩ mãn: 11 thanh niên chen nhau trong cái nhà 10 chỗ
    assertEquals(10, budget.active());
    assertEquals(1, budget.pending());
    assertEquals(11, budget.used());
}

private static final class TransitionGapBudget {
    private final int limit;
    private final AtomicInteger active;
    private final AtomicInteger pending;

    private TransitionGapBudget(int limit, int active, int pending) {
        this.limit = limit;
        this.active = new AtomicInteger(active);
        this.pending = new AtomicInteger(pending);
    }

    void creationSucceeded(
            CountDownLatch gapOpened,
            CountDownLatch reservationFinished
    ) {
        pending.decrementAndGet();
        // Hé cửa sổ cho khe hở xuất hiện
        gapOpened.countDown();
        // Đứng chơi, chờ thằng nhõi kia lách vào xong
        awaitOrFail(reservationFinished);
        // Tăng active cho trọn gói
        active.incrementAndGet();
    }

    boolean tryReserve() {
        if (active.get() + pending.get() >= limit) {
            return false;
        }
        pending.incrementAndGet();
        return true;
    }

    int active() {
        return active.get();
    }

    int pending() {
        return pending.get();
    }

    int used() {
        return active() + pending();
    }
}
```

(Đoạn code trên mượn lại ba cái râu ria của thí nghiệm 1). Ở đây mình bắt quả tang được cái trò tự chia đôi giai đoạn biến 1 cái thành 2 cái tách biệt.

## Thí nghiệm 3: Xem CAS ra tay trừng trị

Mình nhét 9 chú vào nhà trước (bằng API hịn), sau đó lôi ra tận 32 cái Virtual Thread rồi kêu tụi nó ập vào xâu xé giành giật cái slot thứ 10 cuối cùng.

```java
package com.example.connection;

import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class ProviderConnectionBudgetTest {

    @Test
    void exactlyOneActorWinsTheLastSlot() throws Exception {
        // Budget bao bọc bằng xịn (CAS)
        ProviderConnectionBudget budget = new ProviderConnectionBudget(
                new ConnectionBudgetProperties(10)
        );
        for (int index = 0; index < 9; index++) {
            assertTrue(budget.tryReserveCreation());
            budget.creationSucceeded();
        }

        int actorCount = 32;
        CountDownLatch ready = new CountDownLatch(actorCount);
        CountDownLatch start = new CountDownLatch(1);
        List<Future<Boolean>> futures = new ArrayList<>(actorCount);

        try (ExecutorService executor =
                     Executors.newVirtualThreadPerTaskExecutor()) {
            for (int actor = 0; actor < actorCount; actor++) {
                futures.add(executor.submit(() -> {
                    ready.countDown();
                    awaitOrFail(start);
                    return budget.tryReserveCreation();
                }));
            }

            assertTrue(ready.await(5, TimeUnit.SECONDS));
            start.countDown(); // Xung phong!

            int accepted = 0;
            for (Future<Boolean> future : futures) {
                if (future.get(5, TimeUnit.SECONDS)) {
                    accepted++; // Đếm số anh được duyệt
                }
            }
            // Chỉ 1 anh hùng duy nhất sống sót qua cửa tử
            assertEquals(1, accepted);
        }

        BudgetState state = budget.stateForTest();
        assertEquals(9, state.active());
        assertEquals(1, state.pending());
        assertEquals(10, state.used());
    }

    private static void awaitOrFail(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("latch timed out");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("interrupted while waiting", exception);
        }
    }
}
```

Phải đếm coi bao nhiêu đứa win nha! Đừng có chỉ hì hục check mỗi cái `used <= limit` không thôi. Lỡ có bug nào đó chảnh chọe sút hết 32 ông ra ngoài đường dù còn 1 slot trống thì check kiểu kia đâu lòi ra được.

## Thí nghiệm 4: Phép thuật CAS chuyển trạng thái không rơi vãi chỗ

Khởi tạo ở mốc `active=9`, `pending=1`. Mình cho một anh làm nhiệm vụ thăng cấp (pending -> active) xen kẽ cùng lúc với 31 tay săn mồi xông tới giật slot. Kết quả là CAS bảo kê tận răng, không thằng săn mồi nào được vào vì tổng vẫn đang là 10.

```java
@Test
void pendingToActiveTransitionConservesUsedCapacity() throws Exception {
    ProviderConnectionBudget budget = new ProviderConnectionBudget(
            new ConnectionBudgetProperties(10)
    );
    for (int index = 0; index < 9; index++) {
        assertTrue(budget.tryReserveCreation());
        budget.creationSucceeded();
    }
    assertTrue(budget.tryReserveCreation()); // Cục tạ chiếm nốt chỗ cuối

    CountDownLatch start = new CountDownLatch(1);
    List<Future<Boolean>> reservations = new ArrayList<>();

    try (ExecutorService executor =
                 Executors.newVirtualThreadPerTaskExecutor()) {
        Future<?> completion = executor.submit(() -> {
            awaitOrFail(start);
            budget.creationSucceeded(); // Chuyển hóa
        });
        for (int actor = 0; actor < 31; actor++) {
            reservations.add(executor.submit(() -> {
                awaitOrFail(start);
                return budget.tryReserveCreation(); // Đòi giành giật
            }));
        }

        start.countDown();
        completion.get(5, TimeUnit.SECONDS);
        // Buồn thay cho 31 thanh niên
        for (Future<Boolean> reservation : reservations) {
            assertFalse(reservation.get(5, TimeUnit.SECONDS));
        }
    }

    BudgetState state = budget.stateForTest();
    assertEquals(10, state.active());
    assertEquals(0, state.pending());
    assertEquals(10, state.used()); // Vững như kiềng ba chân
}
```

## Thí nghiệm 5: Thẻ xịn (Permit handle) chặn đứng tay lấp lửng đòi nhả vé 2 lần

```java
@Test
void closingPermitTwiceDoesNotCreateExtraCapacity() {
    // Nhà chỉ có 1 ghế
    SemaphoreConnectionBudget budget = new SemaphoreConnectionBudget(1);

    // Xí ghế duy nhất
    ConnectionPermit first = budget.tryAcquire().orElseThrow();
    // Đứa tới sau thì nhịn
    assertTrue(budget.tryAcquire().isEmpty());

    // Nhả ghế 1 lần
    first.close();
    // Chơi dơ, nhả ghế lần 2 coi sao
    first.close();

    // Giờ ghế trống, anh số 2 xin vào
    ConnectionPermit second = budget.tryAcquire().orElseThrow();
    // Vẫn không cho phép ghế đẻ ra thêm
    assertTrue(budget.tryAcquire().isEmpty());
    second.close();
}
```

Chỗ này test độ cứng cựa của cái vòng đời (lifecycle identity). Cái này mấy biến đếm rỗng (aggregate counter) bó tay toàn tập.

## Kiểm tra dọn dẹp lỗi và giới hạn ngầm (Underflow)

Đừng quên viết ba cái unit test cơ bản tuần tự như sau:
- Khai tử (`creationFailed()`) mà chưa có vé pending nào (`pending=0`) là phải sập luôn cái app báo lỗi, không được để lòi ra số âm.
- Đóng kết nối (`connectionClosed()`) mà active bằng 0 cũng phải cho ra chuồng gà.
- Timeout thì gọi quét rác 1 lần duy nhất.
- Báo mồi xong rồi cố tình báo mồi thiu cho cùng 1 vé thì phải bị chặn đứng.
- Gặp lỗi ngoài lề thì cũng không được vác cái CAS ra chạy quay vòng.

Xài lock hay Semaphore thì nhớ phải test cái dụ timeout ngắt quãng (interrupt) để không bị kẹt mãi mãi nha.

## Kiểm thử chịu tải dập tơi tả (Stress test)

Xong xuôi mấy trò ở trên thì mở máy cày, tung hàng chục thằng ập vào làm trò con bò: đứt gánh, nhả vé, chốt vé loạn ngầu. Xong chốt sổ đo đếm xem mấy thứ sau có đúng không:
```text
active >= 0
pending >= 0
active + pending <= limit
Số lượt lấy vé thành công = tổng số win + số đứt gánh + số đang ngâm
Số active lọt vào nhà = số nhả cửa + số active còn sót
```

## Xác minh "hàng real" trong môi trường thực tế

Xách đồ lên ngó dashboard của cái server (instance) nha:
- Chỉ chĩa cam vào snapshot của cụm `active`, `pending`, `used`, `limit` nha.
- Hóng xem số người bị hất hủi ra ngoài (capacity rejection).
- Kiểm số vé thành công / xịt / ngâm dấm timeout.
- Coi thử có thằng nào dính bẫy ngâm mỏ neo chờ lock dài cổ hông.
- Có ai gọi nhầm nhả vé 2 lần chưa.
- Lòi ra cái mụn `used > limit` hay âm tiền thì bắt lỗi tức khắc.

Nếu rò rỉ dung lượng (lifecycle leak) thì lôi log ra điều tra, không rảnh tay mà hack set số tay lại từ đầu đâu, coi chừng ngập hệ thống.

## Danh sách chốt bài (Checklist uy tín)

- [ ] Lập cái bẫy đua (race window) bằng latch, tuyệt đối không xài hàm ngủ (sleep).
- [ ] Gài timeout cho mọi chốt / future.
- [ ] Soi nát đoạn code lỗi bắt được số vượt rào.
- [ ] Test được màn CAS sút văng đám thừa thải, bảo kê 1 tay lên bảng.
- [ ] Test đoạn transition đảm bảo 1 đổi 1.
- [ ] Đã quét sạch mấy mầm mống double-release và số âm.
- [ ] Sạch sẽ ngắt nhịp (interrupt) trong bộ test phụ.
- [ ] Đã shutdown cái executor như dân chuyên.
- [ ] Đọc metric đúng điệu từ chung một khung ảnh snapshot.
- [ ] Không ngáo ngơ nhầm lẫn chuyện chia quota giữa nhiều cái server khác nhau.
