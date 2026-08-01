# Các thử nghiệm kiểm chứng vòng lặp thử lại có giới hạn

## 1. Mục tiêu kiểm thử

- Đảm bảo nhiều thread đồng thời tải chung một phiên bản dữ liệu tại cùng một thời điểm.
- Xác nhận rằng khi xảy ra xung đột, dữ liệu sẽ được rollback và vòng lặp thử lại sẽ sử dụng một định danh transaction hoàn toàn mới.
- Khẳng định tính chính xác của dữ liệu: Tổng điểm cuối cùng khớp với tổng của các lệnh hợp lệ; bảng lịch sử thao tác không chứa dữ liệu trùng lặp.
- Xác thực hệ thống tự động ngừng xử lý khi chạm giới hạn số lần thử hoặc vượt quá thời gian tối đa.
- Kiểm tra tính năng tạo độ trễ ngẫu nhiên hoạt động chuẩn xác mà không phụ thuộc vào `Thread.sleep`.
- Đo lường mức độ khuếch đại tải: So sánh giữa số lượng lệnh ban đầu, số lần thực hiện lại và tần suất xung đột thực tế.

> **Ghi chú quan trọng:** Đảm bảo trạng thái dữ liệu đầu cuối chính xác là chưa đủ. Vòng lặp thử lại có thể cung cấp dữ liệu đúng, nhưng sẽ làm hệ thống cạn kiệt tài nguyên nếu không kiểm soát chặt chẽ số lần thử lại tối đa.

## 2. Thiết lập môi trường bằng PostgreSQL Testcontainers

```java
@Testcontainers
@SpringBootTest
class OptimisticRetryIT {

    @Container
    @ServiceConnection
    // Yêu cầu sử dụng database PostgreSQL thay thế cho H2 trong kiểm thử
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @Autowired
    RewardCreditCoordinator coordinator;

    @Autowired
    JdbcTemplate jdbc;

    // Khởi tạo thread pool với 8 thread đồng thời
    private final ExecutorService executor = Executors.newFixedThreadPool(8);

    @BeforeEach
    void seed() {
        jdbc.update("delete from reward_credit");
        jdbc.update("delete from reward_wallet");
        // Thiết lập ví 77 có 100 điểm, phiên bản khởi tạo là 10
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

Lưu ý: Phương thức thực thi test tuyệt đối **KHÔNG CÓ** annotation `@Transactional` nhằm đảm bảo hệ thống tự quản lý ranh giới transaction một cách thực tế.

## 3. Đồng bộ hóa cạnh tranh

Thiết kế một cơ chế rào chắn dành riêng cho kiểm thử để đồng bộ các thread ở lần tải dữ liệu đầu tiên:

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
        if (number == 1) { // Phát hiện lần tải đầu tiên
            allLoaded.countDown();
            // Đóng băng tiến trình chờ xử lý đồng bộ
            await(continueFlush, "release first-attempt flushes"); 
        }
    }

    void awaitAllAndRelease() {
        // Đợi tất cả 8 thread tải xong phiên bản dữ liệu (phiên bản 10)
        await(allLoaded, "all first attempts loaded same version");
        // Cấp quyền cho 8 thread cùng thực thi đồng bộ đồng thời
        continueFlush.countDown(); 
    }
}

// Phương thức hỗ trợ chờ có áp đặt timeout an toàn
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

Trong môi trường thực tế, cơ chế rào chắn này không ảnh hưởng đến logic. Kỹ thuật này giúp phân tích chỉ số ID của transaction, phiên bản và kết quả transaction.

## 4. Thử nghiệm 1 — Độ phủ hoàn trọn

```java
@Test
void uniqueCommandsEventuallyCommitWithoutLostOrDuplicatePoints()
        throws Exception {
    int actors = 8;
    // Khởi tạo 8 lệnh cộng 10 điểm vào ví 77
    List<CreditCommand> commands = IntStream.range(0, actors)
            .mapToObj(index -> new CreditCommand(
                    UUID.randomUUID(), 77, 10
            ))
            .toList();

    List<Future<CreditResult>> futures = commands.stream()
            .map(command -> executor.submit(() ->
                    coordinator.credit(
                            command,
                            Instant.now().plusSeconds(5) // Thời hạn xử lý 5 giây
                    )
            ))
            .toList();

    gate.awaitAllAndRelease(); // Kích hoạt sự kiện tương tranh
    
    for (Future<CreditResult> future : futures) {
        // Kiểm chứng tính hoàn thiện của transaction
        assertThat(future.get(10, TimeUnit.SECONDS).committed()).isTrue();
    }

    // Đánh giá dữ liệu tổng
    assertThat(points()).isEqualTo(180); // 100 + (8 * 10)
    assertThat(version()).isEqualTo(18); // 10 + 8 lần cập nhật
    assertThat(creditCount()).isEqualTo(8); // Tồn tại 8 dòng lịch sử độc lập
    assertThat(distinctCommandCount()).isEqualTo(8);
    // Đo lường tần suất tải trọng khuếch đại
    assertThat(probe.totalAttempts()).isGreaterThanOrEqualTo(8); 
    // Xác nhận xảy ra xung đột thực tế
    assertThat(probe.optimisticConflicts()).isGreaterThan(0);
    // Tính độc lập của mỗi ID transaction
    assertThat(probe.transactionIds()).doesNotHaveDuplicates();
}
```

Ở bài test này, việc cấu hình hạn mức thời gian và số lần đủ rộng nhằm đảm bảo tiến trình thành công.

## 5. Thử nghiệm 2 — Cập nhật phiên bản

Khi thread A và B cùng tải phiên bản 10. Nếu A commit thành công, phiên bản cập nhật là `v11` với số điểm `110`. Thread B phải đọc lại `110` điểm tại `v11` thay vì thao tác trên đối tượng bộ đệm cũ, sau đó cập nhật thành `130` điểm tại `v12`.

```java
assertThat(probe.loads(commandB))
        .containsExactly(
                new LoadObservation(100, 10), // Trạng thái ban đầu
                new LoadObservation(110, 11)  // Trạng thái được cập nhật
        );
assertThat(probe.transactionIds(commandB))
        .hasSize(2) // Transaction được cấp mới
        .doesNotHaveDuplicates();
assertThat(recordingWaiter.invocations(commandB)).isEqualTo(1); // Yêu cầu chờ 1 lần
```

Kiểm tra này khẳng định transaction mới được tạo thành công và dữ liệu đệm cũ bị xóa sạch.

## 6. Thử nghiệm 3 — Xử lý lỗi cạn kiệt nỗ lực

Mô phỏng phương thức thực thi cố định ném ra ngoại lệ `ObjectOptimisticLockingFailureException`. Nếu cấu hình giới hạn số lần thử là 3:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline))
        .isInstanceOf(WalletContentionException.class); // Trả về lỗi hệ thống
assertThat(attempts.invocations()).isEqualTo(3); // Giới hạn đúng 3 lần thực thi
assertThat(recordingWaiter.invocations()).isEqualTo(2); // Thời gian chờ thực thi 2 lần
// Khẳng định tài nguyên transaction đã được thu hồi
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isFalse();
```

Hệ thống kết thúc dứt điểm và không sinh độ trễ vô ích.

## 7. Thử nghiệm 4 — Ràng buộc hết thời gian chờ

Sử dụng đồng hồ mô phỏng. Sau khi phát hiện lần xung đột đầu tiên, vặn đồng hồ vượt qua thời hạn:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline))
        .isInstanceOf(WalletContentionException.class);
assertThat(attempts.invocations()).isEqualTo(1); // Tiến trình bị cắt ngang
// Đảm bảo thời gian chờ không vượt qua thời hạn còn lại
assertThat(recordingWaiter.totalRequestedDelay())
        .isLessThanOrEqualTo(Duration.between(start, deadline));
```

Thiết kế kiểm thử độc lập với thời gian thực tế của máy chủ.

## 8. Thử nghiệm 5 — Tính toán độ trễ và lệch hướng

Sử dụng đối tượng mô phỏng trả về các giá trị tối thiểu và tối đa:

```java
// Trường hợp không có lệch hướng
assertThat(planWithZeroJitter.delayAfter(1))
        .isEqualTo(Duration.ofMillis(20));
        
// Trường hợp lệch hướng tối đa
assertThat(planWithMaxJitter.delayAfter(1))
        .isBetween(
                Duration.ofMillis(20),
                Duration.ofMillis(30)
        );
        
// Kiểm tra trần tối đa của hệ số lũy thừa
assertThat(plan.delayAfter(20))
        .isLessThanOrEqualTo(Duration.ofMillis(200));
```

Đây là kỹ thuật kiểm thử dựa trên thuộc tính giúp xác nhận giá trị khoảng chờ luôn gia tăng đều đặn, không có giá trị âm và tuân thủ giới hạn tối đa mà không cần phụ thuộc vào `Thread.sleep`.

## 9. Thử nghiệm 6 — Tính lũy đẳng

```java
CreditResult first = coordinator.credit(command, deadline());
long pointsAfterFirst = points();
long versionAfterFirst = version();

// Mô phỏng phía gọi gửi lại yêu cầu với Command ID không đổi
CreditResult replay = coordinator.credit(command, deadline());

assertThat(first.applied()).isTrue(); // Trạng thái transaction hoàn tất
assertThat(replay.replayed()).isTrue(); // Trạng thái xử lý lại
assertThat(points()).isEqualTo(pointsAfterFirst); // Bảo toàn điểm
assertThat(version()).isEqualTo(versionAfterFirst); // Bảo toàn phiên bản
assertThat(creditCountFor(command.commandId())).isEqualTo(1); // Một bản ghi lịch sử
```

Hệ thống cung cấp kết quả đã được xác nhận mà không tạo bản ghi mới hay điều chỉnh lại thông số ví.

## 10. Thử nghiệm 7 — Từ chối quy tắc nghiệp vụ

Thay đổi trạng thái ví thành không hoạt động. Khi phương thức xử lý thực thi, lỗi nghiệp vụ phát sinh:

```java
assertThatThrownBy(() -> coordinator.credit(command, deadline()))
        .isInstanceOf(WalletSuspendedException.class); // Ví bị vô hiệu hóa
assertThat(attempts.invocations()).isEqualTo(1); // Thực thi 1 lần
assertThat(recordingWaiter.invocations()).isZero(); // Bỏ qua quá trình tạm dừng
assertThat(points()).isEqualTo(100); // Dữ liệu nguyên vẹn
```

## 11. Thử nghiệm 8 — Phân định ranh giới transaction

Công cụ đo kiểm chứng thực transaction không bị giữ lại trong suốt khoảng thời gian chờ:

```java
// Transaction phải ở trạng thái đã đóng
assertThat(TransactionSynchronizationManager
        .isActualTransactionActive()).isFalse();
assertThat(probe.previousOutcome()).isEqualTo("ROLLED_BACK");
```

Việc chứng minh trạng thái connection pool TRƯỚC KHI thực hiện chờ có giá trị quan trọng hơn các khai báo tĩnh bằng Annotation.

## 12. Thử nghiệm 9 — Đảm bảo bộ điều phối độc lập transaction

Thiết lập rào chắn để phát hiện cấu hình `@Transactional` bọc bên ngoài lớp điều phối:

```java
// Giả định tiến trình bên ngoài vi phạm ranh giới
assertThatThrownBy(() -> transactionalCaller.call(command))
        .isInstanceOf(IllegalStateException.class);
assertThat(attempts.invocations()).isZero(); // Từ chối tiến hành khởi tạo transaction
```

## 13. Tổng hợp ma trận bao phủ

| Kịch bản kiểm thử | Xác thực tính toàn vẹn | Xác thực khả năng vận hành |
| --- | --- | --- |
| Tương tranh đa luồng | Đồng bộ chính xác điểm / phiên bản / lịch sử | Tổng số transaction và xung đột bị giới hạn |
| Cập nhật bản ghi | Phiên bản được thay mới liên tục | Thời gian chờ được kích hoạt tự động |
| Cạn kiệt số lần | Ngăn chặn ghi dữ liệu cục bộ | Phản hồi lỗi quá tải dứt khoát |
| Vượt thời gian xử lý | Không cho phép thực thi ngoài khung giờ quy định | Dừng quá trình ngay lập tức |
| Điều phối độ trễ | Không áp dụng | Xác nhận thông số thuật toán chính xác |
| Yêu cầu lặp | Thay đổi trạng thái dữ liệu một lần duy nhất | Phản hồi truy xuất ổn định |
| Ngoại lệ nghiệp vụ | Dữ liệu được bảo toàn tuyệt đối | **Cấm kích hoạt cơ chế thử lại** |
| Ranh giới của transaction | Rollback trọn vẹn tiến trình nội bộ | Hệ thống connection pool không bị nghẽn |

## 14. Tiêu chuẩn cho độ tin cậy của bài kiểm thử

- Sử dụng cơ chế rào chắn chỉ trong chu kỳ tải đầu tiên, loại bỏ ràng buộc ở các lần tái lặp.
- Mọi công cụ đồng bộ thread (Latch, Future) phải áp dụng thời hạn, nghiêm cấm sử dụng `Thread.sleep` trực tiếp.
- Thay thế các đối tượng tính thời gian bằng đồng hồ mô phỏng, độ lệch ngẫu nhiên, và hệ thống giả lập chờ để dễ dàng kiểm định logic theo nguyên tắc.
- Testcontainers với database PostgreSQL đóng vai trò thiết yếu để xác minh cơ chế phiên bản/khóa duy nhất và hành vi transaction.
- Hạn chế gán cứng số lần lặp cụ thể khi hệ thống thực tế chạy đa biến.
- Gắn tích hợp công cụ phân tích log SQL tại sự kiện kiểm thử lỗi để dò tìm nguyên nhân chập chờn.
- Trong môi trường thực tế: Bắt buộc tích hợp giám sát các thông số metric như số lượng lệnh, số lần thử lại, tỷ lệ xung đột, lỗi cạn kiệt, tần suất chờ, độ trễ, lặp lệnh, và trạng thái connection pool.
- Một khi tỷ lệ nỗ lực để thành công tăng cao bất thường, hệ thống nên cân nhắc áp dụng kiến trúc chuyển đổi trạng thái bằng cập nhật nguyên tử hoặc hàng đợi thông điệp, thay vì tùy chỉnh tăng số lần thử tối đa.
