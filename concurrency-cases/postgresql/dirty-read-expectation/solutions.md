# Giải Pháp Theo Dõi Tiến Độ Chuẩn Mực (Committed coordination and progress solutions)

## 1. Mục tiêu thiết kế kiến trúc

Một giải pháp thiết kế toàn vẹn phải phân biệt rõ ràng ba nhóm trạng thái sau:

```text
Trạng thái nghiệp vụ cuối cùng (final business state):
  Chỉ được ghi nhận và truy xuất sau khi Giao dịch (transaction) chính thức hoàn tất (commit).

Tín hiệu tiến độ / nhịp tim (attempt progress/heartbeat):
  Hoạt động trên một tiến trình lưu trữ độc lập, tự chủ kết thúc (commit) thông qua phiên bản (generation) và định danh chủ sở hữu (owner).

Cơ chế khôi phục (recovery):
  Quy trình giải cứu hoặc khởi động lại tiến trình phải áp dụng thao tác giành quyền nguyên tử (atomic claim), tránh tình trạng phân tích thiếu chính xác (đọc cũ - giải cứu mù quáng).
```

> **Nói ngắn gọn:** Cơ chế chia sẻ trạng thái tiến độ phải dựa trên việc phát đi một thông điệp đã hoàn thiện việc lưu trữ. Tuyệt đối không biến những bản nháp (uncommitted data) đang xử lý trong một khối giao dịch thành một kênh truyền thông (message bus) của hệ thống.

## 2. Giải Pháp 1 — Chấp nhận "Chỉ đọc dữ liệu đã commit" (Committed-only observation)

Nếu yêu cầu giám sát hoặc hiển thị giao diện không bắt buộc cung cấp trạng thái theo thời gian thực (real-time progress):

```java
@Service
public class JobStatusReader {
    private final JobRunRepository jobs;

    public JobStatusReader(JobRunRepository jobs) {
        this.jobs = jobs;
    }

    // Đọc trạng thái bình thường bằng READ_COMMITTED mặc định
    @Transactional(readOnly = true)
    public JobSnapshot read(UUID jobId) {
        return jobs.findById(jobId)
            .orElseThrow()
            .snapshot();
    }
}
```

Hãy sử dụng mức độ `READ_COMMITTED`. Hãy làm rõ trong tài liệu kỹ thuật: "Trạng thái này chỉ được cập nhật sau khi công việc hoàn tất và chốt sổ." Giao diện (UI) nên hiển thị theo kiểu "Cập nhật lần cuối lúc..." (`lastCommittedAt`), thay vì hiển thị một thanh tiến độ dễ gây hiểu lầm.

Giải pháp này cực kỳ hiệu quả khi tiến trình diễn ra nhanh chóng, và hệ thống chấp nhận khoảng thời gian ngắn có dữ liệu tĩnh (stale-until-commit).

## 3. Giải Pháp 2 — Chia nhỏ giao dịch (Committed checkpoints/Chunks)

Với những công việc diễn ra lâu dài (long-running workflow), phương án tối ưu là chia nhỏ chúng thành nhiều phần (chunks):

```text
BẮT ĐẦU (BEGIN)
  Xử lý phân đoạn công việc hiện tại (đảm bảo tính idempotent)
  Lưu trạng thái hoàn thành phân đoạn (checkpoint)
CHỐT SỔ (COMMIT)
```

Áp dụng truy vấn chèn dữ liệu nguyên tử (atomic INSERT) để xử lý việc giành quyền:

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
        // Thực hiện giành quyền xử lý nguyên tử phân đoạn
        int claimed = results.claim(chunkId, jobId);
        if (claimed == 0) { // Đã có luồng khác giành quyền
            ChunkResult existing =
                results.findByChunkId(chunkId).orElseThrow();
            return ChunkOutcome.replayed(existing); // Nhường quyền, coi như đã hoàn tất
        }

        // Lấy khóa bảo vệ (Row Lock) để cập nhật tiến trình chính
        JobRun job = jobs.findByIdForUpdate(jobId)
            .orElseThrow();
        job.advanceCommittedCheckpoint(checkpoint);
        ChunkResult result =
            results.findByChunkId(chunkId).orElseThrow();
        return ChunkOutcome.applied(result);
    }
}
```

Trường `chunk_id` độc nhất kết hợp cấu trúc `ON CONFLICT DO NOTHING` ngăn ngừa xung đột thi hành kép, chuyển hóa chúng thành cơ chế điều phối cạnh tranh. Các bộ phận theo dõi (Watchdog) lúc này có thể đọc thông tin của Phân Đoạn đã lưu thành công. Trong trường hợp xảy ra lỗi Rollback, trạng thái quyền xử lý phân đoạn đó sẽ tự động bị rút lại.

**Các yêu cầu đi kèm (Trade-off):**

- Phá vỡ nguyên tắc thao tác "Tất cả hoặc không gì cả" (all-or-nothing) đối với quy trình tổng thể;
- Đòi hỏi xây dựng mã nguồn tự kháng lỗi (resumable/idempotent);
- Quản lý mô hình trạng thái (State machine) phức tạp hơn;
- **Đổi lại:** Các giao dịch, Khóa (Lock), và Kết nối (Connection) được giải phóng rất nhanh.

## 4. Giải Pháp 3 — Tạo cấu trúc thông báo Nhịp Tim độc lập (Independent heartbeat record)

Khi tiến trình lớn không thể chia nhỏ, giải pháp tốt nhất là lưu thông tin tiến độ "Nhịp Tim" vào một bảng độc lập:

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

Component báo cáo tín hiệu (Publisher) sử dụng giao dịch tách biệt:

```java
@Service
public class JobHeartbeatPublisher {
    private final JobHeartbeatRepository heartbeats;

    public JobHeartbeatPublisher(
        JobHeartbeatRepository heartbeats
    ) {
        this.heartbeats = heartbeats;
    }

    // Annotation này sinh ra một khối Giao dịch độc lập và đóng ngay lập tức
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void publish(Heartbeat heartbeat) {
        int changed = heartbeats.upsertIfOwnerMatches(
            heartbeat.jobId(),
            heartbeat.generation(),
            heartbeat.ownerToken(),
            heartbeat.progressPercent(),
            heartbeat.observedAt()
        );
        if (changed != 1) { // Không tìm thấy (Mất quyền)
            throw new StaleJobOwnerException(
                heartbeat.jobId(),
                heartbeat.generation()
            );
        }
    }
}
```

Truy vấn Cập Nhật/Thêm mới (Upsert):

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
-- CHỈ CẬP NHẬT KHI LÀ CHÍNH CHỦ VÀ TIẾN ĐỘ ĐANG ĐI TỚI
where job_attempt_heartbeat.owner_token = excluded.owner_token
  and job_attempt_heartbeat.progress_percent
      <= excluded.progress_percent;
```

Dữ liệu nhịp tim sẽ tiếp tục lưu giữ dẫu cho Giao dịch chính (Main Transaction) có gặp lỗi (rollback). Ý nghĩa của chúng là mô tả "Hệ thống đang nỗ lực đạt đến mức độ này" (attempt progress), chứ không đại diện cho "Kết quả chung cuộc hoàn tất" (final success).

### Tránh lỗi vòng khóa đệ quy (Self-deadlock)

Bắt buộc cẩn trọng khi một giao dịch ngoài (outer transaction) đã nắm giữ Khóa cấp dòng, không được sử dụng lệnh gọi `REQUIRES_NEW` để yêu cầu chỉnh sửa ở chính dòng đó:

```text
Giao dịch Cha nắm giữ Khóa Dòng.
Giao dịch Con (REQUIRES_NEW) xếp hàng đợi Giao dịch Cha trả khóa.
Giao dịch Cha đợi Giao dịch Con hoàn tất để thực thi tiếp.
Phát sinh Deadlock.
```

Vì vậy Bảng Nhịp Tim phải nằm ở một Entity riêng biệt, tránh xung đột cấu trúc. (Và nên gia tăng giới hạn Connection Pool nếu hệ thống liên tục khởi tạo luồng `REQUIRES_NEW`).

## 5. Giải Pháp 4 — Tranh Giành Quyền Phục Hồi Nguyên Tử (Atomic recovery claim)

Các cơ chế Giám sát (Watchdog) phải kích hoạt tiến trình giải cứu bằng cách viết một lệnh truy cập nguyên tử để cấp phát quyền (conditional write):

```sql
update job_run
set generation = generation + 1,
    owner_token = :newOwnerToken,
    lease_until = :newLeaseUntil,
    status = 'RUNNING'
where job_id = :jobId
  and generation = :observedGeneration -- Xác thực với thế hệ vừa quét
  and status = 'RUNNING' -- Tiến trình vẫn chưa hoàn tất
  and lease_until < :databaseNow -- Phải qua mức giới hạn hết hạn
returning generation, owner_token, lease_until;
```

Sự quyết định dựa vào số lượng kết quả phản hồi:

| Phản hồi | Quyết định xử lý |
| --- | --- |
| 1 Dòng | Watchdog giành quyền thành công! Xác thực mở phiên bản mới. |
| 0 Dòng | Yêu cầu bị loại. Cơ sở khác đã xử lý hoặc Job đã kết thúc. |

Mọi thay đổi tiếp theo của nhiệm vụ (side effect) sẽ đều gắn với một "Chứng thực Phiên Bản" (generation/fencing token). Trong trường hợp Processor cũ hoạt động lại, yêu cầu đóng sổ (final commit) sẽ bị loại bỏ do giới hạn của chứng thực. Lệnh xử lý nguyên tử (atomic) trên CSDL bảo vệ hệ thống khỏi việc hàng loạt Watchdog cùng khởi chạy tiến trình, điều mà `READ_UNCOMMITTED` không bao giờ đảm nhận.

## 6. Giải Pháp 5 — Chờ Đợi Bằng Cơ Chế Khóa (Locking read)

Trường hợp hệ thống cần luồng đọc đứng chờ để đồng bộ hóa:

```sql
select job_id, status, progress_percent, generation
from job_run
where job_id = :jobId
for update; -- Giữ luồng chờ đồng bộ!
```

Bộ đọc sẽ duy trì trạng thái nghỉ (hoặc đợi lố giờ timeout) cho đến khi bộ Ghi thực hiện thao tác commit hoặc rollback để kết luận giá trị. Quá trình này không hiển thị dữ liệu dở dang (dirty read).

Mô hình này tối ưu cho các luồng tương tác cực ngắn (short critical). NHƯNG Không khuyến khích áp dụng cho Polling trên giao diện vì:

- Hao tốn số lượng Kết Nối trong Pool (Connection hold).
- Gây quá tải Hàng đợi Khóa (lock queue), tăng độ trễ tổng thể hệ thống.
- Yêu cầu cấu trúc ngăn chặn ngoại lệ khi lố giờ (timeout/deadlock).
- Không phản ánh kịp thời Nhịp Tim tiến trình.

## 7. Giải Pháp 6 — Hàng Đợi Thông Điệp Bền Bỉ (Durable event/outbox)

Cung cấp thông tin tiến độ cho bên thứ 3 (Consumers) với độ tin cậy cao:

```text
Giao Dịch Xử Lý (business Tx)
  -> Cập nhật nghiệp vụ Job
  -> Ghi thông báo Tín hiệu (Event) vào Hàng đợi Outbox
ĐÓNG GIAO DỊCH (COMMIT)

Thành phần chuyển phát (Publisher) xử lý gửi Thông điệp ĐÃ CHỐT tới bên ngoài.
```

Thông điệp nằm trong Outbox sẽ chỉ được truy xuất nếu quá trình commit diễn ra trọn vẹn. Bên tiếp nhận chủ động triển khai cơ chế loại bỏ tin trùng lặp (idempotency/Inbox).

Câu lệnh `NOTIFY` của PostgreSQL cũng bắn tín hiệu sau khi Commit, nhưng không cung cấp tính bảo lưu lâu dài (durable). Dữ liệu có thể bị suy giảm trong trường hợp sự cố mạng; do đó, hãy cẩn trọng với tính năng này.

## 8. Tiêu Chuẩn Đồng Bộ Kiến Trúc (Portability contract)

Hệ thống nên tương thích bằng cách định nghĩa các nguyên tắc (guarantees) thay vì lợi dụng kẽ hở DB:

```text
1. Tuyệt đối không dựa vào dữ liệu trung gian chưa Commit.
2. Trạm dừng (Checkpoints) cần phải được hiển thị chính xác ngay sau khi Commit.
3. Yêu cầu Cứu Hộ phải duy trì thông số nguyên tử, với duy nhất MỘT quyền sở hữu (Exclusive Owner).
4. Ngoại lệ Hủy Bỏ (Rollback) thì cấm thông báo quá trình là "Hoàn Thành".
```

Tránh thiết lập điều kiện rẽ nhánh logic phụ thuộc vào chuỗi `"read uncommitted"`. Nếu hệ thống cần tương tác chuyên biệt cho DB, nên tách chúng thành những Adapter riêng lẻ đi kèm định dạng bảo vệ.

## 9. Định Hướng Phân Tích Lỗi Hệ Thống (Failure behavior)

| Hình Thái Sự Cố | Kết Quả / Đánh Giá |
| --- | --- |
| Giao dịch chính (Main Tx) Rollback | Giao dịch không cung cấp Trạng Thái Hoàn Thành (final state); Các phân đoạn Checkpoint vẫn lưu giá trị. |
| Nhịp tim thành công, Main Tx bị Rollback | Trạng Thái Nhịp tim bảo lưu quá trình đã hoạt động; Bảng chính chưa cung cấp Dữ Liệu Hoàn Chỉnh. |
| Publisher của Nhịp Tim gặp lỗi Crash | Luồng Watchdog sẽ giành quyền dựa trên thời hạn hợp đồng (expire lease); Bộ Ghi cần tự vệ thông qua token Fencing. |
| Đồng loạt Watchdog tranh giành quyền | Cấu trúc phân giải nguyên tử từ DB ưu tiên một bên chiến thắng. |
| Bộ chuyển Outbox gửi lặp thông báo | Cơ chế Inbox tự bảo vệ (idempotent rejection). |
| Đọc khóa (FOR UPDATE) Timeout | Chuyển luồng sang ngoại lệ (Explicit timeout exception); Hệ thống bảo mật toàn bộ dữ liệu dở dang. |

## 10. Chiến Lược Đặt Giới Hạn Thời Gian Cứu Hộ (Timeout và recovery budget)

- Khoảng thời gian thông báo (interval) Nhịp Tim phải diễn ra thường xuyên hơn Thời hạn hiệu lực của Token (lease duration) (bổ sung sai số).
- Watchdog phải chia sẻ chung mốc thời gian hệ thống DB.
- Quy trình yêu cầu Quyền Cứu Hộ phải tích hợp giám sát quá tải lố giờ (Bounded transaction/lock timeout).
- Dừng ngay thao tác vã liên tục Truy vấn (Retry storm) khi CSDL từ chối yêu cầu bằng kết quả không có hàng nào (zero-row).
- Tiến trình xử lý (Processor) luôn tự kiểm tra "Quyền chủ sở hữu" trước khi gửi xác nhận cuối (final commit).
- Mọi hoạt động dọn dẹp cấp thiết phải thi hành từ cơ sở CSDL, cấm sử dụng cờ báo động trên RAM.

Mọi khoảng số thời gian giới hạn phải thiết lập cẩn thận dựa theo Biểu đồ phân bổ của cấu trúc hệ thống (SLO Distribution) tại từng Doanh nghiệp.

## 11. Bảng Trọng Tài Cân Nhắc Thiết Kế (Trade-off comparison)

| Cơ Chế | Tầm Nhìn Dữ Liệu (Visibility) | Quy Mô Nguyên Tử | Mức Độ Phức Tạp | Chỉ Định |
| --- | --- | --- | --- | --- |
| Chỉ Đọc Dữ liệu Đã Commit | Chờ Giao dịch xử lý hoàn tất | Toàn bộ đơn vị (Full Unit) | Thấp | Tiến trình xử lý nhanh chóng |
| Chia Đoạn Checkpoint | Chờ mỗi Phân đoạn hoàn tất | Các Phân đoạn (Chunk) độc lập | Vừa | Tiến trình lâu dài, hỗ trợ tiếp tục (resume) |
| Cập nhật Nhịp Tim Độc Lập | Mô phỏng Tiến độ quá trình hoạt động | Trạng thái Nhịp Tim TÁCH BIỆT với Kết quả cuối | Khá-Cao | Trì hoãn thao tác giao dịch rất lớn |
| Giành Quyền Atomic | Giới hạn Một thực thể Khôi phục | Giao dịch Yêu Cầu Cấp Phép (Claim) | Vừa | Quản lý tranh chấp Watchdog |
| Khóa Hàng `FOR UPDATE` | Chờ đợi luồng Đồng bộ | Khóa Hàng Tương Đối (Same row) | Vừa | Giao dịch ngắn, yêu cầu độ trễ thấp |
| Hàng đợi Outbox | Bất đồng bộ sau Commit | Kết hợp Dữ liệu & Sự Kiện | Cao | Hệ thống rộng, nhiều đối tượng Consumer |

## 12. Định Hướng Khuyến Nghị (Recommendation cho case này)

1. Gỡ bỏ triệt để việc thiết lập mức cô lập `READ_UNCOMMITTED` khỏi tư duy kiến trúc.
2. Với các tiến trình kéo dài, triển khai cấu trúc Chia Nhỏ Phân Đoạn (Chunks) hoặc Nhịp Tim (Heartbeat).
3. Làm rõ định nghĩa về thông điệp tiến độ "Đang xử lý" (attempt) và "Kết quả cuối cùng" (final outcome).
4. Nâng cấp mô hình Cứu Hộ Watchdog sang phương pháp Giành Quyền cấp phiên bản (generation/lease claim).
5. Luôn yêu cầu bộ xử lý xác thực phiên bản Chủ Sở Hữu (Fencing) trước thao tác cập nhật dữ liệu cốt lõi.
6. Sử dụng CSDL Thực (như PostgreSQL Testcontainers) trong môi trường phát triển để xác thực các kiểm thử Rollback/Commit.

## 13. Danh Sách Kiểm Tra Trước Khi Triển Khai (Production checklist)

### Tính Chuẩn Xác Dữ Liệu (Semantics)

- [ ] Đồng ý nguyên tắc: Không dựa vào Đọc Bẩn trong bất kỳ quy trình kiểm thử và kinh doanh nào.
- [ ] Xác minh cấu trúc Bảng Dữ Liệu cho Tiến độ, Nhịp Tim và Hoàn Thành xử lý rõ ràng.
- [ ] Kiểm định thao tác Rollback để chắc chắn rằng thông báo "Tiến trình Xong" không bị lọt.
- [ ] Xóa bỏ nhãn `isolation label` để tránh lỗi phát triển tương lai.
- [ ] Dữ liệu bài kiểm thử Tích hợp (Integration test) cần chứng thực tính linh động cấu hình hệ CSDL.

### Khả Năng Điều Phối (Coordination)

- [ ] Cấu trúc Cơ chế Cứu Hộ chuyển sang phương pháp Ghi Nguyên Tử (atomic claim), xóa bỏ kỹ thuật "Đọc cũ rồi Hốt" (check-then-act).
- [ ] Xác minh mã thẻ Sở Hữu (Owner Token) mỗi khi luồng Ghi diễn ra.
- [ ] Thiết lập quy trình truyền thông tin (Chunk/Event) tích hợp cơ chế Chống Giao dịch đúp (Idempotency).
- [ ] Sử dụng Annotation `REQUIRES_NEW` cẩn thận, nhận thức sâu sắc vòng đời hoạt động.
- [ ] Chú ý NGĂN CẢN Giao dịch Nội (inner NEW) cạnh tranh Khóa với Giao dịch Ngoại (outer Tx) khi tương tác vào cùng Khối Dữ Liệu.

### Hoạt Động Giám Sát (Operations)

- [ ] Thời gian gia hạn vòng đời Nhịp Tim/Hợp đồng được cài đặt số khoảng chênh lệch dự phòng (Margin).
- [ ] Khởi chạy công cụ theo dõi tuổi thọ các Giao Dịch Dài Hạn (Long transaction).
- [ ] Thống kê số lượng các Tranh Giành Hợp đồng thành công và Bị đá văng do hết hạn (Metrics).
- [ ] Gán khung định mức giới hạn thời gian chạy (Bounded timeouts) đối với mọi Yêu cầu khóa.
- [ ] Bộ chạy giám sát (Reconciliation) phải rạch ròi quy mô đo lường Dữ liệu Nháp và Dữ Liệu Kết Thúc Hoàn Toàn.
