# Kiểm thử đồng thời và xác minh trong môi trường thực tế

## Chiến lược kiểm thử

Case này chúng ta áp dụng luôn hai đòn hiểm để kiểm thử:

1. Chạy test "đặt bẫy": Dùng `CyclicBarrier` ép hai luồng phải phanh lại sau khi đọc dữ liệu, chờ nhau rồi mới cùng ghi đè. Cố tình gây lỗi để xem nó nát cỡ nào.
2. Chạy test đa luồng đạn đạo: Dùng `CountDownLatch` trên bản code xịn (đã sửa) để spam request xem ID có bị trùng và dữ liệu khách hàng có bị râu ông nọ cắm cằm bà kia không.

Trường hợp này không cần động đến `Testcontainers` (database thật chạy trong docker) vì lỗi này xuất phát từ code trên RAM, không liên quan gì đến DB cả.

## Tái hiện lỗi có kiểm soát

Dưới đây là một đoạn code chèn thêm cái "bốt gác" chỉ dùng để test. Cái rào chắn (barrier) này sẽ chặn cổ 2 luồng lại sau khi đọc dữ liệu, giúp ta chủ động gây ra cái sự cố "đọc-sửa-ghi" cực kỳ ổn định, lần nào chạy cũng lỗi đều như vắt chanh:

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

Cái `CyclicBarrier` đóng vai trò "cảnh sát giao thông", tạo trật tự rõ ràng giữa các thao tác trước và sau hàm `await()`. Thay vì ngồi đoán mò xài `Thread.sleep` (rất dễ xịt), chúng ta điều phối luồng một cách chắc cú.

> **Nói ngắn gọn:** Test này cố tình ép 2 luồng lọt hố cùng lúc, nên kết quả bao giờ cũng sai như dự đoán chứ không phụ thuộc vào hên xui nữa.

## Kiểm thử hồi quy cho code đã sửa

Đoạn test này sẽ nhét 100 ông khách vào chung vạch xuất phát, hô "chạy" một phát là cả 100 ông đồng loạt lao lên gọi service. Cuối cùng, mình chỉ việc chốt sổ xem có đủ 100 kết quả không, ID có ông nào bị trùng không và dữ liệu của ai có về đúng tay người đó không.

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

Dĩ nhiên, `ReceiptDraftService` và `DraftIdGenerator` trong đoạn test này là hàng chuẩn cơm mẹ nấu ở file [solutions](solutions.md) rồi nhé.

## Kiểm thử tải đồng thời bổ sung

Nếu rảnh, bạn có thể bê đoạn code bị lỗi chạy qua nhiều luồng lặp đi lặp lại rồi soi thông số:

```text
tổng số lời gọi
số lượng sequence khác biệt
result.customerId == input customerId
```

Nhưng lưu ý, test tải kiểu này tuỳ máy tuỳ thời điểm, hên thì ra lỗi xui thì nó pass xanh lè. Nên xem nó như đồ chơi xem thêm thôi, không thay thế được cái test gài bẫy có kiểm soát ở trên đâu.

## Kiểm tra lỗi và khả năng tiến triển

- Mấy trò xài latch/barrier wait là **bắt buộc** phải có timeout (thời gian chờ tối đa). Không là treo luôn máy.
- Gắn thêm `Future.get(timeout)` đề phòng luồng nó bị nghẽn đơ ra đó (deadlock).
- Luôn nhớ dọn rác, tắt executor trong khối `finally`.
- Đã test thì phải check kỹ thiếu sót kết quả nữa, chứ đừng chăm chăm mỗi vụ trùng ID.
- Lỡ bắt được `InterruptedException` thì nhớ đánh cờ lại (khôi phục interrupt status) cho luồng nhé.

## Xác minh trong môi trường thực tế

Đem lên production thì ráng mà ngồi canh me các dấu hiệu tà đạo này:

- Database báo lỗi văng miểng vì ràng buộc (constraint) trùng ID;
- Của người này mà tên người kia;
- User bấm cháy cả chuột vì request báo lỗi liên tục;
- Hàng đợi (queue) đầy nhóc, request chạy rùa bò;
- Lâu lâu deploy hoặc khởi động lại, ID lại quay xe về cấp số cũ.

Lưu ý: Đừng có lanh chanh in luôn dữ liệu cá nhân nhạy cảm của khách ra log chỉ để soi cho lẹ. Gom ID request với các cấu trúc log đàng hoàng mà xài.

## Checklist chất lượng của case

- [x] Lắp bẫy gài luồng xịn xò, không thèm xài `sleep` hên xui.
- [x] Chạy đa luồng te tua với code đã sửa.
- [x] Đã chốt kiểm tra tính độc lập và chống trùng lặp.
- [x] Timeout/chống treo đồ đạc dọn dẹp cẩn thận.
- [x] Không nhúng H2/Testcontainers vì lỗi này không ăn nhậu gì tới database.
