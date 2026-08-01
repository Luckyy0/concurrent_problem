# Kiểm thử đồng thời và xác minh trong môi trường thực tế

## Chiến lược kiểm thử

Case dùng hai tầng kiểm thử:

1. kiểm thử có kiểm soát dùng `CyclicBarrier` để buộc hai tác nhân cùng đọc một trạng thái
   trước khi được phép ghi;
2. kiểm thử nhiều luồng dùng `CountDownLatch` trên code đã sửa để kiểm tra ID không
   trùng và dữ liệu của từng request không bị lẫn.

Không cần Testcontainers vì case không phụ thuộc vào hành vi của database.

## Tái hiện lỗi có kiểm soát

Service có điểm điều phối dưới đây chỉ được dùng trong kiểm thử. Barrier dừng hai
luồng sau bước đọc, nhờ đó tái hiện ổn định chuỗi đọc–sửa–ghi bị lỗi:

```java
package com.example.checkout;

import static org.junit.jupiter.api.Assertions.assertEquals;

import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;
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

            // Cả hai kết quả đọc cùng lastCustomerId; đúng tối đa một request.
            assertEquals(1, correctlyIsolatedResults);
        } finally {
            executor.shutdownNow();
        }
    }

    private static Draft get(Future<Draft> future)
            throws InterruptedException, ExecutionException, TimeoutException {
        return future.get(5, TimeUnit.SECONDS);
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
                barrier.await(5, TimeUnit.SECONDS);
            } catch (InterruptedException exception) {
                Thread.currentThread().interrupt();
                throw new IllegalStateException("test interrupted", exception);
            } catch (BrokenBarrierException exception) {
                throw new IllegalStateException("barrier broken", exception);
            } catch (TimeoutException exception) {
                throw new IllegalStateException("barrier timed out", exception);
            }
        }
    }

    private record Draft(long sequence, String customerId) {
    }
}
```

`CyclicBarrier` tạo quan hệ xảy ra-trước giữa các thao tác trước và sau
`await()`. Nhờ vậy, kiểm thử kiểm soát thứ tự bằng cơ chế đồng bộ thay vì đoán thời
điểm bằng `Thread.sleep`.

> **Nói ngắn gọn:** kiểm thử chủ động đặt hai luồng đúng vào cửa sổ gây lỗi, nên kết
> quả không phụ thuộc vào may rủi của bộ lập lịch.

## Kiểm thử hồi quy cho code đã sửa

Kiểm thử này cho 100 lời gọi bắt đầu cùng thời điểm, sau đó kiểm tra các quy tắc bắt
buộc: đủ kết quả, ID không trùng và dữ liệu khách hàng không bị lẫn giữa các request.

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

`ReceiptDraftService` và `DraftIdGenerator` ở kiểm thử dùng đúng mã đã sửa trong
[solutions](solutions.md).

## Kiểm thử tải đồng thời bổ sung

Có thể chạy cách triển khai lỗi với nhiều tác nhân và nhiều vòng lặp rồi
so sánh:

```text
tổng số lời gọi
số lượng sequence khác biệt
result.customerId == input customerId
```

Kiểm thử tải đồng thời có thể không phát hiện lỗi ở mọi máy hoặc mọi lần chạy.
Nó chỉ là bằng chứng bổ sung và không thay thế kiểm thử có thứ tự được kiểm soát.

## Kiểm tra lỗi và khả năng tiến triển

- Mọi latch/barrier wait phải có timeout.
- Dùng `Future.get(timeout)` nếu task có thể bị deadlock.
- Luôn đóng executor trong `finally`.
- Kiểm thử phải thất bại khi thiếu kết quả, không chỉ khi phát hiện ID trùng.
- Phải khôi phục trạng thái ngắt (interrupt status) khi bắt `InterruptedException`.

## Xác minh trong môi trường thực tế

Theo dõi các tín hiệu sau:

- định danh nghiệp vụ hoặc định danh tương quan bị trùng bị ràng buộc database từ chối;
- sự không khớp giữa khách hàng đã xác thực và chủ sở hữu kết quả;
- số lần thử lại cùng request/khóa idempotency;
- hàng đợi của thread pool và độ trễ của request;
- khởi động lại hoặc triển khai có làm sequence cục bộ tái sử dụng hay không.

Không ghi trực tiếp dữ liệu khách hàng nhạy cảm vào log chỉ để chẩn đoán tranh
chấp. Dùng ID của request và trường kiểm toán có cấu trúc phù hợp.

## Checklist chất lượng của case

- [x] Xen kẽ tất định (deterministic interleaving) không dùng sleep.
- [x] Mã đã sửa được chạy với nhiều luồng.
- [x] Khẳng định tính duy nhất và tính độc lập của request.
- [x] Timeout/dọn tiết được khai báo.
- [x] Không dùng H2/Testcontainers vì không có hành vi database.
