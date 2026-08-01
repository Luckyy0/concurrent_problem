# Phòng Thí Nghiệm: Khảo Sát Cơ Chế MVCC Trên PostgreSQL (Deterministic PostgreSQL experiments)

## 1. Mục tiêu kiểm thử

Các bài kiểm thử (Test) cần thể hiện một cách khoa học các nguyên lý sau:

1. Giao dịch A và B khởi tạo và thi hành trên hai Giao dịch (transactions) hoàn toàn riêng biệt.
2. Cả hai thao tác Đọc (SELECT) đều nhận được số liệu gốc là `10`.
3. Giao dịch A hoàn tất (commit) giá trị `13` TRƯỚC KHI giao dịch B ghi đè dữ liệu cũ bằng giá trị `14`.
4. Mọi hàm xử lý trong ứng dụng đều báo cáo hoàn thành, TUY NHIÊN tính toàn vẹn của kết quả (tổng `17`) bị sai lệch hoàn toàn.
5. Việc áp dụng Phép Cộng Nguyên Tử (atomic delta) sẽ điều phối chính xác dữ liệu ra kết quả `17`.
6. Việc sử dụng các giải pháp Bắt lỗi (detectors) sẽ chủ động chặn yêu cầu (block) hoặc gửi cảnh báo Xung đột (conflict), loại trừ tình trạng ghi đè âm thầm (silent overwrite).

> **Nói ngắn gọn:** Chúng ta sẽ thiết lập một Rào cản (sử dụng `CountDownLatch`) để ép các luồng chạy theo đúng trình tự: "Cùng Đọc -> Khác thời điểm Ghi", sau đó sử dụng một Giao dịch Mới hoàn toàn để đối chiếu mức độ toàn vẹn của kết quả.

## 2. Thiết Lập PostgreSQL Thực Tế Bằng Testcontainers

Bắt buộc sử dụng hệ quản trị CSDL thực thay vì các CSDL giả lập bộ nhớ (như H2):

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class LostUpdateIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine"); // PostgreSQL thực tế
}
```

Lưu ý: Không được gán annotation `@Transactional` vào toàn bộ Class hay Phương thức Test. Việc giả lập H2 trong bộ nhớ (In-Memory) không thể phản ánh chính xác các cơ chế MVCC và Quản lý khóa (lock behavior) của PostgreSQL.

Cấu trúc Bảng chứa lỗi (Schema broken variant):

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

Dữ liệu ban đầu (Fixture setup commit):

```sql
insert into job_progress(job_id, completed_units, total_units)
values (:jobId, 10, 100);
```

## 3. Rào Cản Điều Phối Luồng (Gate ép both reads trước first commit)

Rào cản này (Gate) sẽ có vai trò chặn cả hai luồng (threads) để chúng phải hoàn tất thao tác ĐỌC trước khi có thể GHI.

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
        bothLoaded.countDown(); // Thông báo: "Thao tác đọc hoàn tất"
        awaitOrFail(bothLoaded, Duration.ofSeconds(5)); // Chờ luồng còn lại thao tác đọc xong

        if (actorId.equals(actorA)) {
            awaitOrFail(allowA, Duration.ofSeconds(5)); // Chờ chỉ thị tiếp tục
        } else if (actorId.equals(actorB)) {
            awaitOrFail(allowB, Duration.ofSeconds(5)); // Chờ chỉ thị tiếp tục
        } else {
            throw new AssertionError("Luồng định danh không hợp lệ");
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
                throw new AssertionError("Lỗi treo ứng dụng do hết thời gian (Timeout)");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Xảy ra gián đoạn khi đang chờ (Interrupt)", interrupted);
        }
    }
}
```

Thành phần Cổng Rào Cản này là công cụ đặc thù dành riêng cho việc kiểm thử (test-only component). Trong Service sẽ chèn thêm mã thực thi `gate.afterLoad(...)` chen giữa quá trình ĐỌC (Repository SELECT) và GHI ĐÈ.

## 4. Công Cụ Giám Sát Giao Dịch (Transaction observation)

Để đảm bảo mức độ Cô Lập chuẩn xác, chúng ta nhúng một Query sau vào Giao Dịch:

```sql
select pg_current_xact_id()::text,
       current_setting('transaction_isolation');
```

Tạo một cấu trúc lưu trữ thông tin Giám Sát:

```java
public record TransactionObservation(
    UUID actorId,
    String transactionId, // Định danh của Giao dịch
    String isolation,     // Mức độ cách ly
    AtomicInteger completionStatus
) {}
```

Bổ sung tính năng theo dõi thông qua Listener (`TransactionSynchronization.afterCompletion`) để xác thực tiến trình của hai Giao dịch A và B đều kết thúc hợp lệ (`STATUS_COMMITTED`).

## 5. Thí nghiệm 1 — Kiểm Chứng Sự Cố Mất Dữ Liệu Điển Hình

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

        gate.awaitBothLoaded(); // Đảm bảo cả hai luồng đã tiến hành Đọc
        assertThat(gate.observedBy(actorA)).isEqualTo(10); // Đều phải tải lên dữ liệu là 10
        assertThat(gate.observedBy(actorB)).isEqualTo(10);

        gate.releaseA(); // Cấp phát tín hiệu cho A tiếp tục xử lý
        ProgressResult resultA = a.get(5, TimeUnit.SECONDS);
        assertThat(resultA.after()).isEqualTo(13); // A hoàn tất với 13

        gate.releaseB(); // Bây giờ nhả khóa cho B xử lý đè lên
        ProgressResult resultB = b.get(5, TimeUnit.SECONDS);
        assertThat(resultB.after()).isEqualTo(14); // B hoàn tất với 14

        // Đối chiếu hồ sơ Giao dịch
        TransactionObservation txA = probe.transaction(actorA);
        TransactionObservation txB = probe.transaction(actorB);
        assertThat(txA.transactionId())
            .isNotEqualTo(txB.transactionId()); // Khẳng định 2 giao dịch phân biệt
        assertThat(txA.isolation()).isEqualTo("read committed"); // Đúng mức Cô lập yêu cầu
        assertThat(txB.isolation()).isEqualTo("read committed");
        assertThat(txA.completionStatus().get())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED); // Cả hai báo Thành công
        assertThat(txB.completionStatus().get())
            .isEqualTo(TransactionSynchronization.STATUS_COMMITTED);

        // KẾT CỤC: THẤT THOÁT DỮ LIỆU
        JobProgressSnapshot finalState = inspector.read(JOB_ID);
        assertThat(finalState.completedUnits()).isEqualTo(14); // Trạng thái cuối mang số 14
        assertThat(finalState.completedUnits())
            .isNotEqualTo(10 + 3 + 4); // Toàn vẹn hệ thống bị vi phạm, không ra 17
    } finally {
        gate.releaseAll();
        probe.reset();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Hàm `a.get(timeout)` giữ nhiệm vụ chờ cho Trình Chặn (transaction interceptor) của giao dịch A đạt mốc xác nhận (commit) hoàn tất. Nên khi giao dịch B tiếp tục chạy, nó chắc chắn ghi đè trực tiếp kết quả.

## 6. Thí nghiệm 2 — Phép Cộng Nguyên Tử (Atomic delta compose) Khắc Phục Lỗi

Tạo ra một Rào cản Đồng thời ("Cùng bắt đầu"):

```java
final class StartGate {
    private final CountDownLatch ready = new CountDownLatch(2);
    private final CountDownLatch start = new CountDownLatch(1);

    void readyAndAwaitStart() {
        ready.countDown(); // Đã sẵn sàng
        awaitOrFail(start, Duration.ofSeconds(5)); // Chờ chỉ thị tiến hành
    }

    void awaitReady() {
        awaitOrFail(ready, Duration.ofSeconds(5));
    }

    void release() {
        start.countDown(); // Bắt đầu tiến trình
    }
}
```

Tiến hành Kiểm thử:

```java
@Test
void atomicDeltaPreservesBothConcurrentCommands() throws Exception {
    StartGate gate = atomicProbe.install(2);
    ExecutorService actors = Executors.newFixedThreadPool(2);

    try {
        Future<ProgressApplyResult> a = actors.submit(() -> {
            gate.readyAndAwaitStart();
            return atomic.addCompletedUnits(JOB_ID, 3); // Giao phó DB thực hiện phép tính cộng
        });
        Future<ProgressApplyResult> b = actors.submit(() -> {
            gate.readyAndAwaitStart();
            return atomic.addCompletedUnits(JOB_ID, 4); // Giao phó DB thực hiện phép tính cộng
        });

        gate.awaitReady(); // Hai luồng đã sẵn sàng
        gate.release(); // Kích hoạt chạy đồng thời

        assertThat(a.get(5, TimeUnit.SECONDS).isApplied()).isTrue();
        assertThat(b.get(5, TimeUnit.SECONDS).isApplied()).isTrue();

        // KẾT QUẢ ĐẠT TOÀN VẸN: 17
        JobProgressSnapshot finalState = inspector.read(JOB_ID);
        assertThat(finalState.completedUnits()).isEqualTo(17);
    } finally {
        gate.release();
        actors.shutdownNow();
        assertThat(actors.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Kiểm thử này độc lập với việc Luồng nào chiếm giữ Khóa Dòng (Row Lock) trước tiên. Hệ thống DB sẽ giải quyết thứ tự thi hành và duy trì kết quả cộng dồn nguyên tử. Dữ liệu của hai bên hoàn toàn được bảo toàn (compose theo cả hai order).

## 7. Thí nghiệm 3 — Điều Kiện Cản Trở Chống Quá Tải (Conditional cap)

Nếu Số hoàn thành hiện tại là `95`, Hạn mức tổng `100`; Giao dịch A gửi Yêu cầu `3`, Giao dịch B gửi `4`:

```java
@Test
void atomicPredicatePreventsAcceptedTotalFromCrossingCap()
        throws Exception {
    ConcurrentApplyResults results =
        runAtomicDeltasFromSameStart(95, 100, 3, 4);

    // Mặc dù tương tranh cùng lúc, điều kiện DB sẽ chỉ cấp quyền cập nhật cho 1 luồng
    assertThat(results.appliedCount()).isEqualTo(1);
    assertThat(results.notAppliedCount()).isEqualTo(1); // Yêu cầu luồng còn lại bị bác bỏ

    int finalValue = inspector.read(JOB_ID).completedUnits();
    assertThat(finalValue).isIn(98, 99); // Kết quả sẽ phụ thuộc vào luồng được xử lý trước
    assertThat(finalValue).isLessThanOrEqualTo(100); // KHÔNG VƯỢT QUÁ HẠN MỨC
}
```

Nguyên lý bảo vệ ở đây: Luồng nào chịu trách nhiệm chờ (wait) do bị Khóa, lúc Khóa được giải phóng PostgreSQL sẽ tiến hành ĐÁNH GIÁ LẠI ĐIỀU KIỆN (re-evaluate predicate) dựa trên dữ liệu lưu trữ MỚI NHẤT. Nếu giới hạn bị vi phạm, chỉ số dòng thay đổi (affected row) sẽ rơi vào = 0 (not applied). Nhờ đó, ứng dụng hoàn toàn có thể kiểm soát và từ chối cập nhật khi hết dung lượng phân bổ.

## 8. Thí nghiệm 4 — Giải Pháp Khóa Lạc Quan Sinh Xung Đột (Optimistic version conflict)

Khai báo bổ sung trường `version` vào bảng:

```sql
alter table job_progress
add column version bigint not null default 0;
```

Tái sử dụng Rào cản từ kiểm thử số 1 (LostUpdateGate). Ép hai giao dịch A và B tải dữ liệu ở cùng phiên bản Version `7`. A chốt sổ hoàn thành trước (Version tự động tăng lên 8). Đến phiên B (đang mang Version 7 ở dữ liệu đệm) cố gắng đồng bộ thay đổi:

```java
Throwable conflict = catchThrowable(() ->
    b.get(5, TimeUnit.SECONDS)
);

// Bắt ngoại lệ! Yêu cầu từ chối Khóa Lạc Quan
assertThat(rootCause(conflict))
    .isInstanceOfAny(
        OptimisticLockException.class,
        StaleObjectStateException.class
    );
```

Tầng Spring sẽ bao bọc các lỗi ngoại lệ (exception) này dưới cái tên `ObjectOptimisticLockingFailureException`. Sau khi giao dịch của B bị Hủy bỏ (rollback), DB bảo lưu kết quả:

```text
completed=13, version=8 (Phần cập nhật từ A)
```

Lúc này tiến trình ứng dụng bắt buộc B thử lại từ đầu (tạo Transaction mới). B sẽ nạp bộ nhớ với số `13 / Version 8`, chạy tính toán cộng `4` và gửi dữ liệu chốt:

```text
completed=17, version=9 (Hoàn thành một cách hợp lệ)
```

Giám Sát (Probe) xác thực Mã Định Danh Transaction khi B Retry là phiên bản mới. Chúng ta không bao giờ được phép duy trì hay sửa chữa nghiệp vụ ngay trên Giao dịch đã phát sinh Hủy Bỏ (rollback-only).

## 9. Thí nghiệm 5 — Khóa Bi Quan Đóng Chờ Đồng Bộ (Pessimistic read)

Giao dịch A sử dụng `findByIdForUpdate` và tải giá trị `10`. Ngay khi A đang bị giữ ở Rào cản, B cố gắng thực thi truy vấn nhưng ngay lập tức Gặp Tình Trạng Chờ (locked repository query).

Động thái kiểm tra DB sẽ xuất hiện nhãn: `wait_event_type = 'Lock'`. Sau khi A được phép xử lý xong và giải phóng:

```text
A chốt lưu dữ liệu 13.
B nhận quyền Khóa, thực thi Lệnh Query đã lên lịch và lúc này nạp giá trị hiện tại là 13.
B thực hiện tính toán: 13 + 4 = 17 và chốt kết quả.
```

Minh chứng bằng số liệu:

```java
assertThat(probe.loadedValue(actorA)).isEqualTo(10); // A đọc giá trị cũ
assertThat(probe.loadedValue(actorB)).isEqualTo(13); // B đọc giá trị mới (Do đợi A kết thúc)
assertThat(inspector.read(JOB_ID).completedUnits()).isEqualTo(17); // KẾT QUẢ ĐẠT CHUẨN
```

Lưu ý: Luôn áp đặt hạn mức thời gian (timeout) cho các tác vụ xếp hàng Chờ Khóa, và tất cả thao tác mở khóa đều nằm ở khối `finally`.

## 10. Thí nghiệm 6 — Mức Độ Cô Lập `REPEATABLE READ`: Sinh Lỗi Xung Đột

Hai giao dịch áp dụng cấu hình:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

Sử dụng Rào Cản ép Đọc cùng thời điểm, sau đó A cập nhật thành công (commit). Khi B tiến hành cập nhật (UPDATE):

```text
Một giao dịch hoàn tất (one commit)
Một giao dịch từ chối tiến trình (one rollback with serialization failure)
Ghi nhận lỗi kèm theo mã SQLSTATE = 40001
Cơ chế ngăn ngừa toàn diện các rủi ro ghi đè âm thầm (no silent two-success outcome).
```

Vì vấp phải ngoại lệ huỷ bỏ (abort), luồng B phải cấu hình tự gọi Thử lại (retry) trong một Giao dịch Mới hoàn toàn để hệ thống khôi phục kết quả tính toán `17`. Đừng đặt điều kiện xử lý dựa vào Thông Báo Văn Bản (message text), hãy bám sát Mã SQLSTATE (root cause) để thiết kế hệ thống vững chắc.

## 11. Thí nghiệm 7 — Thiết Kế Cụm Ứng Dụng (Multi-instance equivalent)

Trong trường hợp luồng giao dịch được đặt tại 2 Máy Chủ Ứng Dụng riêng biệt truy vấn về một DB duy nhất, Hệ quả Lỗi số liệu `14` VẪN XẢY RA nếu thiếu cơ chế đồng bộ CSDL. Việc ứng dụng mã Java `synchronized` để bảo vệ tài nguyên trở nên vô dụng vì nó chỉ nằm trên máy chủ đơn thuần.

Việc vận dụng các Cú pháp Cập nhật Nguyên Tử, hoặc Khóa trong cấp độ CSDL sẽ giúp mô hình đảm bảo Toàn Vẹn nghiệp vụ ngay trong bất kể khối lượng Máy chủ mở rộng (single or multi-instance).

## 12. Công Cụ Trích Xuất Phân Tích (Inspector)

```java
@Service
class JobProgressInspector {
    private final JdbcTemplate jdbc;

    // YÊU CẦU THIẾT LẬP MỘT GIAO DỊCH MỚI HOÀN TOÀN TRƯỚC KHI TRUY VẤN (REQUIRES_NEW)
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

Luồng (Inspector) đảm bảo dữ liệu khi được trích xuất hoàn toàn không dựa trên các Ảnh Chụp hoặc Bộ Nhớ Đệm Cũ kỹ của Giao dịch trước.

## 13. Phân Phối Ma Trận Kiểm Định (Coverage matrix)

| Kịch Bản Thiết Lập | Kết Quả Đọc | Hành Vi Xử Lý Xung Đột | Kết Quả Chốt |
| --- | --- | --- | --- |
| Quy trình Hỏng (Broken JPA) | A=10, B=10 | Không nhận diện được; Ghi Đè Lỗi | 14 |
| Cộng Nguyên Tử DB (Atomic delta) | Lấy giá trị Hiện Tại trên Dòng | Chờ Khóa rồi Cộng Dồn Nguyên Tử | 17 |
| Ràng Buộc (Conditional cap)| DB Tái Kiểm tra Lại Điều kiện | Luồng đến muộn bị vô hiệu hoá (affected row 0) | 98 hoặc 99 |
| Khóa Lạc Quan `@Version` Không Retry | Đọc Phiên Bản v7 | B vấp Lỗi Xung Đột Lạc Quan | 13/v8 |
| Khóa Lạc Quan `@Version` Tự Retry | B Tải Phiên Bản v8 mới | Khởi tạo Lại Giao Dịch, Xử Lý | 17/v9 |
| Khóa Bi Quan `FOR UPDATE` | A=10, B=13 | B Phải Đợi Ở Điểm Truy Vấn Khóa | 17 |
| Cô lập Repeatable read | Sử Dụng Cùng Ảnh Chụp | B Đối diện Lỗi (SQLSTATE 40001) | Tôn trọng lẽ phải |

## 14. Nguyên Tắc Tránh Kiểm Thử Chập Chờn (Chống flaky)

- Rào cản (Gate) cần phải kiểm soát chu trình theo đúng: `Cả hai Đọc 10 < A Chốt < B Cập nhật`.
- Toàn bộ cơ chế kiểm soát tiến độ chờ (latch, future, Awaitility) PHẢI CÓ GIỚI HẠN GIỜ (timeout).
- Đưa các bước mở khoá, giải phóng luồng vào khối `finally` nhằm tránh gây tắc nghẽn ứng dụng.
- Class chạy kiểm định phải ưu tiên Đồng Luồng (`SAME_THREAD`).
- Nghiêm cấm sử dụng annotation `Transaction` bao bọc bên ngoài phương thức Test.
- Việc tạo dữ liệu Khởi tạo (Fixture setup) hoặc truy xuất (inspector) đòi hỏi Giao dịch Độc Lập.
- KHÔNG sử dụng hệ quản trị CSDL Nhúng giả lập (H2) làm môi trường thực hiện kiểm thử MVC.
- Tính vững chãi được khẳng định từ việc Phục Hồi Chính Xác, chứ không phải dừng lại ở việc Cắt Bắt Lỗi Mù Quáng (exceptions).

## 15. Xác Nhận Kiểm Định Đối Soát (Production verification)

Công cụ rà soát Sổ Sách (Reconciliation) có thể tham chiếu logic:

```sql
select p.job_id,
       p.completed_units,
       coalesce(sum(c.delta), 0) as accepted_delta
from job_progress p
left join progress_command c on c.job_id = p.job_id
where p.job_id = :jobId
group by p.job_id, p.completed_units;
```

Hãy tập trung so khớp kết quả từ các Giao dịch hoàn tất so với các Bản đối chiếu của sự kiện độc lập. Nghiêm túc theo dõi các Chỉ số (Metrics) số lượng Dòng Bị Bác Bỏ (Affected Row = 0), Số lượng xung đột, tỷ lệ lỗi Cô Lập. Sự nguy hiểm của "Ghi Đè Âm Thầm" (Silent lost update) là nó hoàn toàn không sinh ra Đồ thị Log Lỗi ngoại lệ trong Dashboard của ứng dụng.
