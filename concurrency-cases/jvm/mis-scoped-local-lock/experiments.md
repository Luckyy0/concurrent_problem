# Kiểm thử đồng thời và xác minh trong môi trường thực tế

## Chiến lược kiểm thử

Case phải trả lời riêng bốn câu hỏi:

1. hai lock object khác nhau có cho hai actor cùng vào không;
2. hai logical key bằng nhau nhưng khác reference có dùng cùng monitor không;
3. một stable striped lock có bảo vệ cùng key trong một service instance không;
4. hai service instance có phá giả định local lock khi dùng chung store không.

Fake store dùng latch ngay sau `exists` để ép hai actor cùng quan sát “chưa có”.
Mọi latch/future có timeout; không dùng `Thread.sleep`. Hướng dẫn nền nằm tại
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Experiment 1: lock mới mỗi call không serialize

```java
package com.example.settlement;

import org.junit.jupiter.api.Test;

import java.nio.charset.StandardCharsets;
import java.time.Duration;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.assertEquals;

class BrokenSettlementArtifactServiceTest {

    @Test
    void perCallLocksAllowDuplicateGeneration() throws Exception {
        BarrierStore store = new BarrierStore(2);
        AtomicInteger renders = new AtomicInteger();
        SettlementRenderer renderer = key -> {
            renders.incrementAndGet();
            return key.getBytes(StandardCharsets.UTF_8);
        };
        BrokenSettlementArtifactService service =
                new BrokenSettlementArtifactService(store, renderer);

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            Future<ArtifactResult> first = executor.submit(() ->
                    service.generate("settlement/day-1.csv", Duration.ofSeconds(5)));
            Future<ArtifactResult> second = executor.submit(() ->
                    service.generate("settlement/day-1.csv", Duration.ofSeconds(5)));

            first.get(5, TimeUnit.SECONDS);
            second.get(5, TimeUnit.SECONDS);
        }

        assertEquals(2, renders.get());
        assertEquals(2, store.putCount());
    }

    private static final class BarrierStore implements ArtifactStore {
        private final CountDownLatch allExistsCalls;
        private final AtomicInteger puts = new AtomicInteger();

        private BarrierStore(int expectedCallers) {
            this.allExistsCalls = new CountDownLatch(expectedCallers);
        }

        @Override
        public boolean exists(String artifactKey) {
            allExistsCalls.countDown();
            awaitOrFail(allExistsCalls);
            return false;
        }

        @Override
        public void put(String artifactKey, byte[] content) {
            puts.incrementAndGet();
        }

        int putCount() {
            return puts.get();
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

Barrier chỉ về 0 khi cả hai actor đã chạy `exists`. Nếu hai call thật sự dùng
cùng lock, test sẽ timeout ở barrier; với lock-per-call, cả hai đi qua và tạo hai
write một cách deterministic.

> **Nói ngắn gọn:** test không tìm lock object bằng reflection; nó kiểm tra hành
> vi mà lock phải bảo vệ—chỉ một generation workflow cho cùng key.

## Experiment 2: equal key không đồng nghĩa cùng monitor

Test harness nhỏ cho wrong-monitor variant:

```java
final class KeyMonitorGenerator {
    private final ArtifactStore store;
    private final SettlementRenderer renderer;

    KeyMonitorGenerator(ArtifactStore store, SettlementRenderer renderer) {
        this.store = store;
        this.renderer = renderer;
    }

    ArtifactResult generate(String key) {
        synchronized (key) {
            if (store.exists(key)) {
                return ArtifactResult.alreadyExists(key);
            }
            store.put(key, renderer.render(key));
            return ArtifactResult.created(key);
        }
    }
}
```

```java
@Test
void equalStringsWithDifferentIdentityUseDifferentMonitors() throws Exception {
    BarrierStore store = new BarrierStore(2);
    AtomicInteger renders = new AtomicInteger();
    KeyMonitorGenerator generator = new KeyMonitorGenerator(
            store,
            key -> {
                renders.incrementAndGet();
                return new byte[] {1};
            }
    );

    String firstKey = new String("settlement/day-1.csv");
    String secondKey = new String("settlement/day-1.csv");
    assertEquals(firstKey, secondKey);
    assertNotSame(firstKey, secondKey);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<?> first = executor.submit(() -> generator.generate(firstKey));
        Future<?> second = executor.submit(() -> generator.generate(secondKey));
        first.get(5, TimeUnit.SECONDS);
        second.get(5, TimeUnit.SECONDS);
    }

    assertEquals(2, renders.get());
    assertEquals(2, store.putCount());
}
```

Snippet dùng lại `BarrierStore`/imports từ Experiment 1 và static import
`assertNotSame`.

## Experiment 3: striped lock bảo vệ cùng key trong một instance

Renderer đầu tiên bị dừng để request thứ hai chắc chắn overlap. Request thứ hai
phải chờ stripe; sau khi request đầu publish, nó quan sát artifact đã tồn tại.

```java
@Test
void stableStripeAllowsOnlyOneRenderForTheSameKey() throws Exception {
    InMemoryStore store = new InMemoryStore();
    CountDownLatch firstRenderEntered = new CountDownLatch(1);
    CountDownLatch allowFirstRender = new CountDownLatch(1);
    CountDownLatch secondTaskStarted = new CountDownLatch(1);
    AtomicInteger renders = new AtomicInteger();

    SettlementRenderer renderer = key -> {
        if (renders.incrementAndGet() == 1) {
            firstRenderEntered.countDown();
            awaitOrFail(allowFirstRender);
        }
        return key.getBytes(StandardCharsets.UTF_8);
    };
    SettlementArtifactService service =
            new SettlementArtifactService(store, renderer);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<ArtifactResult> first = executor.submit(() ->
                service.generate("settlement/day-1.csv", Duration.ofSeconds(5)));
        assertTrue(firstRenderEntered.await(5, TimeUnit.SECONDS));

        Future<ArtifactResult> second = executor.submit(() -> {
            secondTaskStarted.countDown();
            return service.generate(
                    "settlement/day-1.csv",
                    Duration.ofSeconds(5)
            );
        });
        assertTrue(secondTaskStarted.await(5, TimeUnit.SECONDS));
        allowFirstRender.countDown();

        assertEquals(ArtifactResult.Status.CREATED,
                first.get(5, TimeUnit.SECONDS).status());
        assertEquals(ArtifactResult.Status.ALREADY_EXISTS,
                second.get(5, TimeUnit.SECONDS).status());
    }

    assertEquals(1, renders.get());
    assertEquals(1, store.putCount());
}
```

Fake store không cần barrier:

```java
final class InMemoryStore implements ArtifactStore {
    private final ConcurrentMap<String, byte[]> data = new ConcurrentHashMap<>();
    private final AtomicInteger puts = new AtomicInteger();

    @Override
    public boolean exists(String key) {
        return data.containsKey(key);
    }

    @Override
    public void put(String key, byte[] content) {
        puts.incrementAndGet();
        data.put(key, content);
    }

    int putCount() {
        return puts.get();
    }
}
```

Test assert business outcome, không chỉ `lock.isLocked()`. Lock implementation có
thể thay đổi mà invariant test vẫn giữ giá trị.

## Experiment 4: hai service instance chứng minh local-lock limitation

Mô phỏng hai node bằng hai service object, mỗi object có `StripedKeyLocks` riêng,
nhưng cùng dùng một barrier store:

```java
@Test
void twoInstancesStillGenerateTheSameArtifactTwice() throws Exception {
    BarrierStore sharedStore = new BarrierStore(2);
    AtomicInteger renders = new AtomicInteger();
    SettlementRenderer renderer = key -> {
        renders.incrementAndGet();
        return new byte[] {1};
    };

    SettlementArtifactService nodeA =
            new SettlementArtifactService(sharedStore, renderer);
    SettlementArtifactService nodeB =
            new SettlementArtifactService(sharedStore, renderer);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<?> first = executor.submit(() -> nodeA.generate(
                "settlement/day-1.csv",
                Duration.ofSeconds(5)
        ));
        Future<?> second = executor.submit(() -> nodeB.generate(
                "settlement/day-1.csv",
                Duration.ofSeconds(5)
        ));

        first.get(5, TimeUnit.SECONDS);
        second.get(5, TimeUnit.SECONDS);
    }

    assertEquals(2, renders.get());
    assertEquals(2, sharedStore.putCount());
}
```

Test này bắt buộc khi team định dùng local keyed lock trước shared resource. Một
service instance test pass không phải proof cho deployment nhiều replica.

## Experiment 5: conditional create chọn đúng một winner

Fake authoritative store dùng atomic `putIfAbsent`:

```java
final class ConditionalInMemoryStore {
    private final ConcurrentMap<String, byte[]> data = new ConcurrentHashMap<>();

    PutIfAbsentResult putIfAbsent(String key, byte[] content) {
        return data.putIfAbsent(key, content) == null
                ? PutIfAbsentResult.CREATED
                : PutIfAbsentResult.ALREADY_EXISTS;
    }

    int size() {
        return data.size();
    }
}
```

```java
@Test
void conditionalCreateAllowsExactlyOneWinnerAcrossInstances() throws Exception {
    ConditionalInMemoryStore store = new ConditionalInMemoryStore();
    CountDownLatch start = new CountDownLatch(1);

    Callable<PutIfAbsentResult> attempt = () -> {
        awaitOrFail(start);
        return store.putIfAbsent("settlement/day-1.csv", new byte[] {1});
    };

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<PutIfAbsentResult> first = executor.submit(attempt);
        Future<PutIfAbsentResult> second = executor.submit(attempt);
        start.countDown();

        List<PutIfAbsentResult> outcomes = List.of(
                first.get(5, TimeUnit.SECONDS),
                second.get(5, TimeUnit.SECONDS)
        );

        assertEquals(1, outcomes.stream()
                .filter(result -> result == PutIfAbsentResult.CREATED)
                .count());
        assertEquals(1, outcomes.stream()
                .filter(result -> result == PutIfAbsentResult.ALREADY_EXISTS)
                .count());
    }

    assertEquals(1, store.size());
}
```

Snippet cần imports `Callable`, `List`, concurrent map và static assertions đã
xuất hiện ở các experiment trước. Test fake chỉ mô hình atomic store primitive;
integration test production phải chạy với API conditional-create thật của store.

## Kiểm tra timeout, interruption và failure

Bổ sung tests:

- giữ stripe ở T1; T2 dùng timeout ngắn và nhận `ArtifactBusyException`;
- interrupt T2 đang `tryLock`, xác nhận interrupt status được restore;
- renderer ném exception, request sau vẫn acquire được stripe;
- store `put` ném timeout với outcome unknown, workflow chuyển sang reconcile;
- hai key map cùng stripe có thể serialize nhưng không vi phạm correctness;
- service không unlock từ callback thread khác.

Dùng future timeout như outer watchdog cho mọi test lock. Nếu test fail, thu thread
dump để phân biệt assertion failure với deadlock.

## Xác minh trong môi trường thực tế

Theo dõi theo node và key hash/stripe:

- lock wait duration, acquisition timeout và current in-flight;
- render count so với created artifact count;
- conditional conflict/`ALREADY_EXISTS` count;
- duplicate publish event;
- store timeout có outcome không rõ và reconciliation result;
- stripe hot spot, queue length (diagnostic) và render duration;
- node ID của winner/loser.

Không dùng `ReentrantLock.getQueueLength()` như correctness signal; đây chỉ là
ước lượng phục vụ chẩn đoán.

## Checklist chất lượng của case

- [ ] Lock-per-call race được ép bằng barrier.
- [ ] Equal-but-not-identical key test dùng hai object reference khác nhau.
- [ ] Local fixed test assert một render và một write.
- [ ] Hai service instance test chứng minh local lock limitation.
- [ ] Conditional-create test assert đúng một winner.
- [ ] Không dùng `Thread.sleep`.
- [ ] Mọi latch/future có timeout.
- [ ] Interrupt status và unlock-on-exception được kiểm tra.
- [ ] Executor được đóng sau test.
- [ ] Production verification phân biệt node-local và authoritative outcome.
