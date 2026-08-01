# Thử nghiệm tích hợp PostgreSQL và xác minh trên production

## Chiến lược kiểm thử

Unit test thuần Java không kiểm tra Spring proxy, commit của repository transaction
hay PostgreSQL MVCC. Dùng `@SpringBootTest` + PostgreSQL Testcontainers, gọi bean
từ application context và không annotate phương thức test bằng `@Transactional`.

Một latch probe dừng luồng ghi (writer) sau khi phương thức debit của repository trả về. Với service bị lỗi,
transaction của debit đã commit. Với harness có transaction, SQL của debit nằm
trong transaction bên ngoài (outer transaction) và chưa commit.

Xem [Kiểm thử đồng thời](../../concepts/concurrency-testing.md).

## Test infrastructure

```java
@Testcontainers
@SpringBootTest
class TransactionalSelfInvocationIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }

    @Autowired BrokenTransferService broken;
    @Autowired TransactionalTransferHarness fixed;
    @Autowired LatchingTransferStepProbe probe;
    @Autowired JdbcTemplate jdbc;
    @Autowired PlatformTransactionManager transactionManager;
}
```

Schema/migration tạo `account(id bigint primary key, balance bigint not null
check (balance >= 0))`. Trước mỗi test insert A=100, B=100 trong transaction
thiết lập ban đầu đã commit. Executor dùng hai platform thread và luôn được shutdown/await ở giới hạn.

## Latch probe

```java
final class LatchingTransferStepProbe implements TransferStepProbe {
    private final AtomicReference<Gate> current = new AtomicReference<>();

    Gate arm() {
        Gate gate = new Gate();
        current.set(gate);
        return gate;
    }

    @Override
    public void afterDebit() {
        Gate gate = current.get();
        if (gate == null) return;
        gate.debitReached.countDown();
        awaitOrFail(gate.allowCredit);
    }

    record Gate(
            CountDownLatch debitReached,
            CountDownLatch allowCredit
    ) {
        Gate() { this(new CountDownLatch(1), new CountDownLatch(1)); }
    }
}
```

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

Production bean của `TransferStepProbe` là no-op; cấu hình test thay bằng
`@Primary` latching probe.

## Thử nghiệm 1: self-invocation bị lỗi làm lộ commit một phần

```java
@Test
void readerSeesCommittedDebitBeforeCredit() throws Exception {
    LatchingTransferStepProbe.Gate gate = probe.arm();

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> writer = executor.submit(() -> broken.transfer(
                new TransferCommand(1L, 2L, 10L)
        ));

        assertTrue(gate.debitReached().await(5, TimeUnit.SECONDS));
        assertEquals(190L, readTotalInNewTransaction());

        gate.allowCredit().countDown();
        writer.get(5, TimeUnit.SECONDS);
    }

    assertEquals(200L, readTotalInNewTransaction());
}
```

Business assertion `total=190` chứng minh trạng thái một phần có thể hiển thị bên ngoài (externally visible); chỉ
kiểm tra cờ transaction-active là chưa đủ.

> **Nói ngắn gọn:** test đọc bằng transaction khác, vì cùng persistence context
>hoặc outer test transaction có thể che giấu ranh giới commit thật.

## Thử nghiệm 2: credit thất bại không rollback debit đã lỗi

Dùng destination ID không tồn tại để `credit` trả về số dòng bị ảnh hưởng (affected rows) bằng 0:

```java
@Test
void brokenCreditFailureLeavesDebitCommitted() {
    assertThrows(ExecutionException.class, () -> {
        try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
            Future<?> writer = executor.submit(() -> broken.transfer(
                    new TransferCommand(1L, 999L, 10L)
            ));
            writer.get(5, TimeUnit.SECONDS);
        }
    });

    assertEquals(90L, balanceOf(1L));
    assertEquals(190L, readTotalInNewTransaction());
}
```

Trong mã test thực tế, unwrap `ExecutionException.getCause()` và assert ngoại lệ domain;
dọn dẹp executor đặt trong `finally` nếu không dùng Java 21
try-with-resources.

## Thử nghiệm 3: proxy transactional giữ trạng thái không hiển thị cho tới khi commit

Test harness là Spring bean riêng, public method được annotate `@Transactional` và có
cùng luồng debit → probe → credit. Nó tồn tại chỉ trong `@TestConfiguration`
để đặt rào cản tất định (deterministic barrier) trong transaction.

```java
@Test
void outerTransactionHidesDebitUntilBothUpdatesCommit() throws Exception {
    assertTrue(AopUtils.isAopProxy(fixed));
    LatchingTransferStepProbe.Gate gate = probe.arm();

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> writer = executor.submit(() -> fixed.transferWithPause(
                new TransferCommand(1L, 2L, 10L)
        ));

        assertTrue(gate.debitReached().await(5, TimeUnit.SECONDS));
        assertEquals(200L, readTotalInNewTransaction());

        gate.allowCredit().countDown();
        writer.get(5, TimeUnit.SECONDS);
    }

    assertEquals(90L, balanceOf(1L));
    assertEquals(110L, balanceOf(2L));
    assertEquals(200L, readTotalInNewTransaction());
}
```

Probe có thể ghi lại
`TransactionSynchronizationManager.isActualTransactionActive()` để assert `true`
ở fixed harness và `false` sau debit bị lỗi của repository, nhưng đây là cấu trúc kiểm tra
bổ sung, không thay thế business invariant tổng.

## Thử nghiệm 4: lỗi rollback cả transfer đúng

Fixed harness transfer tới destination không tồn tại phải ném ra ngoại lệ. Sau
khi future hoàn thành, assert A=100, B=100, total=200. Bổ sung biến thể checked
exception nếu production dùng `rollbackFor`; test phải chứng minh chính sách thực tế,
không chỉ đọc annotation.

## Transaction luồng đọc độc lập

```java
long readTotalInNewTransaction() {
    TransactionTemplate reader = new TransactionTemplate(transactionManager);
    reader.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    reader.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    return reader.execute(status -> jdbc.queryForObject(
            "select sum(balance) from account", Long.class));
}
```

Không dùng lại entity đã load trước bulk update; đọc vô hướng SQL (SQL scalar) trong transaction mới
tránh cache của persistence-context.

## Kiểm tra proxy và đường dẫn lời gọi (call path)

- Assert bean lấy từ context là AOP proxy.
- Gọi phương thức public đầu vào qua bean được inject, không khởi tạo (new) và không gọi target unwrap.
- Dùng log/test probe xác nhận transaction đang active ở đúng ranh giới.
- Test self-invocation bị lỗi và bean bên ngoài đã sửa (fixed external-bean) như hai trường hợp riêng biệt.
- Nếu dùng `TransactionTemplate`, assert active bên trong callback và inactive
  sau khi trả về.

## Xác minh trên production

- Bất biến (invariant) bảo toàn tổng số dư qua đối soát;
- transfer một phần/sai lệch affected-row;
- commit/rollback/thời gian chạy của transaction và lỗi rollback-only;
- sử dụng kết nối cơ sở dữ liệu, timeout của statement/lock;
- ID của correlation/transfer xuyên suốt debit-credit;
- sự kiện phát hành trước/sau commit;
- chẩn đoán khởi động (startup diagnostic) xác nhận các bean như kỳ vọng được proxy khi phù hợp.

## Checklist chất lượng

- [ ] PostgreSQL Testcontainers được dùng.
- [ ] Phương thức test không có outer `@Transactional` che giấu commit.
- [ ] Luồng đọc dùng transaction `REQUIRES_NEW` độc lập.
- [ ] Latch dừng đúng sau debit, không dùng sleep.
- [ ] Mọi latch/future có timeout.
- [ ] Test lỗi thấy tổng 190 trước credit.
- [ ] Lỗi ở broken code giữ nguyên debit đã commit.
- [ ] Luồng đọc của code đã sửa chỉ thấy tổng 200 và lỗi sẽ rollback cả hai cập nhật.
- [ ] Proxy/transaction-active assertion bổ sung cho business invariant.
- [ ] Executor và vòng đời container được dọn dẹp.
