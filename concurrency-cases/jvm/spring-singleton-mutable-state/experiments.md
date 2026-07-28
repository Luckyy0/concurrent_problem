# Concurrency tests và production verification

## Testing strategy

Case dùng hai tầng kiểm thử:

1. deterministic test với `CyclicBarrier` để ép cả hai actor read cùng state
   trước khi write;
2. concurrent test trên fixed implementation với `CountDownLatch` để xác minh
   uniqueness và request isolation trên nhiều invocation.

Không cần Testcontainers vì case không phụ thuộc database semantics.

## Deterministic reproduction

Instrumented service dưới đây chỉ thuộc test. Barrier biểu diễn đúng
read-modify-write của broken implementation và tạo interleaving ổn định:

```java
package com.example.checkout;

import static org.junit.jupiter.api.Assertions.assertEquals;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import org.junit.jupiter.api.Test;

class BrokenReceiptDraftServiceConcurrencyTest {

    @Test
    void duplicatesSequenceAndLeaksCustomerAcrossRequests() throws Exception {
        var service = new InstrumentedBrokenService();
        ExecutorService executor = Executors.newFixedThreadPool(2);

        try {
            Future<Draft> aliceFuture =
                    executor.submit(() -> service.createDraft("alice"));
            Future<Draft> bobFuture =
                    executor.submit(() -> service.createDraft("bob"));

            Draft aliceResult = get(aliceFuture);
            Draft bobResult = get(bobFuture);

            assertEquals(42, aliceResult.sequence());
            assertEquals(42, bobResult.sequence());

            int correctlyIsolatedResults =
                    (aliceResult.customerId().equals("alice") ? 1 : 0)
                    + (bobResult.customerId().equals("bob") ? 1 : 0);

            // Cả hai result đọc cùng lastCustomerId; đúng tối đa một request.
            assertEquals(1, correctlyIsolatedResults);
        } finally {
            executor.shutdownNow();
        }
    }

    private static Draft get(Future<Draft> future)
            throws InterruptedException, ExecutionException {
        return future.get();
    }

    private static final class InstrumentedBrokenService {

        private final CyclicBarrier bothActorsHaveRead = new CyclicBarrier(2);
        private long nextSequence = 41;
        private String lastCustomerId;

        Draft createDraft(String customerId) {
            lastCustomerId = customerId;
            long observedSequence = nextSequence;

            await(bothActorsHaveRead);

            nextSequence = observedSequence + 1;
            return new Draft(nextSequence, lastCustomerId);
        }

        private static void await(CyclicBarrier barrier) {
            try {
                barrier.await();
            } catch (InterruptedException exception) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("test interrupted", exception);
            } catch (BrokenBarrierException exception) {
                throw new IllegalStateException("barrier broken", exception);
            }
        }
    }

    private record Draft(long sequence, String customerId) {
    }
}
```

Barrier có memory-consistency effect: action trước `await()` của mỗi actor
happens-before action sau barrier của actor khác. Vì vậy test không dựa vào
`Thread.sleep`.

## Regression test cho fixed implementation

Test này bắt đầu 100 invocation cùng một thời điểm và assert business invariant:

```java
package com.example.checkout;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.time.Duration;
import java.util.UUID;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;

class ReceiptDraftServiceConcurrencyTest {

    @Test
    void keepsEveryInvocationIsolated() throws Exception {
        int actors = 100;
        var ready = new CountDownLatch(actors);
        var start = new CountDownLatch(1);
        var done = new CountDownLatch(actors);
        var invocations = new ConcurrentLinkedQueue<Invocation>();
        var service = new ReceiptDraftService(UUID::randomUUID);
        ExecutorService executor = Executors.newFixedThreadPool(actors);

        try {
            for (int index = 0; index < actors; index++) {
                String inputCustomerId = "customer-" + index;

                executor.submit(() -> {
                    ready.countDown();
                    try {
                        start.await();
                        ReceiptDraft result =
                                service.createDraft(inputCustomerId);
                        invocations.add(
                                new Invocation(inputCustomerId, result)
                        );
                    } catch (InterruptedException exception) {
                        Thread.currentThread().interrupt();
                    } finally {
                        done.countDown();
                    }
                });
            }

            assertTrue(ready.await(5, TimeUnit.SECONDS));
            start.countDown();
            assertTrue(done.await(10, TimeUnit.SECONDS));

            assertEquals(actors, invocations.size());
            assertEquals(
                    actors,
                    invocations.stream()
                            .map(invocation -> invocation.result().id())
                            .distinct()
                            .count()
            );
            assertTrue(
                    invocations.stream().allMatch(invocation ->
                            invocation.inputCustomerId().equals(
                                    invocation.result().customerId()
                            )
                    )
            );
        } finally {
            executor.shutdownNow();
            assertTrue(executor.awaitTermination(
                    Duration.ofSeconds(5).toMillis(),
                    TimeUnit.MILLISECONDS
            ));
        }
    }

    private record Invocation(
            String inputCustomerId,
            ReceiptDraft result
    ) {
    }
}
```

`ReceiptDraftService` và `DraftIdGenerator` ở test dùng đúng fixed code trong
[solutions](solutions.md).

## Stress test bổ sung

Có thể chạy broken implementation thật với nhiều actor và nhiều vòng lặp rồi
so sánh:

```text
total calls
distinct sequences
result.customerId == input customerId
```

Stress test có thể không fail ở mọi máy/lần chạy; nó là diagnostic bổ sung,
không thay deterministic regression test.

## Failure và progress assertions

- Mọi latch/barrier wait phải có timeout.
- `Future.get(timeout)` nên được dùng nếu task có thể deadlock.
- Executor luôn được shutdown trong `finally`.
- Test phải fail nếu thiếu result, không chỉ khi có duplicate.
- Interrupt status phải được khôi phục.

## Production verification

Theo dõi các signal:

- duplicate business/correlation ID bị database constraint reject;
- mismatch giữa authenticated customer và result owner;
- số retry cùng request/idempotency key;
- thread pool queueing và request latency;
- restart/deployment có làm local sequence tái sử dụng hay không.

Không log raw sensitive customer data chỉ để chẩn đoán race. Dùng request ID và
structured audit field phù hợp.

## Quality gate của case

- [x] Deterministic interleaving không dùng sleep.
- [x] Fixed implementation được chạy với nhiều actor.
- [x] Assert uniqueness và request isolation.
- [x] Timeout/cleanup được khai báo.
- [x] Không dùng H2/Testcontainers vì không có database behavior.

