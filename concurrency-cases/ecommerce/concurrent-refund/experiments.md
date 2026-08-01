# Thực nghiệm hoàn tiền đồng thời với PostgreSQL

## 1. Mục tiêu

Bộ test cần chứng minh các bất biến nghiệp vụ, không chỉ kiểm tra một phương thức
trả về:

```text
0 <= completed_refund_amount
completed_refund_amount <= allocated_refund_amount
allocated_refund_amount <= captured_amount

projection = tổng các bút toán ledger
cùng khóa + cùng nội dung = cùng kết quả
cùng khóa + khác nội dung = bị từ chối
mỗi refund được chấp nhận có một allocation entry và một outbox command
```

Test phải chạy trên PostgreSQL thật bằng Testcontainers. H2 không tái hiện đầy
đủ MVCC, khóa dòng, `RETURNING`, SQLSTATE và hành vi ràng buộc của PostgreSQL.

## 2. Cấu hình Testcontainers

```java
@Testcontainers
@SpringBootTest
class ConcurrentRefundIT {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withDatabaseName("refund_test")
                    .withUsername("test")
                    .withPassword("test");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
    }
}
```

Migration của test phải giống migration chạy thật. Mỗi tác nhân đồng thời dùng
một giao dịch và một kết nối riêng; không đặt cả test trong một `@Transactional`
bao ngoài.

## 3. Bộ chạy hai tác nhân

```java
public final class TwoActorRunner implements AutoCloseable {
    private final ExecutorService executor = Executors.newFixedThreadPool(2);

    public <T> List<T> run(
            Callable<T> first,
            Callable<T> second
    ) throws Exception {
        CountDownLatch ready = new CountDownLatch(2);
        CountDownLatch start = new CountDownLatch(1);

        Future<T> a = executor.submit(() -> {
            ready.countDown();
            require(ready.await(5, TimeUnit.SECONDS), "actors not ready");
            require(start.await(5, TimeUnit.SECONDS), "start not released");
            return first.call();
        });
        Future<T> b = executor.submit(() -> {
            ready.countDown();
            require(ready.await(5, TimeUnit.SECONDS), "actors not ready");
            require(start.await(5, TimeUnit.SECONDS), "start not released");
            return second.call();
        });

        require(ready.await(5, TimeUnit.SECONDS), "actors not ready");
        start.countDown();

        return List.of(
                a.get(10, TimeUnit.SECONDS),
                b.get(10, TimeUnit.SECONDS)
        );
    }

    private static void require(boolean value, String message) {
        if (!value) {
            throw new AssertionError(message);
        }
    }

    @Override
    public void close() {
        executor.shutdownNow();
    }
}
```

Các giới hạn thời gian trong test chỉ ngăn CI treo vô hạn; chúng không phải cấu
hình khuyến nghị cho môi trường thật. Không dùng `Thread.sleep` để hy vọng hai
luồng rơi đúng thời điểm.

## 4. Dữ liệu nền và hàm kiểm tra bất biến

```java
private static final UUID MERCHANT_ID = UUID.randomUUID();
private static final UUID CHARGE_ID = UUID.randomUUID();

@BeforeEach
void setUpCharge() {
    jdbc.update("DELETE FROM outbox_event", Map.of());
    jdbc.update("DELETE FROM refund_ledger_entry", Map.of());
    jdbc.update("DELETE FROM refund", Map.of());
    jdbc.update("DELETE FROM refund_request", Map.of());
    jdbc.update("DELETE FROM payment_charge", Map.of());

    jdbc.update("""
            INSERT INTO payment_charge (
                charge_id,
                merchant_id,
                captured_amount,
                allocated_refund_amount,
                completed_refund_amount,
                currency,
                revision,
                captured_at,
                updated_at
            ) VALUES (
                :chargeId,
                :merchantId,
                1000000,
                0,
                0,
                'VND',
                0,
                CURRENT_TIMESTAMP,
                CURRENT_TIMESTAMP
            )
            """, Map.of(
            "chargeId", CHARGE_ID,
            "merchantId", MERCHANT_ID
    ));
}
```

```java
private void assertChargeInvariant() {
    ChargeAmounts charge = loadChargeAmounts(CHARGE_ID);
    LedgerAmounts ledger = sumLedger(CHARGE_ID);

    assertThat(charge.completed()).isGreaterThanOrEqualTo(0);
    assertThat(charge.allocated()).isGreaterThanOrEqualTo(charge.completed());
    assertThat(charge.captured()).isGreaterThanOrEqualTo(charge.allocated());
    assertThat(charge.allocated()).isEqualTo(ledger.allocated());
    assertThat(charge.completed()).isEqualTo(ledger.completed());
}

private void assertAcceptedRefundHasEvidence(UUID refundId) {
    assertThat(countRefund(refundId)).isEqualTo(1);
    assertThat(countLedger(refundId, "REFUND_ALLOCATED")).isEqualTo(1);
    assertThat(countOutbox(refundId, "REFUND_REQUESTED")).isEqualTo(1);
}
```

Mọi test thay đổi dữ liệu đều gọi `assertChargeInvariant()` ở cuối.

## 5. Điểm hẹn tái hiện bản lỗi

Để tái hiện chắc chắn chuỗi đọc–kiểm tra–ghi, chèn một cổng test sau lần đọc:

```java
@FunctionalInterface
interface AfterChargeReadHook {
    void reached();
}

final class BarrierHook implements AfterChargeReadHook {
    private final CyclicBarrier barrier = new CyclicBarrier(2);

    @Override
    public void reached() {
        try {
            barrier.await(5, TimeUnit.SECONDS);
        } catch (Exception failure) {
            throw new IllegalStateException(failure);
        }
    }
}
```

`BrokenRefundService` gọi `hook.reached()` ngay sau khi tải charge và trước khi
kiểm tra hạn mức. Hook mặc định ở mã chạy thật không làm gì; cấu hình test dùng
`BarrierHook`.

## 6. Tái hiện hai refund vượt số tiền đã thu

```java
@Test
void broken_read_check_write_exceeds_captured_amount() throws Exception {
    RefundCommand a = command("KEY-A", 700_000);
    RefundCommand b = command("KEY-B", 600_000);

    try (TwoActorRunner actors = new TwoActorRunner()) {
        actors.run(
                () -> brokenService.refund(a),
                () -> brokenService.refund(b)
        );
    }

    assertThat(sumAcceptedRefundAmounts(CHARGE_ID))
            .isEqualTo(1_300_000);
    assertThat(loadChargeAmounts(CHARGE_ID).allocated())
            .isNotEqualTo(1_300_000);
}
```

Test này cố ý chứng minh bản lỗi. Không chạy `assertChargeInvariant()` vì kết quả
được mong đợi là vi phạm.

## 7. Hai yêu cầu khác nhau tranh hạn mức

```java
@Test
void conditional_allocation_accepts_only_one_when_sum_exceeds_capture()
        throws Exception {
    RefundCommand a = command("KEY-A", 700_000);
    RefundCommand b = command("KEY-B", 600_000);

    List<RefundResult> results;
    try (TwoActorRunner actors = new TwoActorRunner()) {
        results = actors.run(
                () -> coordinator.accept(a),
                () -> coordinator.accept(b)
        );
    }

    assertThat(results)
            .filteredOn(RefundResult.Accepted.class::isInstance)
            .hasSize(1);
    assertThat(results)
            .filteredOn(RefundResult.LimitExceeded.class::isInstance)
            .hasSize(1);

    RefundResult.Accepted accepted = results.stream()
            .filter(RefundResult.Accepted.class::isInstance)
            .map(RefundResult.Accepted.class::cast)
            .findFirst()
            .orElseThrow();

    assertAcceptedRefundHasEvidence(accepted.refundId());
    assertThat(countRefunds(CHARGE_ID)).isEqualTo(1);
    assertChargeInvariant();
}
```

Không cố định A hay B phải thắng. Lịch chạy hợp lệ được phép chọn một trong hai;
bất biến mới là điều test cần khẳng định.

## 8. Hai yêu cầu hợp lệ không bị gom thành một

```java
@Test
void different_keys_can_both_win_when_total_fits() throws Exception {
    RefundCommand a = command("KEY-A", 600_000);
    RefundCommand b = command("KEY-B", 400_000);

    List<RefundResult> results;
    try (TwoActorRunner actors = new TwoActorRunner()) {
        results = actors.run(
                () -> coordinator.accept(a),
                () -> coordinator.accept(b)
        );
    }

    assertThat(results)
            .allMatch(RefundResult.Accepted.class::isInstance);
    assertThat(loadChargeAmounts(CHARGE_ID).allocated())
            .isEqualTo(1_000_000);
    assertThat(countRefunds(CHARGE_ID)).isEqualTo(2);
    assertThat(countLedger(CHARGE_ID, "REFUND_ALLOCATED")).isEqualTo(2);
    assertThat(countOutbox(CHARGE_ID, "REFUND_REQUESTED")).isEqualTo(2);
    assertChargeInvariant();
}
```

Test này ngăn một cách sửa sai thường gặp: dùng `charge_id` làm khóa lũy đẳng và
vô tình chỉ cho một khoản hoàn tồn tại trên mỗi giao dịch.

## 9. Cùng khóa và cùng nội dung chỉ phân bổ một lần

```java
@Test
void duplicate_request_replays_the_same_refund() throws Exception {
    RefundCommand original = command("SAME-KEY", 700_000);
    RefundCommand retry = sameIntentWithNewRequestId(original);

    List<RefundResult> results;
    try (TwoActorRunner actors = new TwoActorRunner()) {
        results = actors.run(
                () -> coordinator.accept(original),
                () -> coordinator.accept(retry)
        );
    }

    List<UUID> refundIds = results.stream()
            .map(RefundResult.Accepted.class::cast)
            .map(RefundResult.Accepted::refundId)
            .toList();

    assertThat(refundIds).hasSize(2);
    assertThat(refundIds.get(0)).isEqualTo(refundIds.get(1));
    assertThat(loadChargeAmounts(CHARGE_ID).allocated()).isEqualTo(700_000);
    assertThat(countRefunds(CHARGE_ID)).isEqualTo(1);
    assertAcceptedRefundHasEvidence(refundIds.get(0));
    assertChargeInvariant();
}
```

`requestId` nội bộ của lần gửi lại không được nằm trong dấu vân tay. Khóa và nội
dung nghiệp vụ mới quyết định đó có phải cùng ý định hay không.

## 10. Cùng khóa nhưng khác nội dung bị từ chối

```java
@Test
void reused_key_with_different_amount_is_rejected() {
    RefundCommand first = command("REUSED-KEY", 700_000);
    RefundResult.Accepted accepted =
            (RefundResult.Accepted) coordinator.accept(first);

    RefundCommand changed = command("REUSED-KEY", 600_000);

    assertThatThrownBy(() -> coordinator.accept(changed))
            .isInstanceOf(IdempotencyKeyReusedException.class);

    assertThat(loadChargeAmounts(CHARGE_ID).allocated()).isEqualTo(700_000);
    assertAcceptedRefundHasEvidence(accepted.refundId());
    assertChargeInvariant();
}
```

Thêm các biến thể đổi `charge_id`, `currency` và `reasonCode` để kiểm tra dấu vân
tay bao phủ mọi trường làm thay đổi ý nghĩa.

## 11. Kết quả vượt hạn mức cũng được phát lại

```java
@Test
void limit_rejection_is_stable_for_the_same_key() {
    accept(command("FIRST", 800_000));
    RefundCommand rejected = command("REJECTED", 300_000);

    assertThat(coordinator.accept(rejected))
            .isInstanceOf(RefundResult.LimitExceeded.class);

    releaseAcceptedRefund("FIRST");

    assertThat(coordinator.accept(sameIntentWithNewRequestId(rejected)))
            .isInstanceOf(RefundResult.LimitExceeded.class);
    assertThat(countRequests("REJECTED")).isEqualTo(1);
    assertChargeInvariant();
}
```

Sau khi hạn mức thay đổi, một ý định mới có thể dùng khóa mới. Không đổi kết quả
đã chốt của khóa cũ.

## 12. Lỗi sau phân bổ phải hoàn tác mọi dữ liệu

Tạo một `RefundOutboxDao` cho test ném ngoại lệ ngay trước khi chèn outbox:

```java
@Test
void outbox_failure_rolls_back_capacity_refund_ledger_and_claim() {
    RefundCommand command = command("ROLLBACK", 700_000);

    assertThatThrownBy(() -> failingCoordinator.accept(command))
            .isInstanceOf(InjectedFailure.class);

    assertThat(loadChargeAmounts(CHARGE_ID).allocated()).isZero();
    assertThat(countRequests("ROLLBACK")).isZero();
    assertThat(countRefunds(CHARGE_ID)).isZero();
    assertThat(countLedgerEntries(CHARGE_ID)).isZero();
    assertThat(countOutboxEvents(CHARGE_ID)).isZero();

    assertThat(coordinator.accept(command))
            .isInstanceOf(RefundResult.Accepted.class);
    assertChargeInvariant();
}
```

Có thể đặt điểm lỗi sau từng bước ghi để bảo đảm mọi nhánh đều hoàn tác.

## 13. Bên giữ khóa chốt, bên chờ kiểm tra lại điều kiện

Dùng hai `Connection` riêng để quan sát trực tiếp:

```java
@Test
void waiter_rechecks_limit_after_holder_commits() throws Exception {
    try (Connection holder = dataSource.getConnection();
         Connection waiter = dataSource.getConnection();
         ExecutorService executor = Executors.newSingleThreadExecutor()) {

        holder.setAutoCommit(false);
        waiter.setAutoCommit(false);

        assertThat(allocate(holder, CHARGE_ID, 700_000)).isEqualTo(1);
        int waiterPid = backendPid(waiter);
        CountDownLatch submitted = new CountDownLatch(1);

        Future<Integer> waitingUpdate = executor.submit(() -> {
            submitted.countDown();
            return allocate(waiter, CHARGE_ID, 600_000);
        });

        assertThat(submitted.await(5, TimeUnit.SECONDS)).isTrue();
        await().atMost(Duration.ofSeconds(5))
                .until(() -> isWaitingOnRowLock(waiterPid));

        holder.commit();

        assertThat(waitingUpdate.get(5, TimeUnit.SECONDS)).isZero();
        waiter.commit();
    }

    assertThat(loadChargeAmounts(CHARGE_ID).allocated()).isEqualTo(700_000);
}
```

`isWaitingOnRowLock` đọc `pg_stat_activity.wait_event_type = 'Lock'`. Awaitility
chỉ quan sát có giới hạn; việc sắp xếp hai tác nhân vẫn do latch và khóa cơ sở dữ
liệu quyết định.

Test JDBC thấp tầng này không chèn ledger, nên hoàn tác dữ liệu sau phần quan sát
hoặc chạy trong fixture riêng. Test dịch vụ ở mục 7 mới khẳng định toàn bộ bất
biến nghiệp vụ.

## 14. Bên giữ khóa hoàn tác, bên chờ được phép tiếp tục

```java
@Test
void waiter_can_allocate_after_holder_rolls_back() throws Exception {
    try (Connection holder = dataSource.getConnection();
         Connection waiter = dataSource.getConnection();
         ExecutorService executor = Executors.newSingleThreadExecutor()) {

        holder.setAutoCommit(false);
        waiter.setAutoCommit(false);
        assertThat(allocate(holder, CHARGE_ID, 700_000)).isEqualTo(1);
        int waiterPid = backendPid(waiter);

        Future<Integer> waitingUpdate = executor.submit(
                () -> allocate(waiter, CHARGE_ID, 600_000)
        );
        await().atMost(Duration.ofSeconds(5))
                .until(() -> isWaitingOnRowLock(waiterPid));

        holder.rollback();

        assertThat(waitingUpdate.get(5, TimeUnit.SECONDS)).isEqualTo(1);
        waiter.commit();
    }

    assertThat(loadChargeAmounts(CHARGE_ID).allocated()).isEqualTo(600_000);
}
```

Điều này chứng minh lần cập nhật chưa chốt của bên giữ khóa không làm mất hạn
mức vĩnh viễn.

## 15. Hết thời gian chờ là lỗi kỹ thuật

```java
@Test
void lock_timeout_is_not_reported_as_limit_exceeded() throws Exception {
    try (Connection holder = dataSource.getConnection()) {
        holder.setAutoCommit(false);
        assertThat(allocate(holder, CHARGE_ID, 700_000)).isEqualTo(1);

        RefundResult result = coordinatorWithShortTestTimeout()
                .accept(command("TIMEOUT", 100_000));

        assertThat(result)
                .isInstanceOf(RefundResult.Busy.class);
        assertThat(countRequests("TIMEOUT")).isZero();

        holder.rollback();
    }

    assertThat(loadChargeAmounts(CHARGE_ID).allocated()).isZero();
}
```

Thời gian chờ ngắn chỉ nằm trong cấu hình test. Sau lỗi PostgreSQL, worker phải
để transaction manager hoàn tác trước khi coordinator phân loại lỗi.

## 16. Hoàn tất trùng chỉ tăng số tiền một lần

```java
@Test
void duplicate_success_notification_completes_once() throws Exception {
    RefundResult.Accepted accepted =
            accept(command("SUCCESS", 700_000));

    try (TwoActorRunner actors = new TwoActorRunner()) {
        actors.run(
                () -> providerResults.markSucceeded(
                        accepted.refundId(), "PR-1"),
                () -> providerResults.markSucceeded(
                        accepted.refundId(), "PR-1")
        );
    }

    ChargeAmounts charge = loadChargeAmounts(CHARGE_ID);
    assertThat(charge.allocated()).isEqualTo(700_000);
    assertThat(charge.completed()).isEqualTo(700_000);
    assertThat(countLedger(
            accepted.refundId(),
            "REFUND_SUCCEEDED"
    )).isEqualTo(1);
    assertThat(loadRefundStatus(accepted.refundId()))
            .isEqualTo("SUCCEEDED");
    assertChargeInvariant();
}
```

## 17. Giải phóng trùng chỉ trả hạn mức một lần

```java
@Test
void duplicate_failure_notification_releases_once() throws Exception {
    RefundResult.Accepted accepted =
            accept(command("FAILURE", 700_000));

    try (TwoActorRunner actors = new TwoActorRunner()) {
        actors.run(
                () -> providerResults.releaseFailed(accepted.refundId()),
                () -> providerResults.releaseFailed(accepted.refundId())
        );
    }

    assertThat(loadChargeAmounts(CHARGE_ID).allocated()).isZero();
    assertThat(countLedger(
            accepted.refundId(),
            "REFUND_RELEASED"
    )).isEqualTo(1);
    assertThat(loadRefundStatus(accepted.refundId()))
            .isEqualTo("FAILED_RELEASED");
    assertChargeInvariant();
}
```

Thêm điểm lỗi sau chuyển trạng thái nhưng trước cập nhật charge để chứng minh cả
hai cùng hoàn tác và lần xử lý sau vẫn có thể giải phóng.

## 18. Thành công và thất bại cùng tranh một refund

```java
@Test
void incompatible_results_cannot_both_apply() throws Exception {
    RefundResult.Accepted accepted =
            accept(command("RESULT-RACE", 700_000));

    try (TwoActorRunner actors = new TwoActorRunner()) {
        actors.run(
                () -> providerResults.markSucceeded(
                        accepted.refundId(), "PR-2"),
                () -> providerResults.releaseFailed(accepted.refundId())
        );
    }

    assertThat(loadRefundStatus(accepted.refundId()))
            .isIn("SUCCEEDED", "FAILED_RELEASED");
    assertThat(countTerminalLedgerEntries(accepted.refundId()))
            .isEqualTo(1);
    assertChargeInvariant();
}
```

Test này chỉ chứng minh hai chuyển trạng thái không cùng thắng. Kết quả nào đúng
về thứ tự sự kiện của nhà cung cấp phải được xác định bằng quy tắc BANK-006.

## 19. Mất phản hồi sau `COMMIT`

Tạo lớp bọc ngoài giao dịch gọi worker thành công rồi ném lỗi mô phỏng mất phản
hồi:

```java
@Test
void retry_after_dropped_response_replays_committed_result() {
    RefundCommand command = command("DROPPED", 700_000);

    assertThatThrownBy(() -> responseDropper.acceptThenDrop(command))
            .isInstanceOf(SimulatedConnectionDrop.class);

    RefundResult.Accepted replayed =
            (RefundResult.Accepted) coordinator.accept(
                    sameIntentWithNewRequestId(command)
            );

    assertThat(loadChargeAmounts(CHARGE_ID).allocated()).isEqualTo(700_000);
    assertThat(countRefunds(CHARGE_ID)).isEqualTo(1);
    assertAcceptedRefundHasEvidence(replayed.refundId());
    assertChargeInvariant();
}
```

Lỗi phải được ném sau khi `RefundAcceptanceTx.accept` đã trả về, tức transaction
proxy đã chốt. Không ném bên trong worker vì khi đó test chỉ kiểm tra rollback.

## 20. Hai máy chủ ứng dụng

Khởi tạo hai `ApplicationContext` cùng trỏ vào PostgreSQL Testcontainer, nhưng
mỗi context có `DataSource`, transaction manager và bean service riêng. Gửi hai
lệnh ở mục 7 qua hai context:

```java
actors.run(
        () -> nodeA.getBean(RefundCoordinator.class).accept(a),
        () -> nodeB.getBean(RefundCoordinator.class).accept(b)
);
```

Khẳng định một yêu cầu được chấp nhận, một yêu cầu vượt hạn mức và đối soát rỗng.
Test này chứng minh kết quả đến từ PostgreSQL, không đến từ khóa trong một JVM.

## 21. Kiểm tra outbox được giao lại

Mô phỏng consumer gọi nhà cung cấp thành công rồi dừng trước khi đánh dấu
`published_at`. Lần chạy sau đọc lại cùng sự kiện và phải dùng cùng `refund_id`
làm khóa lũy đẳng ở nhà cung cấp giả lập:

```java
assertThat(fakeProvider.distinctRefunds()).containsExactly(refundId);
assertThat(fakeProvider.callsFor(refundId)).isGreaterThanOrEqualTo(2);
```

Số lần gọi mạng có thể lớn hơn một; số tác dụng bền vững tại nhà cung cấp phải là
một. Nếu nhà cung cấp thật không hỗ trợ khóa lũy đẳng, thay fake bằng hợp đồng tra
cứu theo tham chiếu và test riêng kết quả chưa rõ.

## 22. Truy vấn đối soát trong test

```java
@Test
void reconciliation_queries_are_empty_after_scenarios() {
    // chạy một tập hợp accept, complete và release hợp lệ

    List<Map<String, Object>> projectionMismatch = jdbc.queryForList("""
            SELECT c.charge_id
            FROM payment_charge c
            LEFT JOIN refund_ledger_entry e
                   ON e.charge_id = c.charge_id
            GROUP BY c.charge_id,
                     c.allocated_refund_amount,
                     c.completed_refund_amount
            HAVING c.allocated_refund_amount <>
                       COALESCE(SUM(e.allocation_delta), 0)
                OR c.completed_refund_amount <>
                       COALESCE(SUM(e.completion_delta), 0)
            """, Map.of());

    List<Map<String, Object>> amountViolation = jdbc.queryForList("""
            SELECT charge_id
            FROM payment_charge
            WHERE allocated_refund_amount < 0
               OR completed_refund_amount < 0
               OR completed_refund_amount > allocated_refund_amount
               OR allocated_refund_amount > captured_amount
            """, Map.of());

    assertThat(projectionMismatch).isEmpty();
    assertThat(amountViolation).isEmpty();
}
```

## 23. Dữ liệu cần giữ khi test thất bại

Khi một test đồng thời thất bại, in hoặc lưu:

- khóa lũy đẳng đã che một phần và dấu vân tay rút gọn;
- `request_id`, `refund_id`, `charge_id`;
- kết quả và ngoại lệ của từng tác nhân;
- SQLSTATE, trạng thái transaction và thời gian chờ khóa;
- dòng `payment_charge` trước và sau;
- toàn bộ refund, ledger và outbox liên quan;
- ảnh chụp `pg_stat_activity` và `pg_locks` cho test khóa;
- seed nếu dữ liệu đầu vào được sinh ngẫu nhiên.

Không bỏ qua test không ổn định. Một test chỉ thỉnh thoảng đỏ thường cho thấy
điểm hẹn chưa kiểm soát đúng hoặc bất biến chưa được kiểm tra ở đúng ranh giới.

## 24. Ma trận thực nghiệm tối thiểu

| Kịch bản | Khẳng định chính |
| --- | --- |
| Bản lỗi `700.000 + 600.000` | Tái hiện vượt hạn mức và projection lệch |
| Bản đúng `700.000 + 600.000` | Một chấp nhận, một từ chối; không vượt captured |
| Hai khóa `600.000 + 400.000` | Cả hai chấp nhận; không bị gom nhầm |
| Cùng khóa, cùng nội dung | Một refund, một allocation, cùng phản hồi |
| Cùng khóa, khác nội dung | Từ chối dùng lại khóa |
| Từ chối rồi hạn mức thay đổi | Khóa cũ vẫn phát lại cùng từ chối |
| Lỗi ledger hoặc outbox | Projection, claim và mọi dữ liệu cùng hoàn tác |
| Bên giữ khóa chốt | Bên chờ kiểm tra lại và có thể thua hạn mức |
| Bên giữ khóa hoàn tác | Bên chờ có thể tiếp tục |
| Hết thời gian chờ | Lỗi kỹ thuật, không phải `LIMIT_EXCEEDED` |
| Thông báo thành công trùng | Tăng completed và thêm ledger một lần |
| Thông báo thất bại trùng | Giảm allocated và thêm bút toán bù một lần |
| Thành công tranh thất bại | Chỉ một trạng thái kết thúc thắng |
| Mất phản hồi sau chốt | Gửi lại cùng khóa không tạo refund mới |
| Hai máy chủ | Bất biến giống một máy chủ |
| Đối soát | Projection bằng ledger, không vi phạm thứ tự số tiền |
