# Môi Trường Thực Nghiệm: Xác Minh Lỗ Hổng Check-Then-Act

## 1. Chiến Lược Kiểm Thử (Testing Strategy)

Mô hình kiểm định khai thác 3 lớp phòng ngự:

1. Thiết lập kịch bản đan xen luồng nhằm ép cấu trúc lỗi sinh ra dư thừa tài nguyên (2 Resource) cho chung 1 Key định danh.
2. Giả lập hiệu ứng bầy đàn (Mass actors) triệu gọi trực diện `computeIfAbsent` để thẩm định rằng cơ chế Factory chỉ bị kích hoạt 1 lần duy nhất.
3. Kịch bản đánh giá quy trình gỡ bỏ đối tượng Điền Trống (Placeholder) trong trường hợp hệ thống sụp đổ (Failed), mở đường cho tiến trình Thử lại (Retry).

Không vận hành `Testcontainers` do trạng thái cấu trúc này giới hạn nghiêm ngặt trong phạm vi Java Heap. Toàn bộ các mỏ neo điều phối (latch, barrier, future) bắt buộc cấp thiết lập ngắt thời gian (timeout); cấm tuyệt đối cơ chế `Thread.sleep` thay thế logic luồng.

## 2. Thí Nghiệm 1: Định Hình Cơ Chế Thất Thoát (Deterministic Lost Update)

Thiết kế mô phỏng (Factory Barrier) trì hoãn các tiến trình cho tới khi 2 luồng cùng giao cắt vào vùng `open(...)`. Bằng chứng đanh thép xác định 2 luồng đã cùng "Nhìn thấy" Key vắng mặt trước khi kịp thực thi `put`:

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

            // Xác thực: Hệ thống phải gánh chịu 2 lần tạo tài nguyên
            assertEquals(2, factory.openCalls());
            // Ảo ảnh: Map chỉ ghi nhận 1
            assertEquals(1, registry.size());
            // Cảnh báo: Các luồng trả về 2 kết quả hoàn toàn phân kỳ
            assertNotSame(first, second);
            // Một đối tượng bị loại trừ ra khỏi Registry vĩnh viễn
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
            await(bothCallsEntered); // Giam lỏng để chờ luồng 2
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

Quan sát thấy chỉ số `map.size() == 1` là một lời nói dối ngụy trang, không thể khẳng định code chạy đúng. Nòng cốt cần khẳng định: Factory đã kích hoạt trái phép 2 lần và chỉ một Caller nhận được tài nguyên còn chịu sự giám sát.

> **Nguyên tắc kỹ thuật:** Không kiểm định việc Map có bị biến dạng vật lý, mà phải kiểm định khâu cấp phát tài nguyên có rò rỉ và có bị xóa sổ danh tính quản trị khỏi Registry hay không.

## 3. Thí Nghiệm 2: Đánh Giá Phương Thức computeIfAbsent

Tạo đợt tấn công từ 32 luồng độc lập đồng bộ thời khắc kích hoạt (Start point). Rào cản Factory chủ đích kéo dãn cửa sổ rủi ro, tuy nhiên phải đảm bảo chỉ định duy nhất 1 Luồng thành công vượt ải:

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
            start.countDown(); // Phát động đồng thanh
            assertTrue(attemptingRegister.await(5, TimeUnit.SECONDS));
            assertTrue(factory.openEntered.await(5, TimeUnit.SECONDS));
            factory.allowOpenToFinish.countDown();
            assertTrue(done.await(10, TimeUnit.SECONDS));

            assertTrue(failures.isEmpty());
            assertEquals(actors, results.size());
            // Tuyệt đối chỉ 1 luồng lọt qua
            assertEquals(1, factory.openCalls());
            assertEquals(1, registry.size());

            // 100% Caller trả về duy nhất 1 Định danh tham chiếu
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

Kịch bản thiết lập: Tác vụ kiến tạo đổ vỡ ngay vòng 1 nhưng phục hồi ở vòng 2. Hệ thống bắt buộc phải giải trừ khối Placeholder độc hại trước khi đón luồng thứ 2.

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

    // Kích hoạt ngoại lệ Lần Đầu
    assertThrows(
            ResourceRegistrationException.class,
            () -> registry.register("tenant-a")
    );

    // Cơ chế gỡ chốt thành công - Phục hồi tiến trình
    ManagedResource recovered = registry.register("tenant-a");

    assertEquals("tenant-a-ready", recovered.id());
    assertEquals(2, attempts.get());
}
```

Môi trường Production yêu cầu hoàn thiện thêm cấu trúc đa Caller đồng loạt hứng chịu Exception từ một Future, kết hợp cấu hình Retry Policy giới hạn cấp số nhân thay vì thả nổi vòng lặp.

## 5. Kiểm Định Các Phương Án Kiến Trúc Khác

### Cơ Chế `putIfAbsent` Cùng Cleanup
Dùng khối Factory có khả năng đếm (open/close counter) và đánh giá:
```text
open count   = 2
close count  = 1
registry size = 1
Mọi Caller quy tụ vào đúng phiên bản hợp pháp của Registry
```
Cảnh báo: Nếu hàm đếm `close count == 0`, hệ thống đã rò rỉ bất kể Map chỉ bảo lưu 1 tài nguyên.

### Khối Độc Quyền `synchronized`
Thiết lập 2 Key độc lập để quan sát độ trễ bị Serialization hóa. Đánh giá này đo đếm mức độ suy giảm Thông lượng (Contention trade-off), không phân định tiêu chí Correctness.

## 6. Giám Sát Môi Trường Khai Thác (Production Metrics)

Các thông số cần đo lường liên tục:

- Biến đếm: Số vòng đời thực thi hàm Factory dựa trên chuẩn `resourceKey`.
- Độ lệch số lượng Active Resource thực tế so sánh với Registry Size.
- Số vụ Cleanup Failure (Hủy tài nguyên dư thất bại).
- Quỹ thời gian Caller nằm chờ cho các Request cùng Key.
- Khối lượng báo lỗi Mapping Function kích hoạt tiến trình Retry.
- Theo dõi Thread/Socket/File descriptor sau các đợt tăng tải đột biến (Traffic spike).

Khai báo Log bắt buộc chứa `resourceKey`, định danh tài nguyên (resource ID) và kết quả đăng ký; Chống chỉ định ghi log định dạng mã hóa Credential hoặc Tenant secret.

## 7. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [x] Broken test phơi bày lỗi 2 lệnh gọi Factory mà không dùng ngắt `sleep`.
- [x] Test Phương án Đúng phải truy vết chỉ một cuộc gọi Factory và đồng bộ Định danh duy nhất (Object Identity).
- [x] Mọi thao tác chờ buộc phải cấp phép Timeout.
- [x] Bộ thử nghiệm đánh giá sự cố Tài Nguyên Rò Rỉ (Orphan), không dựa vào kết quả `map.size()`.
- [x] Đánh giá đầy đủ năng lực Tự phục hồi hệ thống khi dùng khối Placeholder.
- [x] Loại trừ nền tảng `Testcontainers` vì cơ chế hoàn toàn xoay quanh hệ Java Semantics.
