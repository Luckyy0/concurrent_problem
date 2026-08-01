# Môi Trường Thực Nghiệm: Xác Minh Lỗ Hổng Check-Then-Act

## 1. Chiến Lược Kiểm Thử (Testing Strategy)

Để test vụ này, mình sẽ dùng 3 chiêu như sau:

1. Ép các luồng chạy đan xen nhau để cố tình tạo ra tình huống 1 Key đẻ ra tới 2 tài nguyên.
2. Dùng một "bầy" luồng (Mass actors) lao vào gọi hàm `computeIfAbsent` cùng lúc để xem thử Factory có bị gọi lố hơn 1 lần không.
3. Test kịch bản luồng cắm mỏ neo (Placeholder) nhưng bị lỗi giữa chừng xem nó có biết tự nhổ mỏ neo đi để luồng sau còn vào được không.

Bài này mình không cần xài tới `Testcontainers` (do lỗi nằm trong bộ nhớ Java thôi). Chú ý là mọi chỗ đứng đợi (như latch, barrier) đều phải cài thời gian chờ tối đa (timeout) nhé. Tuyệt đối không dùng `Thread.sleep()` để mô phỏng luồng chạy.

## 2. Thí Nghiệm 1: Định Hình Cơ Chế Thất Thoát (Deterministic Lost Update)

Mình sẽ viết một cái rào cản (Barrier) ngầm để ép 2 luồng bị kẹt lại ngay lúc chuẩn bị gọi hàm `open(...)`. Mục đích là làm cho cả 2 luồng cùng thấy Map đang rỗng trước khi kịp lưu vào:

```java
package com.example.registry;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotSame;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Stream;
import org.junit.jupiter.api.Test;

class BrokenManagedResourceRegistryConcurrencyTest {

    @Test
    void createsTwoResourcesForOneKey() throws Exception {
        var factory = new BarrierFactory();
        var registry = new ManagedResourceRegistry(factory);
        ExecutorService executor = Executors.newFixedThreadPool(2);

        try {
            Future<ManagedResource> firstFuture =
                    executor.submit(() -> registry.register("tenant-a"));
            Future<ManagedResource> secondFuture =
                    executor.submit(() -> registry.register("tenant-a"));

            ManagedResource first = firstFuture.get(5, TimeUnit.SECONDS);
            ManagedResource second = secondFuture.get(5, TimeUnit.SECONDS);
            ManagedResource stored = registry.find("tenant-a");

            // Rành rành là Factory bị gọi 2 lần (lỗi nặng)
            assertEquals(2, factory.openCalls());
            // Nhưng check Map thì thấy size = 1 (Lừa đảo thật!)
            assertEquals(1, registry.size());
            // 2 luồng nhận về 2 object khác hẳn nhau
            assertNotSame(first, second);
            // Một đối tượng bị bỏ rơi, không nằm trong Map
            assertTrue(stored == first || stored == second);
            assertEquals(
                    1,
                    Stream.of(first, second)
                            .filter(resource -> resource == stored)
                            .count()
            );
        } finally {
            executor.shutdownNow();
            assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
        }
    }

    private static final class BarrierFactory
            implements ManagedResourceFactory {

        private final AtomicInteger calls = new AtomicInteger();
        private final CountDownLatch bothCallsEntered = new CountDownLatch(2);

        @Override
        public ManagedResource open(String resourceKey) {
            int call = calls.incrementAndGet();
            bothCallsEntered.countDown();
            await(bothCallsEntered); // Bắt thằng 1 đứng đợi thằng 2 vô luôn rồi mới chạy tiếp
            return new TestResource(resourceKey + "-" + call);
        }

        int openCalls() {
            return calls.get();
        }
    }

    private record TestResource(String id) implements ManagedResource {
        @Override
        public void close() {
        }
    }

    private static void await(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("Hết hạn rào cản luồng");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Gián đoạn kiểm thử", exception);
        }
    }
}
```

Kết quả: Chỉ check `map.size() == 1` thì chả nói lên được gì cả. Chỗ chết là Factory bị gọi 2 lần và có 1 object rác nằm trôi nổi ngoài kia.

> **Nguyên tắc kỹ thuật:** Đừng chỉ chăm chăm soi xem Map có lưu đúng 1 dòng hay không. Bạn phải test xem có bị đẻ lố tài nguyên và rò rỉ hay không mới quan trọng.

## 3. Thí Nghiệm 2: Đánh Giá Phương Thức computeIfAbsent

Giờ mình tạo ra tận 32 luồng để spam cùng lúc vào 1 Key. Dùng cái rào cản chặn lại để hở toang cái cửa sổ lỗi ra. Nhưng kết quả phải là chỉ có 1 luồng duy nhất được tạo tài nguyên:

```java
package com.example.registry;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import org.junit.jupiter.api.Test;

class ManagedResourceRegistryConcurrencyTest {

    @Test
    void publishesOneResourceToEveryCaller() throws Exception {
        int actors = 32;
        var factory = new BlockingFactory();
        var registry = new ManagedResourceRegistry(factory);
        var ready = new CountDownLatch(actors);
        var start = new CountDownLatch(1);
        var attemptingRegister = new CountDownLatch(actors);
        var done = new CountDownLatch(actors);
        var results = new ConcurrentLinkedQueue<ManagedResource>();
        var failures = new ConcurrentLinkedQueue<Throwable>();
        ExecutorService executor = Executors.newFixedThreadPool(actors);

        try {
            for (int index = 0; index < actors; index++) {
                executor.submit(() -> {
                    ready.countDown();
                    try {
                        if (!start.await(5, TimeUnit.SECONDS)) {
                            throw new IllegalStateException("Thất bại tín hiệu Khởi tranh");
                        }
                        attemptingRegister.countDown();
                        results.add(registry.register("tenant-a"));
                    } catch (InterruptedException exception) {
                        Thread.currentThread().interrupt();
                        failures.add(exception);
                    } catch (RuntimeException exception) {
                        failures.add(exception);
                    } finally {
                        done.countDown();
                    }
                });
            }

            assertTrue(ready.await(5, TimeUnit.SECONDS));
            start.countDown(); // Phát súng ra lệnh cả 32 thằng cùng xông lên
            assertTrue(attemptingRegister.await(5, TimeUnit.SECONDS));
            assertTrue(factory.openEntered.await(5, TimeUnit.SECONDS));
            factory.allowOpenToFinish.countDown();
            assertTrue(done.await(10, TimeUnit.SECONDS));

            assertTrue(failures.isEmpty());
            assertEquals(actors, results.size());
            // Tuyệt đối chỉ 1 thằng được gọi Factory
            assertEquals(1, factory.openCalls());
            assertEquals(1, registry.size());

            // Đảm bảo 32 anh em đều xài chung 1 object
            ManagedResource expected = results.peek();
            assertTrue(results.stream().allMatch(result -> result == expected));
        } finally {
            factory.allowOpenToFinish.countDown();
            executor.shutdownNow();
            assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
        }
    }

    private static final class BlockingFactory
            implements ManagedResourceFactory {

        private final AtomicInteger calls = new AtomicInteger();
        private final CountDownLatch openEntered = new CountDownLatch(1);
        private final CountDownLatch allowOpenToFinish = new CountDownLatch(1);

        @Override
        public ManagedResource open(String resourceKey) {
            int call = calls.incrementAndGet();
            openEntered.countDown();
            await(allowOpenToFinish);
            return new TestResource(resourceKey + "-" + call);
        }

        int openCalls() {
            return calls.get();
        }
    }

    private record TestResource(String id) implements ManagedResource {
        @Override
        public void close() {
        }
    }

    private static void await(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("Mỏ neo đồng bộ sụp đổ");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Phát sinh gián đoạn môi trường", exception);
        }
    }
}
```

## 4. Thí Nghiệm 3: Thẩm Định Năng Lực Hồi Phục Khối FutureTask

Mình giả vờ test vụ nhổ mỏ neo: Lần 1 gọi lỗi cố ý, lần 2 gọi thì phải tạo thành công. Chứng tỏ mỏ neo lỗi ở lần 1 đã bị nhổ bỏ.

```java
@Test
void removesFailedPlaceholderBeforeNextAttempt() {
    AtomicInteger attempts = new AtomicInteger();

    ManagedResourceFactory flakyFactory = resourceKey -> {
        if (attempts.incrementAndGet() == 1) {
            throw new IllegalStateException("Cố tình báo lỗi Cấp phát lần đầu");
        }
        return new TestResource(resourceKey + "-ready");
    };

    var registry = new FutureManagedResourceRegistry(flakyFactory);

    // Lần đầu quăng lỗi
    assertThrows(
            ResourceRegistrationException.class,
            () -> registry.register("tenant-a")
    );

    // Lần sau chạy lại ngon lành, mỏ neo đã bị xoá
    ManagedResource recovered = registry.register("tenant-a");

    assertEquals("tenant-a-ready", recovered.id());
    assertEquals(2, attempts.get());
}
```

Trong thực tế, bạn cần làm khắt khe hơn: Quản lý xem đám luồng đang đứng đợi có cùng nhận được cái lỗi này không, và nhớ chỉnh Policy để tự động thử lại kiểu thời gian tăng dần (exponential backoff).

## 5. Kiểm Định Các Phương Án Kiến Trúc Khác

### Cơ Chế `putIfAbsent` Cùng Cleanup
Bạn viết cái Factory có thêm bộ đếm (tạo bao nhiêu lần, đóng bao nhiêu lần). Nếu test ngon nó sẽ ra vầy:
```text
open count   = 2
close count  = 1
registry size = 1
Mọi người đều dùng 1 object giống hệt nhau
```
Cảnh báo: Nếu bạn soi thấy `close count == 0` là hiểu luôn rồi đấy, rác vẫn đang đầy đầy ngoài kia kìa!

### Khối Độc Quyền `synchronized`
Muốn test cái này, bạn gọi 2 Key hoàn toàn khác nhau cùng lúc. Bạn sẽ thấy thằng đằng sau phải đợi thằng đằng trước chạy xong mới được chạy. Nghĩa là hiệu suất rất tệ, nhưng chắc chắn là đúng (không có chuyện tạo lố tài nguyên).

## 6. Giám Sát Môi Trường Khai Thác (Production Metrics)

Lên server thực tế thì nhớ đo lường mấy thứ này để còn biết mà đỡ:

- Xem Factory nó được gọi bao nhiêu lần cho cùng 1 key.
- Có bao nhiêu tài nguyên thực tế đang sống so với số lượng báo cáo trong Map.
- Báo động ngay lập tức nếu hàm Cleanup bị lỗi.
- Người dùng (caller) phải đứng chờ bao lâu cho cùng 1 Key.
- Đếm số lần Factory lỗi khiến luồng sau phải Retry lại từ đầu.
- Trực tiếp soi xem Thread/Socket/File descriptor có bị nhảy múa bất thường sau mỗi lần tải cao hay không.

Khi ghi log, nhớ ghi luôn cái `resourceKey`, cái ID của tài nguyên và kết quả. Tuyệt đối không lưu log mấy cái nhạy cảm như Mật khẩu hay Chuỗi khoá bảo mật của Khách hàng nhé!

## 7. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [x] Test ra được cái lỗi 2 thằng gọi Factory bằng rào cản luồng (không dùng trò ngủ `sleep` gian lận).
- [x] Chạy cách giải pháp đúng thì thấy Factory chỉ gọi đúng 1 lần, các anh em cầm đúng 1 ID.
- [x] Mấy cục đợi như latch/barrier đều có gắn ngòi nổ thời gian (Timeout).
- [x] Tập trung rình rập vụ rò rỉ rác (tài nguyên lọt lưới), chả thèm quan tâm cái `map.size()`.
- [x] Test kỹ năng "Tự phục hồi" sau cú vấp ngã khi xài Placeholder.
- [x] Hất cẳng `Testcontainers` ra khỏi danh sách vì lỗi này xử lý nguyên tuý trong não của Java (Heap).
