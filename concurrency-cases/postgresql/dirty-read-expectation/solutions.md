# Tuyệt Chiêu Chốt Sổ: Thiết Kế Đúng Đắn Cho Bài Toán Theo Dõi Tiến Độ (Committed coordination and progress solutions)

## 1. Mục tiêu thiết kế (Thiết kế chuẩn là phải thế nào?)

Một giải pháp (solution) xịn xò phải định nghĩa rạch ròi 3 thứ:

```text
Trạng thái công việc cuối cùng (final business state):
  Chỉ được vinh danh công bố sau khi cái Giao dịch chính (transaction) ĐÃ CHỐT (commit).

Tín hiệu Nhịp tim/Tiến độ đang làm (attempt progress/heartbeat):
  Phải tự mình chốt sổ độc lập. Ghi rõ tên chủ nhân (owner) và thế hệ (generation) để dễ quản lý.

Cứu hộ (recovery):
  Phải là một cú Tranh Giành Độc Quyền (atomic claim). Tuyệt đối cấm cái thói "Đọc thấy cũ rích -> Nhắm mắt nhào vô cứu mù quáng".
```

> **Nói ngắn gọn:** Muốn báo cáo tình hình, hãy dùng một tín hiệu ĐÃ CHỐT một cách đàng hoàng. Đừng có biến cái đống Dữ liệu dở dang chưa chốt thành cái trạm phát thanh (message bus) của hệ thống.

## 2. Giải Pháp 1 — Chấp nhận số phận "Chỉ Đọc Đồ Đã Chốt" (Committed-only observation)

Nếu Watchdog hay cái Dashboard của bạn không quá vã đến mức đòi xem tiến độ nhảy múa từng giây:

```java
@Service
public class JobStatusReader {
    private final JobRunRepository jobs;

    public JobStatusReader(JobRunRepository jobs) {
        this.jobs = jobs;
    }

    // Đọc chay bình thường thôi, kệ tía Đọc Bẩn!
    @Transactional(readOnly = true)
    public JobSnapshot read(UUID jobId) {
        return jobs.findById(jobId)
            .orElseThrow()
            .snapshot();
    }
}
```

Cứ xài `READ_COMMITTED` mặc định. Viết hẳn lên tài liệu cho anh em biết: "Trạng thái này chỉ cập nhật khi Job đã làm Xong Xuôi và Chốt Sổ". Giao diện người dùng (UI) nên hiển thị chữ "Cập nhật lần cuối lúc..." (`lastCommittedAt`), chứ đừng cố vẽ thanh tiến độ mượt mà lừa dối người dùng.

Tuyệt chiêu này xài ngon khi cái Job của bạn xử lý nhanh gọn lẹ, và hệ thống chấp nhận được chuyện dữ liệu hiển thị trễ một chút (stale-until-commit).

## 3. Giải Pháp 2 — Chia để trị: Băm nhỏ việc thành nhiều Trạm Nghỉ (Committed checkpoints)

Nếu workflow chạy dài lê thê, hãy băm nó ra thành nhiều cục (chunks). Mỗi cục làm xong thì:

```text
BẮT ĐẦU (BEGIN)
  Xử lý cục công việc hiện tại (nhớ code sao cho chạy lại không bị lỗi đúp - idempotent)
  Cập nhật tiến độ của cái Trạm Nghỉ này
CHỐT SỔ (COMMIT)
```

Dùng một cú INSERT nguyên tử (atomic) để giành quyền xử lý cái chunk đó:

```sql
insert into chunk_result(chunk_id, job_id, status)
values (:chunkId, :jobId, 'APPLIED')
on conflict (chunk_id) do nothing;
```

```java
@Service
public class JobChunkService {
    private final JobRunRepository jobs;
    private final ChunkResultRepository results;

    public JobChunkService(
        JobRunRepository jobs,
        ChunkResultRepository results
    ) {
        this.jobs = jobs;
        this.results = results;
    }

    @Transactional
    public ChunkOutcome applyChunk(
        UUID jobId,
        UUID chunkId,
        int checkpoint
    ) {
        // Cú này lách khóa nguyên tử: "Thằng nào giành được chunk này?"
        int claimed = results.claim(chunkId, jobId);
        if (claimed == 0) { // Có thằng khác húp rồi!
            ChunkResult existing =
                results.findByChunkId(chunkId).orElseThrow();
            return ChunkOutcome.replayed(existing); // Nhường nó, mình coi như xong
        }

        // Xin Khóa dòng Job chính để cập nhật Trạm Nghỉ
        JobRun job = jobs.findByIdForUpdate(jobId)
            .orElseThrow();
        job.advanceCommittedCheckpoint(checkpoint);
        ChunkResult result =
            results.findByChunkId(chunkId).orElseThrow();
        return ChunkOutcome.applied(result);
    }
}
```

Cái `chunk_id` độc nhất vô nhị cộng thêm trò `ON CONFLICT DO NOTHING` sẽ bóp chết mấy yêu cầu chạy đúp, biến chúng thành một cú Tranh Giành gọn gàng. Watchdog tha hồ đọc Trạm Nghỉ (đã chốt). Lỡ có cục Chunk nào bị lỗi Hủy kèo (rollback) thì nó cũng rút lại lệnh Claim, chả báo cáo láo tiến độ ra ngoài.


Đánh đổi (Trade-off):

- Tạm biệt triết lý "Một sống một còn" (all-or-nothing) cho cả cái quy trình lớn;
- Bắt buộc phải code cho nó tự miễn dịch khi chạy lại (resumable/idempotent);
- Thiết kế Bảng và Máy Trạng Thái nhức não hơn chút;
- Đổi lại: Transaction, Khóa, Kết nối được giải phóng nhanh như chớp.

## 4. Giải Pháp 3 — Xây trạm phát Nhịp Tim riêng lẻ (Independent heartbeat record)

Khi cục Job quá to không băm nhỏ được, nhưng Watchdog lại gào thét đòi tín hiệu sự sống, thì đẻ ra cái Bảng Riêng:

```sql
create table job_attempt_heartbeat (
    job_id uuid not null,
    generation bigint not null,
    owner_token uuid not null,
    progress_percent integer not null,
    last_seen_at timestamptz not null,
    primary key (job_id, generation),
    check (progress_percent between 0 and 100)
);
```

Người báo cáo (Publisher) phải là một Bean xài Transaction độc lập hoàn toàn:

```java
@Service
public class JobHeartbeatPublisher {
    private final JobHeartbeatRepository heartbeats;

    public JobHeartbeatPublisher(
        JobHeartbeatRepository heartbeats
    ) {
        this.heartbeats = heartbeats;
    }

    // Cái thẻ ma thuật này giúp đẻ ra 1 Giao dịch mới toanh, chốt ngay lập tức
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void publish(Heartbeat heartbeat) {
        int changed = heartbeats.upsertIfOwnerMatches(
            heartbeat.jobId(),
            heartbeat.generation(),
            heartbeat.ownerToken(),
            heartbeat.progressPercent(),
            heartbeat.observedAt()
        );
        if (changed != 1) { // Bị đuổi cổ rồi
            throw new StaleJobOwnerException(
                heartbeat.jobId(),
                heartbeat.generation()
            );
        }
    }
}
```

Khúc SQL Upsert xịn xò:

```sql
insert into job_attempt_heartbeat(
    job_id,
    generation,
    owner_token,
    progress_percent,
    last_seen_at
)
values (
    :jobId,
    :generation,
    :ownerToken,
    :progress,
    :observedAt
)
on conflict (job_id, generation)
do update
set progress_percent = excluded.progress_percent,
    last_seen_at = excluded.last_seen_at
-- CHỈ CẬP NHẬT KHI LÀ CHÍNH CHỦ VÀ TIẾN ĐỘ ĐANG ĐI TỚI!
where job_attempt_heartbeat.owner_token = excluded.owner_token
  and job_attempt_heartbeat.progress_percent
      <= excluded.progress_percent;
```

Cái Nhịp tim này sẽ sống thọ mặc kệ cái Job chính có bị Hủy Kèo (rollback) hay không. Do đó, phải hiểu nó mang ý nghĩa là "Tao đang cố làm tới khúc này" (attempt progress), CHỨ KHÔNG PHẢI là "Kết quả cuối cùng đã xong" (final success).

### Tuyệt đối tránh trò tự cắn đuôi (Self-deadlock)

Đừng dại dột để cái Giao dịch lớn bên ngoài (outer Tx) khóa cái Dòng X, rồi thằng bên trong gọi `REQUIRES_NEW` nhảy ra chọt đúng cái Dòng X đó:

```text
Thằng Cha ôm Khóa Dòng X
Thằng Con REQUIRES_NEW đứng chờ Thằng Cha nhả Khóa Dòng X
Thằng Cha đứng ngó Thằng Con trả kết quả
BÙM! Đứng máy cả đôi.
```

Bảng Nhịp tim phải Tách Biệt hoàn toàn (hoặc gọi Publish ở ngoài luồng). Và nhớ tăng sức chứa của Database Pool lên để gánh thêm mấy cái `REQUIRES_NEW` này nhé.

## 5. Giải Pháp 4 — Cuộc Chiến Tranh Giành Quyền Cứu Hộ (Atomic recovery claim)

Watchdog đừng hở tí là bật chế độ Cứu Hộ chỉ bằng cách "Đọc mồm". Phải đập ngay một cú Ghi Có Điều Kiện (conditional write) để phân định thắng thua:

```sql
update job_run
set generation = generation + 1,
    owner_token = :newOwnerToken,
    lease_until = :newLeaseUntil,
    status = 'RUNNING'
where job_id = :jobId
  and generation = :observedGeneration -- Đúng cái thế hệ lúc tao soi
  and status = 'RUNNING' -- Vẫn chưa Xong
  and lease_until < :databaseNow -- Phải hết hạn Thuê rồi tao mới cướp!
returning generation, owner_token, lease_until;
```

Kết quả trả về sẽ định đoạt số phận:

| Kết quả trả về | Số phận |
| --- | --- |
| 1 Dòng | Watchdog ăn kèo! Được quyền Giải Cứu, mở kỷ nguyên Thế hệ mới. |
| 0 Dòng | Trâu chậm uống nước đục! Kẻ khác đã xơi hoặc Job tự xong rồi. Đi về! |

Mọi hậu quả của việc Giải cứu (side effect) đều phải mang theo cái "Giấy Phép Thế Hệ" (generation/fencing token). Lỡ thằng Processor cũ sống lại định Chốt Sổ thì sẽ bị đá văng vì Giấy Phép đã hết hạn. Cái trò Nguyên Tử (atomic) này gánh được cả trăm ông Watchdog cùng lúc; Đọc Bẩn (`READ_UNCOMMITTED`) muôn đời không làm được.

## 6. Giải Pháp 5 — Lấy Khóa ngồi Đợi thằng kia làm xong (Locking read)

Nếu bắt buộc phải đứng đợi thằng Writer hoàn thành nghiệp lớn:

```sql
select job_id, status, progress_percent, generation
from job_run
where job_id = :jobId
for update; -- Đóng cọc ngồi chờ!
```

Thằng Đọc sẽ Đứng Đợi (hoặc lố giờ timeout) cho tới lúc thằng Writer kia Chốt sổ hoặc Hủy kèo, rồi lượm luôn kết quả cuối cùng. Không có miếng dữ liệu chưa chốt nào lòi ra đây cả.

Cách này ngon cho mấy cái Transaction chớp nhoáng (short critical). NHƯNG chống chỉ định làm Polling cho Dashboard hay chờ Job làm quá lâu vì:

- Ngâm cái Connection trong Pool khi đứng chờ;
- Hàng đợi Khóa (lock queue) phình to, kéo lùi tốc độ hệ thống (tail latency);
- Phải tự thiết kế cơ chế dập lửa khi lố giờ (timeout/deadlock);
- Chả cung cấp được cái Nhịp tim nào trong lúc đứng đợi.

## 7. Giải Pháp 6 — Thùng thư Sự kiện Bền bỉ (Durable event/outbox)

Nếu thiên hạ (Consumers) cần hóng tin sau khi bạn Chốt Sổ:

```text
Giao dịch chính (business Tx)
  -> Cập nhật bảng Job
  -> Ném một tin nhắn Sự Kiện (Event) vào bảng Hộp Thư Đi (Outbox)
CHỐT SỔ (COMMIT)

Ông Đưa Thư (publisher) rảo bước đi giao mớ Sự kiện ĐÃ CHỐT.
```

Tin nhắn trong Outbox sẽ không bay ra ngoài lừa tình thiên hạ cho đến khi bạn Chốt Sổ thành công. Người nhận tự xây Inbox để chống chạy đúp (idempotency).

Lệnh `NOTIFY` của PostgreSQL cũng bắn tin ngay lúc Commit đấy, nhưng nó không phải là hàng bền bỉ (durable). Đứt mạng phát là mất tin luôn; chỉ xài nó khi bạn chấp nhận rủi ro này.

## 8. Lời Thề Đồng Bộ (Portability contract)

Database nào muốn được App hỗ trợ thì phải vượt qua bài Test nhân phẩm này:

```text
1. Tuyệt đối KHÔNG DÙNG Dữ Liệu Dở Dang chưa chốt để vẫy cờ.
2. Trạm Nghỉ (checkpoint) Đã Chốt thì phải đọc được.
3. Cú Tranh Giành Cứu Hộ nguyên tử (atomic claim) chỉ có DUY NHẤT một kẻ chiến thắng.
4. Hủy kèo (Rollback) thì cấm tiệt vụ công bố "Job Đã Xong".
```

Đừng rẽ nhánh If/Else hèn hạ chỉ vì cái chuỗi `"read uncommitted"`. Nếu cần dùng tính năng dị biệt của DB, nhét nó vô một cục Adapter rồi viết giải thích đàng hoàng (document guarantees).

## 9. Cẩm nang Sập Nguồn (Failure behavior)

| Kiểu Sập | Hậu quả để lại |
| --- | --- |
| Giao dịch chính (Main Tx) Hủy kèo | Không có Trạng thái Kết thúc (final state); Các Trạm Nghỉ trước đó vẫn sống nguyên. |
| Nhịp tim Chốt, nhưng Main Tx Hủy kèo | Bảng Nhịp Tim vẫn phơi xác số liệu cố gắng; Không có Kết quả cuối cùng. |
| Người đẩy Nhịp tim bị sập chết | Watchdog có thể sẽ nhảy vào cướp Giấy Phép thuê (expire lease); Thằng Processor tự lo mà bảo vệ thân mình (fence). |
| 100 ông Watchdog cùng bu vô | Tranh Giành nguyên tử đập chết 99 ông, chỉ cho 1 ông ăn. |
| Ông Đưa thư (Outbox) phát trùng tin | Inbox của người nhận tự động gạt bỏ (idempotency). |
| Đọc lấy Khóa chờ quá lâu (Timeout) | Văng lỗi lố giờ (explicit timeout); Tuyệt đối không phơi ra Dữ Liệu Dở Dang. |

## 10. Nghệ thuật Hẹn giờ và Ngân sách Cứu hộ (Timeout và recovery budget)

- Thời gian Bắn Nhịp Tim (interval) phải ngắn hơn Thời hạn Thuê (lease duration) để chừa đường lui cho lag mạng (margin).
- Watchdog phải tin vào đồng hồ của Database, hoặc dùng chung một hệ đo lường chuẩn mực.
- Đòi quyền Cứu hộ thì phải canh lố giờ (bounded transaction/lock timeout).
- Lỡ đòi Quyền thất bại (zero-row) thì đừng có điên cuồng vã Retry đấm vào hệ thống (storm).
- Thằng làm việc (Processor) phải ngó lại xem "Mình còn quyền không?" trước khi đi Chốt sổ (final commit).
- Mọi trò dọn dẹp rác khi Crash phải dựa trên Database, cấm dùng biến Cờ (in-memory flag) trong RAM.

Chả có con số Hẹn Giờ nào gọi là "Chuẩn toàn vũ trụ" cả. Hãy vác biểu đồ (distribution) ra mà canh theo SLO của hệ thống nhà bạn.

## 11. Bảng Trọng Tài Cân Đo Đong Đếm (Trade-off comparison)

| Tuyệt chiêu | Tầm Nhìn Dữ Liệu Thấy Được | Quy mô Nguyên tử | Độ Rắc Rối | Phù hợp khi nào? |
| --- | --- | --- | --- | --- |
| Chỉ đọc Đồ Đã Chốt | Sau khi thằng Chính chốt sổ | Trọn gói nguyên con (Full unit) | Dễ ẹc (Thấp) | Job chạy nhanh như gió |
| Băm Trạm Nghỉ (Chunks) | Sau mỗi cục Chunk chốt sổ | Từng cục Chunk riêng lẻ | Thường thường (Vừa) | Workflow dài, có thể chạy tiếp |
| Bơm Nhịp tim Độc Lập | Thấy ngay Tín hiệu Đang Ráng Làm | Trạng thái Nhịp Tim TÁCH BIỆT Kết Quả Cuối | Nhức não (Vừa-cao) | Job ngâm tôm quá lâu |
| Đòi Quyền Thuê Nguyên tử | Chỉ 1 thằng Tướng Cứu Hộ được sinh ra | Cú đòi quyền (Claim only) | Vừa | 100 thằng Watchdog đánh nhau |
| Đọc Lấy Khóa `FOR UPDATE` | Ngồi đợi thằng kia làm xong xuôi | Khóa cùng chung Dòng (Same row lock) | Vừa | Chờ đợi đồng bộ siêu ngắn |
| Thùng thư Outbox | Bất đồng bộ (Async) sau khi Chốt sổ | DB + Sự Kiện chốt cùng lúc | Đổ mồ hôi (Cao) | Đám đông hóng hớt tin tức |

## 12. Lời Khuyên Vàng Ngọc Từ Bậc Tiền Bối (Recommendation cho case này)

1. Vứt ngay cái thói bấu víu vào Đọc Bẩn `READ_UNCOMMITTED` vào sọt rác!
2. Job mà kéo dài, hãy Băm Trạm Nghỉ hoặc quăng Nhịp Tim ra Bảng Riêng.
3. Phải rạch ròi giữa "Tiến độ đang cố làm" (attempt) và "Kết quả chung cuộc đã xong" (final outcome).
4. Watchdog muốn Giải Cứu thì phải đè ra làm cú Claim Tranh Giành Thế Hệ (generation/lease).
5. Luôn check lại "Mình còn là Chính chủ không?" (Fencing) trước khi chốt hạ bất cứ Dữ liệu Vĩnh Viễn nào.
6. Muốn test trò Chốt/Hủy, xách con PostgreSQL thật ra mà chạy.

## 13. Bảng Phong Thần Trước Khi Lên Production (Production checklist)

### Về mặt Luân thường đạo lý (Semantics)

- [ ] Lời thề: Không có quyết định nghiệp vụ nào dùng Đọc Bẩn để phán xét.
- [ ] Tiến độ, Nhịp Tim và Kết Quả Cuối Cùng đã có Bảng Tên và Hợp Đồng rõ ràng.
- [ ] Test thử Hủy Kèo (Rollback) xem nó có lén báo "Xong rồi" không.
- [ ] Vứt cái Nhãn mức Cô Lập (Isolation label) đi, cấm lấy nó làm bằng chứng biện minh.
- [ ] Test thật (Integration test) gánh được trò đổi DB.

### Về mặt Chống Nhau (Coordination)

- [ ] Cứu hộ bằng Cú Đấm Nguyên Tử (atomic claim), dẹp trò "Nhìn trước Cướp sau" (check-then-act).
- [ ] Thế hệ và Token của Chủ Sở Hữu (Owner) được Soi Kỹ trước khi Ghi (validate).
- [ ] Giao việc (Chunk/command delivery) có chìa khóa miễn dịch (idempotency key).
- [ ] Xài `REQUIRES_NEW` là ĐÃ HIỂU LUẬT, tự chịu trách nhiệm.
- [ ] TUYỆT ĐỐI KHÔNG để Giao Dịch Con (inner NEW) đòi chọt Dòng đã bị Giao Dịch Cha (outer Tx) Khóa.

### Về mặt Vận Hành (Operations)

- [ ] Định thời Nhịp Tim/Thuê Mướn (heartbeat/lease timing) đã trừ hao cho đứt cáp mập cắn (failure margin).
- [ ] Giăng lưới bắt bọn Giao Dịch Ngâm Tôm (Long transaction age).
- [ ] Đặt đồng hồ đo số lần Giải cứu chạy đúp và Bị đá văng vì hết quyền (Metrics).
- [ ] Nếu dùng Đọc Ép Khóa thì đã canh thời gian nổ bom (Bounded timeouts).
- [ ] Đối soát dữ liệu (Reconciliation) phân minh rõ ràng đâu là "Cố gắng làm" (attempt) đâu là "Kết liễu" (terminal).
