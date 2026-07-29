# Kiểm thử đồng thời và xác minh trong môi trường thực tế

## Chiến lược kiểm thử

Case cần tách ba câu hỏi:

1. broken implementation có để lộ state đang refresh dở không;
2. immutable snapshot có bảo vệ tính đầy đủ và generation tăng đơn điệu không;
3. collection được chọn có đúng iteration semantics mà caller mong đợi không.

Dùng latch để đặt thread đúng vào cửa sổ tranh chấp. Không dùng `Thread.sleep`
làm điều kiện correctness vì scheduler và tốc độ máy có thể thay đổi. Mọi
`await()` và `Future.get()` đều có timeout để test không treo vô hạn.

Chiến lược chung được mô tả tại
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Experiment 1: tái hiện partial snapshot một cách deterministic

Test harness dưới đây giữ writer lại ngay sau entry đầu tiên của generation mới.
Reader chạy khi writer đang dừng, vì vậy lỗi không phụ thuộc may mắn của
scheduler.

```java
package com.example.routing;

import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class BrokenRoutingRegistryTest {

    @Test
    void readerCanObserveOnlyPartOfTheNewGeneration() throws Exception {
        HookedBrokenRegistry registry = new HookedBrokenRegistry();
        registry.replaceForSetup(routes(41));

        CountDownLatch firstEntryPublished = new CountDownLatch(1);
        CountDownLatch allowRefreshToFinish = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
            Future<?> refresh = executor.submit(() -> registry.refresh(
                    routes(42),
                    () -> {
                        firstEntryPublished.countDown();
                        awaitOrFail(allowRefreshToFinish, Duration.ofSeconds(5));
                    }
            ));

            assertTrue(firstEntryPublished.await(5, TimeUnit.SECONDS));

            Map<String, PaymentRoute> observed = registry.copyForDiagnostic();
            assertEquals(1, observed.size());
            assertTrue(observed.values().stream()
                    .allMatch(route -> route.generation() == 42));

            allowRefreshToFinish.countDown();
            refresh.get(5, TimeUnit.SECONDS);

            assertEquals(2, registry.copyForDiagnostic().size());
        }
    }

    private static Map<String, PaymentRoute> routes(long generation) {
        return Map.of(
                "merchant-a", new PaymentRoute("provider-a", true, generation),
                "merchant-b", new PaymentRoute("provider-b", true, generation)
        );
    }

    private static void awaitOrFail(CountDownLatch latch, Duration timeout) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new IllegalStateException("latch timed out");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("interrupted while waiting", exception);
        }
    }

    private static final class HookedBrokenRegistry {
        private final Map<String, PaymentRoute> routes = new HashMap<>();

        void replaceForSetup(Map<String, PaymentRoute> loaded) {
            routes.clear();
            routes.putAll(loaded);
        }

        void refresh(Map<String, PaymentRoute> loaded, Runnable afterFirstPut) {
            routes.clear();
            Iterator<Map.Entry<String, PaymentRoute>> iterator =
                    loaded.entrySet().iterator();

            Map.Entry<String, PaymentRoute> first = iterator.next();
            routes.put(first.getKey(), first.getValue());
            afterFirstPut.run();

            iterator.forEachRemaining(entry ->
                    routes.put(entry.getKey(), entry.getValue()));
        }

        Map<String, PaymentRoute> copyForDiagnostic() {
            return new HashMap<>(routes);
        }
    }
}
```

Test không cố ép `ConcurrentModificationException`. Exception của fail-fast
iterator chỉ là best-effort; invariant quan trọng hơn là reader đã thấy một map
chỉ có một entry dù mọi generation hợp lệ đều có hai.

> **Nói ngắn gọn:** test khóa đúng “khung hình” state đang dở, thay vì chạy thật
> nhiều lần rồi hy vọng bắt gặp lỗi.

## Experiment 2: xác minh atomic snapshot và chặn stale writer

Test sau cho hai refresh writer cạnh tranh. Writer giữ generation 42 bị dừng;
generation 43 được publish trước. Khi writer cũ tiếp tục, compare-and-set logic
phải từ chối nó.

```java
package com.example.routing;

import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.Map;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

class PaymentRoutingRegistryTest {

    @Test
    void snapshotIsCompleteAndOlderWriterCannotOverwriteNewerGeneration()
            throws Exception {
        PaymentRoutingRegistry registry = registryWithoutRemoteCalls();
        assertTrue(registry.publishIfNewer(snapshot(41)));

        CountDownLatch olderWriterReady = new CountDownLatch(1);
        CountDownLatch releaseOlderWriter = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
            Future<Boolean> olderResult = executor.submit(() -> {
                RoutingSnapshot older = snapshot(42);
                olderWriterReady.countDown();
                awaitOrFail(releaseOlderWriter, Duration.ofSeconds(5));
                return registry.publishIfNewer(older);
            });

            assertTrue(olderWriterReady.await(5, TimeUnit.SECONDS));
            assertTrue(registry.publishIfNewer(snapshot(43)));

            RoutingSnapshot whileOlderWriterIsBlocked = registry.snapshot();
            assertCompleteGeneration(whileOlderWriterIsBlocked, 43);

            releaseOlderWriter.countDown();
            assertFalse(olderResult.get(5, TimeUnit.SECONDS));
            assertCompleteGeneration(registry.snapshot(), 43);
        }
    }

    private static void assertCompleteGeneration(
            RoutingSnapshot snapshot,
            long expectedGeneration
    ) {
        assertEquals(expectedGeneration, snapshot.generation());
        assertEquals(2, snapshot.routes().size());
        assertTrue(snapshot.routes().values().stream()
                .allMatch(route -> route.generation() == expectedGeneration));
    }

    private static RoutingSnapshot snapshot(long generation) {
        return new RoutingSnapshot(generation, Map.of(
                "merchant-a", new PaymentRoute("provider-a", true, generation),
                "merchant-b", new PaymentRoute("provider-b", true, generation)
        ));
    }

    private static PaymentRoutingRegistry registryWithoutRemoteCalls() {
        RoutingConfigClient unusedClient = () -> {
            throw new AssertionError("remote client must not be called");
        };
        return new PaymentRoutingRegistry(unusedClient);
    }

    private static void awaitOrFail(CountDownLatch latch, Duration timeout) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new IllegalStateException("latch timed out");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("interrupted while waiting", exception);
        }
    }
}
```

Để lambda `RoutingConfigClient` trong test khớp code solution, interface có thể
được khai báo như sau:

```java
public interface RoutingConfigClient {
    RoutingSnapshot loadSnapshot();
}
```

Assertion kiểm tra business invariant: snapshot có đủ hai route và mọi value có
cùng generation. Chỉ kiểm tra exception hoặc kích thước map là chưa đủ.

## Experiment 3: kiểm tra refresh failure giữ nguyên last-known-good snapshot

Không cần concurrency để xác minh nhánh failure, nhưng đây là regression test
quan trọng cho thiết kế build-before-publish. Fake client trả snapshot hợp lệ ở
lần đầu và snapshot rỗng ở lần sau:

```java
@Test
void invalidRefreshDoesNotDestroyCurrentSnapshot() {
    java.util.concurrent.atomic.AtomicInteger calls =
            new java.util.concurrent.atomic.AtomicInteger();

    RoutingConfigClient client = () -> {
        if (calls.getAndIncrement() == 0) {
            return snapshot(41);
        }
        return new RoutingSnapshot(42, Map.of());
    };
    PaymentRoutingRegistry registry = new PaymentRoutingRegistry(client);

    registry.refresh();
    assertCompleteGeneration(registry.snapshot(), 41);

    assertThrows(IllegalArgumentException.class, registry::refresh);

    assertCompleteGeneration(registry.snapshot(), 41);
}
```

Snippet dùng lại `snapshot`, `assertCompleteGeneration` và static assertion từ
Experiment 2. Assertion cuối xác nhận refresh thất bại không phá hủy generation
đang phục vụ request.

## Experiment 4: unsafe publication là stress test, không phải unit test ổn định

Không nên viết JUnit yêu cầu “reader phải thấy stale value”, vì scheduler không
tạo được bằng chứng happens-before và test có thể pass trên một CPU nhưng fail ở
CPU khác. Dùng [OpenJDK jcstress](https://openjdk.org/projects/code-tools/jcstress/)
để khảo sát các outcome được Java Memory Model cho phép:

```java
@JCStressTest
@Outcome(id = "0", expect = Expect.ACCEPTABLE_INTERESTING,
        desc = "Reader observed the old snapshot")
@Outcome(id = "1", expect = Expect.ACCEPTABLE,
        desc = "Reader observed the new snapshot")
@State
public class UnsafeSnapshotPublicationStress {

    private Map<String, PaymentRoute> routes = Map.of();

    @Actor
    public void writer() {
        routes = Map.of(
                "merchant-a",
                new PaymentRoute("provider-a", true, 1)
        );
    }

    @Actor
    public void reader(I_Result result) {
        result.r1 = routes.size();
    }
}
```

Outcome `0` không tự nó chứng minh structural corruption; nó cho thấy code không
có contract buộc reader quan sát write cạnh tranh. Sau khi đổi field thành
`volatile` hoặc `AtomicReference`, vẫn phải thiết kế outcome theo ordering mà ứng
dụng thực sự yêu cầu, không diễn giải scheduling order thành memory order.

## Kiểm tra semantics của ConcurrentHashMap

Nếu chọn `ConcurrentHashMap`, test không được assert iterator trả snapshot chính
xác. Contract hợp lệ nên kiểm tra:

- không có structural corruption hoặc `ConcurrentModificationException`;
- mỗi entry quan sát được là một key/value hợp lệ;
- operation trên một key có kết quả atomic theo API đã dùng;
- report toàn bảng được đánh dấu approximate nếu concurrent update được phép.

Nếu nghiệp vụ yêu cầu mọi entry cùng generation, test phải fail phương án
`ConcurrentHashMap` per-key và chuyển sang immutable snapshot hoặc locking.

## Xác minh trong môi trường thực tế

Theo dõi theo từng application instance:

- generation hiện tại và tuổi của snapshot;
- số entry và checksum của snapshot;
- refresh success, failure, timeout và stale publish bị từ chối;
- số request phải fallback vì không tìm thấy route;
- distribution generation giữa các node sau mỗi lần rollout cấu hình.

Log một event tại điểm publish gồm `oldGeneration`, `newGeneration`, `entryCount`
và duration load/validate. Không log toàn bộ routing data nhạy cảm.

## Checklist chất lượng của case

- [ ] Broken test điều phối đúng partial-update window bằng latch.
- [ ] Không dùng `Thread.sleep` để quyết định test pass/fail.
- [ ] Mọi latch và future đều có timeout.
- [ ] Regression test kiểm tra đầy đủ entry và cùng generation.
- [ ] Test có case nhiều writer và stale generation.
- [ ] Refresh failure giữ nguyên last-known-good snapshot.
- [ ] Iterator semantics khớp với lựa chọn collection.
- [ ] Executor được đóng sau test và interrupt status được khôi phục.
- [ ] Production dashboard hiển thị generation theo từng node.
