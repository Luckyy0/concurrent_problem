# Các Giải Pháp Nguyên Tử, Lạc Quan Và Bi Quan (Atomic, optimistic and pessimistic solutions)

## Lựa chọn theo mục tiêu nghiệp vụ (Business intent)

Mục đích thiết kế của kịch bản này là thao tác cộng dồn các khối lượng công việc (commutative deltas) vào cùng một dòng dữ liệu. Vì hệ quản trị CSDL có khả năng xử lý phép cộng một cách trực tiếp, giải pháp SQL nguyên tử (atomic conditional SQL) mang lại mức độ đơn giản và ổn định cao nhất.

Trong trường hợp yêu cầu thay đổi dữ liệu phụ thuộc vào việc tải và tính toán trên nhiều trường hoặc áp dụng nhiều quy tắc nghiệp vụ phức tạp trên Java, phương pháp Khóa Lạc Quan (optimistic `@Version`) hoặc Khóa Bi Quan (pessimistic lock) sẽ là những phương án phù hợp.

> **Nói ngắn gọn:** Hãy truyền chỉ thị `+delta` tới kho lưu trữ thay vì truy xuất dữ liệu gốc và chỉ định một giá trị tuyệt đối.

## Giải Pháp 1 — Cập nhật nguyên tử (Atomic conditional delta)

### Cấu trúc SQL

```sql
update job_progress
set completed_units = completed_units + :delta
where job_id = :jobId
  and :delta > 0
  and completed_units + :delta <= total_units;
```

Cơ chế thực thi (Behavior):

1. Lệnh UPDATE giành quyền chiếm Khóa Dòng (row lock).
2. Các giao dịch cập nhật đồng thời (concurrent updater) buộc phải chuyển vào trạng thái chờ (wait).
3. Sau khi giao dịch giữ khóa hoàn tất (commit), PostgreSQL đánh giá lại điều kiện (re-evaluate predicate) dựa trên dữ liệu lưu trữ mới nhất.
4. Biểu thức thực hiện cộng dồn với giá trị `completed_units` hiện tại.
5. Số lượng dòng trả về (Affected rows) `1` = thao tác được chấp nhận; `0` = thao tác bị từ chối do vi phạm điều kiện.

Nếu luồng A báo cáo `+3`, luồng B báo cáo `+4`:

```text
10 -> 13 -> 17
```

Thứ tự xử lý có thể hoán đổi tùy thuộc luồng nào chiếm khóa trước, nhưng kết quả cuối cùng vẫn là `17` miễn là cả hai đều vượt qua vòng kiểm tra điều kiện (predicates pass).

### Thiết lập ở Spring repository

```java
public interface JobProgressRepository
        extends JpaRepository<JobProgress, UUID> {

    @Modifying(
        flushAutomatically = true,
        clearAutomatically = true
    )
    @Query(
        value = """
            update job_progress
               set completed_units = completed_units + :delta
             where job_id = :jobId
               and :delta > 0
               and completed_units + :delta <= total_units
            """,
        nativeQuery = true
    )
    int addCompletedUnits(UUID jobId, int delta);
}
```

```java
@Service
public class AtomicJobProgressService {
    private final JobProgressRepository progress;

    public AtomicJobProgressService(
        JobProgressRepository progress
    ) {
        this.progress = progress;
    }

    @Transactional
    public ProgressApplyResult addCompletedUnits(
        UUID jobId,
        int delta
    ) {
        if (delta <= 0) {
            throw new IllegalArgumentException(
                "Delta phải nhận giá trị dương"
            );
        }

        int changed = progress.addCompletedUnits(jobId, delta);
        return changed == 1
            ? ProgressApplyResult.applied(jobId, delta)
            : ProgressApplyResult.notApplied(jobId, delta);
    }
}
```

Vì thao tác xử lý bằng mã SQL thuần (native query) bỏ qua khâu quản lý trạng thái entity (managed entity state) trong Hibernate, nên cấu trúc dọn dẹp bộ nhớ (clear/transaction) phải được tinh chỉnh để tránh trường hợp tải lên một phiên bản `JobProgress` đã lỗi thời.

### Sử dụng `RETURNING` để trích xuất giá trị mới

Ứng dụng `JdbcTemplate` / jOOQ / DAO thuần túy có thể tận dụng cú pháp:

```sql
update job_progress
set completed_units = completed_units + :delta
where job_id = :jobId
  and :delta > 0
  and completed_units + :delta <= total_units
returning completed_units, total_units;
```

Không có giá trị trả về (zero returned rows) có nghĩa là thao tác cập nhật đã bị từ chối. Giá trị mới nhất có thể lấy ngay trong cùng một câu lệnh nguyên tử, không cần khởi tạo câu lệnh SELECT sau đó.

### Ràng buộc CSDL (Database constraint)

```sql
alter table job_progress
add constraint ck_job_progress_range
check (
    total_units >= 0
    and completed_units >= 0
    and completed_units <= total_units
);
```

Ràng buộc bảo vệ cấu trúc sâu (defense-in-depth) giúp rào chắn các quy tắc giới hạn. Trong khi phương pháp cập nhật Nguyên tử (atomic UPDATE) sẽ đảm nhận công việc bảo toàn thông số khối lượng làm việc (delta composition).

## Giải Pháp 2 — Khóa Lạc Quan với `@Version` (Optimistic locking)

### Thực Thể (Entity)

```java
@Entity
@Table(name = "job_progress")
public class JobProgress {
    @Id
    private UUID jobId;

    private int completedUnits;
    private int totalUnits;

    @Version
    private long version;

    public void addCompletedUnits(int delta) {
        // Thực hiện kiểm tra nghiệp vụ (validate) và cập nhật nội bộ
    }
}
```

Câu lệnh tự động (Generated update):

```sql
update job_progress
set completed_units = :newValue,
    total_units = :total,
    version = :nextVersion
where job_id = :jobId
  and version = :expectedVersion;
```

Nếu luồng A cập nhật thành công (affected rows = `1`), luồng B sẽ dùng phiên bản (expected version) bị cũ và ghi nhận số dòng thay đổi = `0`. Tín hiệu này dẫn đến việc Hibernate cảnh báo xung đột (optimistic conflict) và giao dịch của luồng B tự động Hủy (rollback).

### Đóng gói Giao dịch riêng biệt trong Thử Lại (Retry)

```java
@Service
public class ProgressAttemptService {
    private final JobProgressRepository progress;
    private final ProgressCommandRepository commands;

    public ProgressAttemptService(
        JobProgressRepository progress,
        ProgressCommandRepository commands
    ) {
        this.progress = progress;
        this.commands = commands;
    }

    @Transactional
    public ProgressResult addOnce(
        UUID commandId,
        UUID jobId,
        int delta
    ) {
        ProgressCommand existing =
            commands.findByCommandId(commandId).orElse(null);
        if (existing != null) {
            return ProgressResult.replayed(existing);
        }

        JobProgress job = progress.findById(jobId)
            .orElseThrow();
        job.addCompletedUnits(delta);
        commands.save(ProgressCommand.applied(
            commandId,
            jobId,
            delta
        ));
        progress.flush();
        return ProgressResult.applied(commandId, jobId);
    }
}
```

```java
@Service
public class ProgressRetryCoordinator {
    private final ProgressAttemptService attempts;

    public ProgressRetryCoordinator(
        ProgressAttemptService attempts
    ) {
        this.attempts = attempts;
    }

    @Retryable(
        retryFor = ObjectOptimisticLockingFailureException.class,
        maxAttempts = 4,
        backoff = @Backoff(
            delay = 20,
            maxDelay = 200,
            multiplier = 2.0,
            random = true
        )
    )
    public ProgressResult add(
        UUID commandId,
        UUID jobId,
        int delta
    ) {
        return attempts.addOnce(commandId, jobId, delta);
    }
}
```

Component Điều phối (Coordinator) không nên tạo Transaction; mỗi lần thực thi hàm ở Component Xử lý (attempt bean) đều phải sinh ra giao dịch mới. Thời gian giới hạn (Example timing) chỉ đóng vai trò minh họa, chưa chuẩn bị cho hệ thống khối lượng lớn (production benchmark). Tiến trình thử lại sẽ tải phiên bản `13` mà luồng A đã ghi nhận, sau đó tự tính toán để hoàn thiện `17`.

Giải pháp Lạc Quan đem lại sức mạnh cho các quy trình nghiệp vụ thay đổi trạng thái phức tạp và xung đột hệ thống chỉ ở mức tương đối thấp. Tại các Điểm dữ liệu Nóng (Hot key), Khóa Lạc Quan có nguy cơ đẩy ứng dụng vào tình trạng quá tải do thử lại nhiều lần (retry amplification).

## Giải Pháp 3 — Khóa Bi Quan Chờ Đồng Bộ (Pessimistic row lock trước read)

```java
public interface JobProgressRepository
        extends JpaRepository<JobProgress, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select j from JobProgress j where j.jobId = :jobId")
    Optional<JobProgress> findByIdForUpdate(UUID jobId);
}
```

```java
@Transactional
public ProgressResult addCompletedUnits(
    UUID jobId,
    int delta
) {
    JobProgress job = progress.findByIdForUpdate(jobId)
        .orElseThrow();
    int before = job.getCompletedUnits();
    job.addCompletedUnits(delta);
    return ProgressResult.applied(
        jobId,
        before,
        job.getCompletedUnits()
    );
}
```

Timeline luồng thực thi:

```text
A thiết lập khóa, đọc giá trị 10 -> thao tác ghi thành 13 -> commit hoàn tất.
B chuyển về chờ trước khi truy vấn đọc -> đọc giá trị 13 (đã được nhả khóa) -> cập nhật 17 -> commit hoàn tất.
```

Luồng đến sau sẽ bị chặn (blocks) thay vì bị bác bỏ hay phải bắt buộc phải chạy lại (fail/retry). Giao dịch cần duy trì quy mô cực ngắn, có trật tự sắp xếp dòng (deterministic order) cẩn thận khi tiến hành Khóa đa dòng, kèm theo việc áp dụng Hạn Mức Chờ của Khóa. Không nên thực hiện truy vấn kết nối mạng với hệ thống bên ngoài (remote I/O) khi đang giữ khóa hệ thống CSDL.

## Giải Pháp 4 — Đối chiếu và Áp dụng (Compare-and-set old value)

Trường hợp mô hình đòi hỏi xử lý Lạc Quan nhưng không có trường version quản lý:

```sql
update job_progress
set completed_units = :newValue
where job_id = :jobId
  and completed_units = :expectedOldValue;
```

Dòng bị tác động (affected rows) `0` thể hiện tình trạng dữ liệu đã có phiên bản khác thay đổi. Ứng dụng phải thao tác Hủy bỏ (rollback) và yêu cầu (retry) từ đầu bằng giao dịch mới toanh (fresh transaction). Cần cực kỳ cân nhắc độ che phủ (compared columns) của các cột, nếu nghiệp vụ chỉ cần thay đổi 1 cột nhỏ thì hệ thống Khóa Phiên Bản (`@Version`) sẽ bảo đảm quản lý đối soát chuẩn xác và an toàn nhất (explicit aggregate version).

## Giải Pháp 5 — Nâng cấp Cấp Độ Cô Lập (Stronger isolation với full retry)

Cấp độ `REPEATABLE READ`/`SERIALIZABLE` chủ động loại trừ các truy vấn thay đổi bằng cách gửi thông điệp lỗi hệ thống (`SQLSTATE 40001`). 
Luồng ứng dụng phải tuân thủ điều kiện:

- Chấp nhận lỗi huỷ giao dịch (failed transaction rollback).
- Khởi động một lần chạy hoàn toàn độc lập với một Giao dịch Mới (fresh transaction).
- Truy xuất CSDL lại (reload) và thiết lập (revalidate) toàn bộ logic.
- Chặt chẽ về chỉ số Giới hạn Thử (attempt/deadline bounded).
- Tránh tác động trùng lặp dữ liệu không cần thiết tới đối tác bên ngoài (external side effects/idempotency).

Khung giải pháp này áp dụng phổ quát khi tính toàn vẹn (invariants) trải rộng nhiều thông số trong dòng dữ liệu hơn một thao tác đơn giản. Với trường hợp bộ đếm dồn số, SQL nguyên tử tối đa công năng và phân định rõ chỉ số luồng (throughput/operational contract).

## Giải Pháp 6 — Nhật Ký Hành Động Bền Bỉ (Durable progress commands/events)

Nếu công tác kiểm toán (reconciliation/audit) được xếp hạng đầu, nên xây dựng cấu trúc theo dõi các thao tác (distinct completion command):

```sql
create table progress_command (
    command_id uuid primary key,
    job_id uuid not null,
    delta integer not null check (delta > 0),
    applied_at timestamptz not null
);
```

Bản chèn lệnh thay đổi (Insert command) cùng cấu hình cập nhật sổ sách (update projection) buộc đặt trong CÙNG MỘT Giao Dịch. Định danh Độc nhất (Unique command ID) triệt trừ lỗi gửi chồng chéo kết quả (duplicate delivery); Cú Pháp Khối lượng Cộng (atomic delta) an toàn hỗ trợ các thao tác đến từ nhiều thực thể chạy song hành.

```text
Chống xử lý lặp lại lệnh (duplicate-command prevention) != An toàn thao tác thay đổi song song (concurrent-mutation safety)
```

Nguồn lệnh nối chuỗi (Append-only) cung cấp tính năng dựng lại số dư tổng kho (rebuild/reconcile projection), tuy sẽ tăng chi phí bảo lưu và tổ chức quản lý luồng quy trình (workflow complexity).

## So sánh Đối tượng Phát hiện và Xử lý Lỗi

| Cơ Chế Giải Pháp | Trình Phát Hiện Xung Đột (Conflict detector) | Quy Trình Cho Bên Thua Cuộc (Loser) |
| --- | --- | --- |
| Cộng Nguyên Tử (Atomic) | Khóa dòng (Row lock) + Kiểm định hiện trạng (current predicate) | Đứng chờ (Wait) sau đó duyệt (apply/no-apply) |
| Khóa Lạc Quan (`@Version`) | Phát hiện trả về dòng (Affected rows) = `0` | Hủy lỗi, thiết lập quy trình gọi lại giới hạn (Fail, rollback, bounded retry) |
| Khóa Bi Quan (`FOR UPDATE`) | Tranh chấp Khóa không tương thích (Incompatible row lock) | Đóng luồng chờ (Block/timeout), tiến hành tải giá trị mới nhất (fresh read) |
| Compare-and-set | Điều kiện Đối chiếu giá trị Cũ (Old-value predicate) | Số lượng Dòng trả = `0`, thao tác gọi lại hệ thống (retry/fail) |
| Serializable | Đồ thị liên kết chuỗi tuần tự (Serialization graph) | Mã Lỗi SQLSTATE `40001`, bắt buộc chạy luồng mới 100% (full retry) |

## Đánh Giá Giữa Các Phương Án (Trade-off comparison)

| Lựa Chọn Phương Án | Độ Chính Xác (Correctness) | Cách Cư Xử Với Xung Đột | Khối Lượng Truy Cập | Khối Lượng Quản Lý (Complexity) |
| --- | --- | --- | --- | --- |
| Cộng Nguyên Tử | Tuyệt vời, chỉ áp dụng trên dòng quy mô đơn (single-row) | Chuỗi chờ ngắn (Short row serialization) | Thấp | Thấp |
| Khóa Lạc Quan | Bảo vệ giá trị toàn thể | Thực hiện Tải Lại khi có Xung Đột | Áp lực số lần tự kích hoạt Retry | Trung bình |
| Khóa Bi Quan | Nhận Kết Quả Tươi dưới lá chắn Khóa | Hàng chờ Đóng (Blocking queue) | Bộ cấp Kết Nối bị tắc nghẽn (Connections held) | Trung bình |
| Strong Isolation | Hàng rào phòng thủ sâu và rộng | Đạp văng và Bắt phải gọi lại (Abort/retry) | Mức có thể Rất Cao | Trung bình đến Cao |
| Sổ Lệnh Hoạt Động (Event ledger) | Phục vụ Kiểm duyệt Sổ Sách (Audit) | Kiểm duyệt lệnh Áp Dụng Chặt Chẽ (atomic/idempotent apply) | Tiêu thụ Bộ Nhớ Tăng | Cao |

## Hướng Đề Xuất Phù Hợp Cho Bài Toán Này

1. Áp dụng phương án SQL Cộng Dồn Nguyên Tử (atomic conditional delta) kết hợp kiểm định qua lượng dòng thay đổi (affected-row count).
2. Xây dựng bổ sung Hàng Rào Trọng Số (database range constraint).
3. Đính kèm định danh Bền Bỉ (durable command ID) khi nguồn kích hoạt thao tác có dấu hiệu trùng.
4. Chỉ vận hành Khóa Lạc Quan, Bi Quan hay Serializable khi việc truy quét phức tạp vượt ngoài tầm xử lý thông số đếm thông thường.
5. Kiểm duyệt hoạt động Hệ thống CSDL sử dụng PostgreSQL thật bằng cấu trúc đan chéo thao tác (concrete two-actor interleaving).
6. Tái đối soát (Reconcile) số dư tổng bằng danh mục thao tác được chấp nhận (accepted delta sum) đối với hệ thống thương mại cốt lõi.

## Danh Sách Kiểm Kê Chuyên Nghiệp (Production checklist)

### Giao Thức Ghi Nhận (Write contract)

- [ ] Mọi hoạt động được đóng gói tại điều kiện logic hoặc phép cộng (delta/conditional mutation) khi có thể.
- [ ] Phân định rõ kết quả số lượng Dòng được trả về (affected-row count) tương đương kết quả quy trình hiển thị.
- [ ] Loại bỏ tuyệt đối phương án viết thao tác bằng Dữ Liệu Rác/Tuyệt đối (stale absolute write) dựa vào mỗi Khóa (ID).
- [ ] Các rào cản ngăn ngừa xung đột (Constraint) rào chắn hoàn toàn cho dòng hiện hành (current-row range).
- [ ] Hành vi Lệnh Trùng (Duplicate command) và Cập nhật Đồng lúc (concurrent mutation) được bóc tách giải quyết khác biệt.

### Điều Phối Mở Rộng (Alternative coordination)

- [ ] Vòng thử lại của Khóa Lạc Quan (Optimistic retry) được đặt trong một Môi Trường và Transaction hoàn toàn mới.
- [ ] Khóa Bi Quan được thiết lập từ xa trước bất kỳ quy trình đọc hay tính toán nào.
- [ ] Dòng chờ Giao dịch Khóa ngắn, đồng thời luôn có hạn mức Chặn thời gian (timeout budget).
- [ ] Giao dịch vi phạm chuỗi thực thi (Serialization failures) được quản lý số lượng và có chính sách phản ứng phù hợp.
- [ ] Kiến trúc đa cấu hình (Multi-instance correctness) nằm trọn trong hệ thống bảo mật từ trung tâm PostgreSQL.

### Vận Hành Hệ Thống (Operations)

- [ ] Thông số đo lường Xung đột/Khoá/Lệnh Gọi Lại (Conflict/lock/retry) phải an toàn trên từng Key độc lập.
- [ ] Hồ sơ theo dõi (Projection) luôn có nguồn truy xuất gốc để cân bằng (reconciliation source).
- [ ] Phương thức Kiểm Thử bám sát giá trị toàn vẹn ở chặng kết thúc (final invariant), tránh chủ quan đo Đóng Dấu Hoàn Thành của máy.
- [ ] Hệ số Cô Lập Tương quan (Effective isolation) được thẩm định.
- [ ] Phương pháp phân tải truy xuất ở Khóa Nóng (Hot-key strategy) cần được kiểm thử mô phỏng chịu tải (benchmark/workload đại diện).
