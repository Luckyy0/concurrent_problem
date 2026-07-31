# Phòng Thí Nghiệm Đập Tan Ảo Tưởng: Chạy Thật Trên PostgreSQL (Deterministic PostgreSQL experiments)

## 1. Mục tiêu (Thí nghiệm để làm gì?)

Các bài Test phải chứng minh rõ ràng như ban ngày:

1. Ông A và B đang chạy trên hai Giao dịch (transactions) độc lập hoàn toàn.
2. Cả hai cú Đọc (SELECT) đều nhìn thấy con số ban đầu là `10`.
3. Ông A nhanh tay chốt số `13` TRƯỚC KHI ông B chốt đè số cũ rích `14`.
4. Mọi hàm gọi đều báo cáo Thành Công rực rỡ, NHƯNG chân lý cuối cùng (con số `17`) đã bị phá vỡ hoàn toàn.
5. Nếu dùng chiêu Cộng Nguyên Tử (atomic delta), hệ thống sẽ gộp đúng ra số `17`.
6. Nếu xài các chiêu Bắt Lỗi khác (detectors), hệ thống sẽ chủ động Chặn (block) hoặc Báo Lỗi Xung Đột (conflict) thay vì lẳng lặng đè bẹp (silent overwrite).

> **Nói ngắn gọn:** Mình sẽ gài bẫy (dùng `CountDownLatch` latches) ép tụi nó phải chạy theo thứ tự: "Cùng Đọc -> Khác lúc Ghi", rồi mở một Giao dịch Mới Toanh để đọc kết quả cuối cùng phơi bày sự thật.

## 2. Dùng Database Thật Bằng Testcontainers

Đừng có dại dột mà lôi H2 ra thử nghiệm trò này!

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class LostUpdateIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine"); // PostgreSQL 100% real
}
```

Nhớ kỹ: File Test này tuyệt đối KHÔNG GẮN `@Transactional` ở mức Class/Method. Dùng H2 In-Memory không thể làm Bằng Chứng Chứng Minh cho cách xử lý MVCC/Khóa (lock behavior) của PostgreSQL.

Bảng thiết kế ngây ngô gây họa (Schema broken variant):

```sql
create table job_progress (
    job_id uuid primary key,
    completed_units integer not null,
    total_units integer not null,
    constraint ck_job_progress_range check (
        total_units >= 0
        and completed_units >= 0
        and completed_units <= total_units
    )
);
```

Bơm dữ liệu ban đầu (Fixture setup commit):

```sql
insert into job_progress(job_id, completed_units, total_units)
values (:jobId, 10, 100);
```

## 3. Chốt Cửa Gài Bẫy (Gate ép both reads trước first commit)

Cái cổng gác này (Gate) làm nhiệm vụ bắt ép cả hai luồng (threads) phải ĐỌC XONG rồi mới được GHI.

```java
final class LostUpdateGate {
    private final UUID actorA;
    private final UUID actorB;
    private final CountDownLatch bothLoaded = new CountDownLatch(2);
    private final CountDownLatch allowA = new CountDownLatch(1);
    private final CountDownLatch allowB = new CountDownLatch(1);
    private final ConcurrentMap<UUID, Integer> observed =
        new ConcurrentHashMap<>();

    LostUpdateGate(UUID actorA, UUID actorB) {
        this.actorA = actorA;
        this.actorB = actorB;
    }

    void afterLoad(UUID actorId, int value) {
        observed.put(actorId, value);
        bothLoaded.countDown(); // Báo cáo: "Tui đọc xong rồi!"
        awaitOrFail(bothLoaded, Duration.ofSeconds(5)); // Đợi thằng kia cũng đọc xong!

        if (actorId.equals(actorA)) {
            awaitOrFail(allowA, Duration.ofSeconds(5)); // Chờ sếp cho phép chạy tiếp
        } else if (actorId.equals(actorB)) {
            awaitOrFail(allowB, Duration.ofSeconds(5)); // Chờ sếp cho phép chạy tiếp
        } else {
            throw new AssertionError("Ủa thằng ất ơ nào đây?");
        }
    }

    void awaitBothLoaded() {
        awaitOrFail(bothLoaded, Duration.ofSeconds(5));
    }

    void releaseA() {
        allowA.countDown();
    }

    void releaseB() {
        allowB.countDown();
    }

    void releaseAll() {
        allowA.countDown();
        allowB.countDown();
    }

    int observedBy(UUID actorId) {
        return Objects.requireNonNull(observed.get(actorId));
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Bị treo cmnr! (Timeout)");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Ai đá dây điện vậy?", interrupted);
        }
    }
}
```

Cái Cổng này chỉ dành riêng cho bài Test thôi nhé (test-only component). Service sẽ bị ép chèn đoạn code gọi `gate.afterLoad(...)` nằm chen giữa bước ĐỌC (Repository SELECT) và BƯỚC GHI ĐÈ.

## 4. Chụp X-Quang Giao Dịch (Transaction observation)

Để chắc chắn chúng ta đang xài đúng cấp độ Cô Lập, chọt thử cái Query này vào giữa Giao Dịch đang chạy:

```sql
select pg_current_xact_id()::text,
       current_setting('transaction_isolation');
```

Lưu thành Bản Cáo Trạng:

```java
public record TransactionObservation(
    UUID actorId,
    String transactionId, // Tên định danh của Giao dịch
    String isolation,     // Mức độ cách ly
    AtomicInteger completionStatus
) {}
```

Chơi lớn bằng cách đăng ký Listener (`TransactionSynchronization.afterCompletion`) để khẳng định rằng cả A và B cuối cùng đều Đã Chốt Sổ Thành Công (`STATUS_COMMITTED`).

## 5. Thí nghiệm 1 — Thảm Họa Mất Dữ Liệu Rõ Ràng Như Ban Ngày

```java
@Test
void staleAbsoluteWriteSilentlyOverwritesCommittedDelta()
        throws Exception {
    UUID actorA = UUID.randomUUID();
    UUID actorB = UUID.randomUUID();
    LostUpdateGate gate = probe.install(actorA, actorB);
    ExecutorService actors = Executors.newFixedThreadPool(2);

    try {
        Future<ProgressResult> a = actors.submit(() ->
            broken.addCompletedUnits(actorA, JOB_ID, 3)
        );
        Future<ProgressResult> b = actors.submit(() ->
            broken.addCompletedUnits(actorB, JOB_ID, 4)
        );

        gate.awaitBothLoaded(); // Ép hai thằng cùng bắt đầu Đọc
        assertThat(gate.observedBy(actorA)).isEqualTo(10); // Cả hai phải nhìn thấy số 10
        assertThat(gate.observedBy(actorB)).isEqualTo(10);

        gate.releaseA(); // Cho phép ông A lụm khóa và chạy trước
        ProgressResult resultA = a.get(5, TimeUnit.SECONDS);
        assertThat(resultA.after()).isEqualTo(13); // Ông A tưởng mình win

        gate.releaseB(); // Bây giờ mới thả cho ông B đè vào
        ProgressResult resultB = b.get(5, TimeUnit.SECONDS);
        assertThat(resultB.after()).isEqualTo(14); // Ông B cũng tưởng mình win!

        // X-Quang lại xem
        TransactionObservation txA = probe.transaction(actorA);
        TransactionObservation txB = probe.transaction(actorB);
        assertThat(txA.transactionId())
            .isNotEqualTo(txB.transactionId()); // Đúng là 2 giao dịch khác nhau
        assertThat(txA.isolation()).isEqualTo("read committed"); // Đúng mức độ
        assertThat(txB.isolation()).isEqualTo("read committed");
        assertThat(txA.completionStatus().get())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED); // Đều báo Thành công
        assertThat(txB.completionStatus().get())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);

        // KẾT CỤC BI THẢM
        JobProgressSnapshot finalState = inspector.read(JOB_ID);
        assertThat(finalState.completedUnits()).isEqualTo(14); // Bằng 14 (của B)
        assertThat(finalState.completedUnits())
            .isNotEqualTo(10 + 3 + 4); // Không bao giờ lên nổi 17
    } finally {
        gate.releaseAll();
        probe.reset();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Đoạn `a.get(timeout)` đảm bảo rằng Trình chặn Giao Dịch (transaction interceptor) của A đã chốt thành công 100% rồi mới cho qua. Vì vậy, B mà chạy sau đó là chắc chắn ốp đè lên bản chốt cuối cùng.

## 6. Thí nghiệm 2 — Phép Cộng Nguyên Tử (Atomic delta compose) Cứu Rỗi Thế Giới

Tạo cái Cổng xuất phát đơn giản thôi: "1,2,3 Chạy nha!":

```java
final class StartGate {
    private final CountDownLatch ready = new CountDownLatch(2);
    private final CountDownLatch start = new CountDownLatch(1);

    void readyAndAwaitStart() {
        ready.countDown(); // Đã sẵn sàng
        awaitOrFail(start, Duration.ofSeconds(5)); // Chờ sếp thổi còi
    }

    void awaitReady() {
        awaitOrFail(ready, Duration.ofSeconds(5));
    }

    void release() {
        start.countDown(); // Thổi còi!
    }
}
```

Chạy Test đập tan nỗi đau:

```java
@Test
void atomicDeltaPreservesBothConcurrentCommands() throws Exception {
    StartGate gate = atomicProbe.install(2);
    ExecutorService actors = Executors.newFixedThreadPool(2);

    try {
        Future<ProgressApplyResult> a = actors.submit(() -> {
            gate.readyAndAwaitStart();
            return atomic.addCompletedUnits(JOB_ID, 3); // Giao việc cho Database!
        });
        Future<ProgressApplyResult> b = actors.submit(() -> {
            gate.readyAndAwaitStart();
            return atomic.addCompletedUnits(JOB_ID, 4); // Giao việc cho Database!
        });

        gate.awaitReady(); // Điểm danh đủ
        gate.release(); // Chạy!

        assertThat(a.get(5, TimeUnit.SECONDS).isApplied()).isTrue();
        assertThat(b.get(5, TimeUnit.SECONDS).isApplied()).isTrue();

        // KẾT QUẢ CÔNG LÝ: 17
        JobProgressSnapshot finalState = inspector.read(JOB_ID);
        assertThat(finalState.completedUnits()).isEqualTo(17);
    } finally {
        gate.release();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Bài test này không thèm quan tâm A hay B ăn được cái Khóa Dòng (Row Lock) trước. Đứa nào tới trước thì được DB cộng dồn trước, đứa tới sau chờ tí rồi cũng được cộng dồn theo. Tuyệt đối không mất đi đâu miếng data nào (compose theo cả hai order).

## 7. Thí nghiệm 3 — Cú Chặn Mức Trần Chống Lố (Conditional cap)

Giả sử Số đã xong `95`, Tổng số `100`; Thằng A gởi tới `3`, thằng B gởi tới `4`:

```java
@Test
void atomicPredicatePreventsAcceptedTotalFromCrossingCap()
        throws Exception {
    ConcurrentApplyResults results =
        runAtomicDeltasFromSameStart(95, 100, 3, 4);

    // Dù có lao vào cùng lúc, chỉ có 1 thằng được duyệt thôi
    assertThat(results.appliedCount()).isEqualTo(1);
    assertThat(results.notAppliedCount()).isEqualTo(1); // Thằng kia bị đá văng

    int finalValue = inspector.read(JOB_ID).completedUnits();
    assertThat(finalValue).isIn(98, 99); // Trúng thằng nào thì lấy số thằng đó
    assertThat(finalValue).isLessThanOrEqualTo(100); // KHÔNG BAO GIỜ lố 100.
}
```

Tuyệt chiêu ở đây là: Thằng nào đứng Đợi Khóa xong, lúc được nhả Khóa ra, PostgreSQL sẽ THẨM ĐỊNH LẠI CÁI ĐIỀU KIỆN (re-evaluate predicate) trên dữ liệu MỚI NHẤT hiện tại. Lố số thì số dòng bị ảnh hưởng (affected row) = 0 => Ăn rổ hành (not applied). Không có thằng nào bị lừa là "Cập nhật rồi nha" trong khi Delta đã bốc hơi cả.

## 8. Thí nghiệm 4 — Bùa Hộ Mệnh `@Version` Gây Bão (Optimistic version conflict)

Thêm cái Cột `version` vào Bảng:

```sql
alter table job_progress
add column version bigint not null default 0;
```

Vẫn dùng cái Cửa Gài Bẫy (LostUpdateGate) ở bài 1, ép A và B cùng đọc lên bản có Version `7`. Ông A Chốt sổ thành công trước (Version sẽ nhích lên 8). Đến lượt ông B (đang ôm Version 7 trong bụng) cố xả hàng (flush):

```java
Throwable conflict = catchThrowable(() ->
    b.get(5, TimeUnit.SECONDS)
);

// Lôi ra ánh sáng! Phải quăng lỗi Lạc Quan
assertThat(rootCause(conflict))
    .isInstanceOfAny(
        OptimisticLockException.class,
        StaleObjectStateException.class
    );
```

Phía Spring sẽ bọc lỗi này thành `ObjectOptimisticLockingFailureException` ở tầng Service. Sau khi B bị Đạp (rollback), kết quả là:

```text
completed=13, version=8 (Chỉ có phần của A)
```

Giờ nếu bắt ông B Thử Lại (Retry) ĐÀNG HOÀNG từ đầu (nhớ khởi tạo Transaction mới hoàn toàn): Tải lại số `13 / Version 8`, rồi ốp cái cộng `4` vào, chốt sổ:

```text
completed=17, version=9 (Thành công mỹ mãn!)
```

Máy chụp X-Quang (Probe) sẽ báo cáo cái transaction ID lúc Retry của B và Hibernate session là khác biệt hoàn toàn với lúc nãy. Đừng bao giờ Cố Đấm Ăn Xôi (không retry) ngay trên cái Transaction đã bị gạch đít rách nát (rollback-only).

## 9. Thí nghiệm 5 — Chờ Đợi Bi Quan (Pessimistic read) Xếp Hàng Trước Khi Tính

Ông A xài bùa `findByIdForUpdate`, tải lên số `10`, rồi đứng im ở Cửa Gác. Ông B lúc này ráng Chạy, nhưng Bị Treo mỏ ngay lập tức ở câu Lệnh Chọn (locked repository query).

Check DB ngầm thì thấy ông B đang ăn bảng hiệu: `wait_event_type = 'Lock'`. Sau khi B thả ông A ra:

```text
Ông A Chốt sổ với con số 13.
Ông B mới được nhả Khóa, Lệnh Query lúc nãy sẽ tải lên được con số TƯƠI RÓI là 13.
Ông B nhẩm tính: 13 + 4 = 17, chốt sổ.
```

Kiểm chứng rành rành:

```java
assertThat(probe.loadedValue(actorA)).isEqualTo(10); // A đọc cũ
assertThat(probe.loadedValue(actorB)).isEqualTo(13); // B đọc mới, vì B phải Đợi A!
assertThat(inspector.read(JOB_ID).completedUnits()).isEqualTo(17); // KẾT QUẢ HOÀN HẢO
```

Nhớ đặt lố giờ (timeout) cho mấy trò Đợi Khóa này, và phải luôn nhả khóa (release) trong lệnh `finally` nhé!

## 10. Thí nghiệm 6 — Nâng Trình Lên `REPEATABLE READ`: Không Thích Ghi Đè, Chửi Nhau Thẳng Mặt

Hai transactions dùng:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

Lại ép cùng Đọc chung 1 hình (snapshot), A chốt trước B Ghi (UPDATE). Mong mỏi:

```text
Một ông ăn mừng rực rỡ (one commit)
Một ông khóc hận văng lỗi (one rollback with serialization failure)
Lỗi chình ình mã SQLSTATE = 40001
Tuyệt đối KHÔNG có chuyện Lặng im đè bẹp rồi cả hai ôm nhau cười (no silent two-success outcome).
```

Ăn lỗi vỡ mặt (abort) rồi, App bắt buộc phải Tự Động Thử Lại (retry) Nguyên 1 Cục Công Việc của B trong một Giao dịch Khác thì mới ra được con số `17`. Đừng có săm soi cái Câu báo lỗi (message text) vì Spring/Hibernate hay đổi lung tung, soi thẳng vào cái Mã Lỗi SQLSTATE (root cause) hoặc Sổ Sách Trạng Thái (committed business state) mới là người chơi chuyên nghiệp.

## 11. Thí nghiệm 7 — Hệ Thống Chạy Nhiều Máy Chủ (Multi-instance equivalent)

Đám Executor Threads nãy giờ xài Giao dịch độc lập rồi đó, nhưng nếu bạn bê kịch bản đó thảy lên 2 cái Máy Chủ Ứng Dụng riêng biệt (trỏ chung 1 con PostgreSQL), thì Kết Quả Thảm Họa `14` VẪN XẢY RA! Trò ôm khóa `synchronized` trong Java đâm ra vô dụng vì chả liên quan gì nhau.

Vì thế, xài Chiêu của Database thì dù 1 máy hay 100 máy chủ (single or multi-instance) kết quả vẫn Đúng-Và-Chuẩn.

## 12. Công Cụ Lấy Số Cuối Cùng (Inspector)

```java
@Service
class JobProgressInspector {
    private final JdbcTemplate jdbc;

    // NHẤT ĐỊNH PHẢI ĐẺ RA MỘT GIAO DỊCH MỚI HOÀN TOÀN (REQUIRES_NEW)
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    JobProgressSnapshot read(UUID jobId) {
        return jdbc.queryForObject(
            """
            select completed_units, total_units
            from job_progress
            where job_id = ?
            """,
            (rs, rowNum) -> new JobProgressSnapshot(
                rs.getInt(1),
                rs.getInt(2)
            ),
            jobId
        );
    }
}
```

Thằng này (Inspector) chỉ được chạy sau khi mấy cái ẩu đả ở trên kết thúc và luôn xài Khung Bộ Nhớ Đệm tươi mới (persistence context mới).

## 13. Bảng Vàng Thành Tích (Coverage matrix)

| Kịch Bản | Dữ Liệu Các Bên Đọc Lên | Hành Vi Xung Đột | Chốt Cuối |
| --- | --- | --- | --- |
| Lỗi ngớ ngẩn (Broken JPA) | A=10, B=10 | Chả có gì; Ôm nhau chết chùm | 14 |
| Cộng DB (Atomic delta) | Lấy giá trị Hiện Tại Của Dòng | Đợi nhau rồi Cộng Dồn mượt mà | 17 |
| Ràng Buộc Trần (Conditional cap)| DB Check Lại Điều kiện | Có 1 ông nhịn đói (affected row 0) | 98 hoặc 99 |
| Thẻ Bùa `@Version` Không Retry | Cùng đọc v7 | B Văng Lỗi Sấp Mặt | 13/v8 |
| Thẻ Bùa `@Version` Tự Retry | B Tải lại bản v8 | Đánh vật rồi Tới Đích thành công | 17/v9 |
| Khóa Bi Quan `FOR UPDATE` | A=10, B=13 | B Bị Dừng Chân Trước Cửa | 17 |
| Cô lập Repeatable read | Cùng 1 ảnh chụp | B Bị nổ Bom (SQLSTATE 40001) | Tôn trọng lẽ phải |

## 14. Nguyên Tắc Cốt Lõi Để Viết Test Không Chập Chờn (Chống flaky)

- Cái cổng (Gate) bảo kê thứ tự phải Chuẩn: `Cả hai Đọc 10 < A Chốt < B Đè Mù Quáng`.
- Cái gì có chờ đợi (latch, future, Awaitility) LÀ PHẢI CÓ GIỚI HẠN GIỜ (timeout).
- Cho vô khung `finally` việc mở cửa, dọn dẹp trước khi sập cầu dao (shutdown).
- Test class phải chạy cùng Luồng Nhất Quán (`SAME_THREAD`); Đồ soi (probes) chỉ chơi kịch bản xài 1 lần.
- Đừng có dán cái outer transaction bọc ngoài cục test method. Cấm!
- Bơm dữ liệu (Fixture setup) và Bới kết quả (inspector) cũng phải dùng transaction mới toanh!
- Xài trò Cộng DB (commutative atomic) thì khỏi cần quan tâm 100% rốt cuộc ai chốt trước.
- **KHÔNG ĐƯỢC XÀI H2 thay thế mặt mũi PostgreSQL.**
- Xác thực chân lý bằng con số tổng cuối cùng (final invariant), chứ đừng dựa vào việc bắt mỗi mấy cái cục Lỗi (exceptions).

## 15. Kiểm Kê Cuối Năm Trêm Production (Production verification)

Muốn Lục lọi đếm xác lại (Reconciliation) mà có lưu Vết thì chơi SQL này:

```sql
select p.job_id,
       p.completed_units,
       coalesce(sum(c.delta), 0) as accepted_delta
from job_progress p
left join progress_command c on c.job_id = p.job_id
where p.job_id = :jobId
group by p.job_id, p.completed_units;
```

So sánh con số trên bảng (projection) với Con Số Đếm Tay (sum accepted distinct deltas) trong sổ. Dõi theo mấy vụ Dòng-Ảnh-Hưởng bằng Không, Các thể loại Đụng Khóa (optimistic/serialization conflicts), Mấy con số nhảy múa điên cuồng (hot keys) hay các màn đánh lừa Cô Lập. Trò Ghi Đè Lẳng Lặng (Silent lost update) MÃI MÃI KHÔNG bao giờ sinh ra Đồ Thị Lỗi (exception metric riêng) để bạn biết mà khóc đâu!
