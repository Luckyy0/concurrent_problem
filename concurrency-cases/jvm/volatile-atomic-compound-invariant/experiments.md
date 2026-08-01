# Kiểm thử đồng thời và xác minh trong môi trường thực tế

## Chiến lược kiểm thử

Tình huống có hai cửa sổ tương tranh (race window) khác nhau, nên cần hai kiểm thử tất định (deterministic test):

1. dừng hai tác nhân (actor) sau bước kiểm tra capacity nhưng trước khi increment;
2. dừng transition sau khi giảm pending nhưng trước khi tăng active.

Kiểm thử hồi quy (regression test) cho giải pháp phải kiểm tra invariant tổng, số lượng actor giành được slot
cuối, tính bảo toàn của transition và hành vi underflow/double-release. Dùng
latch/future có timeout, không dùng `Thread.sleep`.

Nguyên tắc chung nằm tại
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Thí nghiệm 1: hai actor cùng giữ chỗ (reserve) slot cuối

Bộ kiểm thử (test harness) dùng đúng hai `AtomicInteger` như đoạn code lỗi và thêm latch tại race
window. State bắt đầu ở `active=9`, `pending=0`, `limit=10`.

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
        HookedBrokenBudget budget = new HookedBrokenBudget(10, 9, 0);
        CountDownLatch bothPassedCheck = new CountDownLatch(2);
        CountDownLatch allowIncrement = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            Future<Boolean> first = executor.submit(() ->
                    budget.tryReserve(bothPassedCheck, allowIncrement));
            Future<Boolean> second = executor.submit(() ->
                    budget.tryReserve(bothPassedCheck, allowIncrement));

            assertTrue(bothPassedCheck.await(5, TimeUnit.SECONDS));
            allowIncrement.countDown();

            assertTrue(first.get(5, TimeUnit.SECONDS));
            assertTrue(second.get(5, TimeUnit.SECONDS));
        }

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
            if (active.get() + pending.get() >= limit) {
                return false;
            }
            passedCheck.countDown();
            awaitOrFail(allowIncrement);
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

Assertion quan trọng không phải là hai `AtomicInteger` có ném exception hay không; đó là
kiểm tra `used=11` dù limit bằng 10.

> **Nói ngắn gọn:** latch buộc cả hai quyết định dựa trên cùng một state cũ, rồi mới
> cho hai quá trình increment riêng lẻ chạy.

## Thí nghiệm 2: transition mở capacity gap giả

Harness tiếp theo dừng thread hoàn tất (completion thread) sau `pending--`. Request mới chắc chắn sẽ
reserve trong gap trước khi completion thread chạy `active++`.

```java
@Test
void pendingToActiveTransitionCanExposeFalseCapacity() throws Exception {
    TransitionGapBudget budget = new TransitionGapBudget(10, 9, 1);
    CountDownLatch gapOpened = new CountDownLatch(1);
    CountDownLatch reservationFinished = new CountDownLatch(1);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<?> completion = executor.submit(() -> budget.creationSucceeded(
                gapOpened,
                reservationFinished
        ));
        Future<Boolean> reservation = executor.submit(() -> {
            awaitOrFail(gapOpened);
            boolean accepted = budget.tryReserve();
            reservationFinished.countDown();
            return accepted;
        });

        assertTrue(reservation.get(5, TimeUnit.SECONDS));
        completion.get(5, TimeUnit.SECONDS);
    }

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
        gapOpened.countDown();
        awaitOrFail(reservationFinished);
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

Đoạn mã (snippet) dùng lại các phần import, static assertion và `awaitOrFail` từ Thí nghiệm 1.
Đây là một race khác với việc hai actor cùng kiểm tra: chỉ một thread reservation vẫn có thể
vượt quá capacity do transition của nhiều counter bị tách đôi.

## Thí nghiệm 3: CAS chỉ cho một actor lấy slot cuối

Khởi tạo 9 active connection bằng public transition, sau đó cho 32 virtual thread
cạnh tranh slot thứ 10.

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
            start.countDown();

            int accepted = 0;
            for (Future<Boolean> future : futures) {
                if (future.get(5, TimeUnit.SECONDS)) {
                    accepted++;
                }
            }
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

Test sẽ xác nhận (assert) cả số actor thắng và state cuối. Nếu chỉ assert `used <= limit` thì có thể bỏ
sót bug khi tất cả actor đều bị reject dù còn đúng một slot.

## Thí nghiệm 4: CAS transition không tạo capacity gap

Bắt đầu ở `active=9`, `pending=1`. Một actor chuyển pending sang active trong khi
31 actor khác thử reserve. Dù CAS xen kẽ (interleave) theo thứ tự nào, không actor mới nào
được chấp nhận vì tổng luôn bằng 10.

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
    assertTrue(budget.tryReserveCreation());

    CountDownLatch start = new CountDownLatch(1);
    List<Future<Boolean>> reservations = new ArrayList<>();

    try (ExecutorService executor =
                 Executors.newVirtualThreadPerTaskExecutor()) {
        Future<?> completion = executor.submit(() -> {
            awaitOrFail(start);
            budget.creationSucceeded();
        });
        for (int actor = 0; actor < 31; actor++) {
            reservations.add(executor.submit(() -> {
                awaitOrFail(start);
                return budget.tryReserveCreation();
            }));
        }

        start.countDown();
        completion.get(5, TimeUnit.SECONDS);
        for (Future<Boolean> reservation : reservations) {
            assertFalse(reservation.get(5, TimeUnit.SECONDS));
        }
    }

    BudgetState state = budget.stateForTest();
    assertEquals(10, state.active());
    assertEquals(0, state.pending());
    assertEquals(10, state.used());
}
```

Snippet cần static import `assertFalse`; các helper/import khác được dùng lại từ
Thí nghiệm 3.

## Thí nghiệm 5: permit handle chống double release

```java
@Test
void closingPermitTwiceDoesNotCreateExtraCapacity() {
    SemaphoreConnectionBudget budget = new SemaphoreConnectionBudget(1);

    ConnectionPermit first = budget.tryAcquire().orElseThrow();
    assertTrue(budget.tryAcquire().isEmpty());

    first.close();
    first.close();

    ConnectionPermit second = budget.tryAcquire().orElseThrow();
    assertTrue(budget.tryAcquire().isEmpty());
    second.close();
}
```

Nếu `close()` giải phóng (release) semaphore hai lần, lần acquire cuối sẽ thành công sai. Test
này kiểm tra định danh vòng đời (lifecycle identity) mà một aggregate counter đơn thuần không thể hiện.

## Kiểm tra failure và underflow

Bổ sung các unit test tuần tự nhưng quan trọng:

- `creationFailed()` khi `pending=0` phải fail, không tạo counter âm;
- `connectionClosed()` khi `active=0` phải fail;
- creation timeout gọi phần cleanup đúng một lần;
- callback thành công rồi gọi tiếp callback thất bại cho cùng một reservation sẽ bị từ chối bởi
  reservation token/state machine;
- exception bên ngoài CAS loop không khiến transition chạy lặp lại.

Nếu dùng lock với `tryLock`, cần test timeout và khôi phục (restoration) interrupt riêng. Nếu
dùng blocking `Semaphore.acquire`, mọi test phải có bounded future timeout và
cleanup permit trong khối `finally`/try-with-resources.

## Kiểm thử chịu tải (Stress test) bổ sung

Sau khi chạy deterministic test, chạy nhiều actor ngẫu nhiên thực hiện reserve, succeed, fail và
close. Ghi lại mọi state sau transition rồi assert:

```text
active >= 0
pending >= 0
active + pending <= limit
số reservation thành công = success + failure + pending còn lại
số active được tạo = close + active còn lại
```

Dùng seed cố định và state machine model để so sánh. Stress test tăng độ phủ
interleaving nhưng không thay thế được các deterministic race test.

## Xác minh trong môi trường thực tế

Theo dõi theo application instance:

- `active`, `pending`, `used`, `limit` từ cùng một snapshot;
- số lần từ chối (capacity rejection count) và tỷ lệ từ chối (rejection rate);
- handshake thành công/thất bại/timeout;
- phân phối (distribution) của việc retry CAS hoặc thời gian chờ lock;
- reservation age và các pending slot quá hạn;
- các duplicate callback/thử nghiệm double-close;
- vi phạm invariant (violation) `used > limit` hoặc counter âm.

Khi có vi phạm invariant, cần log lại state, loại transition (transition type) và reservation ID, đồng thời
alert ngay. Không tự động reset counter vì việc reset có thể che giấu lỗi rò rỉ vòng đời (lifecycle leak) và
cho phép tình trạng oversubscription lớn hơn.

## Danh sách kiểm tra (Checklist) chất lượng của tình huống

- [ ] Hai cửa sổ tương tranh (race window) được tái hiện bằng latch, không dùng sleep.
- [ ] Mọi latch/future đều có timeout.
- [ ] Đoạn test mã lỗi (broken test) assert được state vượt limit.
- [ ] CAS test assert đúng một actor thắng slot cuối.
- [ ] Transition test assert used capacity được bảo toàn.
- [ ] Lỗi underflow và duplicate release được kiểm tra.
- [ ] Trạng thái ngắt (Interrupt status) được khôi phục trong helper.
- [ ] Executor được đóng sau khi chạy test.
- [ ] Số liệu production (Production metric) được đọc từ một state snapshot.
- [ ] Quota multi-instance không bị nhầm với local invariant.
