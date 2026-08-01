# Thực nghiệm số dư điểm đồng thời với PostgreSQL

## 1. Mục tiêu

Mọi test giải pháp phải giữ đúng:

```text
points_balance >= 0
points_balance = SUM(points_delta)
mỗi command thành công có đúng một bút toán
command thiếu điểm không có bút toán
cùng command không thay đổi số dư lần hai
```

Test không được chỉ đếm ngoại lệ. Phải đọc số dư, sổ cái và bảng lệnh sau khi
các giao dịch kết thúc.

## 2. Cấu hình Testcontainers

```java
@Testcontainers
@SpringBootTest
@ActiveProfiles("postgres-test")
class LoyaltyPointConcurrencyTest {

    @Container
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void databaseProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
        registry.add("spring.flyway.enabled", () -> "true");
    }

    @Autowired
    LoyaltyApplicationService loyalty;

    @Autowired
    JdbcTemplate jdbc;
}
```

Migration trong test phải giống môi trường thật, bao gồm tên ràng buộc và quyền
chỉ thêm mới. H2 không đại diện cho MVCC, khóa dòng, SQLSTATE hoặc
`ON CONFLICT` của PostgreSQL.

## 3. Tiện ích chạy hai tác nhân

```java
final class TwoActors implements AutoCloseable {

    private final ExecutorService executor =
        Executors.newFixedThreadPool(2);

    <T> List<T> runTogether(
        Callable<T> firstAction,
        Callable<T> secondAction
    ) throws Exception {
        CyclicBarrier start = new CyclicBarrier(2);

        Future<T> first = executor.submit(() -> {
            start.await(5, TimeUnit.SECONDS);
            return firstAction.call();
        });
        Future<T> second = executor.submit(() -> {
            start.await(5, TimeUnit.SECONDS);
            return secondAction.call();
        });

        return List.of(
            first.get(10, TimeUnit.SECONDS),
            second.get(10, TimeUnit.SECONDS)
        );
    }

    @Override
    public void close() {
        executor.shutdownNow();
    }
}
```

Mọi lần chờ có thời gian tối đa. Không dùng `Thread.sleep` làm cách chính để đoán
một giao dịch đã đọc, ghi hoặc đang chờ khóa.

## 4. Tái hiện lỗi tiêu hai lần

Biến thể bị lỗi gọi một cổng thử nghiệm sau khi đã đọc số dư và vượt qua bước
`canSpend`, nhưng trước khi thay đổi thực thể:

```java
@FunctionalInterface
interface AfterBalanceCheck {
    void reached();
}
```

Cài đặt test dùng `CyclicBarrier(2)` để cả hai giao dịch cùng dừng ở khoảng hở.

```java
@Test
void readCheckWriteAcceptsTwoSpends() throws Exception {
    UUID customerId = fixtures.accountWithOpeningBalance(1_000);
    SpendPointsCommand first = fixtures.spend(customerId, 800);
    SpendPointsCommand second = fixtures.spend(customerId, 800);

    try (TwoActors actors = new TwoActors()) {
        actors.runTogether(
            () -> brokenService.spend(first),
            () -> brokenService.spend(second)
        );
    }

    assertThat(readBalance(customerId)).isEqualTo(200);
    assertThat(sumLedger(customerId)).isEqualTo(-600);
    assertThat(countRedeemEntries(customerId)).isEqualTo(2);
}
```

Tổng sổ cái gồm bút toán mở `+1.000` và hai bút toán `-800`. Test cố ý chứng minh
bảng số dư dương không đủ để kết luận hệ thống đúng.

## 5. Hai lệnh tiêu chỉ cho một bên thành công

```java
@Test
void conditionalDebitAllowsOnlyOneSpend() throws Exception {
    UUID customerId = fixtures.accountWithOpeningBalance(1_000);
    SpendPointsCommand first = fixtures.spend(customerId, 800);
    SpendPointsCommand second = fixtures.spend(customerId, 800);

    List<LoyaltyResult> results;
    try (TwoActors actors = new TwoActors()) {
        results = actors.runTogether(
            () -> loyalty.spend(first),
            () -> loyalty.spend(second)
        );
    }

    assertThat(results)
        .extracting(LoyaltyResult::outcome)
        .containsExactlyInAnyOrder(
            LoyaltyOutcome.REDEEMED,
            LoyaltyOutcome.INSUFFICIENT_POINTS
        );

    assertThat(readBalance(customerId)).isEqualTo(200);
    assertThat(sumLedger(customerId)).isEqualTo(200);
    assertThat(countRedeemEntries(customerId)).isEqualTo(1);
    assertThat(countCommands(customerId, LoyaltyOutcome.REDEEMED))
        .isEqualTo(1);
    assertThat(countCommands(
        customerId,
        LoyaltyOutcome.INSUFFICIENT_POINTS
    )).isEqualTo(1);
    assertReconciled(customerId);
}
```

Không giả định lệnh nào thắng. Hai lệnh khác nhau chỉ cần tạo một thứ tự tuần tự
hợp lệ trên dòng số dư.

## 6. Cộng và trừ đồng thời không mất chênh lệch

```java
@Test
void concurrentEarnAndSpendKeepBothDeltas() throws Exception {
    UUID customerId = fixtures.accountWithOpeningBalance(1_000);
    EarnPointsCommand earn = fixtures.earn(customerId, 500);
    SpendPointsCommand spend = fixtures.spend(customerId, 300);

    try (TwoActors actors = new TwoActors()) {
        actors.runTogether(
            () -> loyalty.earn(earn),
            () -> loyalty.spend(spend)
        );
    }

    assertThat(readBalance(customerId)).isEqualTo(1_200);
    assertThat(sumLedger(customerId)).isEqualTo(1_200);
    assertThat(countEntriesForCommands(
        customerId,
        earn.commandId(),
        spend.commandId()
    )).isEqualTo(2);
    assertSequenceHasNoDuplicates(customerId);
    assertReconciled(customerId);
}
```

## 7. Cùng mã lệnh được phát lại

```java
@Test
void duplicateSpendReplaysWithoutSecondDebit() throws Exception {
    UUID customerId = fixtures.accountWithOpeningBalance(1_000);
    SpendPointsCommand command = fixtures.spend(customerId, 300);

    List<LoyaltyResult> results;
    try (TwoActors actors = new TwoActors()) {
        results = actors.runTogether(
            () -> loyalty.spend(command),
            () -> loyalty.spend(command)
        );
    }

    assertThat(results)
        .extracting(LoyaltyResult::outcome)
        .containsOnly(LoyaltyOutcome.REDEEMED);
    assertThat(results)
        .extracting(LoyaltyResult::balanceAfter)
        .containsOnly(700L);
    assertThat(results)
        .extracting(LoyaltyResult::replayed)
        .containsExactlyInAnyOrder(false, true);

    assertThat(readBalance(customerId)).isEqualTo(700);
    assertThat(countEntriesForCommand(customerId, command.commandId()))
        .isEqualTo(1);
    assertReconciled(customerId);
}
```

## 8. Cùng mã nhưng khác nội dung

```java
@Test
void reusedCommandWithDifferentPointsIsRejected() {
    UUID customerId = fixtures.accountWithOpeningBalance(1_000);
    SpendPointsCommand original = fixtures.spend(customerId, 300);
    SpendPointsCommand changed = new SpendPointsCommand(
        customerId,
        original.commandId(),
        original.orderId(),
        400
    );

    loyalty.spend(original);

    assertThatThrownBy(() -> loyalty.spend(changed))
        .isInstanceOf(LoyaltyCommandMismatchException.class);

    assertThat(readBalance(customerId)).isEqualTo(700);
    assertThat(countEntriesForCommand(customerId, original.commandId()))
        .isEqualTo(1);
}
```

Thêm biến thể giữ số điểm nhưng đổi `orderId` hoặc đổi `REDEEM` thành `EARN`.
Cả hai đều phải bị từ chối.

## 9. Kết quả thiếu điểm được phát lại

```java
@Test
void rejectedCommandDoesNotBecomeSuccessfulAfterEarn() {
    UUID customerId = fixtures.accountWithOpeningBalance(500);
    SpendPointsCommand rejected = fixtures.spend(customerId, 800);

    LoyaltyResult first = loyalty.spend(rejected);
    assertThat(first.outcome())
        .isEqualTo(LoyaltyOutcome.INSUFFICIENT_POINTS);

    loyalty.earn(fixtures.earn(customerId, 500));
    assertThat(readBalance(customerId)).isEqualTo(1_000);

    LoyaltyResult replay = loyalty.spend(rejected);
    assertThat(replay.outcome())
        .isEqualTo(LoyaltyOutcome.INSUFFICIENT_POINTS);
    assertThat(replay.replayed()).isTrue();
    assertThat(readBalance(customerId)).isEqualTo(1_000);
    assertThat(countEntriesForCommand(customerId, rejected.commandId()))
        .isZero();

    LoyaltyResult newIntent = loyalty.spend(
        fixtures.spend(customerId, 800)
    );
    assertThat(newIntent.outcome()).isEqualTo(LoyaltyOutcome.REDEEMED);
    assertThat(readBalance(customerId)).isEqualTo(200);
    assertReconciled(customerId);
}
```

Test này làm rõ mã lệnh đại diện cho một ý định, không phải một lần truyền HTTP.

## 10. Lỗi sau phép trừ hoàn tác mọi thứ

Đặt điểm gây lỗi sau `debitIfEnough` nhưng trước `ledger.append`:

```java
@Test
void failureBeforeLedgerInsertRollsBackDebitAndClaim() {
    UUID customerId = fixtures.accountWithOpeningBalance(1_000);
    SpendPointsCommand command = fixtures.spend(customerId, 300);

    faults.failOnceAfterDebit();

    assertThatThrownBy(() -> loyalty.spend(command))
        .isInstanceOf(SimulatedTechnicalFailure.class);

    assertThat(readBalance(customerId)).isEqualTo(1_000);
    assertThat(countEntriesForCommand(customerId, command.commandId()))
        .isZero();
    assertThat(findCommand(customerId, command.commandId())).isEmpty();

    LoyaltyResult retry = loyalty.spend(command);
    assertThat(retry.outcome()).isEqualTo(LoyaltyOutcome.REDEEMED);
    assertThat(readBalance(customerId)).isEqualTo(700);
    assertThat(countEntriesForCommand(customerId, command.commandId()))
        .isEqualTo(1);
    assertReconciled(customerId);
}
```

Điểm gây lỗi phải ném ngoại lệ ra ngoài phương thức `@Transactional`. Không bắt
lỗi rồi trả phản hồi thành công.

## 11. Bên giữ khóa hoàn tác

Test JDBC dùng hai kết nối và một kết nối quan sát:

1. A mở giao dịch, trừ `800` từ `1.000` nhưng chưa chốt.
2. B chạy cùng câu trừ trong `Future` và chờ dòng tài khoản.
3. Kết nối quan sát xác nhận `pg_blocking_pids(waiterPid)` chứa PID của A.
4. A gọi `ROLLBACK`.
5. B phải nhận một dòng, số dư sau là `200`.

```java
await()
    .atMost(Duration.ofSeconds(5))
    .untilAsserted(() ->
        assertThat(blockingObserver.blockersOf(waiterPid))
            .contains(holderPid)
    );

holder.rollback();
BalanceChange result = waitingDebit.get(5, TimeUnit.SECONDS);
waiter.commit();

assertThat(result.pointsBalance()).isEqualTo(200);
```

Không dùng thời gian ngủ cố định để kết luận B đã bị chặn.

## 12. Bên giữ khóa chốt

Lặp lại test trên nhưng A gọi `COMMIT`. Câu trừ của B phải trả kết quả rỗng sau
khi kiểm tra lại `200 >= 800`.

Sau khi giao dịch sản phẩm hoàn tất:

```text
một command REDEEMED
một command INSUFFICIENT_POINTS
một bút toán REDEEM
points_balance = 200
tổng sổ cái = 200
```

## 13. Hết thời gian chờ khóa không phải thiếu điểm

Kết nối A giữ dòng tài khoản. B đặt:

```sql
SET LOCAL lock_timeout = '100ms';
```

B phải nhận SQLSTATE `55P03`. Ứng dụng không được lưu
`INSUFFICIENT_POINTS`, vì điều kiện số dư chưa được phân xử bình thường. Sau khi
A kết thúc, một giao dịch mới dùng cùng mã lệnh có thể thử lại.

`100ms` chỉ giúp test chạy nhanh; không phải giá trị đề xuất cho môi trường thật.

## 14. Mất phản hồi sau khi chốt

```java
@Test
void retryAfterLostResponseReadsStoredOutcome() {
    UUID customerId = fixtures.accountWithOpeningBalance(1_000);
    SpendPointsCommand command = fixtures.spend(customerId, 300);

    LoyaltyResult notDelivered = loyalty.spend(command);
    LoyaltyResult replay = loyalty.spend(command);

    assertThat(replay.replayed()).isTrue();
    assertThat(replay.outcome()).isEqualTo(notDelivered.outcome());
    assertThat(replay.balanceAfter()).isEqualTo(notDelivered.balanceAfter());
    assertThat(readBalance(customerId)).isEqualTo(700);
    assertThat(countEntriesForCommand(customerId, command.commandId()))
        .isEqualTo(1);
}
```

Test bỏ phản hồi đầu như thể mạng bị cắt. Không cần đoán trạng thái `COMMIT`; hợp
đồng phát lại là phần cần chứng minh.

## 15. Ràng buộc sổ cái được kiểm tra trực tiếp

```java
@Test
void oneCommandCannotCreateTwoLedgerEntries() {
    LoyaltyFixture fixture = fixtures.completedRedeem();

    assertThatThrownBy(() -> fixtures.insertSecondEntry(
        fixture.customerId(),
        fixture.commandId()
    )).satisfies(error -> assertUniqueViolation(
        error,
        "uk_ledger_customer_command"
    ));
}
```

Thêm test cho `uk_ledger_customer_sequence`, `uk_ledger_single_reversal` và
`ck_loyalty_account_balance`. Trình phân loại kiểm tra SQLSTATE `23505` cùng tên
ràng buộc, không coi mọi lỗi dữ liệu là lệnh trùng.

## 16. Quyền chỉ thêm mới

Kết nối bằng vai trò `app_runtime`:

```text
INSERT loyalty_ledger_entry → được phép
UPDATE loyalty_ledger_entry → bị từ chối
DELETE loyalty_ledger_entry → bị từ chối
```

Kiểm tra SQLSTATE quyền truy cập phù hợp. Test migration dùng vai trò chủ sở hữu
riêng để không vô tình chạy toàn bộ ứng dụng bằng tài khoản có thể sửa lịch sử.

## 17. Hoàn điểm chỉ một lần

Tạo một bút toán `REDEEM -300`, sau đó chạy hai lệnh hoàn đồng thời cùng nhắm
`reverses_entry_id`:

```text
chỉ một REVERSAL +300 được chốt
số dư tăng đúng 300
uk_ledger_single_reversal ngăn lần thứ hai
tổng sổ cái khớp số dư
```

Nếu nhánh hoàn điểm dùng cơ chế chiếm mã lệnh, bên thua nên phát lại kết quả thay
vì để lỗi `23505` lọt ra API.

## 18. Truy vấn đối soát

```sql
SELECT a.customer_id,
       a.points_balance,
       COALESCE(sum(e.points_delta), 0) AS ledger_balance
FROM loyalty_account a
LEFT JOIN loyalty_ledger_entry e
  ON e.customer_id = a.customer_id
GROUP BY a.customer_id, a.points_balance
HAVING a.points_balance <> COALESCE(sum(e.points_delta), 0);
```

Kiểm tra quan hệ lệnh–bút toán:

```sql
SELECT c.customer_id,
       c.command_id,
       c.outcome,
       count(e.entry_id) AS entry_count
FROM loyalty_command c
LEFT JOIN loyalty_ledger_entry e
  ON e.customer_id = c.customer_id
 AND e.command_id = c.command_id
GROUP BY c.customer_id, c.command_id, c.outcome
HAVING (
    c.outcome IN ('EARNED', 'REDEEMED')
    AND count(e.entry_id) <> 1
) OR (
    c.outcome = 'INSUFFICIENT_POINTS'
    AND count(e.entry_id) <> 0
);
```

Cả hai truy vấn phải trả rỗng sau mỗi test.

## 19. Kiểm tra chuỗi `balance_after`

Đọc bút toán theo `account_sequence`. Với bút toán đầu, `balance_after` phải bằng
`points_delta` nếu số dư mở được biểu diễn bằng bút toán `OPENING`. Với các bút
toán sau:

```text
entry[n].balance_after
    = entry[n-1].balance_after + entry[n].points_delta
```

`account_sequence` phải tăng liên tục theo chính sách đã chọn và không trùng.
Không dùng `created_at` làm thứ tự duy nhất.

## 20. Thử nghiệm nhiều tác nhân

Khởi tạo một số dư hữu hạn, rồi chạy nhiều lệnh cộng/trừ khác nhau qua bộ thực
thi. Dùng `CountDownLatch` cho cổng bắt đầu và thu mọi kết quả/ngoại lệ trước khi
khẳng định.

Cuối test:

```text
points_balance >= 0
points_balance = opening + tổng delta đã chốt
số command thành công = số bút toán mới
không command nào có hơn một bút toán
mọi Future kết thúc trong thời gian tối đa
```

Không đặt một con số thông lượng làm chuẩn chung. Test tải dùng để quan sát tranh
chấp và tìm lỗi xác suất, không thay thế các test dòng thời gian có điều khiển.

## 21. Nhiều máy chủ ứng dụng

Hai luồng với hai kết nối đã kiểm tra ranh giới PostgreSQL. Để kiểm tra cấu hình,
có thể khởi động hai `ApplicationContext` cùng trỏ tới một container và không
chia sẻ bean khóa Java.

Kết quả phải giữ nguyên: số dư không âm, mã lệnh duy nhất, sổ cái khớp và không
có bút toán bị ghi đè.

## 22. Dữ liệu quan sát khi test tải

- số lệnh `EARNED`, `REDEEMED`, `INSUFFICIENT_POINTS` và phát lại;
- thời gian chờ khóa dòng tài khoản;
- số lỗi `55P03`, `40P01`, `40001`;
- thời gian giao dịch và số kết nối đang chờ;
- số lần thử lại trên mỗi mã lệnh;
- số chênh lệch đối soát;
- tài khoản có hàng đợi khóa dài nhất, qua log có kiểm soát.

Không dùng `customer_id` hay `command_id` thô làm nhãn metric.

## 23. Danh sách kiểm tra bộ thực nghiệm

- [ ] Dùng PostgreSQL Testcontainers và migration thật.
- [ ] Mỗi tác nhân có giao dịch và kết nối riêng.
- [ ] Rào chắn/chốt đặt tại đúng điểm đọc, ghi, chốt hoặc hoàn tác.
- [ ] Không dùng `Thread.sleep` làm cơ chế đồng bộ chính.
- [ ] Mọi lần chờ có thời gian tối đa.
- [ ] Tái hiện được tiêu hai lần và cộng/trừ ghi đè.
- [ ] Kiểm tra hai lệnh khác nhau và hai bản sao cùng lệnh.
- [ ] Kiểm tra sai dấu vân tay và phát lại kết quả thiếu điểm.
- [ ] Kiểm tra lỗi sau phép trừ, chốt, hoàn tác và hết thời gian chờ.
- [ ] Kiểm tra quyền chỉ thêm mới cùng các ràng buộc sổ cái.
- [ ] Khẳng định số dư, tổng sổ cái và quan hệ lệnh–bút toán.
- [ ] Có kiểm tra hoặc lập luận triển khai cho nhiều máy chủ.
