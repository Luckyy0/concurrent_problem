# DB-010 — Concurrent workers với `FOR UPDATE SKIP LOCKED`

## Tóm tắt

Nhiều tiến trình xử lý (worker) cùng lúc đọc và lấy công việc (job) từ một bảng hàng đợi (queue table). Nếu chúng ta chỉ dùng lệnh `SELECT` thông thường, nhiều worker có thể sẽ lấy trùng một công việc. Nếu chúng ta dùng `FOR UPDATE` để khóa dòng (row) nhưng quên dùng `SKIP LOCKED`, các worker sẽ phải xếp hàng chờ đợi nhau, gây ra tình trạng tắc nghẽn (lock convoy).

Pattern khuyến nghị để giải quyết:

```text
Giao dịch (transaction) cực ngắn: Lấy các công việc ở trạng thái READY bằng FOR UPDATE SKIP LOCKED
→ COMMIT để chốt việc lấy công việc
→ Xử lý công việc ở bên ngoài giao dịch (external work)
→ Giao dịch cực ngắn: Hoàn thành (complete) CHỈ KHI claim_token khớp!
```

Cú pháp `SKIP LOCKED` cho phép các worker bỏ qua những dòng dữ liệu đang bị khóa bởi các giao dịch khác và ngay lập tức lấy dòng tiếp theo. Cơ chế này cố tình tạo ra một góc nhìn không nhất quán (inconsistent view) về dữ liệu, rất phù hợp cho hàng đợi công việc nhưng lại không phù hợp cho các truy vấn mang tính phân tích, tổng hợp toàn cục.

> **Nói ngắn gọn:** Khóa dòng (row lock) chỉ bảo vệ chúng ta ở bước lấy việc (claim); còn để bảo vệ toàn bộ vòng đời của công việc sau khi đã lấy thành công (hoặc khi hệ thống gặp sự cố), chúng ta cần kết hợp thêm cơ chế cho thuê (lease), thẻ định danh quyền sở hữu (owner token) và tính chất lũy đẳng (idempotency) khi xử lý tác vụ bên ngoài.

## Actor và trạng thái dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Bảng `work_job` | Hàng đợi công việc mang tính nguồn chuẩn (Authoritative queue table) |
| Worker A/B/C | Các tiến trình hoặc pod chạy song song để lấy việc |
| External sink | Các API bên ngoài, hệ thống email, cổng thanh toán |
| Recovery sweeper | Tiến trình dọn dẹp, có nhiệm vụ đưa các công việc đã hết hạn thuê (expired lease) trở lại hàng đợi |

Mỗi công việc (job) có các thông tin quan trọng như trạng thái (`status`), độ ưu tiên (`priority`), thời gian sẵn sàng (`available_at`), thẻ định danh lấy việc (`claim_token`), người lấy việc (`claimed_by`), thời hạn thuê (`lease_until`) và số lần thử (`attempt_count`).

## Rào chắn tính đúng đắn (Invariant)

```text
Một công việc ở trạng thái READY chỉ được phép có nhiều nhất một worker lấy và xử lý tại một thời điểm (active claim).

Chỉ worker nào đang giữ claim_token hiện tại mới có quyền hoàn thành hoặc yêu cầu thử lại (retry) công việc đó.

Sự cố hệ thống (crash) không được làm mất công việc; nếu một công việc hết thời hạn thuê, nó phải có cơ chế để được lấy lại (reclaim).

Công việc có thể bị chạy lại (retry), do đó các tác vụ bên ngoài (external effect) phải đảm bảo tính lũy đẳng (idempotent) dựa trên job ID.
```

Ở bài toán này, chúng ta thiết kế hệ thống đảm bảo xử lý "ít nhất một lần" (at-least-once), chứ không cố gắng ép buộc hệ thống bên ngoài phải đạt được trạng thái "chính xác một lần" (exactly-once) ngay từ trong cơ sở dữ liệu.

## Ranh giới transaction

### Lấy việc (Claim)

Một giao dịch `READ COMMITTED` sử dụng CTE:

```sql
with candidates as (
    select job_id
    from work_job
    where status = 'READY'
      and available_at <= clock_timestamp()
    order by priority desc, available_at, job_id
    for update skip locked
    limit :batchSize
)
update work_job j
set status = 'PROCESSING',
    claim_token = gen_random_uuid(),
    claimed_by = :workerId,
    lease_until = clock_timestamp() + :lease,
    attempt_count = attempt_count + 1
from candidates c
where j.job_id = c.job_id
returning j.*;
```

Khóa (lock) sẽ được giữ cho đến khi giao dịch `COMMIT`. Các worker chỉ được bắt đầu xử lý tác vụ bên ngoài SAU KHI quá trình lấy việc đã được `COMMIT` thành công.

### Hoàn thành (Complete)

Một giao dịch cập nhật có điều kiện (conditional update) riêng biệt:

```sql
update work_job
set status = 'DONE',
    completed_at = clock_timestamp(),
    claim_token = null,
    lease_until = null
where job_id = :jobId
  and status = 'PROCESSING'
  and claim_token = :claimToken;
```

Nếu trả về số dòng bị ảnh hưởng (affected-row) là `1` nghĩa là đã hoàn thành công việc. Nếu trả về `0`, điều này có nghĩa là worker hiện tại đã bị tước quyền sở hữu (mất quyền sở hữu công việc này) và không được phép ghi đè trạng thái.

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong ngữ cảnh này |
| --- | --- |
| Atomic work claiming | Việc lấy công việc diễn ra trọn vẹn, không thể chia cắt |
| `SKIP LOCKED` | Cơ chế bỏ qua các dòng đang bị khóa bởi giao dịch khác |
| Lock convoy | Hiện tượng tắc nghẽn khi nhiều worker xếp hàng chờ đợi một dòng đang bị khóa |
| Inconsistent view | Góc nhìn không nhất quán, các worker thấy các tập hợp công việc khác nhau do đã bỏ qua các dòng bị khóa |
| Lease | Khoảng thời gian cho thuê, worker chỉ có quyền sở hữu công việc cho đến mốc `lease_until` |
| Claim token | Thẻ định danh dùng làm rào chắn (fencing token) bảo vệ cho một phiên lấy việc |
| Stale worker | Worker vẫn tiếp tục xử lý công việc dù công việc đó đã bị hệ thống thu hồi và giao cho người khác (reclaim) |
| Starvation | Tình trạng "chết đói", công việc liên tục bị bỏ qua và không bao giờ được xử lý |
| At-least-once | Chấp nhận việc xử lý lại công việc sau khi hệ thống sập hoặc quá trình hoàn thành không rõ ràng |

## Hướng dẫn điều hướng

- [Code lỗi: duplicate và lock convoy](broken-code.md)
- [Phân tích claiming, fairness và crash recovery](analysis.md)
- [Giải pháp atomic claim, owner token và lease recovery](solutions.md)
- [Các thí nghiệm với Testcontainers](experiments.md)
- [Các loại khóa trong PostgreSQL và vòng đời của chúng](../../concepts/postgresql-locks.md)
- [Kiểm thử các vấn đề đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production nếu làm sai

- Các tác vụ bên ngoài sẽ bị thực thi trùng lặp (duplicate external effects) nếu hai worker cùng lấy một công việc.
- Thông lượng (throughput) của hệ thống sẽ giảm xuống chỉ bằng tốc độ của một worker nếu các tiến trình lấy việc phải xếp hàng chờ đợi công việc cũ nhất.
- Cạn kiệt kết nối cơ sở dữ liệu (connection pool exhaustion) nếu các worker giữ giao dịch trong khi đang gọi các dịch vụ bên ngoài (remote service).
- Các công việc bị kẹt vĩnh viễn ở trạng thái `PROCESSING` sau khi tiến trình xử lý bị sập (crash) mà không có cơ chế thu hồi (lease recovery).
- Một worker cũ (stale worker) có thể ghi đè kết quả của một worker mới nếu không kiểm tra điều kiện thẻ định danh (token predicate).
- Các công việc có độ ưu tiên thấp hoặc vô tình bị kẹt khóa liên tục có thể dẫn đến tình trạng không bao giờ được xử lý (starvation).
- Việc mở rộng hệ thống (scale-out) có thể làm quá tải cơ sở dữ liệu do việc lấy việc liên tục nếu không có giới hạn kích thước (batch size) hoặc thời gian chờ (backoff).

## Hướng sửa chữa (Khuyến nghị)

1. Lấy việc (claim) bằng một giao dịch cực ngắn sử dụng thứ tự `ORDER BY` rõ ràng, có giới hạn số lượng và kết hợp cơ chế `FOR UPDATE SKIP LOCKED`.
2. Hoàn thành việc lấy công việc (commit claim) TRƯỚC KHI bắt đầu xử lý tác vụ bên ngoài.
3. Hoàn thành hoặc đánh dấu lỗi công việc bằng điều kiện kết hợp cả `job_id` và `claim_token`.
4. Có cơ chế quét (requeue) định kỳ các công việc đã hết hạn thuê (expired claims), giới hạn số lần thử lại (attempt cap) và đưa vào trạng thái lỗi vĩnh viễn (dead-letter).
5. Các hệ thống bên ngoài cần phải có cơ chế chống trùng lặp (deduplicate) dựa trên một mã định danh công việc hoặc tác vụ cố định (stable job/effect key).
6. Giám sát hệ thống bằng cách đo lường thời gian nằm chờ của công việc cũ nhất (oldest-ready), số lượng công việc hết hạn thuê, tần suất thu hồi công việc (reclaim) và các trường hợp hoàn thành trễ (stale completion).

## Khi nào phù hợp áp dụng

Pattern này rất phù hợp cho các bảng hoạt động như hàng đợi (queue-like table) với lưu lượng ở mức độ trung bình, và khi cơ sở dữ liệu đã là nguồn lưu trữ chuẩn (authoritative store). Tuy nhiên, nếu bạn cần khả năng lưu giữ dữ liệu lâu dài (retention), khả năng mở rộng cực lớn bằng phân vùng (partitioning), các nhóm tiêu thụ dữ liệu (consumer groups) phức tạp hoặc tính năng phát lại luồng sự kiện (replay), thì các hệ thống chuyên dụng như Kafka/RabbitMQ sẽ là sự lựa chọn tốt hơn.

## Phạm vi của bài viết

Bài viết này tập trung phân tích và giải quyết các bài toán liên quan đến việc lấy công việc (work claiming) trong PostgreSQL. Chúng ta không đi sâu vào cơ chế phân vùng của Kafka. Phân tích về lỗi chọn dữ liệu bằng khóa bi quan (pessimistic-lock selection) được trình bày trong case `LOCK-003`; còn các vấn đề chi tiết về đảm bảo việc gửi tin nhắn và outbox pattern thuộc phạm vi của các bài toán về messaging.
