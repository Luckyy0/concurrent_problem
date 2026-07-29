# Kiểm thử đồng thời và xác minh trong môi trường thực tế

## Chiến lược kiểm thử

Case dùng ba lớp kiểm tra:

1. ép broken implementation tạo hai resource cho cùng key;
2. cho nhiều actor gọi `computeIfAbsent` cùng lúc và kiểm tra factory chỉ chạy
   một lần;
3. kiểm tra placeholder thất bại được loại bỏ để lần gọi sau có thể retry.

Không cần Testcontainers vì state chỉ tồn tại trong Java heap. Mọi latch, barrier
và future đều có timeout; test không dùng `Thread.sleep` để điều phối.

## Tái hiện lỗi có kiểm soát

Factory chờ đến khi cả hai thread cùng bước vào `open(...)`. Điều này chứng minh
cả hai đã quan sát key vắng mặt trước khi một thread kịp `put`:

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

            assertEquals(2, factory.openCalls());
            assertEquals(1, registry.size());
            assertNotSame(first, second);
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
            await(bothCallsEntered);
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
                throw new IllegalStateException("latch timed out");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("test interrupted", exception);
        }
    }
}
```

Kết quả `map.size() == 1` không chứng minh implementation đúng. Assertion quan
trọng là factory đã chạy hai lần và chỉ một trong hai object caller nhận được còn
được registry quản lý.

> **Nói ngắn gọn:** test không kiểm tra map có hỏng hay không; nó kiểm tra resource
> có bị tạo dư và thất lạc khỏi registry hay không.

## Kiểm thử hồi quy cho computeIfAbsent

Test cho 32 actor bắt đầu cùng thời điểm. Factory cố ý chờ để kéo dài cửa sổ
tranh chấp, nhưng chỉ một actor được phép chạy factory:

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
                            throw new IllegalStateException("start timed out");
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
            start.countDown();
            assertTrue(attemptingRegister.await(5, TimeUnit.SECONDS));
            assertTrue(factory.openEntered.await(5, TimeUnit.SECONDS));
            factory.allowOpenToFinish.countDown();
            assertTrue(done.await(10, TimeUnit.SECONDS));

            assertTrue(failures.isEmpty());
            assertEquals(actors, results.size());
            assertEquals(1, factory.openCalls());
            assertEquals(1, registry.size());

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
                throw new IllegalStateException("latch timed out");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("test interrupted", exception);
        }
    }
}
```

## Kiểm thử recovery của FutureTask placeholder

Factory thất bại ở lần đầu và thành công ở lần sau. Registry phải remove future
đã thất bại để retry không nhận lại cùng exception mãi mãi:

```java
@Test
void removesFailedPlaceholderBeforeNextAttempt() {
    AtomicInteger attempts = new AtomicInteger();

    ManagedResourceFactory flakyFactory = resourceKey -> {
        if (attempts.incrementAndGet() == 1) {
            throw new IllegalStateException("temporary open failure");
        }
        return new TestResource(resourceKey + "-ready");
    };

    var registry = new FutureManagedResourceRegistry(flakyFactory);

    assertThrows(
            ResourceRegistrationException.class,
            () -> registry.register("tenant-a")
    );

    ManagedResource recovered = registry.register("tenant-a");

    assertEquals("tenant-a-ready", recovered.id());
    assertEquals(2, attempts.get());
}
```

Test production cần bổ sung case nhiều caller cùng nhận exception từ một future,
và case retry policy dừng sau giới hạn đã cấu hình.

## Kiểm tra các phương án khác

### putIfAbsent và cleanup

Dùng factory đếm số resource mở/đóng rồi assert:

```text
open count   = 2
close count  = 1
registry size = 1
mọi caller trả về resource đang nằm trong registry
```

Nếu `close count == 0`, implementation vẫn rò rỉ dù map chỉ có một entry.

### synchronized

Cho hai key khác nhau đăng ký đồng thời và đo/quan sát việc cả hai bị serialize.
Đây là kiểm tra trade-off về contention, không phải lỗi correctness.

## Xác minh trong môi trường thực tế

Theo dõi:

- số lần factory chạy theo `resourceKey`;
- số resource active so với registry size;
- số cleanup failure;
- thời gian chờ đăng ký cùng key;
- số mapping function exception và retry;
- thread/socket/file descriptor count sau traffic spike.

Log phải có `resourceKey`, resource ID và registration outcome, nhưng không ghi
credential hoặc tenant secret.

## Checklist chất lượng của case

- [x] Broken test ép hai factory call mà không dùng sleep.
- [x] Fixed test kiểm tra một factory call và cùng object identity.
- [x] Mọi thao tác chờ đều có timeout.
- [x] Test kiểm tra resource bị orphan, không chỉ `map.size()`.
- [x] Failure recovery của placeholder được xem xét.
- [x] Không dùng Testcontainers vì case không liên quan database semantics.
