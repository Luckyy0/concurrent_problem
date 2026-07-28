# Concurrency testing

## Mục tiêu

Concurrency test không chỉ chứng minh “nhiều thread đã chạy”. Test phải làm lộ
ra một interleaving có ý nghĩa và assert business invariant sau khi mọi actor
hoàn tất.

Hai nhóm thuộc tính cần phân biệt:

- **safety**: điều không được phép xảy ra, ví dụ duplicate ID, số dư âm hoặc hai
  booking cùng giữ một seat;
- **liveness/progress**: hệ thống cuối cùng có tiến triển hay không, ví dụ không
  deadlock, starvation hoặc retry vô hạn.

## Điều phối thay vì đoán timing

Không dùng `Thread.sleep(...)` làm cơ chế synchronization chính. Thời gian chạy
phụ thuộc CPU, CI runner và scheduler nên sleep vừa chậm vừa flaky.

Ưu tiên:

- `CountDownLatch` để chờ actor sẵn sàng và phát tín hiệu bắt đầu chung;
- `CyclicBarrier` hoặc `Phaser` để ép actor gặp nhau tại một bước;
- `Future.get(timeout)` để phát hiện task không hoàn tất;
- `ExecutorService` để quản lý worker có giới hạn;
- test hook nhỏ hoặc instrumented implementation trong test để dừng đúng giữa
  `read → write`.

Skeleton phổ biến:

```java
int actors = 32;
var ready = new CountDownLatch(actors);
var start = new CountDownLatch(1);
var done = new CountDownLatch(actors);
var executor = Executors.newFixedThreadPool(actors);

try {
    for (int i = 0; i < actors; i++) {
        executor.submit(() -> {
            ready.countDown();
            try {
                start.await();
                // operation under test
            } finally {
                done.countDown();
            }
            return null;
        });
    }

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();
    assertTrue(done.await(10, TimeUnit.SECONDS));
} finally {
    executor.shutdownNow();
}
```

Mọi wait phải có timeout để test không treo vô hạn khi implementation deadlock.

## Deterministic test và stress test

### Deterministic test

Chèn coordination point đúng tại cửa sổ race:

```text
T1 read shared state
T2 read shared state
release both writers
T1 write
T2 write
```

Loại test này giải thích root cause tốt và nên là regression test chính.
Coordination hook chỉ thuộc test; production code không nên mang barrier.

### Stress test

Cho nhiều actor bắt đầu cùng lúc và lặp operation nhiều lần. Stress test hữu ích
để phát hiện race trên implementation thật nhưng không chứng minh race vắng mặt
khi test pass. Kết quả phụ thuộc scheduler nên phải:

- assert invariant cuối cùng;
- ghi lại seed/configuration nếu có randomness;
- không tăng số vòng lặp vô hạn để “chữa” flaky test;
- kết hợp với deterministic test.

## Assert invariant

Không dừng ở `assertThrows`. Ví dụ với ID generation:

```java
assertEquals(requestCount, results.size());
assertEquals(
        requestCount,
        results.stream().map(Result::id).distinct().count()
);
assertTrue(results.stream().allMatch(r -> r.customerId().equals(r.inputId())));
```

Nếu có loser do conflict, assert cả:

- số operation thành công;
- số operation bị reject/retry;
- final state;
- không có side effect một phần;
- error được map đúng thành domain outcome.

## Khi cần dependency thật

Test JVM-local có thể chạy thuần JUnit. Các semantics sau phải ưu tiên
PostgreSQL thật qua Testcontainers:

- MVCC và isolation level;
- row lock/table lock;
- `FOR UPDATE`, `SKIP LOCKED`;
- deadlock detection;
- `@Version` flush conflict;
- `SERIALIZABLE` abort/retry.

Không dùng H2 để suy luận PostgreSQL concurrency behavior.

Với Kafka hoặc Redis, integration test phải nêu rõ property đang kiểm tra:
redelivery, partition ordering, atomic Lua, TTL/lease expiry hay recovery.

## Cleanup và diagnostics

- Đóng executor trong `finally`.
- Khôi phục interrupt flag khi bắt `InterruptedException`.
- Đặt tên thread khi thread dump là bằng chứng quan trọng.
- Khi timeout, thu thập thread dump, lock wait hoặc database activity thay vì
  chỉ tăng timeout.
- Tách lỗi infrastructure khỏi invariant failure.

## Liên kết

- Nền tảng JMM: [Java Memory Model và atomicity](java-memory-model-and-atomicity.md)
- Case áp dụng đầu tiên:
  [JVM-001](../jvm/spring-singleton-mutable-state/experiments.md)
- Các case database trong [catalog](../CONCURRENCY_CASE_LIBRARY.md) sẽ mở rộng
  pattern này bằng PostgreSQL Testcontainers.
