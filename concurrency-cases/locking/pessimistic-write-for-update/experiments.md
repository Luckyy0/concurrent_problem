# Phòng Thí Nghiệm: Ép Trái Tim Database Bằng Testcontainers (Pessimistic Lock)

## 1. Mục tiêu thí nghiệm

Bài test này không phải viết cho vui, nó phải chứng minh được:

1. Đọc chay (Plain SELECT) làm 2 thằng cùng tưởng ghế trống.
2. Đọc có Khóa (`FOR UPDATE`) làm thằng đến sau bị block (chặn họng) và bắt buộc phải đọc lại dữ liệu mới nhất.
3. Kẻ trước Hủy kèo (Rollback) thì trả Khóa, thằng đến sau tha hồ chốt đơn.
4. Chờ quá giờ (`lock_timeout`) thì văng lỗi, không có chuyện lưu được 1 nửa.
5. Ai đi ngang qua xem ghế (Plain reader) thì vẫn thấy dữ liệu cũ, chả bị Khóa chặn lại làm gì.
6. Mua nhiều ghế phải xếp hàng (Stable order) để tránh đụng độ (Deadlock).
7. Đoạn code Spring/JPA của mình chạy ngon lành trên PostgreSQL hàng real!

Tất nhiên: TUYỆT ĐỐI KHÔNG xài H2 (cái Database đồ chơi chạy trên RAM) để giả lập mấy trò Khóa Dòng này. H2 và Postgres xử lý Khóa khác hẳn nhau!

## 2. Bày Biện Bàn Chơi (Testcontainers fixture)

Đầu tiên, kéo một Docker Image PostgreSQL xịn xò về:

```java
@Testcontainers
@SpringBootTest
class PessimisticSeatHoldIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:17-alpine"); // Postgres xịn đây!

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

    private ExecutorService executor; // Lò đẻ luồng (Thread)

    @BeforeEach
    void reset() {
        executor = Executors.newFixedThreadPool(4);
        jdbc.update("delete from seat_hold"); // Dọn rác
        // Trả 2 ghế A-10 và A-11 về trạng thái TRỐNG
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
        executor.shutdownNow(); // Test xong thì đập bỏ lò đẻ luồng
        assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Migration thật tạo hai seat rows cố định và các constraints trong `solutions.md`. Mỗi test dùng command IDs mới để không vô tình đi vào replay path.

## 3. Trợ thủ Đắc Lực (Đồng hồ đếm ngược và Mắt thần)

Để điều khiển các luồng (Thread) chạy dừng đúng ý đồ như đạo diễn phim, ta xài `CountDownLatch`. Để chắc chắn 1 luồng ĐANG BỊ KẸT KHÓA dưới DB, ta xài con mắt thần này:

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
        // Soi thẳng vào hệ thống nội tạng của PostgreSQL
        Boolean blocked = jdbc.queryForObject("""
                select exists (
                    select 1
                    from pg_stat_activity
                    where application_name = ?
                      and wait_event_type = 'Lock' /* Bị kẹt Khóa! */
                      and cardinality(pg_blocking_pids(pid)) > 0
                )
                """, Boolean.class, applicationName);
        if (Boolean.TRUE.equals(blocked)) {
            return; // Kẹt thật rồi! Thỏa mãn!
        }

        LockSupport.parkNanos(Duration.ofMillis(10).toNanos()); // Ngủ 10ms rồi soi tiếp
        if (Thread.currentThread().isInterrupted()) {
            throw new AssertionError("interrupted while observing lock wait");
        }
    }
    fail("Chờ 5 giây rồi mà chả thấy nó bị kẹt Khóa gì cả!");
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

Mọi `Future.get`, latch wait và executor shutdown đều có upper bound. Quá giờ là ném lỗi chứ không để treo máy.

## 4. Thí nghiệm 1: Chứng minh Code Ngu (Tái hiện stale decision)

Ta chế một cái "Cổng rào" (Barrier). Bắt 2 Luồng nhào vô đọc ghế, đọc xong KHÔNG CHO LƯU, bắt đứng đó đợi nhau. Khi cả 2 đều đã đọc xong (đều thấy Ghế Trống), ta mới thả cửa cho tụi nó ùa vào Lưu.

```java
@Service
class BarrierBrokenSeatHoldTx {
    private final ShowSeatRepository seats;
    private final SeatHoldRepository holds;
    private final CyclicBarrier bothLoaded = new CyclicBarrier(2);

    // Constructor injection omitted.

    @Transactional
    public UUID hold(HoldSeatCommand command) {
        ShowSeat seat = seats.findById(command.seatId()).orElseThrow();
        assertEquals(SeatState.AVAILABLE, seat.state());

        try {
            bothLoaded.await(5, TimeUnit.SECONDS); // 2 anh em đợi nhau ở đây nhé
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
    // Experiment này chạy legacy schema chưa có uq_seat_hold_one_active.
    Future<UUID> a = executor.submit(
            () -> brokenWorker.hold(command("hold-a", 501))
    );
    Future<UUID> b = executor.submit(
            () -> brokenWorker.hold(command("hold-b", 902))
    );

    UUID holdA = a.get(10, TimeUnit.SECONDS);
    UUID holdB = b.get(10, TimeUnit.SECONDS);

    // KẾT QUẢ KINH HOÀNG:
    assertNotEquals(holdA, holdB); // Đẻ ra 2 cái vé khác nhau!
    assertEquals(2, activeHoldCount(42, "A-10")); // 1 ghế mà 2 thằng ACTIVE
    assertEquals(1, seatProjectionCount(42, "A-10"));
    assertTrue(currentSeatHoldId(42, "A-10").equals(holdA)
            || currentSeatHoldId(42, "A-10").equals(holdB));
}
```

Trọng tâm là bắt được cái Lỗi Chết Người kia (2 vé ACTIVE), chứ không phải đơn thuần là "code chạy lỗi vặt".

## 5. Thí nghiệm 2: Người đi sau phải Kính Trọng người đi trước (Revalidate)

Cho A vào Xin Khóa trước. Xong bắt A dừng lại uống trà (chưa Commit).
Thả B vào. B đòi xin Khóa. B sẽ BỊ ĐÁ BẬT RA HÀNG CHỜ.
Sau khi soi DB thấy B "đang đứng chờ" thật, ta mới kêu A: "Commit đi!".
A commit xong, B vội vàng lao vào lụm Khóa. Nhưng ôi thôi, kết quả lúc này là B báo lỗi vì ghế đã HELD!

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

    awaitDatabaseBlock("lock003-waiter"); // Soi thấy B há mồm chờ dưới DB
    assertThrows(
            TimeoutException.class,
            () -> waiter.get(100, TimeUnit.MILLISECONDS)
    );

    allowHolderCommit.countDown(); // A húp trọn và nhả Khóa

    assertEquals(HoldOutcome.HELD,
            holder.get(5, TimeUnit.SECONDS)); // A thành công
    assertEquals(HoldOutcome.ALREADY_HELD,
            waiter.get(5, TimeUnit.SECONDS)); // B khóc ròng
    assertEquals(1, activeHoldCount(42, "A-10"));
    assertEquals(501L, currentSeatCustomer(42, "A-10")); // Khách A (501) lấy được ghế
}
```

> **Nói ngắn gọn:** Test này chọc thẳng xuống DB chứng minh B thực sự bị Block, và B khôn ngoan đọc lại Dữ liệu Mới để tự dội gáo nước lạnh vào mặt chứ không chèn đè lên A.

## 6. Thí nghiệm 3: Kẻ Hủy Kèo Bỏ Băn Khoăn (Rollback)

Giống hệt Thí nghiệm 2, nhưng thay vì A Commit, ta bắt A tung Exception để Rollback!
Lúc này B lụm Khóa, thấy ghế vẫn `AVAILABLE` -> B húp trọn!

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
        throw new RollbackForTest(); // Giữa chừng A ném vỡ cái ly (Rollback)
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
            waiter.get(5, TimeUnit.SECONDS)); // Lần này B thành công!
    assertEquals(1, activeHoldCount(42, "A-10"));
    assertEquals(902L, currentSeatCustomer(42, "A-10")); // Khách B (902) lên ngôi
    assertEquals(0, holdCountByCommand("hold-a"));
}
```

## 7. Thí nghiệm 4: Chờ lâu quá thì Giải Tán (`lock_timeout`)

Ta ép thằng B chỉ được chờ 150 mili-giây.
Ta cố tình cho thằng A cầm Khóa đi dạo lâu hơn thế.
Băng B đứt bóng ngay lập tức với mã lỗi `55P03`.

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
                // B chạy vào và dặn DB: "Chỉ chờ 150ms thôi nghen!"
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
    assertEquals("55P03", sqlState(failed)); // Lỗi Không Lấy Được Khóa
    assertEquals(0, holdCountByCommand("hold-b"));
    assertEquals("AVAILABLE", currentSeatState(42, "A-10")); // Ghế vẫn an toàn

    releaseHolder.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
}
```

Test phải release holder trong `finally` ở implementation đầy đủ để failure của một assertion không để task treo. Sau timeout, framework rollback transaction trước một retry/mapping mới.

## 8. Thí nghiệm 5: Dân Thường Vẫn Được Đi Ngang (Plain reader)

A đang xin Khóa để sửa ghế. Một câu lệnh Đọc Chay bay ngang qua. Liệu nó có bị cản? Không! Nó vẫn lấy được Dữ Liệu Cũ trước khi A chốt sổ. Đừng biến mấy API hiện danh sách chơi chơi thành bãi kẹt xe!

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
    assertEquals("AVAILABLE", reader.get(2, TimeUnit.SECONDS)); // Đọc thoải mái

    allowCommit.countDown();
    assertNull(holder.get(5, TimeUnit.SECONDS));
    assertEquals("HELD", currentSeatState(42, "A-10"));
    assertEquals(1, activeHoldCount(42, "A-10"));
}
```

## 9. Thí nghiệm 6 & 7: Cuộc Chiến Đỉnh Cao và Tránh Cắn Đuôi (Deadlock)

**Test 6:** Bắn luồng JPA y hệt Production. Bắn 2 luồng cùng lúc. Chắc chắn chỉ 1 luồng Win, luồng kia văng exception.

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

**Test 7:** Bắn Khách 1 mua `[A-10, A-11]`. Bắn Khách 2 mua `[A-11, A-10]`. Nếu code bạn xịn (có thao tác Sắp Xếp thứ tự ghế trước khi khóa), sẽ KHÔNG BAO GIỜ có Lỗi Kẹt Cứng (Deadlock). Khách 1 hoặc 2 sẽ ôm trọn 2 ghế, kẻ kia ra rìa.

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

Ngoài assertions, SQL capture phải cho thấy cả calls dùng `ORDER BY show_id, seat_no FOR UPDATE`. Không biến “không thấy deadlock trong một run” thành bằng chứng duy nhất; audit query/order là primary guarantee.

## 10. Thí nghiệm 8 — Cùng 1 Request, Gửi Đúp 2 Lần

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

Lớp áo giáp Idempotency (Chống Trùng Lặp) này độc lập với cái Khóa Dòng ở trên.

## 11. Đồ Đọc Lõi (Core JDBC helper)

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

Hàm Helper này BẮT BUỘC phải nổ lỗi (fail) nếu dòng ghế Không Tồn Tại. Nếu về 0 dòng, nghĩa là chả có cái Khóa nào được sinh ra cả!

## 12. Bảng Tóm Tắt Chiến Tích (Coverage matrix)

| Trận Đánh | Tình Cảnh Tạo Ra | Kết Quả Bắt Được |
| --- | --- | --- |
| 1 | Hai ông cùng ngó bảng mà chả thèm khóa | Lòi ra ngay vụ 1 ghế 2 chủ |
| 2 | B đứng chờ A giữ khóa | Trả về 1 HELD, một ALREADY_HELD (Chuẩn) |
| 3 | A Hủy kèo trước khi B giật khóa | Vé thuộc về B |
| 4 | B chờ lố giờ `lock_timeout` | Nổ lỗi `55P03`, chả húp được gì |
| 5 | Dân thường xem danh sách khi A chưa chốt | Vẫn đọc được bản cũ bình thường |
| 6 | 2 luồng chọc thẳng vào tầng JPA | Chỉ có 1 kẻ chiến thắng |
| 7 | Nhập lộn xộn danh sách ghế | Khóa theo ID, khỏi kẹt xe, 1 winner |
| 8 | Bấm 2 lần 1 request | Phát hiện trùng, không đẻ thêm rác |

## 13. Đem Kính Lúp Lên Production (Giám sát thật)

Khi đẩy code lên Production, hãy làm cho mình một cái Bảng Theo Dõi xịn sò bằng câu Query này:

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

Bạn sẽ dễ dàng vạch mặt kẻ nào ôm Khóa đi ngủ quá lâu, hay đếm được bao nhiêu ông khách bị văng lỗi vì xếp hàng quá giờ (`55P03`). Giám sát thêm các chỉ số:
- SQL Log xem Hibernate có tự sinh chữ `FOR UPDATE` không.
- Số lượng văng lỗi Timeout (`55P03`) và Deadlock (`40P01`).
- Số lượng Connection nằm lì chờ đợi (Pool idle/pending).
- Khách văng `ALREADY_HELD` nhiều quá chứng tỏ cái Ghế đó cực Hot.

Nhớ là: Tuyệt đối không log thông tin Thanh toán hay Số thẻ của Khách hàng vào mấy cái View theo dõi kỹ thuật này!

## 14. Bí Kíp Chống Test "Bị Sảng" (Flaky)

Mấy bài Test đa luồng rất hay bị "sảng" (lúc xanh lúc đỏ). Để trị tụi nó:
- Dùng `CountDownLatch` bắt chúng nó làm diễn viên xếp hàng chờ đạo diễn hô "Action", không phó mặc cho độ trễ hên xui của Hệ điều hành.
- Bắt buộc kiểm tra `wait_event_type = 'Lock'` trong DB trước khi sang cảnh tiếp theo.
- Mọi cái wait đều phải có GIỜ CHÓT (Timeout bound).
- Nhớ xả `Latch` và dọn dẹp Lò Đẻ Luồng trong khối `finally`, lỡ test tạch thì máy không bị treo vĩnh viễn.
- Dùng ID dữ liệu MỚI TOANH cho mỗi Test để khỏi bị dính rác của bài Test trước.
- Không dùng trò "Treo vòng lặp spam lệnh cả ngày không dính lỗi thì coi như đậu", cái đó không phải là Bằng Chứng Hợp Lệ!
