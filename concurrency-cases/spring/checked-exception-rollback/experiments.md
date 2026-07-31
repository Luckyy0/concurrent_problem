# Các thí nghiệm kiểm chứng với PostgreSQL (Integration experiments)

## Mục tiêu

Các bài kiểm thử (Tests) phải giúp phân biệt rõ ràng các khái niệm sau:

- Kết quả xử lý của phương thức Java (Java method outcome);
- Trạng thái hoàn thành của giao dịch trong Spring (Spring transaction completion status);
- Trạng thái dữ liệu đã thực sự được commit trong PostgreSQL (committed PostgreSQL state);
- Trạng thái mà một tác nhân (actor) khác có thể quan sát thấy được.

Chỉ dùng một kho dữ liệu ảo (Mock repository) hoặc soi mã nguồn tìm annotation chỉ mới giúp xác nhận cấu hình ở bề mặt. Case này bắt buộc phải gọi hạt đậu (bean) xuyên qua proxy và thực hiện truy vấn đọc lại cơ sở dữ liệu bằng một giao dịch mới hoàn toàn.

> **Nói ngắn gọn:** assert mỗi cái ngoại lệ là chưa đủ; bạn phải chứng minh được rằng cơ sở dữ liệu đã commit hay rollback, và các tiến trình điều phối (dispatcher) thực sự nhìn thấy cái gì.

## Cấu trúc bài kiểm thử (Test topology)

```text
test thread
  ├─ request executor -> Spring proxy -> Giao dịch Tx-A -> chạy qua policy gate -> ném exception
  ├─ dispatcher executor -> tạm dừng đợi Tx-A hoàn thành -> Giao dịch Tx-B -> query trạng thái PROCESSING
  └─ inspector bean -> Giao dịch độc lập Tx-C -> chụp lại ảnh dữ liệu đã được commit (committed-state snapshot)
```

Không được gắn annotation `@Transactional` vào các phương thức kiểm thử (test method). Nếu test framework vô tình bọc một giao dịch bên ngoài (outer transaction), phương thức service có thể sẽ âm thầm tham gia (join) vào nó, và rồi mọi kết quả chứng minh dữ liệu đã commit sẽ bị xóa sạch do cơ chế tự động rollback lúc test kết thúc.

## Sử dụng PostgreSQL Testcontainers

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class CheckedExceptionRollbackIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Tuyệt đối không dùng cơ sở dữ liệu trong bộ nhớ H2 để đưa ra kết luận về các hành vi như flush, khóa dòng (row lock) hay tính hiển thị của dữ liệu sau khi đã commit. Lược đồ cơ sở dữ liệu (Schema) tối thiểu cần thiết:

```sql
create table wallet_account (
    id uuid primary key,
    available_balance bigint not null
);

create table payout_request (
    id uuid primary key,
    wallet_id uuid not null references wallet_account(id),
    beneficiary_id uuid not null,
    amount bigint not null check (amount > 0),
    status varchar(32) not null
);

create table ledger_entry (
    id uuid primary key,
    payout_id uuid not null references payout_request(id),
    wallet_id uuid not null references wallet_account(id),
    amount bigint not null check (amount > 0),
    entry_type varchar(32) not null,
    constraint uk_ledger_payout_type
        unique (payout_id, entry_type)
);
```

Dữ liệu khởi tạo (Fixture setup commits):

```text
P-42: trạng thái RECEIVED, số tiền 300, người thụ hưởng đã bị cấm (blocked)
W-7:  số dư khả dụng (available_balance) là 1000
ledger: chưa có bản ghi nào liên quan đến P-42
```

Mỗi bài kiểm thử sẽ dùng các chuỗi ID riêng biệt hoặc sẽ tự động làm sạch (reset) lại dữ liệu ngay trong giao dịch thiết lập khởi tạo (committed setup transaction).

## Thiết bị thăm dò tiến độ giao dịch (Transaction completion probe)

Thiết bị thăm dò (Probe) chuyên dành cho kiểm thử sẽ được kích hoạt để lắng nghe sự kiện phản hồi (callback) của chính xác cái giao dịch đó:

```java
@Component
final class TransactionCompletionProbe {
    private final AtomicInteger completionStatus =
        new AtomicInteger(Integer.MIN_VALUE);

    void register() {
        completionStatus.set(Integer.MIN_VALUE);
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void afterCompletion(int status) {
                    completionStatus.set(status);
                }
            }
        );
    }

    int status() {
        return completionStatus.get();
    }
}
```

Service chịu trách nhiệm thiết lập (Fixture service) sẽ gọi phương thức `completionProbe.register()` ngay sau khi lọt qua (vào proxied method). Lúc lời gọi proxy (proxy call) kết thúc việc trả giá trị hoặc ném lỗi ra bên ngoài phương thức kiểm thử, phương thức `afterCompletion` đã kịp ghi nhận xong xuôi.

Các hằng số cần dùng lệnh xác nhận (assert):

```java
TransactionSynchronization.STATUS_COMMITTED
TransactionSynchronization.STATUS_ROLLED_BACK
```

Lưu ý: Công cụ thăm dò này chỉ nên phục vụ cho việc kiểm thử/chuẩn đoán, chứ tuyệt đối không phải là một cơ chế dùng cho mã nghiệp vụ (business mechanism).

## Kẻ giám sát trạng thái đã commit (Committed-state inspector)

Bản ảnh (Snapshot) bắt buộc phải được đọc thông qua một giao dịch mới hoàn toàn, nghiêm cấm việc tái sử dụng bối cảnh bền vững (persistence context) của cuộc gọi vừa mới bị lỗi:

```java
public record PayoutSnapshot(
    PayoutStatus status,
    long availableBalance,
    long holdCount
) {}
```

```java
@Service
class CommittedStateInspector {
    private final JdbcTemplate jdbc;

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    public PayoutSnapshot read(UUID payoutId, UUID walletId) {
        PayoutStatus status = PayoutStatus.valueOf(
            jdbc.queryForObject(
                "select status from payout_request where id = ?",
                String.class,
                payoutId
            )
        );
        long balance = jdbc.queryForObject(
            "select available_balance from wallet_account where id = ?",
            Long.class,
            walletId
        );
        long holds = jdbc.queryForObject(
            """
            select count(*)
            from ledger_entry
            where payout_id = ?
              and entry_type = 'PAYOUT_HOLD'
            """,
            Long.class,
            payoutId
        );
        return new PayoutSnapshot(status, balance, holds);
    }
}
```

## Thí nghiệm 1 — Quy tắc mặc định tự động commit những xử lý từ chối là lỗi checked

Hàm `broken.prepare()` sử dụng `@Transactional` mặc định và ngay lập tức bị chính sách từ chối (reject):

```java
@Test
void checkedBusinessExceptionCommitsWithDefaultRule() {
    assertThatThrownBy(() -> broken.prepare(PAYOUT_ID))
        .isExactlyInstanceOf(BeneficiaryRejectedException.class);

    PayoutSnapshot committed = inspector.read(PAYOUT_ID, WALLET_ID);

    assertThat(completionProbe.status())
        .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);
    assertThat(committed.status()).isEqualTo(PayoutStatus.PROCESSING);
    assertThat(committed.availableBalance()).isEqualTo(700L);
    assertThat(committed.holdCount()).isEqualTo(1L);
}
```

Bài test cố ý kiểm chứng lại hành vi sai lầm thực tế (asserts broken actual behavior). Nó như một tấm vé bảo hộ cho bộ tài liệu tránh khỏi một sai lầm chết người (test false-positive), nơi người ta chỉ nhăm nhăm kiểm tra có văng ra ngoại lệ là vội vã kết luận rằng "à, nó đã rollback an toàn rồi".

## Thí nghiệm 2 — `rollbackFor` đã khôi phục lại rào chắn tính đúng đắn

Cấu hình sửa lỗi (Fixed fixture) chỉ sửa đổi mỗi đoạn annotation:

```java
@Transactional(
    rollbackFor = BeneficiaryRejectedException.class
)
public void prepare(UUID payoutId)
        throws BeneficiaryRejectedException {
    completionProbe.register();
    preparationWork.prepare(payoutId);
}
```

Test kiểm chứng:

```java
@Test
void explicitRollbackRuleRollsBackEveryDatabaseMutation() {
    assertThatThrownBy(() -> fixed.prepare(PAYOUT_ID))
        .isExactlyInstanceOf(BeneficiaryRejectedException.class);

    PayoutSnapshot committed = inspector.read(PAYOUT_ID, WALLET_ID);

    assertThat(completionProbe.status())
        .isEqualTo(TransactionSynchronization.STATUS_ROLLED_BACK);
    assertThat(committed.status()).isEqualTo(PayoutStatus.RECEIVED);
    assertThat(committed.availableBalance()).isEqualTo(1000L);
    assertThat(committed.holdCount()).isZero();
    assertThat(dispatcher.hasExecutable(PAYOUT_ID)).isFalse();
}
```

Các lệnh xác nhận về nghiệp vụ (Business assertions) đóng vai trò sống còn so với việc chỉ xác nhận ngoại lệ: giao dịch chi trả không được lọt vào trạng thái có thể thực thi, dòng tiền không được phép bị giữ (hold) và bảng sổ cái tuyệt đối không xuất hiện thêm bất kỳ bản ghi nào mới (entry).

## Rào chắn đợi (Gate) nhằm kiểm soát thứ tự xen kẽ Request A / Dispatcher B

```java
final class RejectionGate {
    private final CountDownLatch policyEntered = new CountDownLatch(1);
    private final CountDownLatch allowRejection = new CountDownLatch(1);
    private final CountDownLatch allowDispatcherRead = new CountDownLatch(1);

    void policyEntered() {
        policyEntered.countDown();
        awaitOrFail(allowRejection, Duration.ofSeconds(5));
    }

    void awaitPolicyEntered() {
        awaitOrFail(policyEntered, Duration.ofSeconds(5));
    }

    void releaseRejection() {
        allowRejection.countDown();
    }

    void awaitDispatcherPermission() {
        awaitOrFail(allowDispatcherRead, Duration.ofSeconds(5));
    }

    void releaseDispatcher() {
        allowDispatcherRead.countDown();
    }

    void releaseAll() {
        allowRejection.countDown();
        allowDispatcherRead.countDown();
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for test gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

Một chính sách kiểm tra sẽ kích hoạt `gate.policyEntered()` rồi ném ra ngoại lệ checked. Mọi thao tác chờ (wait) bắt buộc phải có thời hạn (timeout); và trong khối `finally` sẽ thả nổi cả hai chiếc cổng bảo vệ (gates).

## Thí nghiệm 3 — Dispatcher nhìn thấy được trạng thái commit sai lầm

```java
@Test
void dispatcherCanObserveExecutableStateAfterCallerGetsRejection()
        throws Exception {
    RejectionGate gate = new RejectionGate();
    policy.install(gate, PAYOUT_ID);
    ExecutorService actors = Executors.newFixedThreadPool(2);

    try {
        Future<Throwable> requestOutcome = actors.submit(() ->
            catchThrowable(() -> broken.prepare(PAYOUT_ID))
        );

        gate.awaitPolicyEntered();

        Future<Boolean> dispatcherOutcome = actors.submit(() -> {
            gate.awaitDispatcherPermission();
            return dispatcher.hasExecutable(PAYOUT_ID);
        });

        gate.releaseRejection();
        Throwable rejected = requestOutcome.get(5, TimeUnit.SECONDS);

        assertThat(rejected)
            .isExactlyInstanceOf(BeneficiaryRejectedException.class);
        assertThat(completionProbe.status())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);

        gate.releaseDispatcher();
        assertThat(dispatcherOutcome.get(5, TimeUnit.SECONDS)).isTrue();
    } finally {
        gate.releaseAll();
        policy.clear();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Kết quả `requestOutcome` chỉ hoàn tất sau khi trình chặn giao dịch (transaction interceptor) đã commit và vừa văng ngoại lệ ra ngoài (rethrow). Vậy nên, giao dịch Tx-B của Dispatcher sẽ vô tình đọc luôn trạng thái đã được commit; trong bài kiểm tra chúng tôi hoàn toàn không đoán chừng về độ trễ (delay timing).

Khi chạy cùng đoạn lệnh này với `fixed.prepare()`, phải sửa ngay hai biến xác nhận (assertions):

```text
completion = ROLLED_BACK
dispatcherOutcome = false
```

## Thí nghiệm 4 — Có xả dữ liệu (Flush) sớm thì vẫn dư sức rollback lại được

Chúng ta sẽ tạo ra một bộ cấu trúc chỉnh sửa (fixed fixture), ép nó gọi `entityManager.flush()` ngay sau khi mới ghi bảng sổ cái nhưng chưa qua khâu kiểm tra policy:

```java
ledger.save(LedgerEntry.payoutHold(...));
entityManager.flush();
beneficiaryPolicy.verify(beneficiaryId);
```

Nhờ ném ra ngoại lệ checked đi kèm với thẻ `rollbackFor`, bộ kiểm thử vẫn tự tự tin xác nhận được trạng thái khởi tạo ban đầu (initial state) và mốc tín hiệu `STATUS_ROLLED_BACK`. Bài kiểm chứng này vạch trần một sự thật: các lệnh SQL đã được thi hành chưa hẳn đồng nghĩa với việc nó đã được chốt (durable commit).

Nếu một thanh tra (inspector) nhảy vào truy vấn trước khi có quyết định rollback, hắn ta có thể bị vướng phải khóa dòng (row lock) hoặc sẽ chỉ thấy phiên bản cũ đã được commit, tuỳ thuộc vào câu truy vấn; vì vậy, bài test chính luôn nên đọc lại dữ liệu sau khi tiến trình đã kết thúc rành mạch (completion) để còn tập trung soi xét tính bất biến của việc rollback (rollback invariant).

## Thí nghiệm 5 — Khối catch phía ngoài vấp phải cờ rollback-only

Hạt đậu (bean) nội bộ (inner fixed bean) bị gọi thông qua một lớp trung gian (proxy) và buộc phải kết thân (join) với giao dịch ngoài (outer transaction):

```java
@Service
class BatchFacade {
    private final FixedPayoutPreparationService preparation;

    @Transactional
    public BatchResult run(UUID payoutId) {
        try {
            preparation.prepare(payoutId);
            return BatchResult.ready();
        } catch (BeneficiaryRejectedException rejected) {
            return BatchResult.rejected();
        }
    }
}
```

Kết quả kỳ vọng (Expected):

```java
@Test
void swallowingInnerRollbackSurfacesUnexpectedRollbackAtOuterCommit() {
    assertThatThrownBy(() -> batch.run(PAYOUT_ID))
        .isInstanceOf(UnexpectedRollbackException.class);

    PayoutSnapshot committed = inspector.read(PAYOUT_ID, WALLET_ID);
    assertThat(committed.status()).isEqualTo(PayoutStatus.RECEIVED);
    assertThat(committed.availableBalance()).isEqualTo(1000L);
    assertThat(committed.holdCount()).isZero();
}
```

Bản chất ở đây là bảo vệ nguyên vẹn trạng thái chờ rollback (rollback state). Lỗi `UnexpectedRollbackException` là một bản cáo trạng nhắc nhở: mã lệnh ngoài (outer code) đã âm thầm kết thúc trót lọt dẫu cho giao dịch vật lý đã bó tay không thể nào commit nổi.

## Thí nghiệm 6 — Lập ý đồ ghi nhận kết quả là Từ chối (Committed `Rejected` result)

Áp dụng Giải pháp 3 (Solution 3):

```java
@Test
void expectedRejectionCommitsOnlyNonExecutableRejectedState() {
    PreparationResult result = resultBased.prepare(PAYOUT_ID);

    assertThat(result)
        .isEqualTo(new PreparationResult.Rejected(
            PAYOUT_ID,
            "BENEFICIARY_BLOCKED"
        ));

    PayoutSnapshot committed = inspector.read(PAYOUT_ID, WALLET_ID);
    assertThat(committed.status()).isEqualTo(PayoutStatus.REJECTED);
    assertThat(committed.availableBalance()).isEqualTo(1000L);
    assertThat(committed.holdCount()).isZero();
    assertThat(dispatcher.hasExecutable(PAYOUT_ID)).isFalse();
    assertThat(outbox.countReadyEvents(PAYOUT_ID)).isZero();
}
```

Đây là một sự cố tình đẩy dữ liệu đi thành công (commit) nhưng lại mang ý đồ hợp lý. Mã test hoàn toàn không kỳ vọng sẽ thấy một dòng báo lỗi văng ra hay xuất hiện tình huống rollback.

## Bảng đối chiếu các luồng dữ liệu bao phủ (Coverage matrix)

| Kịch bản (Scenario) | Kết quả phương thức (Method outcome) | Kết quả giao dịch (Tx outcome) | Trạng thái đã chốt (Committed state) |
| --- | --- | --- | --- |
| Mặc định + ném lỗi từ chối checked (checked rejection) | Bắn ra checked exception | Tiến hành Commit | Bị dính trạng thái sai lệch `PROCESSING` + kẹt tiền giữ (hold) |
| Dùng `rollbackFor` + ném lỗi từ chối checked | Bắn ra checked exception | Tiến hành Rollback | Nguyên thủy (Initial state) |
| Cố tình Flush sớm + dùng `rollbackFor` | Bắn ra checked exception | Tiến hành Rollback | Nguyên thủy (Initial state) |
| Rollback nội bộ, bên ngoài chặn bắt lại (outer catches) | Chết với `UnexpectedRollbackException` | Tiến hành Rollback | Nguyên thủy (Initial state) |
| Xử lý từ chối bằng kiểu cấu trúc (Result-based rejection) | Trả về chuỗi `Rejected` | Tiến hành Commit | Ghi nhận là `REJECTED`, tuyệt đối không giữ tiền (no hold) |
| Từ chối không kiểm tra (Unchecked rejection) | Văng ngoại lệ thời gian chạy (runtime) | Tiến hành Rollback | Nguyên thủy (Initial state) |

## Chống "nay chạy được, mai báo lỗi" (Chống flaky)

- Các đối tượng rào chắn (latch), hợp đồng hứa hẹn (future) và tiến trình dừng (executor termination) bắt buộc phải có thời hạn chốt chặn (bounded timeout).
- Khối lệnh `finally` cần nhấc toàn bộ cổng rào (release gates) lên trước khi rút ống thở tiến trình (shutdown executor).
- Tuyệt đối cấm lồng phương thức test trong transaction.
- Các thanh tra (Inspector) phải khăng khăng đòi dùng bối cảnh bền vững hoặc giao dịch mới toanh (persistence context/transaction).
- Luôn kiểm chứng trạng thái chốt hạ (transaction completion) cùng với thông tin trạng thái nghiệp vụ.
- Cấu hình chạy các file lớp test theo hình thức cùng tuyến luồng (`SAME_THREAD`) vì cơ chế gate của policy hay công cụ probe đòi hỏi được hoạt động riêng tư trong kịch bản duy nhất.
- Yêu cầu dùng hệ thống khởi tạo cứng (committed fixture setup) có sử dụng các dải số ID khác lạ, không màng đến những file cài cắm ở lớp test khác.
- Tuyệt đối truy vấn dữ liệu của Dispatcher xuyên qua proxy bean chứ chẳng dại gì thông qua bối cảnh quản lý đối tượng `EntityManager` đang dùng dở dang.
- Không được lạm dụng hàm trễ theo thời gian (time-based delay) để chế tác thứ tự chen ngang luồng xử lý (interleaving).

## Đánh giá mức độ xác thực của môi trường Production (Production verification)

Dữ liệu quan trọng nhất cần được bám sát theo dõi:

- Dò tìm phản hồi từ chối (rejected response) được ghim chung cùng payout ID hoặc bộ nhận dạng chuỗi (correlation ID);
- Quan sát tình trạng hoàn tất giao dịch trong môi trường đo đạc/sửa lỗi (diagnostic environment);
- Dò tìm những dòng hóa đơn payout nào bị đánh nhãn `REJECTED` nhưng vẫn còn kẹt tiền (hold) hoặc dính outbox — con số này bắt buộc bằng 0;
- Lượng kết quả hứa trả về toàn là exception nhưng khốn nỗi tình trạng lại là executable — con số này phải khăng khăng bằng 0;
- Đo đếm số lần nổ bom ngầm `UnexpectedRollbackException` xảy ra dọc vùng vành đai giới hạn ngoài cùng (outer boundaries);
- Bắt quả tang luồng dispatcher (tiến trình điều phối) có nhận kèo điều động bằng cách đếm theo state và giao dịch nguồn gây ra lỗi;
- Lỗi vượt rào cản độc quyền ghi (uniqueness violations) trên bảng outbox và sổ cái cùng với tỷ lệ những câu truy vấn bị gọi thành hai lần (duplicate command rate).

Gợi ý về câu lệnh dùng cho các buổi đối chiếu sổ sách (Reconciliation query):

```sql
select p.id, p.status, count(l.id) as hold_count
from payout_request p
left join ledger_entry l
  on l.payout_id = p.id
 and l.entry_type = 'PAYOUT_HOLD'
where p.status = 'REJECTED'
group by p.id, p.status
having count(l.id) > 0;
```

Nếu chạy truy vấn này, nó không có quyền trả về một tệp bất kỳ nào hết (trả về 0 hàng là qua ải). Nhấn mạnh rằng thao tác này chỉ xem là tấm lá chắn hậu cần vận hành bảo vệ, chứ đừng lấy ra làm kim chỉ nam thay thế hẳn cho những giao ước hợp đồng (transaction contract) cũng như các bộ test tích hợp chuyên sâu.
