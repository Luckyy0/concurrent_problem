# Phân tích ranh giới luồng, thứ tự commit và các rủi ro lỗi

## Trạng thái ban đầu

Giả sử đơn hàng (Order) ID 42 chưa tồn tại trong trạng thái dữ liệu đã commit. Luồng yêu cầu (Request thread) mở một giao dịch (Tx-Order), thực hiện `INSERT` đơn hàng 42 và `flush`; trong khi đó, tiến trình xử lý nền (worker) chưa bắt đầu.

## Kịch bản 1: Async reader không nhìn thấy đơn hàng

| Bước | Request thread / Tx-Order | Worker thread / Tx-Async | Trạng thái PostgreSQL đã commit |
| ---: | --- | --- | --- |
| 1 | `INSERT` đơn hàng 42, `flush` | | chưa có 42 |
| 2 | nộp (submit) tác vụ async | | chưa có 42 |
| 3 | dừng lại chờ trước khi commit | bắt đầu một giao dịch độc lập | chưa có 42 |
| 4 | | gọi `findById(42)` → kết quả rỗng | chưa có 42 |
| 5 | thực hiện commit sau đó | worker đã báo lỗi (fail) trước đó | có 42 nhưng công việc của worker đã mất |

Với mức cô lập `READ COMMITTED`, PostgreSQL không cho phép đọc dữ liệu `INSERT` chưa được commit của một giao dịch khác. Thao tác `flush` chỉ đơn thuần là gửi lệnh SQL xuống cơ sở dữ liệu; nó không làm cho dữ liệu đó hiển thị (external visibility) đối với các giao dịch khác.

## Kịch bản 2: Tác động bên ngoài (async side effect) sống sót sau khi giao dịch gốc bị rollback

| Bước | Request thread | Worker thread | Kết quả thực tế |
| ---: | --- | --- | --- |
| 1 | insert đơn hàng trong Tx-Order | | đơn hàng chưa được commit |
| 2 | truyền DTO hoặc ID cho worker | ghi nhận kiểm toán (audit/attempt) trong Tx-Async | đây là giao dịch độc lập |
| 3 | | commit bản ghi kiểm toán | tác động bên ngoài đã được ghi nhận an toàn (durable) |
| 4 | gặp lỗi ném exception, rollback đơn hàng | | bản ghi kiểm toán bị mồ côi (orphan attempt) |

Việc sử dụng khóa ngoại (foreign key) có thể khiến worker bị chặn lại (block) hoặc báo lỗi thay vì commit, nhưng nó không biến thiết kế này thành một khối nguyên tử (atomic). Hành vi này phụ thuộc vào cấu trúc bảng (schema) và thời điểm khóa (lock timing), và vẫn không hề cung cấp một khế ước thứ tự commit (commit-order contract) rõ ràng.

> **Nói ngắn gọn:** Một tác vụ async có thể chạy quá sớm khi caller chưa kịp commit, hoặc chạy "quá thành công" và để lại tàn dư khi caller cuối cùng lại bị rollback.

## Kỳ vọng và Thực tế

| Khía cạnh | Điều mong đợi | Lỗi thực tế (Broken flow) |
| --- | --- | --- |
| Thứ tự (Ordering) | Tác vụ async chỉ chạy sau khi đơn hàng đã commit | Tác vụ bị đẩy đi trước khi biết kết quả cuối cùng (terminal outcome) |
| Khả năng hiển thị (Visibility) | Worker luôn đọc được đơn hàng đã commit | Worker có thể truy vấn vào lúc dữ liệu chưa tồn tại |
| Khôi phục (Rollback) | Không có tác động bên ngoài nếu đơn hàng rollback | Giao dịch của worker có thể vô tình commit |
| Bối cảnh (Context) | Mỗi luồng có một ranh giới rõ ràng | Lập trình viên tưởng nhầm worker sẽ tham gia giao dịch của caller |
| Lỗi (Failure) | Kết quả được ghi nhận và có thể thử lại | Lỗi (Exception) có thể bị chôn vùi trong đối tượng tương lai (future) hoặc chỉ ghi log |
| Độ bền vững (Durability) | Phải tuân theo yêu cầu rõ ràng | Hàng đợi cục bộ có thể làm mất dữ liệu khi hệ thống sập (crash gap) |

## Nguyên nhân gốc theo từng tầng (Layer)

### Bối cảnh giao dịch trong Spring (Spring transaction context)

Trình quản lý giao dịch (Transaction manager) gắn các tài nguyên như kết nối (connection) hay `EntityManager` với luồng yêu cầu (request thread). Các luồng thực thi nền (executor thread) không hề kế thừa sự gắn kết này. Khi bạn đặt `@Transactional` trên một hàm bất đồng bộ, nó sẽ tạo ra hoặc tham gia vào một giao dịch mới của chính luồng worker đó, chứ không phải giao dịch của caller.

### Proxy bất đồng bộ (Async proxy)

Annotation `@Async` hoạt động dựa trên cơ chế proxy của Spring; việc gọi một hàm nội bộ (self-invocation) có thể làm cho hàm đó chạy đồng bộ (như đã phân tích ở SPR-001). Khi gọi đúng qua proxy, trình chặn (async interceptor) sẽ đẩy công việc đi và caller tiếp tục chạy; hoàn toàn không có một ràng buộc ngầm định nào về việc chờ hoàn thành hay đồng bộ thứ tự commit.

### Hibernate/JPA

Các thực thể đang được quản lý (managed entity), proxy tải trễ (lazy proxy) và bối cảnh bền vững (persistence context) không hề an toàn để chia sẻ qua các luồng khác nhau. Bạn chỉ nên truyền dữ liệu sự kiện bất biến (immutable event data) hoặc ID; worker sẽ tự nạp (load) lại dữ liệu gốc trong giao dịch riêng của nó sau khi commit thành công.

### PostgreSQL

Cơ chế MVCC bảo vệ tính cô lập (isolation) giữa các giao dịch, nó không thể tự suy luận mối quan hệ nhân quả (causal relationship) từ một mã đơn hàng (ID) được truyền qua executor. Cơ sở dữ liệu chỉ biết về việc commit hay rollback của từng giao dịch độc lập.

## Cơ chế After-commit

Annotation `@TransactionalEventListener(phase = AFTER_COMMIT)` chỉ gọi trình lắng nghe (listener) sau khi giao dịch commit thành công; nếu giao dịch bị rollback, listener sẽ không được gọi. Mặc định, listener sẽ không chạy nếu sự kiện được phát ra ngoài phạm vi giao dịch, trừ khi bạn bật cờ `fallbackExecution` — tuy nhiên, không nên bật tính năng này một cách tùy tiện vì nó sẽ làm suy yếu các cam kết ràng buộc (contract).

Để việc ghi cơ sở dữ liệu bên trong listener diễn ra rõ ràng, listener sau khi được gọi (after commit) nên tiếp tục gọi đến một bean async khác; hàm worker này sẽ mở một giao dịch mới. Tuyệt đối không tiếp tục ghi dữ liệu bằng các tài nguyên thuộc về giao dịch vừa mới commit trên cùng một luồng.

## Thất bại, hủy bỏ và quá tải luồng (Failure, cancellation và executor overload)

- Một khi giao dịch đã commit, việc executor từ chối tác vụ (rejection) sẽ không thể quay ngược lại để rollback đơn hàng.
- Các sự kiện cục bộ (local after-commit event) có thể bị mất nếu tiến trình bị sập trước hoặc ngay trong lúc chuyển giao (handoff).
- Lệnh `CompletableFuture.cancel(true)` chỉ là một nỗ lực tốt nhất (best-effort); các I/O hoặc giao dịch của worker có thể vẫn tiếp tục chạy và commit.
- Với các hàm `@Async` trả về `void`, bạn cần cấu hình `AsyncUncaughtExceptionHandler`; còn với hàm trả về future, caller hoặc một trình giám sát phải liên tục theo dõi kết quả ngoại lệ (exceptional completion).
- Bắt buộc phải sử dụng bộ thực thi có giới hạn (bounded executor) và chính sách từ chối (rejection policy); xem chi tiết ở JVM-008.
- Cơ chế thử lại (worker retry) yêu cầu tính lũy đẳng (idempotency) vì caller hoặc sự kiện có thể được phát lại ở một quy trình bền vững (durable flow) khác.

## Truyền bối cảnh (Context propagation)

Bạn có thể dùng `TaskDecorator` để sao chép các bối cảnh như MDC, thông tin truy vết (trace) hoặc bảo mật nếu chính sách hệ thống cho phép. Tuy nhiên, tuyệt đối không sao chép `TransactionSynchronizationManager`, `EntityManager` hay kết nối JDBC sang luồng của worker. Việc truyền bối cảnh truy vết không làm thay đổi tính nguyên tử (atomicity) của cơ sở dữ liệu.

## Đa máy chủ và độ bền vững (Multi-instance và durability)

Trình lắng nghe cục bộ (After-commit local listener) đảm bảo đúng thứ tự commit trong nội bộ một máy chủ (node) nhưng lại không bền vững (không durable). Nếu node sập ngay sau khi commit, công việc sẽ tan biến; việc mở rộng hệ thống (scaling) không giúp phục hồi lại các tác vụ này. Nếu công việc bắt buộc phải được thực thi (eventually execute), bạn phải ghi một bản ghi outbox vào cùng giao dịch Tx-Order, và để một cơ chế chuyển tiếp (relay/consumer) xử lý có tính lũy đẳng (được phân tích trong case `MSG-007`).

## Khả năng quan sát (Observability)

Cần giám sát các dấu mốc thời gian: khi đơn hàng được commit → khi lịch async được lên → khi bắt đầu chạy → khi thành công hoặc thất bại. Theo dõi số lần executor từ chối, tuổi của hàng đợi (queue age), các trường hợp đơn hàng bị thiếu (missing-order) hay công việc mồ côi (orphan outcome), sự hoàn thành trễ (late completion) sau khi đã hủy, và độ trễ của outbox (outbox lag) nếu sử dụng luồng xử lý bền vững. Sử dụng ID của sự kiện bất biến (immutable event ID) để tương quan (correlation) dữ liệu.

Kiến thức nền: [Ranh giới giao dịch trong Spring](../../concepts/spring-transaction-boundaries.md).
