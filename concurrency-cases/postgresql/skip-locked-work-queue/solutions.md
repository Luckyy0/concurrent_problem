# Giải pháp atomic claim, owner token và lease recovery

## Mục tiêu thiết kế

Chúng ta cần thiết lập ranh giới rõ ràng giữa ba bước: Lấy việc (Claim), Xử lý (Process), và Hoàn thành (Complete):

```text
Transaction Claim → COMMIT → Xử lý tác vụ bên ngoài → Transaction Complete
```

Tuyệt đối không giữ kết nối cơ sở dữ liệu hoặc khóa dòng (row lock) trong khi xử lý tác vụ bên ngoài. Trạng thái của hàng đợi phải được thiết kế để chấp nhận việc xử lý ít nhất một lần (at-least-once) và từ chối những worker đã quá hạn (stale owner).

## Giải pháp 1 — Atomic batch claim bằng CTE

### DTO

```java
public record ClaimedJob(
        UUID jobId,
        String jobType,
        String payload,
        UUID claimToken,
        Instant leaseUntil,
        int attemptCount
) {
}
```

### SQL

```sql
with candidates as (
    select job_id
    from work_job
    where status = 'READY'
      and available_at <= clock_timestamp()
    order by priority desc, available_at, job_id
    for update skip locked
    limit :batch_size
)
update work_job j
set status = 'PROCESSING',
    claim_token = gen_random_uuid(),
    claimed_by = :worker_id,
    lease_until = clock_timestamp() + (:lease_seconds * interval '1 second'),
    attempt_count = j.attempt_count + 1
from candidates c
where j.job_id = c.job_id
returning j.job_id,
          j.job_type,
          j.payload::text,
          j.claim_token,
          j.lease_until,
          j.attempt_count;
```

Cần đảm bảo thứ tự `ORDER BY` mang tính tất định (deterministic) và sử dụng partial index để tối ưu:

```sql
create index ix_work_job_claim
    on work_job(priority desc, available_at, job_id)
    where status = 'READY';
```

Bạn luôn phải kiểm chứng tính hiệu quả của chỉ mục (index usefulness) thông qua dữ liệu và query plan thực tế (production-like); không nên khiên cưỡng ép buộc query plan chỉ vì muốn tạo ra một hình thái khóa (lock shape) theo ý thích.

### Claimer dùng JDBC cho `UPDATE RETURNING`

```java
@Repository
public class JdbcJobClaimer {

    private final NamedParameterJdbcTemplate jdbc;

    public JdbcJobClaimer(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public List<ClaimedJob> claim(
            String workerId,
            int batchSize,
            Duration lease
    ) {
        if (batchSize < 1 || batchSize > 100) {
            throw new IllegalArgumentException("invalid batch size");
        }
        var parameters = new MapSqlParameterSource()
                .addValue("worker_id", workerId)
                .addValue("batch_size", batchSize)
                .addValue("lease_seconds", lease.toSeconds());

        return jdbc.query(CLAIM_SQL, parameters, (row, index) ->
                new ClaimedJob(
                        row.getObject("job_id", UUID.class),
                        row.getString("job_type"),
                        row.getString("payload"),
                        row.getObject("claim_token", UUID.class),
                        row.getTimestamp("lease_until").toInstant(),
                        row.getInt("attempt_count")
                )
        );
    }
}
```

### Transactional claim service

```java
@Service
public class JobClaimService {

    private final JdbcJobClaimer claimer;

    public JobClaimService(JdbcJobClaimer claimer) {
        this.claimer = claimer;
    }

    @Transactional(
            propagation = Propagation.REQUIRES_NEW,
            isolation = Isolation.READ_COMMITTED
    )
    public List<ClaimedJob> claimBatch(
            String workerId,
            int batchSize,
            Duration lease
    ) {
        return List.copyOf(claimer.claim(workerId, batchSize, lease));
    }
}
```

Caller (tiến trình gọi hàm) chỉ được phép nhận danh sách công việc sau khi proxy đã thực thi commit xong. Cấu hình `REQUIRES_NEW` là rất hợp lý ở đây vì quá trình lấy việc (polling root) không phụ thuộc vào bất kỳ giao dịch nào bên ngoài. Nếu caller vô tình chạy bên trong một giao dịch lớn hơn, kiến trúc này sẽ bảo vệ và giữ cho giao dịch lấy việc luôn độc lập.

> **Nói ngắn gọn:** Chúng ta dùng duy nhất một câu lệnh SQL để chọn, khóa và đổi chủ sở hữu của dòng dữ liệu; sau đó transaction bắt buộc phải commit trước khi công việc này được mang ra khỏi phạm vi bảo vệ của cơ sở dữ liệu.

## Giải pháp 2 — Processor ngoài transaction

```java
@Component
public class JobWorker {

    private final JobClaimService claims;
    private final JobCompletionService completions;
    private final ExternalProcessor processor;

    public JobWorker(
            JobClaimService claims,
            JobCompletionService completions,
            ExternalProcessor processor
    ) {
        this.claims = claims;
        this.completions = completions;
        this.processor = processor;
    }

    public int poll(String workerId) {
        List<ClaimedJob> batch = claims.claimBatch(
                workerId,
                20,
                Duration.ofMinutes(2)
        );

        for (ClaimedJob job : batch) {
            try {
                processor.process(job.jobId(), job.payload());
                completions.complete(job.jobId(), job.claimToken());
            } catch (RetryableProcessingException failure) {
                completions.retryLater(
                        job.jobId(),
                        job.claimToken(),
                        failure.safeCode()
                );
            } catch (RuntimeException failure) {
                completions.deadLetter(
                        job.jobId(),
                        job.claimToken(),
                        failure.getClass().getSimpleName()
                );
            }
        }
        return batch.size();
    }
}
```

Thành phần `processor` sẽ xử lý dựa trên một idempotency key ổn định được sinh ra từ `jobId`. Ví dụ trên mô phỏng việc xử lý tuần tự trong một lần lấy việc; ở thực tế, bạn có thể đẩy lô công việc này vào một luồng xử lý song song (bounded executor) nhưng nhớ là không được vượt quá năng lực của connection pool hay các dịch vụ đích.

Quy tắc quan trọng: Không được lấy số lượng công việc nhiều hơn sức mà worker có thể bắt đầu xử lý trước khi thời hạn thuê (lease) cạn kiệt. Nếu thời gian xử lý biến động mạnh, hãy thu nhỏ kích thước batch, hoặc chỉ lấy việc khi worker đang thực sự rảnh, hoặc phải thiết kế thêm cơ chế bắn "heartbeat" để gia hạn token. Nếu không, những công việc nằm ở cuối lô có thể sẽ bị hệ thống thu hồi (reclaim) trước cả khi worker kịp đụng tới.

## Complete/fail conditional theo token

```java
public interface WorkJobRepository extends JpaRepository<WorkJob, UUID> {

    @Modifying
    @Query(value = """
            update work_job
            set status = 'DONE',
                completed_at = clock_timestamp(),
                claimed_by = null,
                claim_token = null,
                lease_until = null
            where job_id = :jobId
              and status = 'PROCESSING'
              and claim_token = :claimToken
            """, nativeQuery = true)
    int complete(UUID jobId, UUID claimToken);

    @Modifying
    @Query(value = """
            update work_job
            set status = 'READY',
                available_at = clock_timestamp() + interval '30 seconds',
                claimed_by = null,
                claim_token = null,
                lease_until = null,
                last_error = :safeCode
            where job_id = :jobId
              and status = 'PROCESSING'
              and claim_token = :claimToken
            """, nativeQuery = true)
    int retryLater(UUID jobId, UUID claimToken, String safeCode);
}
```

Service:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void complete(UUID jobId, UUID token) {
    if (jobs.complete(jobId, token) != 1) {
        throw new LostJobOwnershipException(jobId, token);
    }
}
```

Cả `retryLater` và `deadLetter` cũng phải kiểm tra số dòng bị ảnh hưởng (affected-row) tương tự như hoàn thành. Thời gian chờ (backoff) nên được tính toán dựa trên số lần đã thử; `30 seconds` trong SQL trên chỉ mang tính minh họa.

## Lease recovery

Tiến trình Sweeper sẽ quét và tự động lấy lại các công việc quá hạn:

```sql
with expired as (
    select job_id
    from work_job
    where status = 'PROCESSING'
      and lease_until < clock_timestamp()
    order by lease_until, job_id
    for update skip locked
    limit :batch_size
)
update work_job j
set status = case
        when attempt_count >= :max_attempts then 'DEAD'
        else 'READY'
    end,
    available_at = clock_timestamp() + interval '30 seconds',
    claimed_by = null,
    claim_token = null,
    lease_until = null,
    last_error = 'LEASE_EXPIRED'
from expired e
where j.job_id = e.job_id
returning j.job_id, j.status;
```

Nhiều tiến trình sweeper có thể cùng chạy song song; nhờ `SKIP LOCKED`, chúng sẽ tự chia nhau các dòng hết hạn. Token cũ sẽ tự khắc mất hiệu lực.
Nếu bạn cần cơ chế "heartbeat" cho các công việc dài hơi, hãy dùng truy vấn sau:

```sql
update work_job
set lease_until = clock_timestamp() + :lease
where job_id = :job_id
  and status = 'PROCESSING'
  and claim_token = :claim_token
  and lease_until >= clock_timestamp();
```

Nếu trả về affected-row `0` nghĩa là quyền sở hữu đã mất; worker cần phải lập tức dừng công việc đang làm dở (nếu có thể).

## Empty poll và backpressure

Khi kết quả trả về lô trống (batch rỗng), điều đó không chứng minh rằng hàng đợi đã hết việc; có thể các dòng đó đang bị khóa bởi người khác hoặc chưa đến thời điểm `available_at`. Do đó, worker cần có thời gian chờ linh hoạt (bounded polling delay) bằng cơ chế jitter hoặc adaptive backoff.
Luôn phải có cơ chế kiểm soát giới hạn (Admission control):

- Giới hạn số lượng worker cùng lấy việc.
- Giới hạn kích thước lô công việc.
- Giới hạn số lượng yêu cầu gọi dịch vụ bên ngoài đang chờ xử lý.
- Có hạn mức (quota) theo từng loại công việc hoặc người dùng.
- Giới hạn tổng số kết nối cơ sở dữ liệu.

Tuyệt đối không chạy vòng lặp liên tục (busy-loop) với `SKIP LOCKED` khi toàn bộ hàng đợi đang bận rộn.

## Fairness policy

Thứ tự ưu tiên cơ bản:

```sql
order by priority desc, available_at, job_id
```

Nếu bạn không thể chấp nhận việc các công việc độ ưu tiên thấp bị "chết đói" (starvation), hãy áp dụng kỹ thuật aging:

```text
độ ưu tiên thực tế (effective priority) = độ ưu tiên cơ bản + giới hạn cộng thêm theo thời gian chờ
```

Hoặc cách khác là chia tách các hàng đợi riêng biệt/dành ra các worker dự phòng. Mọi chính sách phải đi kèm với việc xem xét chỉ mục và query plan, đồng thời theo dõi độ tuổi công việc chờ lâu nhất (oldest age) cho từng phân khúc; bạn không thể đòi hỏi một thứ tự strict FIFO tuyệt đối khi đã sử dụng `SKIP LOCKED`.

## Failure behavior

| Tình huống | Cách worker xử lý |
| --- | --- |
| Kết quả lấy việc trả về rỗng | Áp dụng thời gian chờ (backoff); không vội kết luận hàng đợi trống |
| Transaction lấy việc bị rollback | Không xử lý các đối tượng được trả về |
| Dịch vụ bên ngoài lỗi (retryable) | Cập nhật lại hàng đợi kèm token và thời gian chờ (backoff) |
| Lỗi vĩnh viễn hoặc hết số lần thử | Đánh dấu DEAD kèm token |
| Thời hạn thuê (lease) cạn kiệt | Tiến trình Sweeper sẽ thu hồi hoặc đánh dấu DEAD |
| Hoàn thành trễ (stale complete) | affected-row trả về `0`, không ghi đè trạng thái |
| Hệ thống sập sau khi đã gọi dịch vụ | Giao lại việc (redelivery); hệ thống đích tự chống trùng lặp |
| Gặp deadlock `40P01` hoặc timeout | Rollback transaction ngắn này; thử lại theo chính sách an toàn |

## Các phương án khác

### `FOR UPDATE` không skip

Chỉ nên dùng khi yêu cầu thứ tự nghiêm ngặt (strict ordering) quan trọng hơn hiệu suất tổng thể, và thời gian chờ được giới hạn chặt chẽ. Nó không hề phù hợp nếu bạn có số lượng lớn các công việc hoàn toàn độc lập với nhau.

### Atomic `UPDATE ... WHERE status='READY'`

Việc lấy một công việc khi đã biết chính xác ID thông qua điều kiện `affected-row` là cực kỳ tốt. Nhưng đối với bài toán "cho tôi N công việc tiếp theo", CTE kết hợp `SKIP LOCKED` là cách tốt nhất để giải quyết bài toán lựa chọn và chiếm quyền sở hữu (selection + ownership) ngay trong lòng cơ sở dữ liệu.

### Advisory lock

Giải pháp này đòi hỏi một giao thức quản lý riêng (protocol key, unlock, crash complexity) và bắt buộc mọi client phải tuân thủ nghiêm ngặt; trong khi đó, trạng thái, thẻ token và thời hạn thuê (lease) vẫn là điều bắt buộc. Do đó, nó không mang lại sự gọn nhẹ cho bài toán lấy công việc như cách làm ở trên.

### Message broker

Các hệ thống chuyên biệt như Kafka, RabbitMQ, SQS cung cấp khả năng lưu trữ (retention), điều phối (backpressure) và hỗ trợ phân nhóm tiêu thụ dữ liệu (consumer groups) rất hoàn hảo. Tuy nhiên, nếu bạn vẫn phải thực thi các tác vụ gọi ra ngoài (side effects) từ database, bạn vẫn sẽ cần đến các cơ chế idempotency hoặc outbox pattern. Hãy chọn giải pháp dựa trên mô hình xử lý lỗi (failure model) của hệ thống, chứ đừng từ bỏ database chỉ vì sợ `SKIP LOCKED` "không scale vô hạn".

## So sánh trade-off

| Phương pháp | Chống lấy trùng (Duplicate claim) | Thông lượng / Độ trễ | Công bằng (Fairness) | Phục hồi sự cố | Vận hành | Hỗ trợ nhiều máy chủ |
| --- | --- | --- | --- | --- | --- | --- |
| CTE `SKIP LOCKED` kèm token/lease | Ngăn chặn trực tiếp qua transaction | Chạy song song, transaction ngắn | Tương đối (Best-effort) | Dùng sweeper | Mức trung bình | Tốt |
| Blocking `FOR UPDATE` | Ngăn chặn qua transaction | Xếp hàng chờ, tắc nghẽn | Gần với thứ tự gốc | Vẫn cần status/lease | Đơn giản | Tốt |
| Plain SELECT | Không thể ngăn chặn | Nhanh nhưng sai lệch dữ liệu | Không đáng tin cậy | Không có | Rất thấp | Rủi ro cao |
| Conditional UPDATE theo ID biết trước | Dùng affected-row để bảo vệ | Tốt nếu đã biết trước ID | Caller tự quyết định | Vẫn cần lease | Thấp | Tốt |
| Message Broker | Dựa trên cơ chế của broker | Khả năng scale mạnh mẽ nhất | Phụ thuộc vào phân vùng (partition) | Gọi lại, gửi vào DLQ | Phức tạp (Cao) | Rất tốt |

## Checklist trước production

- [ ] Lệnh SQL lấy việc (claim SQL) có điều kiện rõ ràng, `ORDER BY` mang tính tất định, có `LIMIT` và kết hợp với partial index phù hợp.
- [ ] Việc lấy việc phải được COMMIT trót lọt TRƯỚC KHI bắt đầu xử lý tác vụ bên ngoài.
- [ ] Các thao tác báo hoàn thành/báo lỗi/gia hạn (heartbeat) đều phải kiểm tra thẻ token hiện tại.
- [ ] Phải quy định rõ thời hạn thuê (lease expiry), số lần thử tối đa và cơ chế xử lý DEAD.
- [ ] Hệ thống bên ngoài (downstream) xử lý chống trùng lặp dựa trên khóa tác vụ (stable job/effect ID).
- [ ] Áp dụng thời gian chờ (jitter/backoff) nếu kết quả lấy việc rỗng; phải có trần giới hạn số lượng worker chạy đồng thời.
- [ ] Có hệ thống cảnh báo (alert) cho các công việc chờ lâu (oldest READY/PROCESSING age) và lỗi hoàn thành trễ (stale completion).
- [ ] Dù đã dùng skip lock, vẫn cần có giới hạn thời gian (timeout/coordination) đề phòng các thao tác DDL hoặc khóa bảng.
- [ ] Có các kịch bản Testcontainers chứng minh được việc lấy việc độc lập (disjoint claims), không bị tắc nghẽn, tính năng bỏ qua (skip) và chặn worker quá hạn.
