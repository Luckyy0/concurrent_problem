# Các thử nghiệm tích hợp PostgreSQL và xác minh trên production

## Chiến lược kiểm thử

Dùng `@SpringBootTest` + PostgreSQL Testcontainers để quan sát các commit thực sự,
rollback-only và `UnexpectedRollbackException`. Phương thức kiểm thử không đánh dấu
`@Transactional`; tiến trình đọc dùng transaction độc lập. `CheckoutProbe` dừng
thread ngoài sau khi `REQUIRES_NEW` bên trong đã trả về/commit.

Không dùng `Thread.sleep`; mọi chốt (latch)/future đều có thời gian chờ (timeout). Xem
[Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Hạ tầng (Infrastructure)

```java
@Testcontainers
@SpringBootTest
class TransactionPropagationPartialCommitIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }

    @Autowired CheckoutService requiresNewCheckout;
    @Autowired AtomicCheckoutService requiredCheckout;
    @Autowired CheckoutWithCaughtFailure caughtFailureCheckout;
    @Autowired CheckoutProbe probe;
    @Autowired PlatformTransactionManager transactionManager;
    @Autowired JdbcTemplate jdbc;
}
```

Mỗi bài kiểm thử thiết lập order 42 đã commit với trạng thái `PENDING`, xóa các dòng audit/risk. Bảng
audit dùng khóa duy nhất `(operation_id, event_type)`; việc kiểm thử ngữ nghĩa từng phần (partial semantics) không phụ
thuộc vào bộ đệm persistence-context.

## Probe có giới hạn (Bounded probe)

```java
final class CheckoutProbe {
    private final AtomicReference<Gate> current = new AtomicReference<>();

    Gate arm() {
        Gate gate = new Gate();
        current.set(gate);
        return gate;
    }

    void afterAuditCommit() {
        Gate gate = current.get();
        if (gate == null) return;
        gate.innerReturned.countDown();
        awaitOrFail(gate.allowOuterFinish);
    }

    record Gate(
            CountDownLatch innerReturned,
            CountDownLatch allowOuterFinish
    ) {
        Gate() { this(new CountDownLatch(1), new CountDownLatch(1)); }
    }
}

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

## Thử nghiệm 1: REQUIRES_NEW audit commit trước khi rollback ở khối ngoài

```java
@Test
void completedAuditIsVisibleBeforeOuterOutcomeAndSurvivesRollback()
        throws Exception {
    CheckoutProbe.Gate gate = probe.arm();

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> writer = executor.submit(() ->
                requiresNewCheckout.complete(42L, true));

        assertTrue(gate.innerReturned().await(5, TimeUnit.SECONDS));

        Snapshot duringOuter = readSnapshotInNewTransaction(42L);
        assertEquals("PENDING", duringOuter.orderStatus());
        assertEquals(List.of("PAYMENT_COMPLETED"), duringOuter.auditTypes());

        gate.allowOuterFinish().countDown();
        assertThrows(ExecutionException.class,
                () -> writer.get(5, TimeUnit.SECONDS));
    }

    Snapshot afterRollback = readSnapshotInNewTransaction(42L);
    assertEquals("PENDING", afterRollback.orderStatus());
    assertEquals(List.of("PAYMENT_COMPLETED"), afterRollback.auditTypes());
}
```

Probe nằm sau khi `recordPaymentCompleted` trả về, nên transaction bên trong đã commit.
Tiến trình đọc thấy được sự mâu thuẫn ngữ nghĩa một cách xác định (deterministic), không phụ thuộc vào may rủi của race condition.

> **Nói ngắn gọn:** thử nghiệm chứng minh partial commit bằng trạng thái nghiệp vụ đã commit,
> không chỉ bằng việc hai phương thức báo rằng transaction đang hoạt động.

## Thử nghiệm 2: REQUIRED bản ghi thành công rollback cùng khối ngoài

`AtomicCheckoutService` giống khối ngoài bị lỗi nhưng gọi audit `REQUIRED`. Sau khi audit,
probe giữ khối ngoài rồi kiểm tra tiến trình đọc phải thấy trạng thái đã commit cũ và không có bản ghi audit.
Cho khối ngoài gặp lỗi, trạng thái cuối cùng vẫn là PENDING và không có bản ghi audit:

```java
@Test
void requiredAuditRollsBackWithCheckout() throws Exception {
    CheckoutProbe.Gate gate = probe.arm();

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> writer = executor.submit(() ->
                requiredCheckout.complete(42L, true));

        assertTrue(gate.innerReturned().await(5, TimeUnit.SECONDS));
        Snapshot duringOuter = readSnapshotInNewTransaction(42L);
        assertEquals("PENDING", duringOuter.orderStatus());
        assertTrue(duringOuter.auditTypes().isEmpty());

        gate.allowOuterFinish().countDown();
        assertThrows(ExecutionException.class,
                () -> writer.get(5, TimeUnit.SECONDS));
    }

    Snapshot afterRollback = readSnapshotInNewTransaction(42L);
    assertEquals("PENDING", afterRollback.orderStatus());
    assertTrue(afterRollback.auditTypes().isEmpty());
}
```

Cùng chốt (latch) `innerReturned` có ngữ nghĩa khác nhau theo propagation: `REQUIRES_NEW`
trả về sau khi commit bên trong, còn `REQUIRED` trả về khi transaction vật lý vẫn mở.
Việc kiểm tra (assertion) cơ sở dữ liệu phân biệt hai trường hợp này.

## Thử nghiệm 3: REQUIRED ngoại lệ bị bắt (catch) vẫn rollback-only

```java
@Test
void caughtInnerRequiredFailureCausesUnexpectedRollback() {
    UnexpectedRollbackException failure = assertThrows(
            UnexpectedRollbackException.class,
            () -> caughtFailureCheckout.complete(42L)
    );

    assertNotNull(failure);
    Snapshot snapshot = readSnapshotInNewTransaction(42L);
    assertEquals("PENDING", snapshot.orderStatus());
    assertTrue(snapshot.auditTypes().isEmpty());
}
```

Kiểm thử phải gọi Spring proxy bean. Service bên trong cũng là một bean/proxy khác để ngoại lệ
đi qua transactional interceptor và đánh dấu rollback-only. Không làm giả (mock) transaction
manager trong kiểm thử này.

## Thử nghiệm 4: REQUIRES_NEW ghi nhận bản ghi thử nghiệm (attempt) một cách trung thực

Khối ngoài rollback nhưng dòng độc lập phải có loại `ATTEMPT_STARTED`, không có
`PAYMENT_COMPLETED`. Thử lại (retry) cùng `operationId` hai lần rồi kiểm tra (assert) rằng ràng buộc duy nhất (unique constraint) hoặc
`insertIfAbsent` chỉ tạo một lần thử. Kiểm thử này xác nhận ngữ nghĩa partial commit dự kiến
và tính lũy đẳng (idempotency), không chỉ là kỹ thuật propagation.

## Thử nghiệm 5: chính sách xử lý lỗi bên trong

Bổ sung ma trận:

| Propagation/kết quả bên trong | Xử lý khối ngoài | Kết quả mong đợi |
| --- | --- | --- |
| REQUIRED lỗi runtime | lan truyền | toàn bộ rollback, nguyên nhân gốc |
| REQUIRED lỗi runtime | bắt lỗi (catch) | khối ngoài rollback + `UnexpectedRollbackException` |
| REQUIRED giá trị từ chối dự kiến | rẽ nhánh rõ ràng | commit/rollback theo quyết định của khối ngoài |
| REQUIRES_NEW lỗi | bắt lỗi (catch) | khối ngoài có thể commit; khối trong rollback |
| REQUIRES_NEW thành công, khối ngoài lỗi | lan truyền lỗi ở khối ngoài | khối trong commit vẫn tồn tại |

Mỗi dòng cần kiểm tra (assertion) tích hợp với cơ sở dữ liệu, không chỉ dùng Mockito để xác minh (verify).

## Transaction độc lập của tiến trình đọc

```java
Snapshot readSnapshotInNewTransaction(long orderId) {
    TransactionTemplate reader = new TransactionTemplate(transactionManager);
    reader.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    reader.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    return reader.execute(status -> querySnapshot(jdbc, orderId));
}
```

## Thử nghiệm chẩn đoán Pool/lock

Với kiểm thử pool nhỏ, nhiều transaction ngoài đồng thời giữ connection rồi gọi
`REQUIRES_NEW` có thể phải chờ connection thứ hai. Kiểm thử dùng future timeout có giới hạn và các chỉ số pool
để minh họa sự gia tăng tiêu thụ tài nguyên (resource amplification), nhưng không biến đây thành bài đánh giá hiệu năng (sizing
benchmark). Việc cạn kiệt toàn bộ connection-pool thuộc `SPR-007`.

Transaction bên trong truy cập dòng (row) mà khối ngoài đã cập nhật có thể bị chặn (block). Dùng thông số
`lock_timeout` nhỏ của PostgreSQL trong kiểm thử chẩn đoán, kiểm tra (assert) timeout/rollback và thu thập
PID đang gây block; không dùng sleep.

## Xác minh trên production

- loại audit so với trạng thái order đã commit;
- `UnexpectedRollbackException`, rollback-only và nguyên nhân gốc bên trong;
- ID transaction ngoài/trong, thời lượng, thời gian tạm dừng;
- connection đang hoạt động/chờ cấp phát và pool timeout;
- timeout của lock bên trong/deadlock;
- ID thao tác bị trùng lặp/xung đột;
- kết quả after-commit/outbox khi việc xuất bản thành công được tách riêng.

## Danh sách kiểm tra chất lượng

- [ ] PostgreSQL Testcontainers được sử dụng.
- [ ] Kiểm thử không dùng `@Transactional` ở khối ngoài.
- [ ] Việc commit REQUIRES_NEW bên trong được quan sát thấy khi khối ngoài còn mở.
- [ ] Rollback ở khối ngoài giữ lại bản ghi audit độc lập nhưng rollback order.
- [ ] Bản ghi audit REQUIRED không thể nhìn thấy và rollback cùng khối ngoài.
- [ ] Lỗi REQUIRED bị bắt (catch) sẽ ném `UnexpectedRollbackException`.
- [ ] Tiến trình đọc dùng transaction `REQUIRES_NEW` độc lập.
- [ ] Mọi latch/future đều có thời gian chờ (timeout); không dùng sleep.
- [ ] Ngữ nghĩa lần thử trung thực và tính lũy đẳng (idempotency) được kiểm thử.
- [ ] Sự gia tăng connection/lock được phân biệt rõ với tính đúng đắn của logic.
