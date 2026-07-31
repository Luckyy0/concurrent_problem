# Các thí nghiệm Kiểm chứng Vòng lặp Thử lại (Bounded Retry)

## Mục tiêu của chúng ta là gì?

- Ép nhiều luồng (actors) cùng tải chung 1 phiên bản dữ liệu ở cùng 1 tích tắc.
- Chứng minh rằng khi có đụng độ (conflicts), dữ liệu sẽ bị Hủy (rollback) và vòng lặp thử lại (retries) sẽ dùng ID Giao dịch hoàn toàn MỚI.
- Đảm bảo Điểm tổng cuối cùng bằng chính xác tổng các lệnh; và bảng lịch sử (credit records) không bị đẻ ra dòng rác (duplicate).
- Xác nhận hệ thống sẽ Tự động Dừng lại khi chạm ngưỡng giới hạn số lần thử (attempt cap) hoặc vượt quá thời gian tối đa (deadline).
- Kiểm tra tính năng "Ngủ ngẫu nhiên" (Backoff/jitter) hoạt động chuẩn xác mà không cần dùng lệnh `Thread.sleep` cứng ngắc.
- Đo lường mức độ "Bão": số lệnh ban đầu, số lần phải thử lại, số lần đụng độ thực tế.

> **Nói ngắn gọn:** Thấy số dư cuối cùng đúng là CHƯA ĐỦ. Vòng lặp thử lại có thể cho kết quả dữ liệu đúng, nhưng nó sẽ "vắt kiệt" hệ thống (phá capacity) nếu bạn không kiểm soát số lần đập đầu húc tường của nó.

## Khởi tạo Môi trường xịn với PostgreSQL Testcontainers

```java
@Testcontainers
@SpringBootTest
class OptimisticRetryIT {

    @Container
    @ServiceConnection
    // Dùng đồ xịn PostgreSQL thật thay vì H2 ảo
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    RewardCreditCoordinator coordinator;

    @Autowired
    JdbcTemplate jdbc;

    // Hồ bơi chứa 8 thợ xây cùng lúc
    private final ExecutorService executor = Executors.newFixedThreadPool(8);

    @BeforeEach
    void seed() {
        jdbc.update("delete from reward_credit");
        jdbc.update("delete from reward_wallet");
        // Khởi tạo ví 77 với 100 điểm, version đang là 10
        jdbc.update("""
                insert into reward_wallet(wallet_id, points, version, active)
                values (77, 100, 10, true)
                """);
    }

    @AfterEach
    void shutdown() throws InterruptedException {
        executor.shutdownNow();
        assertThat(executor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Nhớ nhé, method dùng để viết Test tuyệt đối **KHÔNG CÓ** `@Transactional` bên trên, để hệ thống tự quản lý ranh giới thật sự của nó.

## Cánh cổng Ma Thuật - Ép chúng nó chạy cùng nhịp (Deterministic first-attempt gate)

Chúng ta làm một cái barie (chỉ dành cho Test) ngay trong vòng thử đầu tiên, ngay sau khi tụi nó tải cái Ví lên:

```java
final class FirstAttemptGate {
    private final CountDownLatch allLoaded;
    private final CountDownLatch continueFlush = new CountDownLatch(1);
    private final ConcurrentMap<UUID, AtomicInteger> attempts =
            new ConcurrentHashMap<>();

    FirstAttemptGate(int actors) {
        this.allLoaded = new CountDownLatch(actors);
    }

    void afterLoad(UUID commandId) {
        int number = attempts.computeIfAbsent(
                commandId,
                ignored -> new AtomicInteger()
        ).incrementAndGet();
        if (number == 1) { // Lần thử đầu tiên
            allLoaded.countDown();
            // Đứng đây đợi, không cho thằng nào nhảy qua Flush
            await(continueFlush, "release first-attempt flushes"); 
        }
    }

    void awaitAllAndRelease() {
        // Chủ mưu đứng ngoài đợi 8 thằng tải xong v10
        await(allLoaded, "all first attempts loaded same version");
        // Giật chốt cho 8 thằng cùng lúc nhảy vào tranh nhau Flush!
        continueFlush.countDown(); 
    }
}

// Hàm hỗ trợ chờ đợi có mốc thời gian an toàn
static void await(CountDownLatch latch, String description) {
    try {
        if (!latch.await(5, TimeUnit.SECONDS)) {
            throw new AssertionError("Timed out: " + description);
        }
    } catch (InterruptedException interrupted) {
        Thread.currentThread().interrupt();
        throw new AssertionError("Interrupted: " + description, interrupted);
    }
}
```

Ở môi trường Production thực tế, cái Barie này chỉ là bù nhìn (no-op). Nhờ có trò này, chúng ta đo lường được `txid_current()` (ID Giao dịch), version và kết quả thực sự.

## Thí nghiệm 1 — Trận hỗn chiến: 8 Lệnh cùng cộng điểm qua các lần Thử lại sạch (fresh retries)

```java
@Test
void uniqueCommandsEventuallyCommitWithoutLostOrDuplicatePoints()
        throws Exception {
    int actors = 8;
    // Bơm 8 lệnh cộng 10 điểm vào chung 1 cái ví (ID 77)
    List<CreditCommand> commands = IntStream.range(0, actors)
            .mapToObj(index -> new CreditCommand(
                    UUID.randomUUID(), 77, 10
            ))
            .toList();

    List<Future<CreditResult>> futures = commands.stream()
            .map(command -> executor.submit(() ->
                    coordinator.credit(
                            command,
                            Instant.now().plusSeconds(5) // Hẹn deadline 5s
                    )
            ))
            .toList();

    gate.awaitAllAndRelease(); // Giật chốt hỗn chiến!
    
    for (Future<CreditResult> future : futures) {
        // Ép tất cả đều phải Thành Công!
        assertThat(future.get(10, TimeUnit.SECONDS).committed()).isTrue();
    }

    // Xác nhận kết quả
    assertThat(points()).isEqualTo(180); // 100 + 8*10
    assertThat(version()).isEqualTo(18); // 10 + 8
    assertThat(creditCount()).isEqualTo(8); // Phải có 8 dòng lịch sử
    assertThat(distinctCommandCount()).isEqualTo(8);
    // Số lần thử húc tường chắc chắn phải >= 8
    assertThat(probe.totalAttempts()).isGreaterThanOrEqualTo(8); 
    // Chắc chắn phải có đụng độ vỡ đầu
    assertThat(probe.optimisticConflicts()).isGreaterThan(0);
    // Mỗi lần thử phải sinh ra 1 Mã Giao dịch mới tinh
    assertThat(probe.transactionIds()).doesNotHaveDuplicates();
}
```

Ở đây chúng ta cấp cho tụi nó 1 cái ngân sách retry (retry budget) đủ lớn để yên tâm là đằng nào cũng chốt sổ hết.

## Thí nghiệm 2 — Khi Thử Lại (Retry), nó sẽ đọc lại Dữ liệu của Kẻ Thắng Cuộc (reloads winner state)

Hai thằng A và B cùng tải lên Version 10. Thằng A thắng, cộng thành 110 điểm / Bản v11. Thằng B ở lần húc đầu tiên bị đánh gục; Lúc thức dậy ở lần thứ hai, thằng B phải đọc lên được `110 điểm`, bản `v11`, rồi từ đó mới cộng lên `130 điểm / v12`.

```java
assertThat(probe.loads(commandB))
        .containsExactly(
                new LoadObservation(100, 10), // Đọc lần đầu
                new LoadObservation(110, 11)  // Tỉnh dậy đọc lần hai
        );
assertThat(probe.transactionIds(commandB))
        .hasSize(2) // Tạo 2 Giao dịch khác nhau
        .doesNotHaveDuplicates();
assertThat(recordingWaiter.invocations(commandB)).isEqualTo(1); // Mới chỉ chờ 1 lần
```

Đây là bằng chứng rành rành cho thấy Giao dịch mới được tạo (bộ đệm sạch sẽ) chứ không phải chỉ nhét khối `catch` bắt Exception.

## Thí nghiệm 3 — Đụng rào cản Số Lần (Attempt cap) dẫn đến Kiệt Sức

Dùng Test double để ép Lính Đánh Thuê luôn văng lỗi đụng độ `ObjectOptimisticLockingFailureException`. Dù ngân sách cho Thời Gian (deadline) còn dài dằng dặc, nhưng giới hạn lần thử là `maxAttempts=3`:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline))
        .isInstanceOf(WalletContentionException.class); // Văng lỗi Bận
assertThat(attempts.invocations()).isEqualTo(3); // Gọi húc đầu 3 lần
assertThat(recordingWaiter.invocations()).isEqualTo(2); // Chỉ ngủ 2 lần (lần cuối văng luôn)
// Không ngâm Giao dịch nào!
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isFalse();
```

Sau lần thử cuối cùng thì cút luôn chứ không có đi ngủ vớ vẩn nữa.

## Thí nghiệm 4 — Quá Hạn (Deadline) cướp cờ dù chưa hết Số Lần (Attempt)

Bơm 1 cái Đồng hồ giả (Fake `Clock`). Ngay sau lần đụng độ đầu tiên, vặn đồng hồ chạy vèo qua Deadline luôn:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline))
        .isInstanceOf(WalletContentionException.class);
assertThat(attempts.invocations()).isEqualTo(1); // Chỉ mới húc 1 lần
// Thời gian ngủ phải bị CẮT BỚT không cho lố Deadline
assertThat(recordingWaiter.totalRequestedDelay())
        .isLessThanOrEqualTo(Duration.between(start, deadline));
```

Test này chạy cực chuẩn, không cần quan tâm đến thời gian máy chủ thực tế trôi qua bao nhiêu (wall clock).

## Thí nghiệm 5 — Chờ Đợi và Độ Lệch Ngẫu Nhiên (Backoff và jitter bounds)

Bơm một bộ sinh Số Ngẫu Nhiên giả `JitterSource` luôn trả về con số Min hoặc Max:

```java
// Nếu độ lệch bằng 0 (zero jitter), lần ngủ đầu tiên tốn 20ms
assertThat(planWithZeroJitter.delayAfter(1))
        .isEqualTo(Duration.ofMillis(20));
        
// Nếu độ lệch lớn nhất, lần ngủ tốn ngẫu nhiên từ 20ms đến 30ms
assertThat(planWithMaxJitter.delayAfter(1))
        .isBetween(
                Duration.ofMillis(20),
                Duration.ofMillis(30)
        );
        
// Dù có ngủ ở lần thứ 20 thì không bao giờ được phép quá 200ms
assertThat(plan.delayAfter(20))
        .isLessThanOrEqualTo(Duration.ofMillis(200));
```

Việc test này là cực kỳ khoa học (Property test), giúp kiểm tra xem thời gian tính toán ra không bị âm, luôn tăng (nondecreasing base) và không vượt quá Trần tối đa. Tuyệt đối không dùng `Thread.sleep` mù quáng để test thời gian chờ.

## Thí nghiệm 6 — Khách bấm gửi 2 lần (Same command replay)

```java
CreditResult first = coordinator.credit(command, deadline());
long pointsAfterFirst = points();
long versionAfterFirst = version();

// Chơi ác, gọi lại lần nữa Y CHANG như cũ
CreditResult replay = coordinator.credit(command, deadline());

assertThat(first.applied()).isTrue(); // Lần đầu áp dụng
assertThat(replay.replayed()).isTrue(); // Lần sau chỉ "nhại lại kết quả"
assertThat(points()).isEqualTo(pointsAfterFirst); // Điểm không tăng
assertThat(version()).isEqualTo(versionAfterFirst); // Version không nhảy
assertThat(creditCountFor(command.commandId())).isEqualTo(1); // Chỉ có 1 dòng Lịch sử
```

Nếu chạy đa luồng (Concurrent same-ID), thì luồng đụng độ bị hủy (rollback), lần húc tiếp theo nó cũng móc kết quả ra (fresh replay) chứ tuyệt đối không đẻ ra lịch sử hay tự cộng tiền vào ví.

## Thí nghiệm 7 — Lỗi Luật kinh doanh (Business rejection) thì Cấm Thử Lại

Phá hoại bằng cách gán cái ví đó `active=false`. Khi Lính Đánh Thuê vào làm việc sẽ ném ra lỗi Cấm Cửa:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline()))
        .isInstanceOf(WalletSuspendedException.class); // Ví đã bị phong ấn!
assertThat(attempts.invocations()).isEqualTo(1); // Thử 1 lần
assertThat(recordingWaiter.invocations()).isZero(); // Đéo cho đi ngủ chờ
assertThat(points()).isEqualTo(100); // Dữ liệu còn nguyên
```

## Thí nghiệm 8 — Đi ngủ (Backoff) không được ôm chung chăn (Transaction)

Cái máy đo thời gian ngủ sẽ kiểm chứng thẳng ở mỗi lần kêu gọi:

```java
// Đang ngủ thì Giao dịch PHẢI ĐÃ BỊ CẮT!
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isFalse();
assertThat(probe.previousOutcome()).isEqualTo("ROLLED_BACK");
```

Phải dùng Metric của hồ bơi hoặc Test probe để chứng minh rành rành rằng Connection đã được chui tọt vào lại Hồ (pool) TRƯỚC KHI đi ngủ. Đừng có nhìn vào mấy cái chữ `@Annotation` trên đầu Hàm rồi tưởng bở.

## Thí nghiệm 9 — Đặt Giao dịch Bọc Ở Vòng Ngoài (Outer transaction guard)

Nếu có một Class cẩu thả nào đó tự tiện chêm `@Transactional` từ tít bên ngoài rồi gọi Class Chỉ Huy (coordinator), thì Cửa gác của mình (Guard) phải đạp văng ra ngay trước khi nó kịp đi lính:

```java
// Class ngoài vi phạm ranh giới
assertThatThrownBy(() -> transactionalCaller.call(command))
        .isInstanceOf(IllegalStateException.class);
assertThat(attempts.invocations()).isZero(); // Chưa làm ăn gì cả
```

## Bảng Ma Trận Tóm tắt Các Tình Huống

| Tên Test | An Toàn (Safety) - Có chuẩn không? | Khả năng sống sót (Liveness/capacity) |
| --- | --- | --- |
| 8 luồng tranh giành (8 unique commands) | Điểm/Lịch sử/Version chuẩn 100% | Số lần húc và đụng độ phải Bị Giới Hạn |
| Đọc lại khi có 2 luồng đâm nhau | Dữ liệu/Bản version sạch sẽ (Fresh) | Ngủ (Backoff) 1 lần |
| Chạm giới hạn lần húc (Attempt cap) | Tuyệt đối không Ghi dữ liệu nửa vời (partial write) | Đúng 3 lần là Báo Cáo "Kiệt sức" |
| Hết Giờ (Deadline) | Không thử dở dang giữa chừng | Dừng ngay lập tức |
| Ngẫu Nhiên (Jitter) | N/A | Đo nằm trong ngưỡng Giới Hạn Delay |
| Trùng lặp mã Lệnh (Replay) | 1 dòng điểm delta/ 1 dòng lịch sử | Kết quả trả ra nhanh và ổn định |
| Lỗi Cấm Cửa (Rejection) | Dữ liệu bất di bất dịch | KHÔNG ĐƯỢC THỬ LẠI |
| Ranh giới của việc chờ (Waiter boundary) | Hủy bỏ trọn vẹn | Không ngâm kết nối Database |

## Bí quyết chống Code Chập chờn (Flaky) và Bài học Lên Production

- Chỉ dùng Barie (Gate) giữ chân luồng ở lần tải dữ liệu đầu tiên, còn mấy lần thử sau cho chạy đua thả ga.
- Bất cứ Cờ chốt (Latch) hoặc Biến Tương lai (Future) nào cũng phải Gắn Hạn Giờ (bounded). Nói không với ngủ ngu ngốc `Thread.sleep`.
- Xài Đồng hồ giả / Độ lệch giả / Kẻ chờ giả (Recording clock/jitter/waiter) để kiểm chứng rạch ròi bộ Luật (Policy).
- Dùng hàng thật PostgreSQL Testcontainers để phân định ranh giới Version/Khóa duy nhất/Transaction.
- Đừng có bắt Assert thằng nào thắng, cũng đừng ép con số Lần Thử quá khắt khe sau khi chạy Barie (môi trường OS không biết trước được).
- Gắn thêm lệnh vớt rác (bắt SQL/txid/probe) vào lúc cái Lính Canh Chó (watchdog) kêu gào vì Fail Test.
- Khi đưa lên Production (Môi trường Thật): Bắt buộc Giám Sát chỉ số Lệnh (commands), Số Lần Đâm (attempts), Đụng độ, Đồ Thị số lần húc Thành công, Hết giờ/Kiệt sức, Ngủ (backoff), Độ trễ (latency), Mấy cha nội hối bấm nhiều lần (duplicate replay), và đống rác đang kẹt trong Hồ Bơi (pool pending).
- Một khi thấy Tỷ lệ húc thành công tăng cao bất thường (báo động hot wallet kéo dài), Mời anh em xách balo nhảy sang học bài `LOCK-005` (atomic delta, queue, pessimistic) chứ đừng nhắm mắt đưa chân đi TĂNG cái maxAttempts lên nữa nhé!
