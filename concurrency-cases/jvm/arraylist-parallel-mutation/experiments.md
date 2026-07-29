# Kiểm thử đồng thời và xác minh trong môi trường thực tế

## Chiến lược kiểm thử

Không nên chỉ chạy `parallelStream()` một lần rồi kết luận `ArrayList` an toàn vì
size tình cờ đúng. Case dùng ba tầng:

1. mô hình interleaving có hook để chứng minh cơ chế lost update một cách
   deterministic;
2. stress test trên `ArrayList` thật để quan sát biểu hiện trên runtime/JDK đang
   dùng;
3. regression test cho solution, assertion trực tiếp cardinality, uniqueness và
   ordering invariant.

Mọi latch, barrier và future đều có timeout. Không dùng `Thread.sleep` làm điều
kiện pass/fail. Xem thêm
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Experiment 1: mô hình hóa lost update một cách deterministic

Không thể chèn latch vào giữa các instruction private của `ArrayList.add` mà
không instrument JDK. Test harness sau mô hình hóa đúng compound action “đọc
size → ghi slot → cập nhật size” và ép hai writer cùng chọn index 0.

```java
package com.example.quote;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class LostUpdateInterleavingTest {

    @Test
    void twoWritersCanOverwriteTheSameLogicalSlot() throws Exception {
        HookedAccumulator accumulator = new HookedAccumulator(2);
        CountDownLatch bothCapturedIndex = new CountDownLatch(2);
        CountDownLatch allowWrites = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            Future<?> first = executor.submit(() -> accumulator.add(
                    "result-a", bothCapturedIndex, allowWrites));
            Future<?> second = executor.submit(() -> accumulator.add(
                    "result-b", bothCapturedIndex, allowWrites));

            assertTrue(bothCapturedIndex.await(5, TimeUnit.SECONDS));
            allowWrites.countDown();

            first.get(5, TimeUnit.SECONDS);
            second.get(5, TimeUnit.SECONDS);
        }

        assertEquals(1, accumulator.size());
        assertEquals(1, accumulator.values().size());
    }

    private static final class HookedAccumulator {
        private final Object[] elements;
        private int size;

        private HookedAccumulator(int capacity) {
            this.elements = new Object[capacity];
        }

        void add(
                String value,
                CountDownLatch captured,
                CountDownLatch allowWrites
        ) {
            int index = size;
            captured.countDown();
            awaitOrFail(allowWrites);
            elements[index] = value;
            size = index + 1;
        }

        int size() {
            return size;
        }

        List<Object> values() {
            return java.util.Arrays.stream(elements, 0, size).toList();
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

Đây là executable model của race, không phải test xác nhận implementation nội bộ
của một phiên bản JDK. Nó giúp code review nhìn thấy non-atomic operation mà
không phụ thuộc scheduler.

> **Nói ngắn gọn:** deterministic test chứng minh cơ chế; stress test tiếp theo
> mới khảo sát biểu hiện của `ArrayList` thật.

## Experiment 2: stress test trên ArrayList thật

Test này có tính xác suất và nên đặt trong test suite riêng như `@Tag("stress")`.
Nếu nó không fail, code production vẫn sai vì API không có thread-safety
contract.

```java
package com.example.quote;

import org.junit.jupiter.api.RepeatedTest;
import org.junit.jupiter.api.Tag;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.stream.IntStream;

import static org.junit.jupiter.api.Assertions.assertEquals;

@Tag("stress")
class ArrayListConcurrentAddStressTest {

    @RepeatedTest(200)
    void concurrentAddsMustNotBeTrusted() throws Exception {
        int itemCount = 2_000;
        ArrayList<Integer> results = new ArrayList<>(itemCount);
        CountDownLatch start = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newFixedThreadPool(32)) {
            List<Future<?>> futures = IntStream.range(0, itemCount)
                    .mapToObj(value -> executor.submit(() -> {
                        awaitOrFail(start);
                        results.add(value);
                    }))
                    .toList();

            start.countDown();
            for (Future<?> future : futures) {
                future.get(5, TimeUnit.SECONDS);
            }
        }

        assertEquals(itemCount, results.size());
        assertEquals(itemCount, results.stream().distinct().count());
    }

    private static void awaitOrFail(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("start gate timed out");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("interrupted while waiting", exception);
        }
    }
}
```

Test có thể fail vì size/uniqueness sai hoặc một future bọc exception từ task.
Không dùng test này làm bằng chứng duy nhất và không tăng số vòng lặp vô hạn trong
CI. Lưu seed, JDK, CPU count và failure round khi dùng harness stress riêng.

## Experiment 3: regression test giữ cardinality và input order

Fake client cố tình hoàn tất `request-b` trước `request-a`. Solution vẫn phải trả
output theo input order vì coordinator collect future theo thứ tự submit.

```java
package com.example.quote;

import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.assertEquals;

class BatchQuoteServiceTest {

    @Test
    void returnsExactlyOneResultPerInputInInputOrder() {
        CountDownLatch secondCompleted = new CountDownLatch(1);

        QuoteClient client = request -> {
            if (request.requestId().equals("request-a")) {
                awaitOrFail(secondCompleted);
            } else {
                secondCompleted.countDown();
            }
            return new QuoteResult(
                    request.requestId(),
                    request.amountInMinorUnits() + 10
            );
        };

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            BatchQuoteService service = new BatchQuoteService(client, executor);
            List<QuoteRequest> input = List.of(
                    new QuoteRequest("request-a", 100),
                    new QuoteRequest("request-b", 200)
            );

            List<QuoteResult> result = service.quoteBatch(
                    input,
                    Duration.ofSeconds(5)
            );

            assertEquals(2, result.size());
            assertEquals(
                    List.of("request-a", "request-b"),
                    result.stream().map(QuoteResult::requestId).toList()
            );
            assertEquals(2, result.stream()
                    .map(QuoteResult::requestId)
                    .distinct()
                    .count());
        }
    }

    private static void awaitOrFail(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("dependency timed out");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("interrupted while waiting", exception);
        }
    }
}
```

Assertion dùng business identity `requestId`, không chỉ kiểm tra list không ném
exception. Nếu API không yêu cầu input order, bỏ assertion order nhưng vẫn giữ
cardinality và uniqueness.

## Experiment 4: failure phải cancel task còn lại

Test điều phối để task chậm chắc chắn đã chạy trước khi task còn lại fail. Khi
batch fail, `finally` phải gửi interrupt tới task chậm.

```java
@Test
void failureCancelsUnfinishedTaskAndPublishesNoPartialResult() throws Exception {
    CountDownLatch slowTaskStarted = new CountDownLatch(1);
    CountDownLatch interrupted = new CountDownLatch(1);
    CountDownLatch neverReleasedNormally = new CountDownLatch(1);

    QuoteClient client = request -> {
        if (request.requestId().equals("slow")) {
            slowTaskStarted.countDown();
            try {
                neverReleasedNormally.await(30, TimeUnit.SECONDS);
                throw new AssertionError("slow task should have been cancelled");
            } catch (InterruptedException exception) {
                interrupted.countDown();
                Thread.currentThread().interrupt();
                throw new IllegalStateException("slow task interrupted", exception);
            }
        }

        awaitOrFail(slowTaskStarted);
        throw new IllegalStateException("provider failed");
    };

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        BatchQuoteService service = new BatchQuoteService(client, executor);

        assertThrows(BatchQuoteException.class, () -> service.quoteBatch(
                List.of(
                        new QuoteRequest("fail", 100),
                        new QuoteRequest("slow", 200)
                ),
                Duration.ofSeconds(5)
        ));

        assertTrue(interrupted.await(5, TimeUnit.SECONDS));
    }
}
```

Để compile snippet, dùng static imports cho `assertThrows` và `assertTrue`; các
type/helper còn lại lấy từ Experiment 3. Test xác nhận cancellation signal, không
khẳng định remote system đã rollback. Client timeout và idempotency vẫn phải được
kiểm thử ở integration boundary.

## Kiểm thử progress semantics

Nếu dùng `ConcurrentLinkedQueue`, test progress reader chỉ nên yêu cầu:

- mọi item quan sát được đều hợp lệ;
- không có duplicate `requestId` nếu mỗi task publish một lần;
- progress count không vượt input count;
- final snapshot sau completion barrier có đủ item.

Không assert một progress snapshot giữa chừng phải chứa result vừa hoàn tất nếu
contract của iterator là weakly consistent. Nếu product cần acknowledgement mạnh
hơn, dùng event/channel protocol có sequence thay vì suy diễn từ collection
iterator.

## Xác minh trong môi trường thực tế

Ghi metric theo batch:

- `input_count`, `completed_count`, `failed_count`, `cancelled_count`;
- cardinality mismatch và duplicate `requestId` trước publish;
- batch deadline exceeded và remote timeout;
- executor active count, queue depth hoặc concurrency permit;
- task latency distribution và batch latency;
- số task vẫn chạy sau khi batch đã fail/cancel.

Log `batchId`, input count, outcome và elapsed time. Không log toàn bộ quote data
nhạy cảm. Với stress failure, lưu JDK version, processor count và iteration để có
thể tái hiện gần nhất.

## Checklist chất lượng của case

- [ ] Deterministic model ép hai writer dùng cùng logical index.
- [ ] Stress test thật được tách khỏi test suite ổn định.
- [ ] Không dùng `Thread.sleep` làm synchronization.
- [ ] Mọi latch và future đều có timeout.
- [ ] Fixed test kiểm tra cardinality, uniqueness và ordering.
- [ ] Failure test xác nhận unfinished task nhận cancellation signal.
- [ ] Interrupt status được khôi phục.
- [ ] Executor được đóng sau test.
- [ ] Progress assertion khớp collection semantics.
- [ ] Production validation kiểm tra invariant trước khi publish response.
