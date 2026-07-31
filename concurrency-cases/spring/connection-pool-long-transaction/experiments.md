# Các thí nghiệm với sự quá tải có giới hạn (Bounded saturation experiments)

## Mục tiêu

Bộ kiểm thử (test) phải chứng minh được:

1. Việc đứng chờ hệ thống từ xa (remote wait) ngay trong lòng giao dịch sẽ tiếp tục chiếm giữ các kết nối Hikari;
2. Kẻ đứng đợi khóa (lock waiter) của PostgreSQL cũng sẽ trắng trợn ngậm chặt một kết nối;
3. Một truy vấn ngắn không liên quan (unrelated short query) sẽ sụp đổ (fail) ngay ở khâu lấy kết nối (pool acquisition), chứ không phải do lỗi khi thực thi lệnh SQL;
4. Việc bóc tách ranh giới (split boundary) sẽ giải phóng hoàn toàn hồ kết nối trong suốt thời gian chờ hệ thống từ xa;
5. Giao dịch ở khâu chốt (commit phase) có khả năng từ chối (revalidate) các bản ảnh cũ kỹ rác rưởi (stale snapshot);
6. Tất thảy mọi tác vụ chờ đợi hay dọn dẹp kiểm thử (wait/test cleanup) đều phải có giới hạn điểm dừng (bounded).

> **Nói ngắn gọn:** Chúng ta sẽ chủ động dồn ép một hồ kết nối có dung lượng giới hạn (finite pool) phải đầy ứ ự bằng các cánh cổng rào (gates), tiến hành đo đạc lượng kết nối đang bận/đang chờ (active/pending connections), rồi dõng dạc xác minh trạng thái nghiệp vụ cuối cùng đã được commit (committed business state) sau khi đã dọn dẹp (cleanup).

## Cấu trúc bài kiểm thử (Test topology)

```text
sức chứa của hồ (pool max) = 2

Bị lỗi (broken):
  tác nhân A (actor A) -> Giao dịch Tx/conn-1 -> khóa dòng (row lock) -> đụng cửa rào chặn hệ thống từ xa (remote gate)
  tác nhân B (actor B) -> Giao dịch Tx/conn-2 -> khóa dòng -> đụng cửa rào hệ thống từ xa hoặc đứng ngây ra đợi khóa (lock wait)
  tác nhân U (actor U) -> đứng chờ hồ kết nối -> hết thời gian chờ mượn (acquisition timeout)

Đã sửa (fixed):
  tác nhân A/B -> giao dịch chụp ảnh (snapshot Tx) hoàn tất -> đụng cửa rào hệ thống từ xa (và hoàn toàn không giữ kết nối - no connection)
  tác nhân U   -> câu truy vấn ngắn gọn vượt qua trót lọt (success)
  thả tác nhân A/B (release A/B) -> chạy tiếp các giao dịch áp dụng ngắn ngủi (short apply transactions)
```

## PostgreSQL Testcontainers và thông số thiết lập Hikari (Hikari test profile)

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest(properties = {
    "spring.datasource.hikari.maximum-pool-size=2",
    "spring.datasource.hikari.minimum-idle=0",
    "spring.datasource.hikari.connection-timeout=400ms"
})
class LongTransactionPoolIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Mức ngắt ngưỡng `400ms` chỉ để bảo chứng cho bài test nhanh chóng sụp đổ (fail-fast) một cách rành rọt (deterministic); tuyệt đối đừng mang nó đem cài vào cấu hình thực tế (production setting).
Nghiêm cấm dùng annotation `@Transactional` trên các phương thức kiểm thử (test method).

Lược đồ (Schema):

```sql
create table payment_order (
    id uuid primary key,
    customer_id uuid not null,
    amount bigint not null check (amount > 0),
    status varchar(32) not null,
    risk_reason varchar(80),
    version bigint not null
);
```

Ở mỗi vòng thi, bộ kiểm thử sẽ rải dữ liệu mồi (insert committed fixtures) với mã P-42/P-99 ở trạng thái `RISK_PENDING`.

## Cổng chặn điều hướng hệ thống từ xa (Controlled remote gate)

```java
final class RemoteScenario {
    private final CountDownLatch entered;
    private final CountDownLatch release = new CountDownLatch(1);

    RemoteScenario(int expectedCalls) {
        this.entered = new CountDownLatch(expectedCalls);
    }

    void enterAndWait() {
        entered.countDown();
        awaitOrFail(release, Duration.ofSeconds(5));
    }

    void awaitAllEntered() {
        awaitOrFail(entered, Duration.ofSeconds(5));
    }

    void releaseAll() {
        release.countDown();
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for remote gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

```java
@Component
@Primary
class ControlledRiskClient implements RiskClient {
    private final AtomicReference<RemoteScenario> current =
        new AtomicReference<>();
    private final AtomicInteger calls = new AtomicInteger();

    @Override
    public RiskDecision assess(
        RiskSubject subject,
        Duration timeout
    ) {
        calls.incrementAndGet();
        RemoteScenario scenario = current.get();
        if (scenario != null) {
            scenario.enterAndWait();
        }
        return RiskDecision.approved(subject);
    }
}
```

Khối lệnh `finally` luôn bảo đảm sẽ hạ barie thả trôi lưu lượng (release scenario) trước lúc kết liễu bộ điều phối tác nhân (shutdown actor executor).

## Dụng cụ đo lường trạng thái hồ (Pool probe)

```java
@Component
final class PoolProbe {
    private final HikariDataSource hikari;

    PoolProbe(DataSource dataSource) throws SQLException {
        this.hikari = dataSource.unwrap(HikariDataSource.class);
    }

    int active() {
        return hikari.getHikariPoolMXBean().getActiveConnections();
    }

    int idle() {
        return hikari.getHikariPoolMXBean().getIdleConnections();
    }

    int pending() {
        return hikari.getHikariPoolMXBean()
            .getThreadsAwaitingConnection();
    }
}
```

Các chỉ số của hồ chứa (Pool metrics) có bản chất là những quan sát bị trễ nhịp (eventual observations). Vì thế ta phải bám nhờ thư viện Awaitility có giới hạn thời gian (bounded):

```java
await()
    .atMost(Duration.ofSeconds(2))
    .untilAsserted(() -> {
        assertThat(pool.active()).isEqualTo(2);
        assertThat(pool.idle()).isZero();
    });
```

Tuy vòng lặp kiểm tra của Awaitility (polling) có thời hạn (deadline) riêng; nhưng cổng chặn (gate) vẫn là cơ chế then chốt dệt nên trật tự xen kẽ các luồng (primary interleaving mechanism).

## Thí nghiệm 1 — Hai khoảng chờ từ xa (remote waits) vắt cạn cả hồ chứa

Ta rắp tâm sử dụng hai mã thanh toán (payment IDs) trái ngược nhau để triệt tiêu bài toán cạnh tranh dòng dữ liệu (row contention):

```java
@Test
void remoteWaitInsideTransactionsStarvesUnrelatedQuery()
        throws Exception {
    RemoteScenario scenario = riskClient.install(2);
    ExecutorService actors = Executors.newFixedThreadPool(3);

    try {
        Future<ApprovalResult> a = actors.submit(() ->
            broken.assessAndApprove(PAYMENT_42)
        );
        Future<ApprovalResult> b = actors.submit(() ->
            broken.assessAndApprove(PAYMENT_99)
        );

        scenario.awaitAllEntered();
        await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
            assertThat(pool.active()).isEqualTo(2);
            assertThat(pool.idle()).isZero();
        });

        Future<Throwable> unrelated = actors.submit(() ->
            catchThrowable(shortQuery::selectOne)
        );

        await().atMost(Duration.ofSeconds(2)).untilAsserted(() ->
            assertThat(pool.pending()).isEqualTo(1)
        );

        Throwable acquisitionFailure =
            unrelated.get(2, TimeUnit.SECONDS);

        assertThat(acquisitionFailure)
            .isInstanceOf(CannotGetJdbcConnectionException.class)
            .hasRootCauseInstanceOf(SQLTransientConnectionException.class);

        scenario.releaseAll();

        assertThat(a.get(5, TimeUnit.SECONDS).isApproved()).isTrue();
        assertThat(b.get(5, TimeUnit.SECONDS).isApproved()).isTrue();

        await().atMost(Duration.ofSeconds(2)).untilAsserted(() ->
            assertThat(pool.active()).isZero()
        );
    } finally {
        scenario.releaseAll();
        riskClient.clear();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Hàm `shortQuery.selectOne()` thậm chí không hề được cấp quyền phun một dòng SQL nào do nó bất lực trong việc mượn hồ (connection). Kiểm tra xác nhận ở tầng gốc rễ (Assertion root cause) rạch ròi phân loại lỗi không mượn được kết nối (pool acquisition failure) so với lỗi hết giờ chạy lệnh tại chính PostgreSQL (statement timeout).

## Thí nghiệm 2 — Đứng đợi khóa mà cũng trơ trẽn chiếm luôn kết nối (Lock waiter chiếm connection)

Cả A và B xâu xé chung một nạn nhân P-42. Thằng A đoạt được khóa rồi lẩn vào trốn trong cổng chặn (remote gate); Thằng B lôi được kết nối thứ hai từ hồ ra và rồi bị chôn chân cứng ngắc ở lệnh `FOR UPDATE`.

Chúng ta sử dụng một kết nối ngầm (admin connection) độc lập hoàn toàn khỏi application pool chuyên dành cho thao tác chuẩn đoán (diagnostics):

```java
long lockWaiterCount() {
    try (
        Connection connection = DriverManager.getConnection(
            POSTGRES.getJdbcUrl(),
            POSTGRES.getUsername(),
            POSTGRES.getPassword()
        );
        PreparedStatement statement = connection.prepareStatement(
            """
            select count(*)
            from pg_stat_activity
            where datname = current_database()
              and wait_event_type = 'Lock'
            """
        );
        ResultSet rows = statement.executeQuery()
    ) {
        rows.next();
        return rows.getLong(1);
    }
}
```

Trình tự test (Test sequence):

```java
@Test
void rowLockWaiterConsumesSecondPoolConnection() throws Exception {
    RemoteScenario firstCall = riskClient.install(1);
    ExecutorService actors = Executors.newFixedThreadPool(3);

    try {
        Future<ApprovalResult> holder = actors.submit(() ->
            broken.assessAndApprove(PAYMENT_42)
        );
        firstCall.awaitAllEntered(); // conn-1 ôm khóa dòng (owns row lock)

        Future<Throwable> waiter = actors.submit(() ->
            catchThrowable(() ->
                broken.assessAndApprove(PAYMENT_42)
            )
        );

        await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
            assertThat(pool.active()).isEqualTo(2);
            assertThat(lockWaiterCount()).isGreaterThanOrEqualTo(1);
            assertThat(riskClient.callCount()).isEqualTo(1);
        });

        Future<Throwable> unrelated = actors.submit(() ->
            catchThrowable(shortQuery::selectOne)
        );
        assertThat(unrelated.get(2, TimeUnit.SECONDS))
            .isInstanceOf(CannotGetJdbcConnectionException.class);

        firstCall.releaseAll();
        assertThat(holder.get(5, TimeUnit.SECONDS).isApproved()).isTrue();

        Throwable duplicateOutcome =
            waiter.get(5, TimeUnit.SECONDS);
        assertThat(duplicateOutcome)
            .isInstanceOf(IllegalStateException.class);
    } finally {
        firstCall.releaseAll();
        riskClient.clear();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Đoạn máy trạm ảo diệu (Controlled client) chỉ chặn gác cái yêu cầu thứ nhất thôi; kẻ trùng lặp (duplicate) sẽ tự động phi qua sau khi tên ôm khóa chịu nhả (holder commit). Sau đó mã bị lỗi (broken code) lại ngang nhiên đánh đu thực hiện lời gọi rủi ro ảo (unnecessary risk call) và rồi tức tưởi sụp đổ ở vòng thẩm định trạng thái (fails state validation). Điểm khẳng định mạnh mẽ nhất ở bài kiểm tra (Main assertion) là bằng chứng rõ mười mươi: 1 người cầm khóa đợi + 1 kẻ đứng xếp hàng đòi DB lock là đã dư sức thổi tung bóp chết cả cái pool (đã full pool).

Cái kết nối quản trị (Admin connection) đó không phải được dùng hòng phác thảo lấp liếm lỗi của môi trường thực (production workaround); nó chỉ tồn tại với tư cách là con mắt quan sát độc lập từ bên ngoài nhìn soi vào application pool.

## Thí nghiệm 3 — Khoảng chờ hệ thống từ xa sau khi sửa lỗi không còn ngâm kết nối nữa (Fixed remote waits không giữ connection)

```java
@Test
void splitBoundaryLeavesPoolAvailableDuringRemoteWait()
        throws Exception {
    RemoteScenario scenario = riskClient.install(2);
    ExecutorService actors = Executors.newFixedThreadPool(3);

    try {
        Future<ApprovalResult> a = actors.submit(() ->
            fixed.assessAndApprove(
                PAYMENT_42,
                testDeadline()
            )
        );
        Future<ApprovalResult> b = actors.submit(() ->
            fixed.assessAndApprove(
                PAYMENT_99,
                testDeadline()
            )
        );

        scenario.awaitAllEntered();

        await().atMost(Duration.ofSeconds(2)).untilAsserted(() -> {
            assertThat(pool.active()).isZero();
            assertThat(pool.pending()).isZero();
        });

        assertThat(shortQuery.selectOne()).isEqualTo(1);

        scenario.releaseAll();
        assertThat(a.get(5, TimeUnit.SECONDS).isApproved()).isTrue();
        assertThat(b.get(5, TimeUnit.SECONDS).isApproved()).isTrue();

        assertThat(inspector.status(PAYMENT_42))
            .isEqualTo(PaymentStatus.APPROVED);
        assertThat(inspector.status(PAYMENT_99))
            .isEqualTo(PaymentStatus.APPROVED);
    } finally {
        scenario.releaseAll();
        riskClient.clear();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Những giao dịch mang sứ mệnh chớp nhoáng (Snapshot transactions) đã tự động bay màu (end) nhường lối lại cho luồng tác nhân được rẽ bước vào cánh cổng trễ mạng (actors enter remote gate). Việc phát hiện pool active chỉ có `0` là những luận điểm mang đầy tính kỹ thuật sắc bén (technical evidence); đồng thời truy vấn qua đường không vướng bận chút nào (unrelated query success) là minh chứng hùng hồn nhất dưới con mắt thực thi (business/service evidence).

## Thí nghiệm 4 — Quá trình Thẩm định lại khước từ mọi loại Quyết định mốc meo (Revalidation từ chối stale decision)

```java
@Test
void concurrentChangeMakesRemoteDecisionStale() throws Exception {
    RemoteScenario scenario = riskClient.install(1);
    ExecutorService actor = Executors.newSingleThreadExecutor();

    try {
        Future<ApprovalResult> assessment = actor.submit(() ->
            fixed.assessAndApprove(PAYMENT_42, testDeadline())
        );

        scenario.awaitAllEntered(); // lúc này snapshot đang version 12, không có Tx mở
        canceller.cancel(PAYMENT_42); // bồi thêm short independent Tx -> chuyển version 13
        scenario.releaseAll();

        ApprovalResult result =
            assessment.get(5, TimeUnit.SECONDS);

        assertThat(result.isStaleDecision()).isTrue();
        assertThat(inspector.status(PAYMENT_42))
            .isEqualTo(PaymentStatus.CANCELLED);
        assertThat(inspector.version(PAYMENT_42)).isEqualTo(13L);
    } finally {
        scenario.releaseAll();
        riskClient.clear();
        actor.shutdownNow();
        assertThat(actor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Bài test thâm thúy này đã chặn đứng mầm bệnh thoái trào (regression) tai hại: “Đã bê remote ra ngoài Tx rồi mà não lại quên tái thẩm định lại dữ liệu (revalidate)”.

## Thí nghiệm 5 — Thời gian chờ từ xa báo lỗi (Remote timeout) nhưng không hề kéo theo sự mở ra của một luồng Giao dịch cập nhật (DB transaction)

Cài bẫy sao cho Client của bài test bị nổ lỗi `RiskDependencyTimeoutException` chủ động sau khi đập mặt vào ngưỡng giới hạn (bounded gate):

```java
@Test
void remoteTimeoutDoesNotOpenCommitTransaction() {
    assertThatThrownBy(() ->
        fixed.assessAndApprove(PAYMENT_42, expiredRemoteDeadline())
    ).isInstanceOf(RiskDependencyTimeoutException.class);

    assertThat(pool.active()).isZero();
    assertThat(inspector.status(PAYMENT_42))
        .isEqualTo(PaymentStatus.RISK_PENDING);
    assertThat(decisionWriter.invocationCount()).isZero();
}
```

Sứ mệnh đọc bản ảnh giao dịch (Snapshot read transaction) đã kịp thời đóng chốt (commit) trước khi lời gọi từ xa (remote call) gieo mình xuất phát; và giao dịch áp dụng cập nhật sau chót (apply transaction) sẽ chẳng dại gì mà mở bung ra một khi nhận thấy hệ thống đối tác vừa lăn đùng ra chết (dependency fails).

## Thí nghiệm 6 — Dọn dẹp tàn cuộc khi bị ngắt bởi quá hạn khóa dòng (Bounded lock timeout cleanup)

Thêm một bức tường thành bảo vệ vô cùng cứng cáp (trusted test guardrail) cho tiến trình (apply transaction) ở môi trường cấu hình test (test profile):

```sql
set local lock_timeout = '300ms';
```

Thử dùng một rào chắn độc lập (independent transaction) ôm chặt lấy P-42, và vẫy gọi luồng mã chữa cháy (fixed writer) rồi tiến hành giám sát (assert):

- Mũi tên exception/nguyên nhân bốc cháy (cause) đều khớp chính xác vào thông báo trạng thái nghẹt thở của PostgreSQL lock-timeout SQLSTATE;
- Giao dịch cập nhật dứt khoát rút lui (apply transaction rollback);
- Trạng thái bận của hồ lập tức nhảy lùi về lại mốc sàn ổn định (pool active trở về baseline);
- Danh tính và số hiệu trạng thái của dữ liệu bất di bất dịch (status/version không đổi);
- Quá trình đè nhấn thử lại (retry) cấm tuyệt đối việc mò vào thực thi ngay trên nền tảng của giao dịch chết yếu (failed transaction) vừa rồi.

Con số về thời lượng bị ngắt (Giá trị timeout) ở đây hoàn toàn là chiêu trò bày biện cho bộ kiểm thử (test fixture). Để ấn định những số liệu quy chuẩn ngoài đời thực (Production classification) phải luôn bám rễ dựa dẫm (derive) vào cội nguồn phát động mã trình điều khiển (driver cause/SQLSTATE) cộng với giới hạn còn sót lại của thời gian sống (remaining deadline).

## Thí nghiệm 7 — Kích thước của Pool (Pool size) mãi vô vọng trong việc trị bệnh thâm niên về thời gian dài cổ (long duration)

Cho phóng rầm rầm tập hợp các mô phỏng (parameterized test) bằng vô vàn các cỡ hồ (pool capacities) khiêm tốn khác nhau, luôn nhắm tới việc tạo ra chính xác lượng thợ chầu trực hệ thống xa (remote waiters) ăn khớp trọn vẹn với kích cỡ dung lượng (capacity). Đều đặn với mỗi mốc sức chứa:

```text
số kết nối bận (active) == kích thước cực đại (maximumPoolSize)
kẻ tới sau nhòm ngó mượn ké phải chết dí (unrelated borrower times out)
tuổi thọ của giao dịch (transaction duration) tò tò chạy theo y chang tuổi thọ của nút thắt ở cổng xa (tracks remote gate duration)
```

Không hề lợi dụng mớ kết quả (test) đong đếm này hòng xưng hùng xưng bá vạch ra một quy chuẩn (pool size tối ưu) cho thiên hạ. Tham vọng duy nhất là vạch trần chân lý: Dù có lấp đầy một cái rổ (finite capacity) lớn đến cỡ nào thì cũng chỉ là gom thêm một rổ những sự cố giam hãm (long transactions) trước khi cỗ máy gầm rú quá tải (saturation).

## Bảng đối chiếu các luồng dữ liệu bao phủ (Coverage matrix)

| Kịch bản (Scenario) | Kết nối đang bận trong lúc chờ từ xa (Active connections trong remote wait) | Kẻ đợi khóa (Lock waiter) | Truy vấn ất ơ (Unrelated query) |
| --- | --- | --- | --- |
| Lỗi (Broken), khác dòng xử lý | Chết đứng 2/2 | Báo Không | Quá hạn chờ mượn hồ (Acquisition timeout) |
| Lỗi (Broken), nã vào một dòng | Chết đứng 2/2 | Báo Có | Quá hạn chờ mượn hồ (Acquisition timeout) |
| Đã sửa: Chia đôi ranh giới | Dư giả 0/2 | Báo Không | Êm ru, trót lọt (Success) |
| Đã sửa: Chặn lén bản ảnh lỗi | Chỉ 0 lúc đang đợi; và áp dụng chớp nhoáng (short apply) | Mất một lúc (Short) | Phát hiện rác, chối ngay (Stale/no-op) |
| Lỗi ngắt chờ từ xa ngoài vùng Tx | Chỉ 0 tăm hơi sau khi ảnh chốt (sau snapshot) | Báo Không | Không mảy may tổn hại (Unaffected) |
| Sập bẫy ngắt khóa cập nhật (Apply lock timeout) | Bị bó gọn, chớp nhoáng (Bounded short Tx) | Báo Có | Hồ lại hồi phục sinh lực (Pool recovers) |

## Chống "nay chạy được, mai báo lỗi" (Chống flaky)

- Các đối tượng chốt chặn từ xa (Remote latches) như lời cam kết đinh ninh diễn viên đã trót lọt vượt qua lệnh truy vấn cơ sở dữ liệu (actors đã qua DB query) trước khi bị ép đông đá (block).
- Mọi mốc thời gian chốt chặn (latch), mốc chờ từ lai (future), công cụ mỏ neo Awaitility, và điểm triệt hạ đường dẫn executor đều trang bị rành rọt đếm ngược ngắt bỏ (timeout).
- Lệnh cấm kỵ: phải dỡ lệnh chặn hãm (release remote gate) trong khối `finally` trước khi nổ mìn dẹp tiệm (executor shutdown).
- Cấu hình file lớp test mặc định cắm rễ luôn theo con đường duy nhất `SAME_THREAD`; thiết bị ngụy trang kiểm soát (controlled client) có bản chất duy nhất kịch bản (single-scenario).
- Các phương thức kiểm nghiệm (Test method) sạch bóng lớp mạng che đậy bên ngoài (outer transaction).
- Sức chứa (Pool capacity) cũng như khung cài đặt cấu hình chỉ có tiếng nói độc quyền trong địa hạt bao trọn của bài kiểm tra (test context).
- Đầu mối móc ngoặc quản trị mạng (Direct admin connection) lúc nào cũng tuân thủ nghiêm ngặt tháo ngòi thắt nút khi hết việc bằng try-with-resources.
- Khối dựng mô hình rập khuôn (Fixtures) bắt buộc dùng khung đã đóng ván đóng đinh (committed setup) với những con số thẻ căn cước (IDs) rạch ròi không dính lứu.
- Luôn phải soi thấu tâm can (Assert) trên đủ 3 mặt trận: sự trinh bạch của hồ kết nối (pool state), bề mặt của các điểm bùng nổ (exception layer) và toàn vẹn nội dung của báo cáo tài chính đã an bài (committed payment state).

## Đánh giá mức độ xác thực của môi trường Production (Production verification)

Dashboard cần liên kết móc nối vạn vật trong một khung thời gian (correlate cùng time window):

```text
Chỉ số căng cứng Hikari active/max, số lượng vật vờ chờ mượn (pending), số lượng cạn sức từ bỏ (acquisition timeout)
Tốc độ hao mòn PostgreSQL xact age, hiện tượng thả rông idle-in-tx, đám tắc đường chờ khóa (lock waits/blockers)
Tốc độ ngậm từ xa (remote latency/timeouts), mức độ chen lấn của cổng vách ngăn (bulkhead active/queued/rejected)
Tổng hành dinh hoạt động luồng xử lý yêu cầu/ngồi chờ, hiện tượng đuổi việc cự tuyệt tiếp khách (rejection)
Tổng quân số máy móc (instance count) và toàn bộ những đường đi nước bước đã quy định (total configured connections)
```

Nghệ thuật soi mói hệ thống (Diagnostic SQL):

```sql
select a.pid,
       a.application_name,
       a.state,
       a.wait_event_type,
       a.wait_event,
       now() - a.xact_start as transaction_age,
       pg_blocking_pids(a.pid) as blockers
from pg_stat_activity a
where a.datname = current_database()
  and a.xact_start is not null
order by a.xact_start;
```

Còi báo động ở môi trường thực tế (Production alert) phải nhạy bén đôn sự ưu tiên phát hỏa trước những biểu đồ nhấp nhô của mốc thời gian nằm lì (rising transaction/connection usage duration) và đám lâu la chầu rìa đói khát (pending borrowers). Lấy số liệu quá hạn thời gian chầu chực xin hồ (Pool timeout) làm cơ sở chỉ huy là nước cờ muộn màng tụt hậu, báo hiệu cái bẫy sập đổ (cascade) đã thành hình.
