# Các thí nghiệm kiểm chứng với PostgreSQL (Integration experiments)

## Chiến lược kiểm thử

Chúng ta sử dụng `@SpringBootTest`, PostgreSQL Testcontainers và một bộ thực thi (executor) thật. Các hàm kiểm thử (test method) sẽ không được gắn `@Transactional`. Một test harness (được cấu hình dưới dạng Spring bean) sẽ mở một giao dịch bên ngoài, gọi `saveAndFlush` để chèn đơn hàng rồi dùng rào chắn (latch) để tạm dừng. Luồng kiểm thử sẽ gọi trực tiếp qua async proxy, giữ lại đối tượng `future` trả về nhằm quan sát kết quả xử lý của worker trước khi quyết định cho giao dịch bên ngoài được tiếp tục commit hay rollback.

Tuyệt đối không dùng lệnh `Thread.sleep`; tất cả các rào chắn (latch) hay future đều phải thiết lập thời gian chờ (timeout). Kiến thức nền: [Kiểm thử các vấn đề đồng thời](../../concepts/concurrency-testing.md).

## Hạ tầng kiểm thử (Infrastructure)

```java
@Testcontainers
@SpringBootTest
class AsyncTransactionBoundaryIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }

    @Autowired TransactionalOrderHarness harness;
    @Autowired AsyncOrderProcessor brokenProcessor;
    @Autowired OrderPlacementService fixedPlacement;
    @Autowired AsyncProcessingProbe processingProbe;
    @Autowired ThreadPoolTaskExecutor orderExecutor;
    @Autowired OrderRepository orders;
    @Autowired DispatchAttemptRepository attempts;
}
```

Lược đồ cơ sở dữ liệu (Schema) dùng chuỗi tuần tự (sequence/identity) cho ID đơn hàng. Trong bài kiểm thử tạo bản ghi mồ côi (orphan test), bảng `dispatch_attempt` cố ý bỏ qua khóa ngoại (FK) để minh họa rõ tác động bên ngoài độc lập; trong thực tế, nếu có khóa ngoại, thao tác này có thể bị chặn hoặc báo lỗi, nhưng bài toán cốt lõi về sự thiếu vắng thứ tự commit chuẩn xác vẫn tồn tại.

## Giao dịch kiểm thử (Transactional harness)

```java
@Service
public class TransactionalOrderHarness {

    @Transactional
    public long insertAndPause(
            PlaceOrderCommand command,
            Gate gate,
            boolean rollbackAfterPause
    ) {
        Order order = orders.saveAndFlush(Order.pending(command.customerId()));
        gate.orderId().set(order.getId());
        gate.flushed().countDown();
        awaitOrFail(gate.allowCompletion());
        if (rollbackAfterPause) {
            throw new IllegalStateException("rollback requested");
        }
        return order.getId();
    }
}

record Gate(
        AtomicLong orderId,
        CountDownLatch flushed,
        CountDownLatch allowCompletion
) {
    Gate() {
        this(new AtomicLong(), new CountDownLatch(1), new CountDownLatch(1));
    }
}
```

Hàm `awaitOrFail`:

```java
private static void awaitOrFail(CountDownLatch latch) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new IllegalStateException("latch timed out");
        }
    } catch (InterruptedException exception) {
        Thread.currentThread().interrupt();
        throw new IllegalStateException("interrupted", exception);
    }
}
```

## Thí nghiệm 1: Async reader không thể thấy đơn hàng chưa commit

```java
@Test
void asyncTransactionCannotReadUncommittedOuterInsert() throws Exception {
    Gate gate = new Gate();

    try (ExecutorService caller = Executors.newSingleThreadExecutor()) {
        Future<Long> writer = caller.submit(() -> harness.insertAndPause(
                new PlaceOrderCommand(7L, false), gate, false));

        assertTrue(gate.flushed().await(5, TimeUnit.SECONDS));
        long orderId = gate.orderId().get();

        CompletableFuture<Void> async = brokenProcessor.process(orderId);
        ExecutionException failure = assertThrows(
                ExecutionException.class,
                () -> async.get(5, TimeUnit.SECONDS)
        );
        assertInstanceOf(IllegalStateException.class, failure.getCause());
        assertFalse(orders.existsById(orderId));

        gate.allowCompletion().countDown();
        assertEquals(orderId, writer.get(5, TimeUnit.SECONDS));
        assertTrue(orders.existsById(orderId));
    }
}
```

Chúng ta giữ future của tiến trình async lại để các ngoại lệ không bị lọt ra ngoài. Đồng thời, giao dịch ghi bên ngoài (writer) vẫn bị chặn lại bởi cổng (gate), do đó lỗi "không tìm thấy dữ liệu" (missing read) là tất yếu, không hề phụ thuộc vào bộ lên lịch (scheduler).

> **Nói ngắn gọn:** Hàm `saveAndFlush` đã cấp ID và gửi câu lệnh `INSERT` xuống cơ sở dữ liệu, nhưng đối tượng future chạy trên một luồng khác chỉ có thể nhìn thấy dữ liệu trạng thái đã được commit.

## Thí nghiệm 2: Tác động bên ngoài (async side effect) sống sót sau khi giao dịch gốc bị rollback

Một bộ ghi bất đồng bộ đặc thù dành cho kiểm thử nhận một ID đơn hàng bất biến (immutable), rồi thực hiện lệnh chèn vào bảng `dispatch_attempt` trong một giao dịch worker độc lập, mà không hề thực hiện truy vấn hay bị ràng buộc khóa ngoại tới bảng đơn hàng:

```java
@Test
void asyncSideEffectSurvivesOuterRollback() throws Exception {
    Gate gate = new Gate();

    try (ExecutorService caller = Executors.newSingleThreadExecutor()) {
        Future<Long> writer = caller.submit(() -> harness.insertAndPause(
                new PlaceOrderCommand(7L, false), gate, true));

        assertTrue(gate.flushed().await(5, TimeUnit.SECONDS));
        long orderId = gate.orderId().get();
        orphanAttemptWriter.write(orderId).get(5, TimeUnit.SECONDS);
        assertEquals(1, attempts.countByOrderId(orderId));

        gate.allowCompletion().countDown();
        assertThrows(ExecutionException.class,
                () -> writer.get(5, TimeUnit.SECONDS));

        assertFalse(orders.existsById(orderId));
        assertEquals(1, attempts.countByOrderId(orderId));
    }
}
```

Kiểm chứng nghiệp vụ (Business assertion) ở đây là một bản ghi kiểm toán tồn tại trong khi đơn hàng gốc không còn tồn tại, chứ không chỉ đơn thuần là so sánh xem hai luồng có ID giao dịch khác nhau hay không.

## Thí nghiệm 3: Trình lắng nghe after-commit phải đợi đến khi commit thành công

Một test harness khác tiến hành lưu đơn hàng, phát hành sự kiện `OrderPlacedEvent`, bắn tín hiệu `eventPublished` rồi mới chặn lại đợi lệnh `allowCommit`; toàn bộ quy trình này nằm gọn trong một hàm `@Transactional` công khai. Lấy trình lắng nghe và trình xử lý từ các giải pháp (solution).

```java
@Test
void afterCommitDispatchWaitsForSuccessfulCommit() throws Exception {
    CommitGate gate = new CommitGate();
    processingProbe.reset();

    try (ExecutorService caller = Executors.newSingleThreadExecutor()) {
        Future<Long> writer = caller.submit(() ->
                afterCommitHarness.placeAndPause(gate, false));

        assertTrue(gate.eventPublished().await(5, TimeUnit.SECONDS));
        assertEquals(0, processingProbe.startedCount());

        gate.allowCommit().countDown();
        long orderId = writer.get(5, TimeUnit.SECONDS);
        assertTrue(processingProbe.started().await(5, TimeUnit.SECONDS));
        assertTrue(processingProbe.succeeded().await(5, TimeUnit.SECONDS));
        assertEquals(orderId, processingProbe.lastOrderId());
        assertTrue(orders.existsById(orderId));
    }
}
```

Chúng ta tiến hành đọc giá trị `startedCount=0` trong khi chắc chắn rằng giao dịch vẫn đang bị rào chắn (gate) kìm lại; do đó, không cần dùng phương pháp rủi ro như chờ một khoảng thời gian giả định (negative sleep assertion).

## Thí nghiệm 4: Giao dịch bị rollback thì không kích hoạt trình lắng nghe

Harness tiến hành phát hành sự kiện nhưng sau đó lại rollback. Sau khi future của giao dịch ghi thất bại, chúng ta gửi một tác vụ đánh dấu (sentinel) vào cùng một executor và đợi nó hoàn thành để chắc chắn đã "rút cạn" (drain) mọi tác vụ đang nằm trong hàng đợi trước nó:

```java
@Test
void rollbackDoesNotScheduleAfterCommitProcessor() throws Exception {
    processingProbe.reset();
    assertThrows(ExecutionException.class, () -> {
        try (ExecutorService caller = Executors.newSingleThreadExecutor()) {
            Future<Long> writer = caller.submit(() ->
                    afterCommitHarness.placeAndRollback());
            writer.get(5, TimeUnit.SECONDS);
        }
    });

    Future<?> sentinel = orderExecutor.getThreadPoolExecutor().submit(() -> {});
    sentinel.get(5, TimeUnit.SECONDS);

    assertEquals(0, processingProbe.startedCount());
    assertEquals(0, orders.count());
}
```

Sentinel giúp loại bỏ thói quen code lỗi "đợi 300ms rồi cầu nguyện hy vọng trình lắng nghe không chạy". Nếu sự kiện vô tình bị lên lịch (schedule) trước sentinel trên một single-worker executor của môi trường kiểm thử, biến đếm của probe chắc chắn sẽ tăng lên trước khi ta kiểm tra xác nhận (assertion).

## Thí nghiệm 5: Executor từ chối tác vụ (rejection) sau khi đã commit

Cấu hình bộ thực thi (executor) trong môi trường kiểm thử với chỉ một worker duy nhất, hàng đợi chỉ chứa được một slot và áp dụng chính sách `AbortPolicy`; sau đó, dùng rào chắn giữ chân worker và hàng đợi, rồi tiến hành commit đơn hàng/sự kiện thứ ba. Kết quả là, dù việc chuyển giao sau commit (after-commit dispatch) ném ra ngoại lệ `TaskRejectedException` và bộ đo lường từ chối (rejection metric) tăng lên, đơn hàng vẫn được commit thành công trong cơ sở dữ liệu. Điều này minh chứng rõ ràng việc thiết kế sau-commit cục bộ (local after-commit) không hề đảm bảo tính bền vững/nguyên tử (durable/atomic) đi kèm với bước chuyển giao.

Đừng để trường hợp từ chối tác vụ làm luồng kiểm thử kẹt lại với giao dịch mở vô tận; hãy cẩn thận mở rào chắn (release blocker) và gọi hàm chờ bộ thực thi ngưng hoạt động (await executor termination) bên trong khối `finally`.

## Xác minh bối cảnh và giao dịch

Bộ đo lường của worker (Probe worker) ghi nhận các trường hợp sau:

```text
callerThread != workerThread
TransactionSynchronizationManager.isActualTransactionActive() == true
```

Giá trị `true` khẳng định rằng đây là một giao dịch riêng rẽ do bộ xử lý bất đồng bộ tạo ra. Bạn cũng cần bổ sung thêm bài kiểm thử với `TaskDecorator` cho việc dọn dẹp MDC/trace, nhưng tuyệt đối không sao chép các tài nguyên giao dịch (transaction resource). Khẳng định lại một lần nữa: không bao giờ truyền một thực thể `Order` đang được quản lý (managed) vào trình lắng nghe; sự kiện chỉ được phép chứa dữ liệu định danh (ID).

## Kiểm chứng trên môi trường Production

- Các đơn hàng đã commit không được phép rơi vào trạng thái có thời gian hoàn thành (processing outcome) vượt quá SLA.
- Giám sát số lượng các trường hợp đơn hàng bị thiếu (missing-order) hoặc lỗi xử lý mồ côi (orphan-attempt).
- Theo dõi vòng đời after-commit: đã lên lịch (scheduled), bắt đầu (started), thành công (succeeded), thất bại (failed), bị từ chối (rejected).
- Đánh giá sức khỏe của executor: trạng thái hoạt động, thời gian nằm trong hàng đợi (queue age), sức chứa (capacity), tỷ lệ bị từ chối và lượng dữ liệu thất thoát khi hệ thống tắt (shutdown loss).
- Các giao dịch của worker hoàn thành commit, bị rollback hay hết thời gian (timeout).
- Xử lý trễ (late completion) sau khi tiến trình đã bị hủy.
- Hiện tượng tồn đọng hàng đợi (outbox backlog), cơ chế gửi lại (redelivery) và tính lũy đẳng (idempotency) nếu đang dùng giải pháp xử lý bền vững (durable solution).

## Danh sách kiểm tra chất lượng (Checklist)

- [ ] Đã sử dụng PostgreSQL Testcontainers.
- [ ] Lớp kiểm thử (Test) không dùng `@Transactional` bọc bên ngoài.
- [ ] Thao tác insert bên ngoài đã được flush nhưng được giữ cho chưa commit nhờ cơ chế rào chắn (latch).
- [ ] Future bất đồng bộ có phương thức `get` kèm giới hạn thời gian (bounded `get`) và các ngoại lệ được quan sát rõ ràng.
- [ ] Phải có assert để kiểm chứng hiện tượng đọc hụt (missing read) và sinh lỗi mồ côi (orphan side effect).
- [ ] Trình lắng nghe after-commit không được phép khởi chạy trước commit, mà chỉ được start sau khi commit.
- [ ] Bài kiểm thử rollback sử dụng tác vụ đánh dấu (executor sentinel), tuyệt đối không dùng sleep.
- [ ] Lỗ hổng từ chối tác vụ sau khi commit (Rejection-after-commit gap) đã được kiểm chứng.
- [ ] Luồng của caller và executor phải được dọn dẹp cẩn thận, có giới hạn đàng hoàng.
- [ ] Bối cảnh giao dịch (Transaction context) không được phép lan truyền (propagate) xuyên qua các luồng.
