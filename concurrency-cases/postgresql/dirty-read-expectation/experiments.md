# Phòng Thí Nghiệm Đập Tan Ảo Tưởng (Deterministic visibility experiments)

## 1. Mục tiêu tối thượng

Mấy bài Tests này được sinh ra để vả mặt những ai còn mù quáng tin vào Đọc Bẩn ở PostgreSQL. Chúng ta phải chứng minh được:

1. Ông nhà văn (writer) đã gõ lệnh UPDATE và xả (flush) xuống DB, nhưng kiên quyết ôm Khóa không thèm chốt sổ.
2. Thằng đọc (reader) dù xin xỏ Đọc Bẩn (RU) nhưng vẫn lướt qua cái Khóa đó mượt mà (không bị block) bằng lệnh Đọc Chay (plain SELECT).
3. Đọc xong thì thằng reader CHỈ THẤY số `20` cũ rích, cấm tuyệt đối không lòi ra số `80`.
4. Nếu writer Hủy Kèo (rollback), con số vĩnh viễn là `20`. Còn nếu writer Chốt Sổ (commit), thằng reader chạy lại từ đầu mới thấy được `80`.
5. Cái Nhãn mức cô lập (isolation label) DB báo về chỉ để trưng cho vui, KHÔNG ĐƯỢC lấy nó làm bằng chứng "Đã Đọc Bẩn".
6. Các giải pháp thay thế như Nhịp Tim (heartbeat) hay Máy Trạng Thái phải chứng minh được tính hiệu quả và đọc được Dữ Liệu Đã Chốt.

> **Nói ngắn gọn:** Cách bắt quả tang chuẩn nhất là nhốt thằng writer lại giữa chừng bằng một cái Cổng (gate), mở một Kết nối (connection) thứ 2 đi vào đọc, đọc xong mới mở cổng cho thằng writer chốt sổ hay tự tử (rollback) tùy thích.

## 2. Dựng rạp xiếc với PostgreSQL Testcontainers

Tuyệt đối cấm xài H2 để test trò này! Bắt buộc phải dựng một con PostgreSQL hàng real lên:

```java
@Testcontainers
@Execution(ExecutionMode.SAME_THREAD)
@SpringBootTest
class DirtyReadExpectationIntegrationTest {

    @Container
    @ServiceConnection
    static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine");
}
```

Nhớ kỹ: Hàm Test tuyệt đối KHÔNG ĐƯỢC gắn `@Transactional`. Ông Writer, ông Reader và ông Thanh Tra (Inspector) phải là 3 cục Bean độc lập để bắt 3 Kết nối (connections) riêng biệt.

Cấu trúc Bảng:

```sql
create table job_run (
    job_id uuid primary key,
    status varchar(32) not null,
    progress_percent integer not null
        check (progress_percent between 0 and 100),
    generation bigint not null,
    owner_token uuid,
    lease_until timestamptz
);

create table job_attempt_heartbeat (
    -- Bảng riêng dùng để lưu Nhịp Tim tách biệt với Bảng Chính
    job_id uuid not null,
    generation bigint not null,
    owner_token uuid not null,
    progress_percent integer not null
        check (progress_percent between 0 and 100),
    last_seen_at timestamptz not null,
    primary key (job_id, generation)
);
```

Bơm sẵn dữ liệu mồi:

```sql
insert into job_run(
    job_id, status, progress_percent, generation
)
values (:jobId, 'RUNNING', 20, 7);
```

## 3. Lập rào chắn bắt quả tang (Writer gate)

Dùng trò đếm ngược (CountDownLatch) của Java để bắt thằng Writer phải đứng im sau khi đã Xả dữ liệu:

```java
final class WriterGate {
    private final CountDownLatch flushed = new CountDownLatch(1);
    private final CountDownLatch allowCompletion =
        new CountDownLatch(1);

    // Kêu thằng Writer xả xong đứng đây chờ
    void afterFlush() {
        flushed.countDown(); // Báo hiệu: "Tao xả rồi nha!"
        awaitOrFail(allowCompletion, Duration.ofSeconds(5)); // Đợi cấp lệnh mới được đi tiếp
    }

    // Kẻ đứng ngoài (Test) canh me
    void awaitFlushed() {
        awaitOrFail(flushed, Duration.ofSeconds(5));
    }

    // Mở cổng thả cho đi
    void release() {
        allowCompletion.countDown();
    }

    private static void awaitOrFail(
        CountDownLatch latch,
        Duration timeout
    ) {
        try {
            if (!latch.await(timeout.toMillis(), TimeUnit.MILLISECONDS)) {
                throw new AssertionError("Timed out waiting for writer gate");
            }
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new AssertionError("Interrupted while waiting", interrupted);
        }
    }
}
```

Writer gọi `entityManager.flush()`/repository `flush()` trước
khi gọi `gate.afterFlush()`. Khúc `finally` luôn nhớ gọi `release()` nếu không rác vương vãi Thread chạy mãi không tắt nhé.

## 4. Con mắt của người phán xử (Reader observation)

```java
public record ReadObservation(
    int progress,
    String reportedIsolation,
    int jdbcIsolation
) {}
```

```java
@Transactional(
    isolation = Isolation.READ_UNCOMMITTED,
    readOnly = true
)
public ReadObservation observe(UUID jobId) {
    // 1. Hỏi coi JDBC mầy xin Mức độ nào?
    int jdbcIsolation = jdbc.execute(
        (ConnectionCallback<Integer>)
            Connection::getTransactionIsolation
    );
    // 2. Hỏi coi Database mầy chém gió ra cái Nhãn gì?
    String reported = jdbc.queryForObject(
        "select current_setting('transaction_isolation')",
        String.class
    );
    // 3. Và ĐÂY MỚI LÀ CHÂN LÝ: Đọc con số Tiền độ thật sự!
    int progress = jdbc.queryForObject(
        """
        select progress_percent
        from job_run
        where job_id = ?
        """,
        Integer.class,
        jobId
    );
    return new ReadObservation(
        progress,
        reported,
        jdbcIsolation
    );
}
```

Phiên bản DB có thể phịa ra cái Nhãn khác nhau (read committed hay uncommitted), cứ Log hết ra mà xem. Nhưng để kết luận đúng sai (correctness) thì CHỈ ĐƯỢC tin vào con số `progress` móc lên thôi.

## 5. Thí nghiệm 1 — Mặc áo Đọc Bẩn vẫn không húp được đồ bẩn

```java
@Test
void readUncommittedRequestStillCannotSeeDirtyRow()
        throws Exception {
    WriterGate gate = writerProbe.install();
    ExecutorService actor = Executors.newSingleThreadExecutor();

    try {
        // 1. Nhốt thằng A vào 1 Thread riêng, kêu làm việc rồi tự sát (rollback)
        Future<Throwable> writerOutcome = actor.submit(() ->
            catchThrowable(() ->
                processor.processCurrentUnit(
                    JOB_ID,
                    true // failAfterFlush = true
                )
            )
        );

        // 2. Đợi nó kêu Xả xong
        gate.awaitFlushed();

        // 3. Cho B nhào vô đọc thử
        ReadObservation observed = watchdog.observe(JOB_ID);

        // KẾT QUẢ ĐÂY: Dù B xin Đọc Bẩn, nhưng vẫn lãnh số 20 cũ mèm!
        assertThat(observed.progress()).isEqualTo(20);
        // Cười mỉa mai cái Nhãn DB trả về:
        assertThat(observed.reportedIsolation())
            .isIn("read uncommitted", "read committed");
        assertThat(observed.jdbcIsolation()).isIn(
            Connection.TRANSACTION_READ_UNCOMMITTED,
            Connection.TRANSACTION_READ_COMMITTED
        );

        // 4. Mở cổng cho thằng A tự sát
        gate.release();
        Throwable failure =
            writerOutcome.get(5, TimeUnit.SECONDS);
        assertThat(failure)
            .isInstanceOf(ProcessingFailedException.class);

        // Cuối cùng: Chốt sổ vẫn là 20.
        assertThat(inspector.progress(JOB_ID)).isEqualTo(20);
    } finally {
        gate.release();
        writerProbe.clear();
        actor.shutdownNow();
        assertThat(actor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Ông Watchdog về nhà ngủ trong khi Cổng của thằng Writer vẫn đang Đóng Kín, chứng minh rõ ràng Lệnh Đọc Chay không thèm đợi Writer nhả Khóa mà nhảy thẳng vào lấy kết quả cũ.

## 6. Thí nghiệm 2 — Phải đợi Cơm Chín mới được ăn (Before commit old, after commit new)

Nếu ông A không chết mà sống nhăn nhở Chốt Sổ (commit) thì sao?

```java
@Test
void laterStatementSeesValueOnlyAfterWriterCommits()
        throws Exception {
    WriterGate gate = writerProbe.install();
    ExecutorService actor = Executors.newSingleThreadExecutor();

    try {
        Future<Void> writer = actor.submit(() -> {
            processor.processCurrentUnit(JOB_ID, false);
            return null;
        });

        gate.awaitFlushed();
        assertThat(watchdog.observe(JOB_ID).progress())
            .isEqualTo(20);

        gate.release();
        writer.get(5, TimeUnit.SECONDS);

        assertThat(watchdog.observe(JOB_ID).progress())
            .isEqualTo(80);
    } finally {
        gate.release();
        writerProbe.clear();
        actor.shutdownNow();
        assertThat(actor.awaitTermination(5, TimeUnit.SECONDS)).isTrue();
    }
}
```

Phát gọi thứ 2 của Watchdog tạo một Bức Ảnh Chụp (Snapshot) hoàn toàn Mới sau khi ông A chốt. Trò này chứng minh luân thường đạo lý (normal committed visibility), chả dính dáng gì tới "Đọc bẩn".

## 7. Thí nghiệm 3 — Đồ nặn ra (INSERT) cũng tàng hình nốt

Đừng nghĩ Đọc Bẩn chỉ áp dụng cho UPDATE. Kênh của ông A:

```sql
insert into job_run(
    job_id, status, progress_percent, generation
)
values (:newJob, 'RUNNING', 1, 1);
-- flush, no commit
```

Thì thằng Reader xin Đọc Bẩn cũng mù dở:

```java
assertThat(watchdog.find(NEW_JOB_ID)).isEmpty();
```

Thí nghiệm này nhằm bắt bài mấy ông ngây thơ code logic bắt Lỗi "Đọc Bẩn Dòng Cũ" nhưng lại lơ đẹp "Đọc Bẩn Dòng Mới". Chỉ khi A Chốt xong và B chạy lại lệnh mới thì mới thấy dòng đó xuất hiện.

## 8. Thí nghiệm 4 — Đọc ép Khóa (FOR UPDATE) thì Đứng Đợi

Thay vì Đọc chay, ông B dùng hàm:

```java
@Transactional
public JobSnapshot lockAndRead(UUID jobId) {
    return jobs.findByIdForUpdate(jobId)
        .orElseThrow()
        .snapshot();
}
```

Nhốt thằng A ở Cổng, chọt Query thẳng xuống hệ thống Postgres xem có án mạng không:

```sql
select count(*)
from pg_stat_activity
where datname = current_database()
  and wait_event_type = 'Lock';
```

KẾT QUẢ: Ít nhất 1 thằng đứng ngáp ruồi! (waiter count >= 1). 
Ông B bị kẹt cứng ngắc cho đến khi A chốt/hủy. Xong xuôi, B lượm được cái kết quả cuối cùng (outcome).
Khẳng định lại: Đọc lấy Khóa là "Đợi kết quả cuối", chứ KHÔNG phơi ra đồ Đang làm dở (dirty visibility). Trò ép Khóa này không được đem ra để chạy làm Polling trên Dashboard đâu nha!

## 9. Thí nghiệm 5 — Nhịp tim riêng lẻ (Independent heartbeat visible)

Ông A đang ôm cái Transaction bự chảng, nhưng lâu lâu lén thả "Nhịp Tim" (heartbeat) ra ngoài bằng cái lệnh `REQUIRES_NEW`:

```java
heartbeatPublisher.publish(new Heartbeat(
    JOB_ID,
    7,
    OWNER_TOKEN,
    80,
    TEST_NOW
));
```

Cái này vứt vào cái Bảng Mới rồi tự chốt trước khi quay về. Kết quả là trong lúc Giao dịch chính của A còn đang ngâm tôm:

```java
// Bảng chính vẫn lỳ lợm số 20
assertThat(watchdog.observe(JOB_ID).progress()).isEqualTo(20);
// Nhịp tim bên Bảng phụ đã nảy lên số 80
assertThat(heartbeatReader.read(JOB_ID, 7).progressPercent())
    .isEqualTo(80);
```

Sau khi Giao dịch chính nổ (Rollback):

```java
assertThat(inspector.progress(JOB_ID)).isEqualTo(20);
assertThat(heartbeatReader.read(JOB_ID, 7).progressPercent())
    .isEqualTo(80);
```

Bài học: Nhịp tim báo cáo Trạng thái "Đang ráng chạy" là hoàn toàn hợp pháp, nhưng phải vứt nó ra một mảnh đất riêng, chốt sổ nhanh gọn.

## 10. Thí nghiệm 6 — Mở REQUIRES_NEW chọt vô cùng 1 Dòng là NGU Y HỆT NHAU

Đang ôm Transaction ngoài (Outer), khóa cái Dòng Job đó lại. Xong ngứa tay mở thêm cái Transaction nhỏ ở trong (`REQUIRES_NEW`), và đòi chọt vô cùng Dòng đó! 
Kết quả:

```text
Thằng Cha ôm Khóa Dòng
Thằng Cha đứng ngó Thằng Con trả kết quả
Thằng Con đứng mỏi giò đợi Thằng Cha nhả Khóa Dòng
```

Cắn Đuôi (Deadlock) nổ tung đầu! Không cần cái Server khác can thiệp, Code của bạn tự bóp dái nó luôn. Nhịp tim phải vứt ra Bảng Riêng là vì vậy!

## 11. Thí nghiệm 7 — Cuộc chiến Tranh Giành (Atomic recovery claim)

Chờ hết giờ Thuê, lùa 2 ông Watchdog chạy thẳng vào một cửa ải duy nhất để đua:

```sql
update job_run
set generation = generation + 1,
    owner_token = :owner,
    lease_until = :leaseUntil
where job_id = :jobId
  and generation = 7
  and status = 'RUNNING'
  and lease_until < :databaseNow -- Hết hạn Thuê rồi!
returning generation, owner_token;
```

Kết quả:

```java
assertThat(results.successCount()).isEqualTo(1); // 1 thằng Thắng
assertThat(results.noOpCount()).isEqualTo(1); // 1 thằng Khóc
assertThat(inspector.generation(JOB_ID)).isEqualTo(8);
assertThat(inspector.ownerToken(JOB_ID))
    .isEqualTo(results.winnerToken());
```

Không cần mơ mộng Đọc Bẩn, cứ nhào vô Ghi Tranh Giành (atomic write) là giải quyết êm đẹp ai là người Giải Cứu.

## 12. Thí nghiệm 8 — Đừng tin lời đồn (Label test không thay behavior test)

Chỉ dùng bộ Log ghi lại cái Nhãn Cô Lập (Isolation label):

```text
PostgreSQL server version
pgJDBC version
requested Spring isolation
Connection.getTransactionIsolation()
current_setting('transaction_isolation')
observed progress during uncommitted writer
```

Test chạy sai hay đúng là phải dựa vào con số Tiến độ Thật Sự Húp Được (visibility correctness). Mấy cái Label thay đổi chóng mặt tùy theo máy chủ chỉ dùng để làm cảnh tra cứu tương thích, tuyệt đối không được xài làm bằng chứng!

## 13. Con mắt Thanh Tra (Inspector)

```java
@Service
class CommittedJobInspector {
    private final JdbcTemplate jdbc;

    // Đẩy xa bụi trần bằng REQUIRES_NEW
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    int progress(UUID jobId) {
        return jdbc.queryForObject(
            """
            select progress_percent
            from job_run
            where job_id = ?
            """,
            Integer.class,
            jobId
        );
    }
}
```

Đây là hàm dành riêng cho cuối giờ để chốt sổ, mở Transaction Độc Lập hoàn toàn để săm soi kết quả cuối.

## 14. Bảng Chân Lý Trọn Vẹn (Coverage matrix)

| Chuyện gì xảy ra? | Thời điểm ông Đọc nhào vô | Số Tiến Độ Thấy Được |
| --- | --- | --- |
| Xin RU (Đọc Bẩn) Đọc Chay, A Đang mở | Khi A chưa làm xong | 20 (Cũ rích) |
| Xin RU (Đọc Bẩn) Đọc Chay, A Hủy Kèo | Sau khi A Hủy | 20 (Y nguyên) |
| Xin RU (Đọc Bẩn) Đọc Chay, A Chốt sổ | Bắt đầu Lệnh Đọc SAU KHI A Chốt | 80 (Thơm bơ) |
| Đọc Chay trò nặn đồ mới (INSERT) | Khi A chưa làm xong | Trắng xóa (Không có dòng nào) |
| Đọc ép Khóa `FOR UPDATE`, A Chốt | Đợi mòn mỏi xong | 80 |
| Đọc ép Khóa `FOR UPDATE`, A Hủy | Đợi mòn mỏi xong | 20 |
| Nhịp tim chốt riêng lẻ | A đang ngâm tôm Bảng chính | Nhịp tim 80, Bảng chính 20 |

## 15. Bí kíp chống Flaky (Code chạy hên xui)

- Cổng rào (Writer flush gate) ép tụi nó chạy đúng thứ tự.
- Chơi Latch hay Future là phải có Hẹn Giờ Bom Nổ (timeout).
- Lệnh `finally` bắt buộc mở cổng thả chó, không là cháy máy.
- Bỏ H2 đi, Test trò Tầm Nhìn Dữ Liệu là phải xài đồ real.
- Luôn kiểm tra Dữ Liệu Dòng (Row behavior), không phải cái Label mõm.
- Thí nghiệm Lấy Khóa bắt buộc phải có timeout và rình ở Admin Diagnostics (pg_stat_activity).

## 16. Lên Production soi gì? (Production verification)

Ra trận thì phải dán mắt vào mấy cái đồ thị này:

- Tuổi thọ của Transaction (Ai mở qua 5 giây thì chửi).
- Dấu Nhịp Tim cuối là mấy giờ.
- Số thằng thắng / thua trong cuộc chiến Đòi Quyền (recovery claims).
- Dấu hiệu từ chối nhịp tim cũ (generation rejection).
- Ghi nhận những ông Chốt Sổ Thành Công / Những ông Rollback xối xả.

Thằng nào cãi chày cãi cối "Sao tôi chỉ đọc được đồ cũ?" thì đập vô mặt nó tài liệu này. Đó là Lẽ Thường Của MVCC, không phải Bug Cache hay Trễ Mạng (Replication lag)!
