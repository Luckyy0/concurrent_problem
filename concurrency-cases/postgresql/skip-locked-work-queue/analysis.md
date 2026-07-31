# Phân tích claiming, fairness và crash recovery

## Trạng thái ban đầu

Giả sử chúng ta có các công việc (jobs) từ J1 đến J4 đều đang ở trạng thái `READY`, cùng độ ưu tiên (priority), và thời gian sẵn sàng (`available_at`) tăng dần. Có hai worker là Worker A và Worker B đang liên tục kiểm tra hàng đợi (poll) trên hai kết nối hoặc giao dịch hoàn toàn độc lập với nhau.

## Timeline plain `SELECT` tạo duplicate

Nếu chúng ta chỉ dùng lệnh `SELECT` thông thường, kịch bản sau sẽ xảy ra:

| Bước | Worker A | Worker B |
| ---: | --- | --- |
| 1 | đọc J1 | |
| 2 | | đọc J1 |
| 3 | gọi dịch vụ bên ngoài (external effect) cho J1 | gọi cùng dịch vụ đó cho J1 |
| 4 | cập nhật trạng thái thành DONE | cập nhật trạng thái thành DONE |

Ở đây, chuỗi hành động Đọc → Xử lý → Ghi không phải là một khối nguyên tử (không atomic). Dù cuối cùng dòng dữ liệu chỉ ghi trạng thái `DONE` một lần, nhưng thực tế tác vụ gọi dịch vụ bên ngoài đã bị thực thi trùng lặp.

## Timeline `FOR UPDATE` tạo convoy

Nếu chúng ta dùng `FOR UPDATE` để khóa dòng nhưng không dùng `SKIP LOCKED`:

| Bước | Worker A | Worker B |
| ---: | --- | --- |
| 1 | khóa J1 | |
| 2 | giữ J1 để xử lý | yêu cầu J1 và bị chặn lại (block) |
| 3 | J2 và J3 vẫn đang READY | không thể bỏ qua để xử lý J2 |
| 4 | commit giao dịch | bây giờ mới lấy được J1 hoặc đánh giá lại |

Trường hợp này không gây ra deadlock (vì không có vòng lặp chờ nhau), nhưng một dòng dữ liệu đang xử lý chậm (hot/slow first row) sẽ làm tất cả các worker khác phải xếp hàng chờ đợi theo thứ tự (serialize). Hậu quả là làm cạn kiệt các kết nối trong pool.

## Timeline với `SKIP LOCKED`

Khi chúng ta thêm `SKIP LOCKED` vào truy vấn:

| Bước | Worker A | Worker B |
| ---: | --- | --- |
| 1 | khóa J1 | |
| 2 | | bỏ qua J1 đang bị khóa, tiến tới khóa J2 |
| 3 | cập nhật J1 thành PROCESSING | cập nhật J2 thành PROCESSING |
| 4 | commit việc lấy J1 | commit việc lấy J2 |
| 5 | xử lý J1 bên ngoài giao dịch | xử lý J2 bên ngoài giao dịch |

Hai worker sẽ lấy được hai dòng dữ liệu riêng biệt. Lưu ý rằng khóa dòng (row lock) chỉ tồn tại trong vòng đời của giao dịch lấy việc (claim transaction), hoàn toàn không kéo dài đến bước xử lý công việc bên ngoài.

> **Nói ngắn gọn:** `SKIP LOCKED` thay đổi tư duy từ "chờ đúng dòng đầu tiên" thành "bỏ qua và lấy một dòng đang rảnh rỗi". Bù lại, chúng ta phải chấp nhận việc hệ thống không còn đảm bảo thứ tự vào trước ra trước (strict FIFO) một cách tuyệt đối nữa.

## MVCC và lock behavior

Với mức cô lập `READ COMMITTED`, câu lệnh lấy việc sẽ đọc dữ liệu tại thời điểm câu lệnh bắt đầu (statement snapshot). Quá trình quét tìm ứng viên diễn ra như sau:

1. Lọc các dòng đã được commit thỏa mãn điều kiện `READY` và `available_at`.
2. Duyệt theo thứ tự `ORDER BY` để tìm dòng phù hợp.
3. Thử đặt khóa `FOR UPDATE` lên dòng đó.
4. Nếu dòng đang bị khóa bởi giao dịch khác, ngay lập tức bỏ qua (skip).
5. Dừng lại khi đã lấy đủ số lượng theo `LIMIT`.
6. Thực thi `UPDATE ... RETURNING` để đổi quyền sở hữu (ownership) ngay trong cùng một câu lệnh.

Worker khác sẽ không nhìn thấy trạng thái `PROCESSING` đang cập nhật dở dang (uncommitted) của worker này, nhưng khi chạm vào khóa dòng, nó sẽ lướt qua luôn. Khi giao dịch commit, dòng dữ liệu đó sẽ không còn thỏa mãn điều kiện `READY` nữa.

Cần nhớ rằng `SKIP LOCKED` chỉ bỏ qua các khóa cấp độ dòng (row-level lock). Lệnh `SELECT FOR UPDATE` vẫn yêu cầu khóa cấp độ bảng là `ROW SHARE`; do đó, nếu có một thao tác DDL (chẳng hạn `ACCESS EXCLUSIVE`) đang chạy, truy vấn vẫn sẽ bị chặn. Hơn nữa, thời gian chờ lấy kết nối (pool wait), I/O, thời gian thực thi truy vấn và commit vẫn tạo ra độ trễ (latency).

## Vì sao CTE claim atomic?

Việc sử dụng Common Table Expression (CTE) giúp gom mọi thứ vào một khối lệnh duy nhất:

```text
Khóa các dòng ứng viên
→ Cập nhật chính các dòng vừa khóa
→ Trả về token và trạng thái
→ Giao dịch commit
```

Cấu trúc này không chừa ra khe hở nào để ứng dụng kịp lấy ID về rồi một giao dịch khác lại chen vào lấy mất trước khi có lệnh UPDATE.
Nếu giao dịch bị rollback hoặc tiến trình sập trước khi commit, toàn bộ trạng thái, token, số lần thử đều được hoàn tác, và các khóa cũng được giải phóng sạch sẽ.

Mỗi dòng trả về cho ứng dụng chắc chắn đã được gắn thẻ định danh (token) mới. Ứng dụng chỉ bắt đầu xử lý bên ngoài khi quá trình commit thành công trót lọt.

## View không nhất quán là feature có chủ ý

Khi dùng `SKIP LOCKED`, một worker sẽ không thấy các dòng `READY` đang bị khóa trong kết quả trả về. Vì thế:

- Không nên dùng kết quả này để tính toán tổng số lượng trong hàng đợi (queue depth), để tính tiền (billing) hay báo cáo doanh nghiệp.
- Đừng vội kết luận "không còn công việc" nếu kết quả trả về là rỗng; có thể các công việc chỉ đang tạm thời bị khóa.
- Đừng áp đặt yêu cầu strict FIFO làm tiêu chuẩn bắt buộc (correctness invariant).
- Cơ chế này cực kỳ tối ưu khi mục tiêu chính là phân phối các công việc đang khả dụng cho nhiều consumer cùng một lúc.

Để xây dựng dashboard báo cáo, bạn nên dùng các truy vấn đếm (aggregate) thông thường hoặc sử dụng bản sao cơ sở dữ liệu (read replica) riêng biệt, không nên dựa vào kết quả của truy vấn `SKIP LOCKED`.

## Fairness và starvation

Câu lệnh `ORDER BY priority DESC, available_at, job_id` thiết lập một thứ tự ưu tiên ổn định, nhưng không mang lại sự công bằng tuyệt đối. Công việc J1 có thể liên tục bị bỏ qua nếu:

- Một giao dịch khác đang giữ khóa quá lâu.
- J1 liên tục bị lỗi và được trả lại hàng đợi ngay lập tức với độ ưu tiên cao.
- Các worker luôn lấp đầy lô công việc (batch) bằng những công việc mới có độ ưu tiên cao.
- Có sự bất đồng bộ giữa kế hoạch thực thi truy vấn (query plan) và chỉ mục (index).

Để kiểm soát rủi ro này (Containment), chúng ta cần:

- Giữ giao dịch lấy việc thật ngắn.
- Giới hạn số lượng công việc trong mỗi lần lấy (bounded batch).
- Giãn cách thời gian thử lại bằng backoff (cập nhật lại `available_at`).
- Giới hạn số lần thử (max attempts) và đưa vào trạng thái `DEAD` (dead-letter).
- Tăng dần độ ưu tiên theo thời gian chờ (aging) hoặc dành riêng worker cho các độ ưu tiên thấp.
- Giám sát độ tuổi của công việc chờ lâu nhất (oldest READY age), chứ không chỉ đếm số lượng hàng đợi.
- Dùng một tiến trình quét (sweeper) để phát hiện và xử lý các công việc bị ngâm quá hạn (expired lease).

Tóm lại, sự công bằng là một chính sách bạn phải tự thiết kế và giám sát, không phải là thứ có sẵn miễn phí khi dùng `SKIP LOCKED`.

## Lease và ownership epoch

Mỗi lần lấy việc thành công, chúng ta sinh ra một `claim_token` mới và cập nhật `lease_until`. Token này đại diện cho một chu kỳ sở hữu (ownership epoch):

```text
J1/token-A hết thời hạn (expires)
→ Tiến trình sweeper trả J1 về READY
→ Worker B lấy J1, nhận token-B
→ Worker A đột nhiên tỉnh lại và tiếp tục xử lý với token-A
→ Khi Worker A gửi lệnh hoàn thành (kèm token-A), cơ sở dữ liệu trả về affected-row = 0 (thất bại)
```

Thẻ token ngăn chặn các worker quá hạn (stale worker) ghi đè lên hàng đợi. Tuy nhiên, nó không thể ngăn hệ thống bên ngoài nhận lại lời gọi cũ; do đó, các dịch vụ bên ngoài vẫn phải tự xử lý tính lũy đẳng (idempotency key/fencing support).

Thời hạn thuê (lease duration) phải đủ dài để xử lý công việc bình thường, nhưng không được quá dài để tránh việc phục hồi sau sự cố (crash recovery) trở nên quá chậm. Đối với các công việc cần thời gian dài, bạn nên có cơ chế "bắn tim" (heartbeat) để gia hạn thuê, và phải kiểm tra điều kiện token hiện tại, đồng thời áp dụng chính sách thời gian chạy tối đa (maximum execution policy).

## Failure matrix

| Điểm gặp sự cố | Trạng thái cơ sở dữ liệu | Phục hồi |
| --- | --- | --- |
| Trước khi commit claim | Hoàn tác, công việc vẫn là READY | Worker khác lấy việc |
| Sau khi commit claim, trước khi xử lý | Nằm ở PROCESSING cho đến hết lease | Sweeper sẽ đưa lại vào hàng đợi |
| Đang xử lý bên ngoài | Kết quả xử lý có thể bị lỗi hoặc không rõ | Thử lại với cơ chế idempotency |
| Xử lý xong nhưng chưa complete | Công việc sẽ được chạy lại | Dịch vụ bên ngoài sẽ tự lọc trùng dựa trên job/effect key |
| Stale worker gọi lệnh complete | Token không khớp, affected-row = `0` | Worker bỏ qua kết quả, ghi log cảnh báo |
| Commit complete xong nhưng rớt mạng | Đã ghi nhận DONE an toàn | Hệ thống đọc lại trạng thái/token để đồng bộ |
| Tiến trình Sweeper bị sập | Các công việc hết hạn vẫn kẹt ở PROCESSING | Sweeper khác sẽ xử lý hoặc chờ lần chạy tiếp theo |

## At-least-once, không phải exactly-once

Giao dịch cơ sở dữ liệu không bao trùm các dịch vụ bên ngoài. Nếu hệ thống sập sau khi đã gọi dịch vụ nhưng chưa kịp ghi `DONE`, việc gọi lại (redelivery) là hoàn toàn hợp lệ. Các giải pháp xử lý bao gồm:

- Gửi kèm mã nhận diện lũy đẳng (idempotency key) tương ứng với ID của công việc.
- Ghi dữ liệu vào outbox/inbox cục bộ trước rồi mới giao tiếp với bên ngoài.
- Dịch vụ đích (sink) có hỗ trợ khóa có điều kiện (conditional/fenced operation).
- Đối soát (reconciliation) định kỳ để phát hiện độ lệch giữa hai bên.

Đừng bao giờ cố gắng gọi dịch vụ bên ngoài ngay bên trong giao dịch cơ sở dữ liệu với hy vọng đạt được "chính xác một lần" (exactly once). Cách làm này chỉ khiến khóa cơ sở dữ liệu kéo dài và vẫn không thể đảm bảo an toàn như giao dịch phân tán (distributed atomic commit).

## Complete, fail và retry

Cả hai thao tác báo hoàn thành hoặc báo lỗi đều phải kèm theo token:

```sql
where job_id = :jobId
  and status = 'PROCESSING'
  and claim_token = :claimToken
```

Xử lý các tình huống lỗi:

- Lỗi có thể thử lại (retryable): Chuyển trạng thái về READY, cộng thêm thời gian chờ `available_at = now() + backoff`, và xóa quyền sở hữu.
- Lỗi vĩnh viễn (terminal): Chuyển trạng thái sang DEAD, lưu lại thông tin lỗi đã che giấu thông tin nhạy cảm.
- Mất quyền sở hữu (lost ownership): Khi cập nhật trả về `0` dòng, không được phép thay đổi gì thêm.

Cơ chế chờ (backoff) sẽ giúp hàng đợi không bị tắc nghẽn bởi những công việc liên tục bị lỗi (poison job). Các bản ghi lỗi (error payload/log) tuyệt đối không được chứa thông tin bảo mật hay dữ liệu nhạy cảm.

## Timeout, deadlock và isolation

Truy vấn lấy việc thường dùng mức cô lập `READ COMMITTED`. Dù `SKIP LOCKED` giúp giảm bớt thời gian chờ khóa dòng, nó không thể loại trừ hoàn toàn các vấn đề deadlock hoặc chờ khóa bảng (table wait). Luôn thiết lập giới hạn thời gian thực thi `statement_timeout`; còn `lock_timeout` thì vẫn rất hữu ích cho các loại khóa khác.

Nếu thao tác lấy việc vô tình kích hoạt các bảng khác (thông qua trigger/foreign key), hãy quy định rõ thứ tự khóa (lock order) và chuẩn bị xử lý mã lỗi `40P01`. Mức cô lập `SERIALIZABLE` là không cần thiết cho mục đích lấy việc, và nó còn có thể làm tăng tỷ lệ hủy giao dịch (abort/retry).

## Crash và transaction release

Nếu kết nối bị đứt trước khi commit, PostgreSQL sẽ tự động rollback và giải phóng các khóa dòng. Nhưng sau khi đã commit, nếu tiến trình xử lý sập, cơ sở dữ liệu sẽ không tự rollback quyền sở hữu vì dữ liệu đã được ghi nhận an toàn; lúc này cơ chế phục hồi qua thời hạn thuê (lease recovery) là bắt buộc.

Luôn sử dụng thời gian của cơ sở dữ liệu (`clock_timestamp()` hoặc `CURRENT_TIMESTAMP` tùy theo nhu cầu) để tính toán, giúp tránh được độ trễ thời gian (clock skew) giữa các máy chủ (pods). Bạn cần cân nhắc kỹ khi nào dùng thời gian của giao dịch (transaction timestamp) và khi nào dùng thời gian thực (wall-clock) trong cùng một câu lệnh.

## Multi-instance

Các khóa dòng của PostgreSQL tự động phối hợp mọi bản sao ứng dụng (application instance). Việc dùng mutex ở cấp độ JVM là không cần thiết, và đôi khi còn làm giảm hiệu năng vì bắt một máy chủ phải chạy tuần tự không đáng có.

Khi mở rộng hệ thống (scale-out), số lượng truy vấn lấy việc sẽ tăng lên. Bạn phải giới hạn số lượng worker, kích thước lô công việc (batch size), khoảng thời gian giữa các lần poll (polling interval/jitter) và tổng số kết nối (connection budget). Nếu lấy về lô rỗng (empty poll), cần giãn cách thời gian chờ (backoff), tuyệt đối không vòng lặp liên tục (busy-loop).

## Nguyên nhân gốc theo layer

| Tầng ứng dụng (Layer) | Vấn đề / Cơ chế |
| --- | --- |
| Application | Lỗi phổ biến là Đọc/Xử/Ghi tuần tự, hoặc giữ giao dịch quá lâu khi gọi dịch vụ bên ngoài. |
| Spring | Proxy chịu trách nhiệm đảm bảo giao dịch lấy việc đã commit TRƯỚC KHI xử lý bên ngoài. |
| Hibernate/JDBC | Đảm bảo map đúng dữ liệu trả về từ Native CTE, affected-row và mapping Token. |
| PostgreSQL | Chịu trách nhiệm quản lý MVCC, row locks, cơ chế skip semantics và lệnh UPDATE nguyên tử. |
| External sink | Quyết định tính an toàn thông qua tính năng kiểm tra tính lũy đẳng (idempotency). |

## Khả năng quan sát (`observability`)

Cần theo dõi các chỉ số sau:

- Độ sâu của hàng đợi (READY depth) và tuổi của công việc chờ lâu nhất (oldest READY age) theo từng độ ưu tiên.
- Kích thước lô công việc, độ trễ và số lần truy vấn trả về rỗng (empty polls).
- Thời gian xử lý công việc (PROCESSING age), số công việc hết hạn (expired leases), số lần lấy lại công việc (reclaim) và số công việc thất bại vĩnh viễn (DEAD count).
- Số lần thao tác hoàn thành bị từ chối do quá hạn (stale completion với affected-row = `0`).
- Phân bố số lần thử lại (attempt distribution) và số lần tái thực thi (replay) ở hệ thống bên ngoài.
- Số lượng kết nối đang hoạt động/chờ trong pool, số lượng câu lệnh bị hủy do timeout và mã lỗi deadlock `40P01`.
- Tra cứu `pg_stat_activity`, `pg_locks`, `pg_blocking_pids` khi truy vấn lấy việc bị treo bất thường.

Tuyệt đối không sử dụng ID của worker hoặc job ID làm nhãn giám sát (metric label) vì sẽ gây bùng nổ dữ liệu (high-cardinality); thay vào đó, hãy lưu chúng trong log hoặc trace có cấu trúc kèm theo cơ chế lấy mẫu (sampling).
