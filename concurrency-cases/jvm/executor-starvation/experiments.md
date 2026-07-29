# Deterministic experiments và production verification

## Experiment 1: tái hiện self-starvation

Hai parent dùng latch để chắc chắn cùng chiếm worker trước khi submit child:

```java
@Test
void parentsStarveChildrenOnTheSamePool() throws Exception {
    ThreadPoolExecutor pool = new ThreadPoolExecutor(
            2, 2, 0, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(10)
    );
    CountDownLatch bothParentsRunning = new CountDownLatch(2);

    Callable<String> parent = () -> {
        bothParentsRunning.countDown();
        if (!bothParentsRunning.await(5, TimeUnit.SECONDS)) {
            throw new IllegalStateException("parents did not overlap");
        }
        Future<String> child = pool.submit(() -> "child-done");
        return child.get(30, TimeUnit.SECONDS);
    };

    Future<String> first = pool.submit(parent);
    Future<String> second = pool.submit(parent);
    try {
        assertThrows(TimeoutException.class,
                () -> first.get(300, TimeUnit.MILLISECONDS));
        assertEquals(2, pool.getActiveCount());
        assertEquals(2, pool.getQueue().size());
        assertEquals(0, pool.getCompletedTaskCount());
    } finally {
        first.cancel(true);
        second.cancel(true);
        pool.shutdownNow();
        assertTrue(pool.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Timeout là watchdog/assertion, còn latch tạo topology deterministic. Test cleanup
parent; interruption làm `child.get()` thoát.

> **Nói ngắn gọn:** assertion kết hợp “worker đều active, child đều queued,
>completed bằng 0” mô tả starvation rõ hơn một timeout đơn lẻ.

## Experiment 2: direct call giữ progress

```java
@Test
void directDependencyCallCompletesAtPoolCapacity() throws Exception {
    ExecutorService pool = Executors.newFixedThreadPool(2);
    CountDownLatch start = new CountDownLatch(1);
    Callable<String> task = () -> {
        if (!start.await(5, TimeUnit.SECONDS)) throw new IllegalStateException();
        return loadPriceDirectly();
    };
    Future<String> first = pool.submit(task);
    Future<String> second = pool.submit(task);
    start.countDown();
    try {
        assertEquals("price", first.get(5, TimeUnit.SECONDS));
        assertEquals("price", second.get(5, TimeUnit.SECONDS));
    } finally {
        pool.shutdownNow();
        assertTrue(pool.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Fake dependency phải bounded/deterministic; không dùng sleep.

## Experiment 3: rejection và cancellation

Với pool 1 + queue 1, giữ worker bằng latch, enqueue một task rồi submit task thứ
ba. Assert `RejectedExecutionException` ngay tại admission boundary. Sau đó
release latch và xác nhận hai accepted tasks hoàn tất. Test timeout path phải
assert child future nhận `cancel(true)` và worker interrupt status được xử lý.

## Load/stress checks

Tăng request rate qua admission capacity và assert: queue không vượt bound,
rejection xuất hiện trước memory growth, completed count tiếp tục tăng, p99 queue
age nằm trong deadline, retry không quay ngay vào cùng node. Stress không thay
thế deterministic nested-blocking test.

## Production verification

- active/max worker, queue size/capacity và oldest task age;
- submitted/started/completed/rejected/cancelled delta;
- queue wait, execution, dependency latency và end-to-end deadline;
- thread dump worker stack ở `Future.get`/remote I/O;
- downstream connection/quota utilization;
- graceful shutdown elapsed và unfinished task count.

## Checklist

- [ ] Latch chứng minh cả parent giữ worker trước child submit.
- [ ] Mọi await/get có timeout.
- [ ] Không dùng `Thread.sleep`.
- [ ] Starvation assertion kiểm tra active, queued và completed.
- [ ] Cleanup cancel task và await executor termination.
- [ ] Fixed test chứng minh progress tại cùng pool size.
- [ ] Rejection policy được test tại overload boundary.
- [ ] Production metric phân biệt queue wait và execution time.
