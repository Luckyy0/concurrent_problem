# Môi Trường Thực Nghiệm: Kiểm Chứng Khóa Bi Quan Với Testcontainers

## 1. Mục Tiêu Thực Nghiệm

Bộ kiểm thử này được thiết kế nhằm xác thực các đặc tả hệ thống sau:

1. Hệ quả của việc truy xuất dữ liệu không khóa (Plain SELECT) dẫn đến sai lệch phán quyết đồng thời.
2. Xác nhận cơ chế Khóa Bi Quan (`FOR UPDATE`) đưa các tiến trình cạnh tranh vào trạng thái chờ, ép buộc tiến trình sau phải truy xuất dữ liệu mới nhất.
3. Kịch bản hủy giao dịch (Rollback) giải phóng khóa, cho phép tiến trình chờ tiếp quản tài nguyên toàn vẹn.
4. Cơ chế ngắt giới hạn thời gian chờ (`lock_timeout`) ngăn chặn treo hệ thống, hủy giao dịch an toàn.
5. Kiểm chứng tính độc lập của các luồng truy xuất thông thường (Plain reader) không bị cản trở bởi Khóa Bi Quan.
6. Xác thực nguyên lý sắp xếp thứ tự tài nguyên (Stable order) nhằm ngăn chặn Khóa Chéo (Deadlock) khi tương tác đa bản ghi.
7. Triển khai cấu hình Spring/JPA thực tế tương thích hoàn toàn với hệ quản trị PostgreSQL vật lý.

Khuyến cáo: Tuyệt đối không sử dụng các CSDL in-memory (như H2) cho các kịch bản kiểm thử tương tranh phức tạp, do sự khác biệt cốt lõi trong cơ chế quản trị Khóa (Lock Manager).

## 2. Thiết Lập Môi Trường (Testcontainers Fixture)

Triển khai container PostgreSQL thông qua Testcontainers:

```java
@Testcontainers
@SpringBootTest
class PessimisticSeatHoldIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17-alpine");

    @DynamicPropertySource
    static void datasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
    }

    @Autowired JdbcTemplate jdbc;
    @Autowired PlatformTransactionManager transactionManager;
    @Autowired SeatHoldCoordinator coordinator;

    private ExecutorService executor;

    @BeforeEach
    void reset() {
        executor = Executors.newFixedThreadPool(4);
        jdbc.update("delete from seat_hold");
        // Thiết lập trạng thái ban đầu cho các ghế khảo sát
        jdbc.update("""
                update show_seat
                set state = 'AVAILABLE',
                    hold_id = null,
                    holder_customer_id = null,
                    hold_until = null
                where show_id = 42 and seat_no in ('A-10', 'A-11')
                """);
    }

    @AfterEach
    void stopExecutor() throws InterruptedException {
        executor.shutdownNow();
        assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Kiểm thử được thiết kế dựa trên các định danh lệnh (Command ID) hoàn toàn mới nhằm loại trừ nguy cơ xung đột từ dữ liệu lịch sử.

## 3. Hệ Thống Điều Phối Và Giám Sát (Concurrency Utilities)

Sử dụng `CountDownLatch` để kiểm soát nhịp đồng bộ của luồng, đồng thời truy vấn hệ thống giám sát nội bộ của PostgreSQL nhằm xác nhận trạng thái Kẹt Khóa:

```java
@FunctionalInterface
interface TransactionWork<T> {
    T run();
}

private <T> T inTransaction(TransactionWork<T> work) {
    TransactionTemplate template =
            new TransactionTemplate(transactionManager);
    template.setIsolationLevel(
            TransactionDefinition.ISOLATION_READ_COMMITTED
    );
    return template.execute(status -> work.run());
}

private void awaitLatch(CountDownLatch latch) {
    try {
        assertTrue(latch.await(5, TimeUnit.SECONDS));
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError(interrupted);
    }
}

private void useApplicationName(String name) {
    jdbc.queryForObject(
            "select set_config('application_name', ?, true)",
            String.class,
            name
    );
}

private void awaitDatabaseBlock(String applicationName) {
    long deadline = System.nanoTime() + Duration.ofSeconds(5).toNanos();

    while (System.nanoTime() < deadline) {
        // Kiểm tra trực tiếp bảng hệ thống để xác nhận trạng thái Lock Wait
        Boolean blocked = jdbc.queryForObject("""
                select exists (
                    select 1
                    from pg_stat_activity
                    where application_name = ?
                      and wait_event_type = 'Lock' 
                      and cardinality(pg_blocking_pids(pid)) > 0
                )
                """, Boolean.class, applicationName);
        if (Boolean.TRUE.equals(blocked)) {
            return; // Xác nhận tiến trình đã bị đưa vào hàng đợi
        }

        LockSupport.parkNanos(Duration.ofMillis(10).toNanos());
        if (Thread.currentThread().isInterrupted()) {
            throw new AssertionError("interrupted while observing lock wait");
        }
    }
    fail("Vượt giới hạn thời gian nhưng tiến trình chưa bị khóa.");
}

private String sqlState(Throwable failure) {
    for (Throwable current = failure;
         current != null;
         current = current.getCause()) {
        if (current instanceof SQLException sql) {
            return sql.getSQLState();
        }
    }
    return null;
}
```

Mọi tác vụ chờ (`Future.get`, `Latch`) đều được giới hạn ngưỡng thời gian (Upper bound) nhằm ngăn chặn tình trạng treo tiến trình vô hạn.

## 4. Thí Nghiệm 1: Mô Phỏng Kiến Trúc Thiếu Đồng Bộ (Stale Decision)

Kịch bản thiết lập một Rào chắn (Barrier) buộc hai giao dịch phải hoàn thành khâu truy xuất dữ liệu ban đầu trước khi tiếp tục lệnh ghi.

```java
@Service
class BarrierBrokenSeatHoldTx {
    private final ShowSeatRepository seats;
    private final SeatHoldRepository holds;
    private final CyclicBarrier bothLoaded = new CyclicBarrier(2);

    // Bỏ qua phần Constructor Injection

    @Transactional
    public UUID hold(HoldSeatCommand command) {
        ShowSeat seat = seats.findById(command.seatId()).orElseThrow();
        assertEquals(SeatState.AVAILABLE, seat.state());

        try {
            bothLoaded.await(5, TimeUnit.SECONDS); // Đồng bộ hai luồng tại điểm này
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException(interrupted);
        } catch (BrokenBarrierException | TimeoutException failure) {
            throw new IllegalStateException(failure);
        }

        UUID holdId = UUID.randomUUID();
        seat.hold(holdId, command.customerId(), Instant.now().plusSeconds(120));
        holds.save(SeatHold.active(holdId, command));
        return holdId;
    }
}
```

```java
@Test
void plainLoadLetsTwoBusinessDecisionsPass() throws Exception {
    Future<UUID> a = executor.submit(
            () -> brokenWorker.hold(command("hold-a", 501))
    );
    Future<UUID> b = executor.submit(
            () -> brokenWorker.hold(command("hold-b", 902))
    );

    UUID holdA = a.get(10, TimeUnit.SECONDS);
    UUID holdB = b.get(10, TimeUnit.SECONDS);

    // KẾT QUẢ ĐỐI CHIẾU:
    assertNotEquals(holdA, holdB); // Xác định sự tồn tại của hai định danh độc lập
    assertEquals(2, activeHoldCount(42, "A-10")); // Phát hiện lỗi phân bổ trùng lặp
    assertEquals(1, seatProjectionCount(42, "A-10"));
    assertTrue(currentSeatHoldId(42, "A-10").equals(holdA)
            || currentSeatHoldId(42, "A-10").equals(holdB));
}
```

Trọng điểm là mô phỏng thành công sai sót nghiệp vụ (Double booking), chứng minh tác hại của quy trình phân giải dữ liệu cũ (Stale snapshot).

## 5. Thí Nghiệm 2: Tái Thẩm Định Sau Hàng Chờ (Revalidation)

- Giao dịch A nắm giữ Khóa nhưng chưa hoàn tất (Uncommitted).
- Giao dịch B yêu cầu Khóa và bị hệ thống điều hướng vào hàng đợi.
- Xác nhận B đã nằm trong hàng đợi tại cấp độ CSDL.
- Tiến hành Commit cho Giao dịch A.
- B tiếp quản Khóa, đọc trạng thái dữ liệu mới và kích hoạt kiểm định từ chối do trạng thái đã chuyển thành `HELD`.

```java
@Test
void waiterBlocksThenRejectsCommittedHold() throws Exception {
    CountDownLatch holderHasLock = new CountDownLatch(1);
    CountDownLatch allowHolderCommit = new CountDownLatch(1);

    Future<HoldOutcome> holder = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-holder");
        lockSeat(42, "A-10");
        createHold("hold-a", 501);
        holderHasLock.countDown();
        awaitLatch(allowHolderCommit);
        return HoldOutcome.HELD;
    }));

    assertTrue(holderHasLock.await(5, TimeUnit.SECONDS));

    Future<HoldOutcome> waiter = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-waiter");
        ShowSeatRow seat = lockSeat(42, "A-10");
        return seat.state().equals("AVAILABLE")
                ? createHoldAndReturn("hold-b", 902)
                : HoldOutcome.ALREADY_HELD;
    }));

    awaitDatabaseBlock("lock003-waiter"); // Giám sát tình trạng chờ từ DB
    assertThrows(
            TimeoutException.class,
            () -> waiter.get(100, TimeUnit.MILLISECONDS)
    );

    allowHolderCommit.countDown(); // A hoàn tất Commit

    assertEquals(HoldOutcome.HELD,
            holder.get(5, TimeUnit.SECONDS)); // A thành công
    assertEquals(HoldOutcome.ALREADY_HELD,
            waiter.get(5, TimeUnit.SECONDS)); // B chủ động từ chối
    assertEquals(1, activeHoldCount(42, "A-10"));
    assertEquals(501L, currentSeatCustomer(42, "A-10")); // Tài nguyên thuộc về Khách 501
}
```

Kiểm thử minh chứng luồng B bị phong tỏa vật lý, sau đó được cập nhật Snapshot mới nhất để tự đình chỉ xử lý mà không ghi đè dữ liệu.

## 6. Thí Nghiệm 3: Tiến Trình Hủy Bỏ (Rollback) Khôi Phục Trạng Thái

Thay vì Commit, A ném ngoại lệ mô phỏng quá trình Rollback.
B tiếp quản Khóa, xác minh dữ liệu ở trạng thái `AVAILABLE` và hoàn tất cập nhật.

```java
@Test
void waiterCanWinAfterHolderRollsBack() throws Exception {
    CountDownLatch holderHasLock = new CountDownLatch(1);
    CountDownLatch forceRollback = new CountDownLatch(1);

    Future<HoldOutcome> holder = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-rollback-holder");
        lockSeat(42, "A-10");
        createHold("hold-a", 501);
        holderHasLock.countDown();
        awaitLatch(forceRollback);
        throw new RollbackForTest(); // Kích hoạt hoàn tác
    }));

    assertTrue(holderHasLock.await(5, TimeUnit.SECONDS));

    Future<HoldOutcome> waiter = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-rollback-waiter");
        ShowSeatRow seat = lockSeat(42, "A-10");
        assertEquals("AVAILABLE", seat.state());
        return createHoldAndReturn("hold-b", 902);
    }));

    awaitDatabaseBlock("lock003-rollback-waiter");
    forceRollback.countDown();

    ExecutionException rolledBack =
            assertThrows(ExecutionException.class,
                    () -> holder.get(5, TimeUnit.SECONDS));
    assertInstanceOf(RollbackForTest.class, rolledBack.getCause());

    assertEquals(HoldOutcome.HELD,
            waiter.get(5, TimeUnit.SECONDS)); // B xác nhận thành công
    assertEquals(1, activeHoldCount(42, "A-10"));
    assertEquals(902L, currentSeatCustomer(42, "A-10"));
    assertEquals(0, holdCountByCommand("hold-a"));
}
```

## 7. Thí Nghiệm 4: Giới Hạn Thời Gian Chờ (`lock_timeout`)

Bị cấu hình giới hạn chờ 150ms. A duy trì Khóa trong thời gian dài hơn. Giao dịch B buộc phải hủy bỏ với mã lỗi nội bộ `55P03`.

```java
@Test
void lockTimeoutRollsBackWithoutPartialHold() throws Exception {
    CountDownLatch holderHasLock = new CountDownLatch(1);
    CountDownLatch releaseHolder = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> inTransaction(() -> {
        useApplicationName("lock003-timeout-holder");
        lockSeat(42, "A-10");
        holderHasLock.countDown();
        awaitLatch(releaseHolder);
        return null;
    }));

    assertTrue(holderHasLock.await(5, TimeUnit.SECONDS));

    Future<HoldOutcome> contender = executor.submit(() ->
            inTransaction(() -> {
                useApplicationName("lock003-timeout-contender");
                // Cấu hình Timeout chuyên biệt cho Giao dịch B
                jdbc.queryForObject(
                        "select set_config('lock_timeout', '150ms', true)",
                        String.class
                );
                lockSeat(42, "A-10");
                return createHoldAndReturn("hold-b", 902);
            })
    );

    ExecutionException failed =
            assertThrows(ExecutionException.class,
                    () -> contender.get(5, TimeUnit.SECONDS));
    assertEquals("55P03", sqlState(failed)); // Đối sánh Lỗi Không Lấy Được Khóa
    assertEquals(0, holdCountByCommand("hold-b"));
    assertEquals("AVAILABLE", currentSeatState(42, "A-10"));

    releaseHolder.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
}
```

Lưu ý: Yêu cầu giải phóng Khóa của A (Release holder) phải luôn được định tuyến an toàn để tránh treo ứng dụng trong trường hợp Assertion của B thất bại.

## 8. Thí Nghiệm 5: Sự Độc Lập Của Luồng Truy Xuất Dữ Liệu Thông Thường (Plain Reader)

Khóa Bi Quan không tác động đến các truy vấn `SELECT` tiêu chuẩn. Quá trình tra cứu dữ liệu (Hiển thị) vẫn được phản hồi từ Snapshot mới nhất đã Commit.

```java
@Test
void plainReaderSeesLastCommittedVersion() throws Exception {
    CountDownLatch uncommittedUpdateReady = new CountDownLatch(1);
    CountDownLatch allowCommit = new CountDownLatch(1);

    Future<Void> holder = executor.submit(() -> inTransaction(() -> {
        lockSeat(42, "A-10");
        createHold("hold-a", 501);
        uncommittedUpdateReady.countDown();
        awaitLatch(allowCommit);
        return null;
    }));

    assertTrue(uncommittedUpdateReady.await(5, TimeUnit.SECONDS));

    Future<String> reader = executor.submit(
            () -> currentSeatState(42, "A-10")
    );
    assertEquals("AVAILABLE", reader.get(2, TimeUnit.SECONDS)); // Truy cập Snapshot quá khứ

    allowCommit.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
    assertEquals("HELD", currentSeatState(42, "A-10"));
    assertEquals(1, activeHoldCount(42, "A-10"));
}
```

## 9. Thí Nghiệm 6 & 7: Tương Tranh Tích Hợp Và Ngăn Chặn Deadlock

**Thí nghiệm 6:** Khảo sát tương tranh chuẩn tích hợp qua JPA Repository.

```java
@Test
void twoJpaRequestsProduceOneHoldAndOneRejection() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Callable<HoldResult> actorA = () -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return coordinator.hold(command("hold-a", 501));
    };
    Callable<HoldResult> actorB = () -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return coordinator.hold(command("hold-b", 902));
    };

    Future<HoldResult> a = executor.submit(actorA);
    Future<HoldResult> b = executor.submit(actorB);

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    List<HoldResult> results = List.of(
            a.get(10, TimeUnit.SECONDS),
            b.get(10, TimeUnit.SECONDS)
    );

    assertEquals(1, results.stream().filter(HoldResult::held).count());
    assertEquals(1,
            results.stream().filter(HoldResult::alreadyHeld).count());
    assertEquals(1, activeHoldCount(42, "A-10"));
    assertTrue(projectionMatchesActiveHold(42, "A-10"));
}
```

**Thí nghiệm 7:** Cấp phát đồng thời nhóm tài nguyên (Nhiều ghế). Giao dịch áp dụng chính sách Sắp xếp (Sort) trước khi Yêu cầu Khóa để ngăn chặn triệt để rủi ro Deadlock.

```java
@Test
void reverseInputUsesSameDatabaseLockOrder() throws Exception {
    CountDownLatch ready = new CountDownLatch(2);
    CountDownLatch start = new CountDownLatch(1);

    Future<GroupHoldResult> x = executor.submit(() -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return groupService.hold(List.of("A-10", "A-11"), commandId());
    });
    Future<GroupHoldResult> y = executor.submit(() -> {
        ready.countDown();
        assertTrue(start.await(5, TimeUnit.SECONDS));
        return groupService.hold(List.of("A-11", "A-10"), commandId());
    });

    assertTrue(ready.await(5, TimeUnit.SECONDS));
    start.countDown();

    List<GroupHoldResult> results = List.of(
            x.get(10, TimeUnit.SECONDS),
            y.get(10, TimeUnit.SECONDS)
    );

    assertEquals(1, results.stream().filter(GroupHoldResult::held).count());
    assertEquals(1,
            results.stream().filter(GroupHoldResult::unavailable).count());
    assertEquals(2, activeHoldCountForSeats("A-10", "A-11"));
    assertEquals(0, deadlockCountForTestRun());
}
```

Bên cạnh Assertions, cần phân tích cấu trúc SQL để kiểm định mệnh đề `ORDER BY show_id, seat_no FOR UPDATE` được áp dụng chuẩn xác.

## 10. Thí Nghiệm 8 — Xử Lý Yêu Cầu Trùng Lặp (Idempotency)

```java
@Test
void sameCommandReplaysCommittedHold() {
    UUID commandId = UUID.randomUUID();
    HoldSeatCommand first = command(commandId, 501, "A-10");

    HoldResult accepted = coordinator.hold(first);
    HoldResult replayed = coordinator.hold(first);

    assertTrue(accepted.held());
    assertTrue(replayed.replayed());
    assertEquals(accepted.holdId(), replayed.holdId());
    assertEquals(1, holdCountByCommand(commandId));
    assertEquals(1, activeHoldCount(42, "A-10"));
}

@Test
void reusedCommandWithDifferentCustomerIsRejected() {
    UUID commandId = UUID.randomUUID();
    coordinator.hold(command(commandId, 501, "A-10"));

    assertThrows(
            IdempotencyMismatchException.class,
            () -> coordinator.hold(command(commandId, 902, "A-10"))
    );
    assertEquals(1, holdCountByCommand(commandId));
}
```

Cơ chế Idempotency hoạt động độc lập và bổ trợ cho Khóa Dòng vật lý tại hệ thống Cơ sở dữ liệu.

## 11. Cấu Trúc Khối Truy Vấn JDBC Cốt Lõi

```java
private ShowSeatRow lockSeat(long showId, String seatNo) {
    return jdbc.queryForObject("""
            select show_id, seat_no, state, hold_id, holder_customer_id
            from show_seat
            where show_id = ? and seat_no = ?
            for update
            """, SHOW_SEAT_ROW_MAPPER, showId, seatNo);
}
```

Truy vấn này buộc phải kích hoạt ngoại lệ nếu bản ghi tham chiếu không tồn tại, bởi việc tác động vào tập rỗng (0 rows) sẽ không thiết lập bất kỳ Khóa bảo vệ nào.

## 12. Ma Trận Đánh Giá Mức Độ Bao Phủ (Coverage Matrix)

| Kịch Bản Mô Phỏng | Điều Kiện Thiếu Khuyết | Kết Quả Đo Lường |
| --- | --- | --- |
| 1 | Truy xuất thiếu Khóa | Tái hiện thành công lỗi Double booking |
| 2 | Chờ phân giải trạng thái | 1 Luồng thành công, 1 Luồng từ chối hợp lệ |
| 3 | Hủy Giao dịch trước khi cấp khóa | Giao dịch chờ giành quyền kiểm soát |
| 4 | Cạnh tranh vượt thời hạn chờ | Kích hoạt ngoại lệ `55P03`, hủy an toàn |
| 5 | Truy vấn hiển thị | Không bị chặn, trả về trạng thái Snapshot cũ |
| 6 | Giao tiếp tầng JPA | Kiểm soát độc quyền luồng xử lý |
| 7 | Cấp phát đa tài nguyên vô trật tự | Sắp xếp mảng tham số ngăn ngừa rủi ro Deadlock |
| 8 | Lặp luồng Request | Xử lý chống trùng (Idempotency), loại trừ rác dữ liệu |

## 13. Khuyến Nghị Phân Tích Môi Trường Khai Thác (Production Monitoring)

Hệ thống nên tích hợp Dashboard theo dõi cấu trúc chờ (Wait-state monitoring) thông qua công cụ nội tại của PostgreSQL:

```sql
select application_name,
       state,
       wait_event_type,
       wait_event,
       pg_blocking_pids(pid) as blockers,
       clock_timestamp() - xact_start as transaction_age,
       query
from pg_stat_activity
where datname = current_database()
  and state <> 'idle';
```

Bộ giám sát hiệu năng cần phân tích:
- Sự hiện diện của mệnh đề `FOR UPDATE` trong Log truy vấn ORM.
- Tần suất ném mã ngoại lệ `55P03` (Timeout) và `40P01` (Deadlock).
- Tình trạng nghẽn cổ chai (Exhaustion) của Connection Pool.
- Tỷ lệ từ chối nghiệp vụ (Ví dụ: `ALREADY_HELD`) để đo lường độ phổ biến của tài nguyên (Điểm nóng tranh chấp).

Lưu ý: Bảng giám sát không được lưu trữ hoặc ghi Log các thông tin nhạy cảm.

## 14. Bộ Quy Tắc Ổn Định Môi Trường Kiểm Thử (Anti-Flaky Guidelines)

- Sử dụng rào chắn luồng như `CountDownLatch` để tổ chức và đồng bộ các pha kiểm thử, thay cho lệnh `Thread.sleep` phụ thuộc chu kỳ phần cứng.
- Truy vấn khẳng định trạng thái `wait_event_type = 'Lock'` trong CSDL trước khi tiếp tục.
- Thiết lập giới hạn thời gian (Timeout bound) cho mọi lệnh chờ.
- Đảm bảo giải phóng tài nguyên hệ thống (Latch, Executor) trong khối `finally` của phương thức vòng đời.
- Định danh bộ tham số Test hoàn toàn độc lập (Unique Seed Data) để hạn chế nhiễm chéo ngữ cảnh.
- Tránh việc áp dụng lặp vòng tuần hoàn (Loop-retry assert) để thay thế tính xác định logic (Deterministic rules).
