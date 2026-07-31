# Phân tích hiện tượng chết đói kết nối (Pool-starvation analysis)

## Trạng thái ban đầu

Giả sử một máy chủ (instance) có sức chứa hồ chứa kết nối (pool capacity) là `3` trong một kịch bản minh họa:

```text
số kết nối nhàn rỗi (idle connections)   = 3
số kết nối đang bận (active connections) = 0
số người đứng xếp hàng mượn (pending borrowers)  = 0

P-42 status = RISK_PENDING, version = 12
P-99 status = RISK_PENDING, version = 4
```

Hệ thống đánh giá rủi ro từ xa (Remote risk dependency) đang chạy rất chậm nhưng chưa đến nỗi chết hẳn. Đây là một sự chờ đợi chậm chạp có thời hạn hữu hạn (finite slow wait), chứ không phải là tình trạng rò rỉ kết nối (connection leak); các kết nối rồi sẽ được trả về nếu những giao dịch này cuối cùng cũng chịu kết thúc.

## Tiến trình sụp đổ dây chuyền (Timeline cascading failure)

| Bước | Yêu cầu A (Request A) — P-42 | Yêu cầu trùng lặp B (Duplicate B) — P-42 | Yêu cầu C (Request C) — P-99 | Yêu cầu không liên quan U (Unrelated U) |
| ---: | --- | --- | --- | --- |
| T0 | Mượn được kết nối số 1 (conn-1) | | | |
| T1 | Khóa đơn hàng P-42 | | | |
| T2 | Đứng chờ hệ thống từ xa | Mượn được kết nối số 2 (conn-2) | | |
| T3 | | Đứng chờ giành khóa P-42 | Mượn được kết nối số 3 (conn-3) | |
| T4 | | | Khóa đơn hàng P-99, chờ từ xa | |
| T5 | Pool báo đang bận (active) = 3 | | | Hết kết nối, đành đứng chờ mượn |
| T6 | Vẫn mòn mỏi chờ | Vẫn mòn mỏi chờ | Vẫn mòn mỏi chờ | Hết thời gian chờ (Acquisition timeout) |
| T7 | Hệ thống từ xa timeout / trả kết quả | | | Người gọi yêu cầu U đã nhận được báo lỗi |
| T8 | Rollback/commit, cuối cùng cũng nhả conn-1 | B mới bắt đầu chiếm được khóa | | Lưu lượng truy cập thử lại (Retry traffic) có thể sẽ ập đến |

Ở đây không hề có vòng lặp chờ đợi tuần hoàn (circular wait) nào cả:

```text
B đứng chờ A nhả khóa dòng
A đứng chờ dịch vụ từ xa trả lời
C cũng đứng chờ dịch vụ từ xa trả lời
U tuyệt vọng chờ lấy bất kỳ kết nối nào từ hồ chứa (pool connection)
```

Bộ phát hiện bế tắc của PostgreSQL (deadlock detector) không thể phá vỡ chuỗi này vì dịch vụ từ xa và hồ chứa kết nối hoàn toàn không nằm trong vòng lặp chờ đợi (wait-for cycle) của cơ sở dữ liệu.

> **Nói ngắn gọn:** Mọi sự chờ đợi đều có vẻ "hợp lệ" nếu xét riêng lẻ, nhưng khi chúng cùng chiếm giữ các tài nguyên hữu hạn (finite resources) theo một trật tự tồi tệ, nó tạo ra thảm họa chết đói dây chuyền (cascading starvation).

## Kết nối được chiếm giữ và nắm thóp khi nào?

Spring có thể tạo ra một giao dịch logic (logical transaction) từ trước khi kết nối JDBC thực sự được mượn một cách lười biếng (lazily). Trong trường hợp này, lệnh `findByIdForUpdate()` chắc chắn đòi hỏi một kết nối. Ngay sau câu lệnh SELECT:

- EntityManager / Giao dịch sẽ trói chặt (bind) kết nối đó vào luồng yêu cầu (request thread);
- Hibernate sẽ không bao giờ thả kết nối về lại hồ chứa giữa chừng một giao dịch;
- Khóa dòng (row lock) là tài sản trực thuộc giao dịch PostgreSQL;
- Việc gọi qua hệ thống từ xa (remote call) không hề làm cho giao dịch kết thúc;
- Kết nối chỉ lết về lại hồ chứa sau khi các lệnh commit/rollback/cleanup hoàn tất.

Đừng vội đánh đồng một EntityManager đang mở đồng nghĩa với một kết nối JDBC đang bận trong mọi kiểu cấu hình. Bằng chứng thép phải được trích xuất từ các chỉ số đo lường của hồ chứa (pool metrics) và các phiên làm việc (sessions) của PostgreSQL, chứ không chỉ nhìn suông vào mấy cái dòng annotation.

## Hoạt động của PostgreSQL phản ánh điều gì?

Trong tiến trình thời gian ở trên:

- A có thể xuất hiện trạng thái `idle in transaction` (rảnh rỗi trong giao dịch) sau lệnh SELECT, do Java đang mải mê đứng chờ tín hiệu HTTP;
- B sẽ báo trạng thái `active`, với `wait_event_type = 'Lock'` khi bị lệnh `FOR UPDATE` chặn lại;
- C có thể báo `idle in transaction` trên một đơn hàng khác;
- U chưa có bất kỳ tiến trình xử lý dưới nền hay câu truy vấn nào ở PostgreSQL vì nó vẫn còn đang mòn mỏi chờ lấy kết nối trong hồ Hikari pool.

Câu truy vấn chẩn đoán chuyên sâu bằng một kết nối admin độc lập (diagnostic query):

```sql
select pid,
       application_name,
       state,
       wait_event_type,
       wait_event,
       xact_start,
       query_start,
       pg_blocking_pids(pid) as blockers,
       query
from pg_stat_activity
where datname = current_database()
order by xact_start nulls last;
```

Cần lưu ý: Trạng thái `idle in transaction` cũng có thể là hệ quả của một lỗi ứng dụng hay nhánh mã lệnh khác; bạn phải xâu chuỗi dữ liệu mã tương quan (correlation ID), luồng theo dấu (stack trace/telemetry) và mức độ sử dụng hồ chứa để tìm ra bằng chứng chính xác.

## Tuổi thọ của khóa (Lock lifetime) đi đôi với tuổi thọ của giao dịch (Transaction lifetime)

A chiếm giữ khóa dòng ở mốc T1. Khóa này sẽ không được thả ra khi:

- Phương thức trong repository hoàn tất (return);
- Thực thể JPA được ánh xạ (map) sang đối tượng trao đổi dữ liệu (DTO);
- Lệnh gọi hệ thống từ xa vừa mới khởi hành;
- Luồng yêu cầu (request thread) đang không dùng đến CPU;
- Kể cả khi một phương thức khác có annotation `@Transactional` đã hoàn tất (return), nếu nó chỉ đang tham gia (join) vào cùng một giao dịch vật lý ban đầu.

Khóa chỉ được giải phóng ở thời khắc commit/rollback. Chính vì vậy, độ trễ p99 của hệ thống từ xa sẽ trực tiếp kéo dài thời gian giữ khóa p99 và nối dài thêm hàng đợi của các lệnh trùng lặp.

## Chết đói do các dòng khác nhau (Different-row starvation)

Sự tranh chấp khóa dòng (Row lock contention) không phải là điều kiện bắt buộc duy nhất gây lỗi. Nếu mỗi yêu cầu chiếm khóa một đơn hàng khác nhau rồi cùng gọi chung vào một hệ thống phụ thuộc chậm chạp, thì toàn bộ kết nối vẫn sẽ ở trạng thái bận (active).

Mối tương quan về mặt định tính (Qualitative capacity relation):

```text
số kết nối đang được sử dụng (in-use connections)
  ≈ tỷ lệ giao dịch gửi tới (transaction arrival rate) × thời gian trung bình của giao dịch (average transaction duration)
```

Thời gian của giao dịch (Transaction duration) cứ phình to ra theo độ trễ chờ đợi hệ thống từ xa (remote wait), khiến cho mức độ đồng thời cần thiết (required concurrency) tăng vọt, dẫu cho tỷ lệ truy vấn và tải CPU của PostgreSQL chẳng mảy may thay đổi. Nên nhớ, Hồ chứa kết nối (Pool) chỉ là một hàng đợi (queue) hoặc một ranh giới giới hạn dung lượng (capacity boundary), nó không có phép màu để tự thân tạo ra thêm thông lượng (throughput) cho cơ sở dữ liệu.

## Hiệu ứng khuếch đại trên cùng một dòng (Same-row amplification)

Trên một giao dịch thanh toán nóng (hot payment):

1. Yêu cầu A giữ khóa và gọi hệ thống từ xa.
2. Các yêu cầu B..N mượn sạch kết nối rồi bị chặn đứng bởi lệnh `FOR UPDATE`.
3. Trong số đó, chỉ duy nhất một yêu cầu là đang thực hiện tác vụ gọi từ xa có ích (useful remote work).
4. Những kẻ chầu rìa (Waiters) đã vắt kiệt hồ chứa kết nối.
5. Khi A chạy xong, các yêu cầu còn lại lần lượt giành được khóa nhưng bọn chúng có thể vẫn ngây ngô gọi lại hệ thống từ xa mặc dù trạng thái đã thay đổi, nếu như mã lệnh không chịu thẩm định lại (revalidate) từ sớm.

Các luồng đang chờ khóa (Lock waiter) bắt buộc phải tải lại/kiểm tra (reload/check status) ngay tắp lự sau khi giành được khóa và phải trả về kết quả đã lưu trữ hoặc không làm gì cả (stored/no-op result) nếu tiến trình đó đã ở trạng thái kết thúc (terminal). Nếu không làm thế, những công việc nặng nhọc bị nhân bản (duplicated expensive work) sẽ càng kéo lê thê thêm hàng đợi.

## Cạn thời gian chờ mượn kết nối chỉ là triệu chứng (Pool acquisition timeout là symptom)

Yêu cầu U có thể sẽ nhận về chuỗi lỗi:

```text
SQLTransientConnectionException
  -> CannotGetJdbcConnectionException
  -> HTTP 5xx/timeout
```

Chẳng có câu SQL nào của U để mà bạn có thể tối ưu (tune). Vốn dĩ câu truy vấn cực kỳ ngắn gọn nhưng lại đen đủi không được cấp phép kết nối.

Nếu bạn thử tăng thời gian chờ mượn (acquisition timeout):

- Nó chỉ làm tăng thời gian chờ mòn mỏi của request/thread;
- Rất có thể sẽ vô tình vượt qua luôn thời hạn đã cam kết với upstream (upstream deadline);
- Chẳng giúp giải phóng được bất kỳ kết nối đang bận nào;
- Mà chỉ tổ làm cho hàng đợi ngày một dài ngoẵng.

Thời gian chờ (Timeout) ngắn hơn sẽ giúp cơ chế fail-fast/cô lập sự cố hoạt động tốt hơn, nhưng cốt lõi vẫn không thể sửa được cái ranh giới giao dịch lủng củng gốc rễ.

## Phình to Hồ chứa có khi mang họa vào thân (Tăng pool size có thể làm xấu hơn)

Một hồ chứa (Pool) lớn hơn sẽ châm ngòi cho hàng loạt các giao dịch kéo dài (long transactions) sinh sôi:

- Đẻ thêm hàng loạt các backends/tiêu tốn bộ nhớ dưới PostgreSQL;
- Bồi thêm vô số kẻ chầu rìa chờ khóa dòng (row-lock waiters);
- Khơi mào thêm nhiều yêu cầu từ xa;
- Khiến cho các bản ảnh giao dịch (transaction snapshots) phải tồn tại lâu hơn;
- Cú sốc phục hồi (recovery burst) sẽ kinh hoàng hơn khi hệ thống phụ thuộc sống lại.

Nếu triển khai trên cấu trúc đa máy chủ (multi-instance deployment):

```text
tổng số máy chủ (instances) × sức chứa tối đa của hồ (maximumPoolSize)
```

Hoàn toàn có nguy cơ đánh sập ngân sách kết nối tối đa của cơ sở dữ liệu (database connection budget). Việc tự động co giãn quy mô (Autoscaling) dựa vào độ trễ (latency) đôi khi lại dội thêm máy chủ/pool mới vào đúng cái khoảnh khắc cơ sở dữ liệu đang ngắc ngoải, tạo ra vòng lặp phản hồi dương (positive feedback) kéo nhau cùng chết.

Quy mô hồ chứa (Pool size) bắt buộc phải dựa trên sức mạnh của cơ sở dữ liệu và tải trọng đã đo lường (measured workload); ranh giới/áp lực ngược (boundary/backpressure) mới là chìa khóa xử lý tận gốc bài toán thời gian (duration) và mức độ đồng thời (concurrency).

## Hệ lụy từ luồng thực thi (Executor coupling)

Luồng yêu cầu đang làm chủ giao dịch (Transaction-owning request thread) có thể rơi vào tình cảnh:

- Trình lên (submit) một tác vụ đánh giá rủi ro (risk task);
- Tay vẫn bo bo giữ kết nối cơ sở dữ liệu;
- Đứng cứng ngắc (block) chờ kết quả tương lai (future);
- Bỗng dưng hàng đợi của luồng xử lý rủi ro (risk executor queue) báo đầy;
- Càng có thêm các luồng yêu cầu tiếp tục nhảy vào mượn kết nối rồi tiếp tục... đứng chờ.

Nếu tác vụ rủi ro đó lại đi đòi hỏi đúng cái luồng yêu cầu ấy (request executor) hoặc một kết nối cơ sở dữ liệu nào đó, thì biểu đồ phụ thuộc tài nguyên (resource dependency graph) sẽ trở nên cực kỳ nguy hiểm. Chống chỉ định việc đứng chặn (block) bên trong giao dịch cho những tác vụ vốn có hàng đợi hoặc khả năng xử lý độc lập (queue/capacity độc lập).

Vách ngăn (Bulkhead) sẽ giúp khống chế số lượng truy cập đồng thời vào hệ thống từ xa từ trước khi giao dịch cơ sở dữ liệu kịp bắt đầu. Khi vách ngăn đầy, cứ dứt khoát từ chối hoặc xếp hàng có giới hạn (reject/queue bounded) mà không phải ngậm ngùi giữ một kết nối JDBC làm con tin.

## Ngân sách thời gian (Timeout budget)

Một thời hạn cam kết tổng thể (overall deadline) `D` bắt buộc phải bao gồm:

```text
thời gian chờ xếp hàng/vách ngăn (admission/bulkhead wait)
+ thời gian kết nối/đọc từ xa (remote connect/read)
+ thời gian chờ lấy kết nối DB (DB pool acquisition)
+ thời gian khóa/thực thi lệnh (lock/statement work)
+ thời gian chốt/xóa sổ/dọn dẹp (commit/rollback cleanup)
+ thời gian biên độ dự phòng phản hồi (response margin)
<= D (tổng thời gian D)
```

Không được lấy cộng dồn các giới hạn thời gian rời rạc độc lập (đôi khi còn lớn hơn cả deadline từ caller). Mỗi công đoạn (phase) chỉ được phép lấy quỹ thời gian còn lại (remaining budget), và một công đoạn mới không được phép bắt đầu nếu không cầm chắc quỹ thời gian đủ để hoàn tất an toàn.

Việc hết hạn giao dịch của Spring (Spring transaction timeout) không phải là cái công tắc điện đa năng (general-purpose interrupt) có thể tự động ngắt bất kỳ cái HTTP hay future wait ngẫu nhiên nào. Phía gọi máy chủ từ xa (Remote client) và luồng đứng chờ executor bắt buộc phải có thời hạn (deadline) và quyền chủ động hủy bỏ (cancellation) riêng rẽ của chúng.

## Hành xử khi có lệnh Rollback và Timeout

Nếu sự cố hết hạn (timeout) từ xa ném ra một ngoại lệ không kiểm tra (unchecked exception) xuyên rách ranh giới giao dịch:

- Spring tiến hành rollback;
- PostgreSQL nhả các khóa dòng (row lock);
- Hikari mừng rỡ nhận lại kết nối;
- Người đứng đợi tiếp theo (waiter) tiến lên thực thi lệnh.

Nhưng khổ nỗi, khâu dọn dẹp chỉ được diễn ra sau khi quá trình chờ đợi từ xa (remote wait) chính thức kết thúc hoặc bị gián đoạn (interrupt). Một máy khách HTTP (HTTP client) nếu không được trang bị thời hạn kết nối/đọc (connect/read timeout) có giới hạn rõ ràng hoàn toàn có thể treo giao dịch đến vô tận.

Nếu mã lệnh của bạn chụp (catch) lỗi timeout rồi vẫn ngoan cố tiếp tục hoặc commit một trạng thái lưng chừng (intermediate state), thì cam kết khi gặp lỗi (failure contract) đó phải được nêu tường minh. Kịch bản khuyến nghị là tuyệt đối không được khởi động giao dịch từ trước khi chuỗi chờ đợi từ xa kết thúc.

## Quyền chủ động hủy bỏ của người gọi (Client cancellation)

Đừng nghĩ máy khách phía thượng nguồn (Upstream client timeout) chủ động ngắt là máy chủ (server) cũng sẽ tự động chịu dừng tay theo. Nếu máy chủ không tự giác truyền tiếp quyền hủy bỏ hoặc giới hạn thời gian (propagate cancellation/deadline):

- Cuộc gọi tới hệ thống từ xa (remote call) vẫn lầm lũi chạy;
- Giao dịch/kết nối vẫn bị nắm giữ làm con tin;
- Kết quả trả về sau này bị ném sọt rác (bỏ qua), nhưng cái giá phải trả (resource cost) thì vẫn trừ đủ;
- Cơn lốc các yêu cầu thử lại (retry request) có khi còn ập đến song song cùng lúc.

Quyền hủy bỏ phải châm ngòi cho quá trình dọn dẹp (cleanup) nhưng cấm tuyệt đối việc bỏ mặc giao dịch lửng lơ (transaction lửng). Hãy luôn luôn chốt lại (end) bằng một lệnh commit hoặc rollback nằm trong lộ trình giới hạn an toàn.

## Ranh giới được chỉnh đốn cần phải thẩm định lại (Fixed boundary cần revalidation)

Việc bứng cuộc gọi từ xa (remote call) ra khỏi phạm vi giao dịch sẽ để hở sườn một khoảng hở đồng thời (concurrency window):

```text
đọc bản ảnh (snapshot) phiên bản 12
gọi hệ thống đánh giá rủi ro risk(với thông số snapshot v12)
một tác nhân khác (another actor) nẫng tay trên, đổi trạng thái payment thành CANCELLED hoặc lên phiên bản v13
đến lúc vào lại giao dịch (commit phase) thì lại nạp phiên bản v13
```

Tuyệt đối không được nhắm mắt làm ngơ (apply decision v12 mù quáng). Giao dịch chốt (Short commit transaction) bắt buộc phải xét nét (check):

- Đơn hàng (payment) phải vẫn còn là `RISK_PENDING`;
- Phiên bản (version)/số tiền/khách hàng hiện tại có ăn khớp (match) với đối tượng đã được đem đi đánh giá hay không;
- Quyết định có còn hiệu lực không (decision chưa expired);
- Khóa lệnh hoặc trạng thái chống trùng lặp (command/idempotency state);
- Chưa có một kết quả chốt cuối (terminal result) nào ngự trị ở đó.

Nếu phát hiện vênh nhau (mismatch), cứ trả về lỗi (stale) hoặc không thao tác gì (no-op), hoặc cho chạy lại quy trình điều phối (orchestration lại) hoàn toàn nằm ngoài giao dịch, tuân thủ đúng chính sách có giới hạn (bounded policy).

## Luồng công việc bất đồng bộ (Async workflow)

Khi độ trễ từ xa quá dai dẳng hoặc bắt buộc phải có cơ chế thử lại bền vững (durable retry):

1. Tx-1: Chuyển sang `RISK_PENDING` và chèn một sự kiện vào hộp thư đi (outbox event).
2. Nhanh chóng Commit/giải phóng kết nối.
3. Luồng công nhân (Worker) gọi sang hệ thống từ xa mà không ngâm trong giao dịch cơ sở dữ liệu.
4. Tx-2: Chỉ áp dụng kết quả (conditionally apply decision) nếu trạng thái/phiên bản vẫn còn khớp.
5. Lưu trữ kết quả/khóa lũy đẳng đề phòng trường hợp phải gọi lại lệnh phân phối (Store outcome/idempotency for redelivery).

Dòng công việc (Workflow) không bao giờ giữ transaction vắt ngang qua ranh giới mạng (network boundary) hay thông điệp (message). Nó đánh đổi sự chậm trễ đồng bộ (synchronous latency) và đòi hỏi phải có một cỗ máy trạng thái (state-machine) cùng sự phức tạp của cơ chế phục hồi (recovery complexity).

## Độ sẵn sàng (Readiness) và các hoạt động dây chuyền (cascading operations)

Chết đói hồ chứa kết nối (Pool starvation) có sức tàn phá dẫn đến:

- Các đầu dò sức khỏe (readiness) dựa vào DB đồng loạt báo lỗi (fail);
- Trình điều phối vòng ngoài (orchestrator) đành phải khởi động lại (restart) cả những pod đang khỏe nhưng lỡ bị quá tải;
- Những pod mới mọc lên lại lăm le mở thêm hàng loạt hồ chứa (pools) mới;
- Các tác vụ di chuyển dữ liệu (migrations)/quản trị (admin)/đối soát (reconciliation) phải khóc thét vì thiếu connection;
- Những nỗ lực thử lại (retries) càng đắp thêm núi tải (load).

Tuyệt đối không dùng kế lấp liếm bằng một hồ kết nối (special admin pool) ưu tiên để che đậy sự thối rữa của service failure. Nếu có chăng phải duy trì một sức chứa vận hành dự phòng (reserved operational capacity), nó phải thật tinh gọn, hoàn toàn cô lập (isolated) và nằm lọt thỏm trong ngân sách kết nối cơ sở dữ liệu (database connection budget); còn phần lưu lượng nghiệp vụ (business traffic) thì bắt buộc phải gánh chịu áp lực ngược (backpressure).

## Khoanh vùng: Cạn kiệt (starvation), chứ không phải Rò rỉ (leak) hay Bế tắc (deadlock)

- **Rò rỉ (Leak):** Tình trạng kết nối không được trả về hồ do mã nguồn bị lỗi ở khâu dọn dẹp (cleanup bug).
- **Cạn kiệt/Chết đói (Starvation):** Các kết nối vẫn đang được sử dụng một cách danh chính ngôn thuận (hợp lệ) nhưng với thời gian ngâm giấm quá dài.
- **Bế tắc (Deadlock):** Vòng lặp chờ đợi xoay vòng khép kín (wait-for cycle) bắt buộc phải có nạn nhân bị chém (victim) hoặc phải sắp xếp lại thứ tự (ordering).
- **Tràn hồ (Pool exhaustion):** Hoàn toàn không bói đâu ra kết nối nhàn rỗi (idle connection) cho người cần mượn (borrower).

Nhìn lướt qua các chỉ số đo lường (Metrics) có thể thấy chúng từa tựa nhau ở bề nổi. Nhưng hãy đào sâu vào tuổi thọ của giao dịch (transaction age), sơ đồ khóa (lock graph), trạng thái luồng (thread state) và độ trễ của hệ thống từ xa (remote latency) để truy tìm ngọn nguồn (root cause) đích thực.

## Những hàm ý khi chạy đa máy chủ (Multi-instance implications)

Công cụ khóa luồng (JVM semaphore) gắn trên một máy chủ (instance) chỉ đủ sức cản bước tải trọng cục bộ (local load) của máy chủ đó. Lớp bảo vệ bao quát đa máy chủ (Cross-instance protection) cần hội đủ:

- Vách ngăn từng máy (per-instance bulkhead) cộng với hoạch định sức tải cho các hệ thống phụ thuộc toàn cục (global dependency capacity planning);
- Bộ cân bằng tải (load balancer) hoặc kiểm soát xét duyệt đầu vào (admission control);
- Ngân sách kết nối tổng thể `max_connections` dưới database;
- Hàng đợi có định danh (keyed queue) hoặc cỗ máy trạng thái (state machine) khi cần thiết phải đồng bộ hóa một thực thể (serialize aggregate);
- Bảng đo lường chỉ số (metrics) được tổng hợp trên toàn bộ các máy chủ (aggregated across instances).

Cái khóa của JVM tuyệt nhiên không có năng lực phá vỡ được cái khóa ở tận PostgreSQL, và nó càng bất lực trong việc cản các máy chủ khác (node khác) xông vào mượn kết nối (borrow connection).

## Khả năng quan sát (Observability)

Phải ráp nối (Correlate) thông tin:

- Bảng trạng thái Hikari (active/idle/pending/max) cùng với hiện tượng quá hạn mượn (acquisition timeout);
- Biểu đồ phân bổ việc sử dụng kết nối cũng như thời lượng giao dịch (connection usage/transaction duration histogram);
- Độ trễ của hệ thống từ xa (remote latency), số lỗi (timeout), số vách ngăn đang hoạt động/chờ/bị từ chối (bulkhead active/queued/rejected);
- Hoạt động của luồng thực thi (executor active/queue/rejection);
- Tuổi thọ của giao dịch PostgreSQL (transaction age), số kẻ ở không rảnh rỗi `idle in transaction`, số đang chờ khóa (lock waits) hoặc đang làm kỳ đà cản mũi (blockers);
- Quỹ thời gian còn sót lại của toàn bộ yêu cầu (request deadline remaining);
- Đảm bảo ID định danh của đơn hàng/mã lệnh (order/command ID) phải được ẩn/băm (redact/hash) một cách khéo léo phù hợp;
- Lấy tổng số máy chủ (instance count) nhân với sức chứa đã cấu hình cho từng hồ (configured pool capacity).

Luôn giương cao ngọn cờ cảnh báo (Alert) nhắm vào những tín hiệu báo trước (leading indicators) như tình trạng phình to thời lượng sử dụng (rising usage duration) hoặc đống lệnh chờ lấy kết nối (pending borrowers), chứ đừng dại dột đứng chờ cho tới khi tỷ lệ lỗi pool (pool timeout rate) dâng ngập đầu rồi mới cuống cuồng lo ứng cứu (phản ứng).
