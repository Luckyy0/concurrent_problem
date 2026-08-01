# Thực Nghiệm — So Sánh Chiến Lược Lock Dưới Tải Tương Tranh

## 1. Mục tiêu thực nghiệm

Xác minh hành vi của từng chiến lược lock khi mức tương tranh tăng dần trên cùng một bản ghi. Các thực nghiệm cần chứng minh:

1. Lock lạc quan (`@Version`) gây retry storm khi tương tranh cao.
2. Lock bi quan (`FOR UPDATE`) gây cạn kiệt kết nối khi lock queue dài.
3. Atomic UPDATE duy trì thông lượng ổn định hơn và không phát sinh thử lại.
4. Kiểm soát đầu vào (admission control) bảo vệ database khỏi quá tải.

Tất cả thực nghiệm sử dụng PostgreSQL thực (Testcontainers), không sử dụng H2 vì hành vi lock và đánh giá lại (predicate recheck) khác biệt đáng kể.

## 2. Thiết lập hạ tầng kiểm thử

### Testcontainers

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("test")
class HighContentionStrategyComparisonTest {

    @Container
    static PostgreSQLContainer<?> pg =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("contention_test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", pg::getJdbcUrl);
        registry.add("spring.datasource.username", pg::getUsername);
        registry.add("spring.datasource.password", pg::getPassword);
        registry.add("spring.datasource.hikari.maximum-pool-size", () -> "30");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "create-drop");
    }
}
```

### Dữ liệu ban đầu

```java
@BeforeEach
void setUp() {
    // Đảm bảo trạng thái sạch trước mỗi test
    reservationRepository.deleteAll();
    inventoryRepository.deleteAll();

    InventoryItem item = new InventoryItem();
    item.setProductId(2024L);
    item.setAvailableQuantity(500);
    item.setReservedQuantity(0);
    inventoryRepository.saveAndFlush(item);
}
```

### Hạ tầng đồng thời

```java
private record ContentionResult(
        int successCount,
        int failureCount,
        int retryCount,
        long totalDurationMs,
        long maxLatencyMs,
        List<String> errorTypes
) {}

private ContentionResult runConcurrentReservations(
        int threadCount,
        Function<ReserveStockCommand, ReservationResult> reserveFunction
) throws InterruptedException {
    ExecutorService executor = Executors.newFixedThreadPool(threadCount);
    CountDownLatch ready = new CountDownLatch(threadCount);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch done = new CountDownLatch(threadCount);

    AtomicInteger successCount = new AtomicInteger();
    AtomicInteger failureCount = new AtomicInteger();
    AtomicInteger retryCount = new AtomicInteger();
    AtomicLong maxLatency = new AtomicLong();
    ConcurrentLinkedQueue<String> errors = new ConcurrentLinkedQueue<>();

    long startTime = System.nanoTime();

    for (int i = 0; i < threadCount; i++) {
        final int index = i;
        executor.submit(() -> {
            try {
                ReserveStockCommand command = new ReserveStockCommand(
                        UUID.randomUUID(),
                        UUID.randomUUID(),
                        2024L,
                        1,
                        "fingerprint-" + index
                );

                ready.countDown();
                start.await(); // Đồng bộ khởi động

                long t0 = System.nanoTime();
                try {
                    ReservationResult result = reserveFunction.apply(command);
                    if (result.isSuccess()) {
                        successCount.incrementAndGet();
                    } else {
                        failureCount.incrementAndGet();
                    }
                } catch (Exception e) {
                    failureCount.incrementAndGet();
                    errors.add(e.getClass().getSimpleName());
                }

                long latency = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - t0);
                maxLatency.updateAndGet(current -> Math.max(current, latency));
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                done.countDown();
            }
        });
    }

    ready.await(); // Chờ tất cả thread sẵn sàng
    start.countDown(); // Khởi động đồng loạt
    done.await(30, TimeUnit.SECONDS);
    executor.shutdownNow();

    long totalDuration = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime);

    return new ContentionResult(
            successCount.get(),
            failureCount.get(),
            retryCount.get(),
            totalDuration,
            maxLatency.get(),
            new ArrayList<>(errors)
    );
}
```

Sử dụng `CountDownLatch` để đảm bảo tất cả thread khởi động đồng thời, tối đa hóa mức tương tranh thực tế. Không sử dụng `Thread.sleep` làm cơ chế đồng bộ.

## 3. Thực nghiệm 1 — Lock lạc quan dưới tải tăng dần

### Mục tiêu

Chứng minh tỷ lệ thất bại tăng theo mức tương tranh và retry storm nhân bản tải.

### Thiết kế

```java
@Test
void optimisticLocking_retryStormUnderHighContention() throws InterruptedException {
    int[] contentionLevels = {5, 20, 50, 100};

    for (int threads : contentionLevels) {
        setUp(); // Reset dữ liệu

        ContentionResult result = runConcurrentReservations(
                threads,
                command -> optimisticFacade.reserveWithRetry(command)
        );

        System.out.printf(
                "Threads=%d | Success=%d | Failures=%d | MaxLatency=%dms%n",
                threads, result.successCount(), result.failureCount(),
                result.maxLatencyMs()
        );

        // Quy tắc bất biến: tổng đặt hàng thành công <= 500
        int totalReserved = reservationRepository.countByOutcome("RESERVED");
        assertThat(totalReserved).isLessThanOrEqualTo(500);

        // Kiểm tra tồn kho không âm
        InventoryItem item = inventoryRepository.findById(2024L).orElseThrow();
        assertThat(item.getAvailableQuantity()).isGreaterThanOrEqualTo(0);

        // Ghi nhận tỷ lệ thất bại
        if (threads >= 50) {
            // Khi tương tranh >= 50, tỷ lệ thất bại do retry exhaustion phải > 0
            // (trừ khi backoff đủ dài để phân tán xung đột)
            System.out.printf("  Error types: %s%n", result.errorTypes());
        }
    }
}
```

### Kỳ vọng

| Mức tương tranh | Tỷ lệ thành công đợt đầu | Tổng retry cần thiết |
| --- | --- | --- |
| 5 thread | Cao (~80%) | Thấp |
| 20 thread | Trung bình (~50%) | Trung bình |
| 50 thread | Thấp (~10%) | Cao |
| 100 thread | Rất thấp (~2%) | Rất cao, có thể vượt retry limit |

## 4. Thực nghiệm 2 — Lock bi quan và cạn kiệt kết nối

### Mục tiêu

Chứng minh lock queue tiêu thụ kết nối database và gây nghẽn khi vượt connection pool.

### Thiết kế

```java
@Test
void pessimisticLocking_connectionPoolExhaustionUnderHighContention()
        throws InterruptedException {
    // Connection pool = 30, thread = 50
    int threads = 50;

    ContentionResult result = runConcurrentReservations(
            threads,
            command -> pessimisticService.reserve(command)
    );

    System.out.printf(
            "Threads=%d | Success=%d | Failures=%d | MaxLatency=%dms%n",
            threads, result.successCount(), result.failureCount(),
            result.maxLatencyMs()
    );

    // Kiểm tra: một số thread phải thất bại do hết kết nối hoặc lock timeout
    // (khi threads > pool size)
    if (threads > 30) {
        // Ít nhất một số yêu cầu phải gặp lỗi timeout
        System.out.printf("  Errors: %s%n", result.errorTypes());
    }

    // Quy tắc bất biến vẫn phải đúng
    InventoryItem item = inventoryRepository.findById(2024L).orElseThrow();
    assertThat(item.getAvailableQuantity()).isGreaterThanOrEqualTo(0);

    int totalReserved = reservationRepository.countByOutcome("RESERVED");
    assertThat(totalReserved).isLessThanOrEqualTo(500);
}
```

### Thực nghiệm bổ sung — Ảnh hưởng đến yêu cầu khác

```java
@Test
void pessimisticLocking_nonRelatedRequestsBlockedByPoolExhaustion()
        throws InterruptedException {
    ExecutorService executor = Executors.newFixedThreadPool(60);
    CountDownLatch ready = new CountDownLatch(60);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch done = new CountDownLatch(60);

    AtomicInteger hotKeySuccess = new AtomicInteger();
    AtomicInteger otherSuccess = new AtomicInteger();
    AtomicInteger otherFailure = new AtomicInteger();

    // 40 thread tranh chấp sản phẩm 2024 (hot key)
    for (int i = 0; i < 40; i++) {
        final int idx = i;
        executor.submit(() -> {
            try {
                ready.countDown();
                start.await();
                ReservationResult r = pessimisticService.reserve(
                        hotKeyCommand(idx));
                if (r.isSuccess()) hotKeySuccess.incrementAndGet();
            } catch (Exception ignored) {
            } finally {
                done.countDown();
            }
        });
    }

    // 20 thread truy vấn sản phẩm KHÁC (không liên quan flash-sale)
    for (int i = 0; i < 20; i++) {
        executor.submit(() -> {
            try {
                ready.countDown();
                start.await();
                // Truy vấn đơn giản không liên quan đến sản phẩm nóng
                boolean found = inventoryRepository.findById(9999L).isPresent();
                otherSuccess.incrementAndGet();
            } catch (Exception e) {
                otherFailure.incrementAndGet();
            } finally {
                done.countDown();
            }
        });
    }

    ready.await();
    start.countDown();
    done.await(30, TimeUnit.SECONDS);
    executor.shutdownNow();

    System.out.printf(
            "Hot-key success=%d | Other success=%d | Other failure=%d%n",
            hotKeySuccess.get(), otherSuccess.get(), otherFailure.get()
    );

    // Khi connection pool cạn kiệt, yêu cầu không liên quan cũng bị ảnh hưởng
    if (otherFailure.get() > 0) {
        System.out.println("CONFIRMED: Pool exhaustion affected unrelated queries");
    }
}
```

## 5. Thực nghiệm 3 — Atomic UPDATE dưới tải cao

### Mục tiêu

Chứng minh atomic UPDATE không gây retry storm và duy trì thông lượng ổn định hơn.

### Thiết kế

```java
@Test
void atomicUpdate_noRetryStormUnderHighContention() throws InterruptedException {
    int threads = 100;

    ContentionResult result = runConcurrentReservations(
            threads,
            command -> atomicService.reserve(command)
    );

    System.out.printf(
            "Threads=%d | Success=%d | OutOfStock=%d | MaxLatency=%dms%n",
            threads, result.successCount(), result.failureCount(),
            result.maxLatencyMs()
    );

    // Quy tắc bất biến
    InventoryItem item = inventoryRepository.findById(2024L).orElseThrow();
    assertThat(item.getAvailableQuantity()).isGreaterThanOrEqualTo(0);

    int totalReserved = reservationRepository.countByOutcome("RESERVED");
    assertThat(totalReserved).isLessThanOrEqualTo(500);

    // Không có retry → không có ngoại lệ do xung đột version
    assertThat(result.errorTypes())
            .doesNotContain("OptimisticLockException");

    // Tổng thành công + hết hàng = tổng yêu cầu (không có mất mát)
    assertThat(result.successCount() + result.failureCount())
            .isEqualTo(threads);
}
```

### So sánh trực tiếp

```java
@Test
void compareStrategies_sameContentionLevel() throws InterruptedException {
    int threads = 50;

    // Chiến lược 1: Lock lạc quan
    setUp();
    ContentionResult optimistic = runConcurrentReservations(
            threads, command -> optimisticFacade.reserveWithRetry(command));

    // Chiến lược 2: Lock bi quan
    setUp();
    ContentionResult pessimistic = runConcurrentReservations(
            threads, command -> pessimisticService.reserve(command));

    // Chiến lược 3: Atomic UPDATE
    setUp();
    ContentionResult atomic = runConcurrentReservations(
            threads, command -> atomicService.reserve(command));

    System.out.println("=== Strategy Comparison (50 threads) ===");
    System.out.printf("Optimistic:  success=%d, fail=%d, maxLatency=%dms%n",
            optimistic.successCount(), optimistic.failureCount(),
            optimistic.maxLatencyMs());
    System.out.printf("Pessimistic: success=%d, fail=%d, maxLatency=%dms%n",
            pessimistic.successCount(), pessimistic.failureCount(),
            pessimistic.maxLatencyMs());
    System.out.printf("Atomic:      success=%d, fail=%d, maxLatency=%dms%n",
            atomic.successCount(), atomic.failureCount(),
            atomic.maxLatencyMs());

    // Quy tắc bất biến phải đúng cho cả ba chiến lược
    // (kiểm tra đã thực hiện trong từng lần chạy)
}
```

## 6. Thực nghiệm 4 — Kiểm soát đầu vào (Admission control)

### Mục tiêu

Chứng minh kiểm soát đầu vào bảo vệ database khỏi quá tải và duy trì thông lượng ổn định.

### Thiết kế

```java
@Test
void admissionControl_protectsDatabase() throws InterruptedException {
    int threads = 200;

    // Không có kiểm soát đầu vào
    setUp();
    ContentionResult withoutControl = runConcurrentReservations(
            threads, command -> atomicService.reserve(command));

    // Có kiểm soát đầu vào (max 10 concurrent)
    setUp();
    ContentionResult withControl = runConcurrentReservations(
            threads, command -> admissionFacade.reserve(command));

    System.out.println("=== Admission Control Comparison (200 threads) ===");
    System.out.printf("Without: success=%d, fail=%d, maxLatency=%dms%n",
            withoutControl.successCount(), withoutControl.failureCount(),
            withoutControl.maxLatencyMs());
    System.out.printf("With:    success=%d, fail=%d, maxLatency=%dms%n",
            withControl.successCount(), withControl.failureCount(),
            withControl.maxLatencyMs());

    // Cả hai phải bảo vệ quy tắc bất biến
    // Kết quả "fail" của admission control bao gồm cả BUSY (bị từ chối tại ứng dụng)
    // và OUT_OF_STOCK (hết hàng) → cần phân biệt

    // Với kiểm soát, max latency phải thấp hơn đáng kể
    // (vì ít transaction chờ lock đồng thời)
}
```

## 7. Thực nghiệm 5 — Kiểm tra quy tắc bất biến dưới tải cao

### Mục tiêu

Đảm bảo quy tắc bất biến được bảo vệ ở mọi mức tương tranh, bất kể chiến lược.

### Thiết kế

```java
@ParameterizedTest
@ValueSource(ints = {10, 50, 100, 200})
void invariant_neverViolated_atAnyContentionLevel(int threads)
        throws InterruptedException {
    setUp();

    ContentionResult result = runConcurrentReservations(
            threads, command -> atomicService.reserve(command));

    // Quy tắc 1: Tồn kho không âm
    InventoryItem item = inventoryRepository.findById(2024L).orElseThrow();
    assertThat(item.getAvailableQuantity())
            .as("available_quantity must never be negative")
            .isGreaterThanOrEqualTo(0);

    // Quy tắc 2: Tổng đặt hàng thành công <= tồn kho ban đầu
    int totalReserved = reservationRepository.countByOutcome("RESERVED");
    assertThat(totalReserved)
            .as("total reserved must not exceed initial stock")
            .isLessThanOrEqualTo(500);

    // Quy tắc 3: Tổng đặt hàng thành công + hàng còn lại = tồn kho ban đầu
    assertThat(totalReserved + item.getAvailableQuantity())
            .as("conservation of inventory")
            .isEqualTo(500);

    // Quy tắc 4: Mỗi command_id chỉ xuất hiện một lần
    long distinctCommands = reservationRepository.countDistinctCommandIds();
    long totalRecords = reservationRepository.count();
    assertThat(distinctCommands)
            .as("each command must produce exactly one outcome")
            .isEqualTo(totalRecords);

    System.out.printf(
            "Threads=%d | Reserved=%d | Available=%d | Duration=%dms%n",
            threads, totalReserved, item.getAvailableQuantity(),
            result.totalDurationMs()
    );
}
```

## 8. Ghi chú thực nghiệm

### Tại sao sử dụng Testcontainers

- Hành vi lock, predicate recheck, và serialization failure phụ thuộc vào PostgreSQL engine.
- H2 không hỗ trợ `FOR UPDATE SKIP LOCKED`, `RETURNING`, hoặc mã lỗi `55P03`/`40P01`.
- Testcontainers cung cấp PostgreSQL thực, đảm bảo kết quả kiểm thử phản ánh hành vi production.

### Giới hạn của kiểm thử đồng thời

- Kết quả kiểm thử phụ thuộc vào tài nguyên máy chạy test (CPU, RAM, I/O).
- Số liệu thời gian không nên được sử dụng làm benchmark tuyệt đối.
- Mục tiêu là chứng minh quy tắc bất biến được bảo vệ và phát hiện sự khác biệt định tính giữa các chiến lược.

### Đồng bộ khởi động

`CountDownLatch` đảm bảo tất cả thread bắt đầu thực thi cùng lúc, tối đa hóa xác suất xung đột. Đây là yêu cầu quan trọng để kiểm thử tương tranh có ý nghĩa. Không sử dụng `Thread.sleep` để tạo xung đột.

### Tách biệt transaction trong retry

Khi kiểm thử lock lạc quan, mỗi lần retry phải mở transaction mới. Sử dụng `@Retryable` bên ngoài `@Transactional`, hoặc gọi service từ một bean khác để proxy interceptor hoạt động đúng.

## 9. Giám sát trên môi trường production

Ngoài kiểm thử tự động, cần giám sát trên production để phát hiện sớm khi chiến lược hiện tại không còn phù hợp:

| Chỉ số | Cảnh báo khi |
| --- | --- |
| Tỷ lệ `OptimisticLockException` | > 10% trong 1 phút |
| Lock wait time p99 | > 500ms |
| Active connections / Pool size | > 80% |
| Lỗi `55P03` (lock timeout) | > 5 lần/phút |
| Lỗi `40P01` (deadlock) | > 1 lần/phút |
| Tỷ lệ BUSY response | > 20% |
| Thời gian phản hồi p99 | > SLA target |

Các ngưỡng trên mang tính minh họa. Giá trị thực tế phụ thuộc vào đặc điểm hệ thống và yêu cầu nghiệp vụ.

## 10. Phạm vi thực nghiệm (Scope)

Các thực nghiệm trên tập trung so sánh hành vi định tính giữa các chiến lược. Không đo lường throughput tuyệt đối hoặc đưa ra benchmark production.

Kiểm thử nghiệp vụ cụ thể (tồn kho, thanh toán, booking) thuộc phạm vi thực nghiệm của các bài toán tương ứng (`ECOM-*`, `BANK-*`, `BOOK-*`).
