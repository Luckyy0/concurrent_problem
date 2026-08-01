# DB-010 — Xử lý đồng thời các tiến trình với `FOR UPDATE SKIP LOCKED`

## Tóm tắt

Nhiều tiến trình xử lý cùng lúc đọc và lấy công việc từ một bảng hàng đợi. Nếu chúng ta chỉ dùng lệnh `SELECT` thông thường, nhiều tiến trình có thể sẽ lấy trùng một công việc. Nếu chúng ta dùng `FOR UPDATE` để khóa dòng nhưng quên dùng `SKIP LOCKED`, các tiến trình sẽ phải xếp hàng chờ đợi nhau, gây ra tình trạng lock convoy.

Pattern khuyến nghị để giải quyết:

```text
transaction cực ngắn: Lấy các công việc ở trạng thái READY bằng FOR UPDATE SKIP LOCKED
→ commit để chốt việc lấy công việc
→ Xử lý công việc ở bên ngoài transaction
→ transaction cực ngắn: Hoàn thành CHỈ KHI claim_token khớp!
```

Cú pháp `SKIP LOCKED` cho phép các tiến trình bỏ qua những dòng dữ liệu đang bị khóa bởi các transaction khác và ngay lập tức lấy dòng tiếp theo. Cơ chế này cố tình tạo ra một góc nhìn không nhất quán về dữ liệu, rất phù hợp cho hàng đợi công việc nhưng lại không phù hợp cho các truy vấn mang tính phân tích, tổng hợp toàn cục.

> **Nói ngắn gọn:** Khóa dòng chỉ bảo vệ chúng ta ở bước lấy việc; còn để bảo vệ toàn bộ vòng đời của công việc sau khi đã lấy thành công (hoặc khi hệ thống gặp sự cố), chúng ta cần kết hợp thêm cơ chế cho thuê, thẻ định danh quyền sở hữu và tính chất lũy đẳng khi xử lý tác vụ bên ngoài.

## Actor và trạng thái dùng chung

| Thành phần | Vai trò |
| --- | --- |
| Bảng `work_job` | Hàng đợi công việc mang tính nguồn chuẩn |
| Tiến trình A/B/C | Các tiến trình hoặc pod chạy song song để lấy việc |
| Điểm đến bên ngoài | Các API bên ngoài, hệ thống email, cổng thanh toán |
| Tiến trình dọn dẹp | Tiến trình dọn dẹp, có nhiệm vụ đưa các công việc đã hết hạn thuê trở lại hàng đợi |

Mỗi công việc có các thông tin quan trọng như trạng thái (`status`), độ ưu tiên (`priority`), thời gian sẵn sàng (`available_at`), thẻ định danh lấy việc (`claim_token`), người lấy việc (`claimed_by`), thời hạn thuê (`lease_until`) và số lần thử (`attempt_count`).

## Rào chắn tính đúng đắn

```text
Một công việc ở trạng thái READY chỉ được phép có nhiều nhất một tiến trình lấy và xử lý tại một thời điểm.

Chỉ tiến trình nào đang giữ claim_token hiện tại mới có quyền hoàn thành hoặc yêu cầu thử lại công việc đó.

Sự cố hệ thống không được làm mất công việc; nếu một công việc hết thời hạn thuê, nó phải có cơ chế để được thu hồi.

Công việc có thể bị chạy lại, do đó các tác vụ bên ngoài phải đảm bảo tính lũy đẳng dựa trên ID của công việc.
```

Ở bài toán này, chúng ta thiết kế hệ thống đảm bảo xử lý "ít nhất một lần", chứ không cố gắng ép buộc hệ thống bên ngoài phải đạt được trạng thái "chính xác một lần" ngay từ trong database.

## Ranh giới transaction

### Lấy việc

Một transaction `READ COMMITTED` sử dụng CTE:

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

Khóa sẽ được giữ cho đến khi transaction `commit`. Các tiến trình chỉ được bắt đầu xử lý tác vụ bên ngoài SAU KHI quá trình lấy việc đã được `commit` thành công.

### Hoàn thành

Một transaction cập nhật có điều kiện riêng biệt:

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

Nếu trả về số dòng bị ảnh hưởng là `1` nghĩa là đã hoàn thành công việc. Nếu trả về `0`, điều này có nghĩa là tiến trình hiện tại đã bị tước quyền sở hữu (mất quyền sở hữu công việc này) và không được phép ghi đè trạng thái.

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong ngữ cảnh này |
| --- | --- |
| Lấy việc nguyên tử | Việc lấy công việc diễn ra trọn vẹn, không thể chia cắt |
| `SKIP LOCKED` | Cơ chế bỏ qua các dòng đang bị khóa bởi transaction khác |
| Lock convoy | Hiện tượng tắc nghẽn khi nhiều tiến trình xếp hàng chờ đợi một dòng đang bị khóa |
| Góc nhìn không nhất quán | Các tiến trình thấy các tập hợp công việc khác nhau do đã bỏ qua các dòng bị khóa |
| Thời hạn thuê | Khoảng thời gian cho thuê, tiến trình chỉ có quyền sở hữu công việc cho đến mốc `lease_until` |
| Thẻ định danh | Thẻ định danh dùng làm rào chắn bảo vệ cho một phiên lấy việc |
| Tiến trình cũ | Tiến trình vẫn tiếp tục xử lý công việc dù công việc đó đã bị hệ thống thu hồi và giao cho người khác |
| Chết đói | Tình trạng công việc liên tục bị bỏ qua và không bao giờ được xử lý |
| Ít nhất một lần | Chấp nhận việc xử lý lại công việc sau khi hệ thống gặp sự cố hoặc quá trình hoàn thành không rõ ràng |

## Hướng dẫn điều hướng

- [Code lỗi: duplicate và lock convoy](broken-code.md)
- [Phân tích quá trình lấy việc, tính công bằng và phục hồi sự cố](analysis.md)
- [Giải pháp lấy việc nguyên tử, thẻ định danh quyền sở hữu và phục hồi thời hạn thuê](solutions.md)
- [Các thí nghiệm với Testcontainers](experiments.md)
- [Các loại khóa trong PostgreSQL và vòng đời của chúng](../../concepts/postgresql-locks.md)
- [Kiểm thử các vấn đề đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production nếu làm sai

- Các tác vụ bên ngoài sẽ bị thực thi trùng lặp nếu hai tiến trình cùng lấy một công việc.
- Thông lượng của hệ thống sẽ giảm xuống chỉ bằng tốc độ của một tiến trình nếu các tiến trình lấy việc phải xếp hàng chờ đợi công việc cũ nhất.
- Cạn kiệt connection pool nếu các tiến trình giữ transaction trong khi đang gọi các dịch vụ bên ngoài.
- Các công việc bị kẹt vĩnh viễn ở trạng thái `PROCESSING` sau khi tiến trình xử lý gặp sự cố mà không có cơ chế phục hồi thời hạn thuê.
- Một tiến trình cũ có thể ghi đè kết quả của một tiến trình mới nếu không kiểm tra điều kiện thẻ định danh.
- Các công việc có độ ưu tiên thấp hoặc vô tình bị kẹt khóa liên tục có thể dẫn đến tình trạng không bao giờ được xử lý.
- Việc mở rộng hệ thống có thể làm quá tải database do việc lấy việc liên tục nếu không có giới hạn kích thước lô hoặc thời gian chờ.

## Hướng sửa chữa (Khuyến nghị)

1. Lấy việc bằng một transaction cực ngắn sử dụng thứ tự `ORDER BY` rõ ràng, có giới hạn số lượng và kết hợp cơ chế `FOR UPDATE SKIP LOCKED`.
2. Hoàn thành việc lấy công việc TRƯỚC KHI bắt đầu xử lý tác vụ bên ngoài.
3. Hoàn thành hoặc đánh dấu lỗi công việc bằng điều kiện kết hợp cả `job_id` và `claim_token`.
4. Có cơ chế quét định kỳ các công việc đã hết hạn thuê, giới hạn số lần thử lại và đưa vào trạng thái lỗi vĩnh viễn.
5. Các hệ thống bên ngoài cần phải có cơ chế chống trùng lặp dựa trên một mã định danh công việc hoặc tác vụ cố định.
6. Giám sát hệ thống bằng cách đo lường thời gian nằm chờ của công việc cũ nhất, số lượng công việc hết hạn thuê, tần suất thu hồi công việc và các trường hợp hoàn thành trễ.

## Khi nào phù hợp áp dụng

Pattern này rất phù hợp cho các bảng hoạt động như hàng đợi với lưu lượng ở mức độ trung bình, và khi database đã là nguồn lưu trữ chuẩn. Tuy nhiên, nếu bạn cần khả năng lưu giữ dữ liệu lâu dài, khả năng mở rộng cực lớn bằng phân vùng, các nhóm tiêu thụ dữ liệu phức tạp hoặc tính năng phát lại luồng sự kiện, thì các hệ thống chuyên dụng như Kafka/RabbitMQ sẽ là sự lựa chọn tốt hơn.

## Phạm vi của bài viết

Bài viết này tập trung phân tích và giải quyết các bài toán liên quan đến việc lấy công việc trong PostgreSQL. Chúng ta không đi sâu vào cơ chế phân vùng của Kafka. Phân tích về lỗi chọn dữ liệu bằng khóa bi quan được trình bày trong case `LOCK-003`; còn các vấn đề chi tiết về đảm bảo việc gửi thông điệp và outbox pattern thuộc phạm vi của các bài toán về hệ thống truyền thông điệp.
