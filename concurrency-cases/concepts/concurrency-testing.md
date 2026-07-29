# Kiểm thử đồng thời

## Mục tiêu

Kiểm thử đồng thời (`concurrency testing`) không chỉ chứng minh rằng nhiều luồng
đã chạy. Test phải tạo được một thứ tự thực thi xen kẽ có ý nghĩa và kiểm tra
quy tắc nghiệp vụ sau khi mọi actor hoàn tất.

Có hai nhóm thuộc tính cần phân biệt:

- **tính an toàn** (`safety`): điều không bao giờ được phép xảy ra, ví dụ ID bị
  trùng, số dư âm hoặc hai booking cùng giữ một seat;
- **khả năng tiến triển** (`liveness/progress`): hệ thống cuối cùng phải tiếp tục
  xử lý, không bị deadlock, starvation hoặc retry vô hạn.

## Thuật ngữ chính

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| `interleaving` | Thứ tự các bước của nhiều luồng đan xen với nhau |
| `deterministic test` | Test chủ động ép lỗi xảy ra theo một thứ tự xác định |
| `stress test` | Test tạo nhiều tác vụ đồng thời để tăng khả năng gặp lỗi |
| `safety` | Điều kiện sai không được phép xuất hiện |
| `liveness` | Khả năng hệ thống tiếp tục tiến triển |
| `barrier` | Điểm hẹn buộc nhiều luồng chờ nhau |
| `test invariant` | Quy tắc cuối cùng mà test phải kiểm tra |

## Điều phối luồng thay vì đoán thời điểm

Không dùng `Thread.sleep(...)` làm cơ chế đồng bộ chính. Thời gian chạy thay đổi
theo CPU, CI runner và scheduler, vì vậy sleep vừa làm test chậm vừa dễ tạo kết
quả thất thường.

Ưu tiên các công cụ sau:

- `CountDownLatch` để chờ actor sẵn sàng và phát tín hiệu bắt đầu chung;
- `CyclicBarrier` hoặc `Phaser` để buộc actor gặp nhau tại một bước cụ thể;
- `Future.get(timeout)` để phát hiện task không hoàn tất;
- `ExecutorService` để quản lý số worker có giới hạn;
- test hook nhỏ hoặc implementation dành riêng cho test để dừng chính xác giữa
  bước đọc và bước ghi.

Mẫu điều phối phổ biến:

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

Mọi thao tác chờ phải có timeout để test không treo vô hạn khi implementation bị
deadlock.

## Hai tầng kiểm thử

### Tái hiện lỗi có kiểm soát

Đặt điểm điều phối đúng tại cửa sổ xảy ra tranh chấp:

```text
T1 đọc trạng thái dùng chung
T2 đọc cùng trạng thái
cho phép cả hai tiếp tục ghi
T1 ghi kết quả
T2 ghi kết quả
```

Cách này giải thích nguyên nhân gốc rõ ràng và nên là regression test chính.
Barrier chỉ thuộc test; production code không nên chứa cơ chế chờ phục vụ kiểm
thử.

### Kiểm thử tải đồng thời

Cho nhiều actor bắt đầu cùng lúc và lặp operation nhiều lần. Cách này hữu ích để
phát hiện lỗi trên implementation thật, nhưng một lần test pass không chứng minh
rằng hệ thống không còn race condition.

Một stress test tốt cần:

- kiểm tra quy tắc cuối cùng;
- ghi lại seed hoặc configuration nếu có randomness;
- không tăng số vòng lặp vô hạn để che giấu test thất thường;
- chạy cùng deterministic test, không thay thế nó.

> **Nói ngắn gọn:** deterministic test giải thích lỗi; stress test kiểm tra xem
> lỗi có xuất hiện trong implementation gần production hay không.

## Kiểm tra quy tắc nghiệp vụ

Không dừng ở `assertThrows`. Ví dụ với việc tạo ID:

```java
assertEquals(requestCount, results.size());
assertEquals(
        requestCount,
        results.stream().map(Result::id).distinct().count()
);
assertTrue(results.stream().allMatch(r -> r.customerId().equals(r.inputId())));
```

Nếu một actor thua do conflict, test cần kiểm tra cả:

- số operation thành công;
- số operation bị từ chối hoặc phải thử lại;
- trạng thái cuối cùng;
- không có side effect dở dang;
- lỗi kỹ thuật được chuyển thành domain outcome phù hợp.

## Khi cần dependency thật

Test chỉ liên quan đến Java heap có thể chạy bằng JUnit. Các hành vi sau phải ưu
tiên PostgreSQL thật qua Testcontainers:

- MVCC và isolation level;
- row lock hoặc table lock;
- `FOR UPDATE`, `SKIP LOCKED`;
- deadlock detection;
- conflict khi Hibernate flush entity có `@Version`;
- transaction bị hủy ở `SERIALIZABLE` và cách retry.

Không dùng H2 để suy luận hành vi đồng thời của PostgreSQL.

Với Kafka hoặc Redis, integration test phải nói rõ đặc tính đang kiểm tra:
redelivery, partition ordering, Lua atomicity, TTL, lease expiry hay recovery.

## Dọn dẹp tài nguyên và thu thập chẩn đoán

- Đóng executor trong `finally`.
- Khôi phục interrupt flag khi bắt `InterruptedException`.
- Đặt tên thread khi thread dump là bằng chứng quan trọng.
- Khi timeout, thu thập thread dump, lock wait hoặc database activity thay vì
  chỉ tăng timeout.
- Phân biệt lỗi infrastructure với lỗi vi phạm quy tắc nghiệp vụ.

## Liên kết

- Nền tảng JMM:
  [Java Memory Model và tính nguyên tử](java-memory-model-and-atomicity.md)
- Case áp dụng đầu tiên:
  [JVM-001](../jvm/spring-singleton-mutable-state/experiments.md)
- Atomic-claim database experiments:
  [DB-006](../postgresql/unique-constraint-concurrency/experiments.md)
- Pessimistic row-lock experiments: [LOCK-003](../locking/pessimistic-write-for-update/experiments.md)
- Conditional atomic-update experiments: [LOCK-004](../locking/conditional-atomic-update/experiments.md)
- Các case database trong [catalog](../CONCURRENCY_CASE_LIBRARY.md) sẽ mở rộng
- Work-claiming database experiments: [DB-010](../postgresql/skip-locked-work-queue/experiments.md)
  pattern này bằng PostgreSQL Testcontainers.
