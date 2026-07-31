# SPR-007 — Cạn kiệt Connection Pool do giao dịch kéo dài (Long transactions)

## Tóm tắt

Hàm `PaymentRiskService.assessAndApprove()` mở một giao dịch, khóa dòng thanh toán (payment row) bằng chế độ `PESSIMISTIC_WRITE`, sau đó gọi sang API kiểm tra rủi ro từ xa (remote risk API). Khi độ trễ (latency) của mạng hoặc hệ thống từ xa tăng lên:

- Tiến trình đang giữ khóa (lock holder) sẽ ôm luôn cả kết nối cơ sở dữ liệu (connection) và khóa dòng (row lock);
- Các yêu cầu trùng lặp (duplicate requests) bị xếp hàng chờ cùng một khóa dòng, nhưng chúng vẫn chiếm dụng các kết nối khác;
- Các yêu cầu xử lý thanh toán cho những đơn hàng khác nhau cũng bị treo lại và giữ kết nối trong lúc chờ hệ thống từ xa phản hồi;
- Kết cục là hồ chứa kết nối (pool) cạn sạch;
- Các luồng truy cập cơ sở dữ liệu hoàn toàn không liên quan (unrelated database traffic) bỗng dưng bị hết thời gian chờ (timeout) khi cố gắng lấy kết nối.

Rào chắn tính đúng đắn (Invariant):

```text
Tuyệt đối không được giữ kết nối cơ sở dữ liệu / giao dịch / khóa dòng trong lúc chờ đợi I/O từ xa, chờ tài nguyên của luồng thực thi (executor capacity), hoặc tham gia vào một quá trình phối hợp không xác định thời gian (unbounded coordination).
Mọi quá trình chờ đợi bên thứ ba phải có thời hạn (timeout) và cơ chế vách ngăn (bulkhead) riêng biệt.
Giai đoạn chốt giao dịch dữ liệu (Database commit phase) phải cực kỳ ngắn, có giới hạn thời gian và bắt buộc phải kiểm tra lại (revalidate) tính hợp lệ của trạng thái trước khi thay đổi.
Việc hồ chứa kết nối bị quá tải (Pool saturation) phải tạo ra áp lực ngược (backpressure) hoặc phải thất bại nhanh (fail-fast) trước khi tạo ra hiệu ứng sụp đổ dây chuyền (cascading failure).
```

> **Nói ngắn gọn:** Hồ chứa kết nối (connection pool) không chỉ cạn kiệt vì các câu truy vấn cơ sở dữ liệu chạy quá chậm; mã lệnh dù không đụng chạm gì đến cơ sở dữ liệu, nhưng chỉ cần nằm kẹt bên trong một giao dịch (transaction) thì nó vẫn nghiễm nhiên chiếm dụng một kết nối.

## Các thành phần và tài nguyên dùng chung (Actors và shared resources)

| Thành phần | Vai trò |
| --- | --- |
| Yêu cầu A (Request A) | Khóa đơn hàng P-42 rồi đứng chờ quyết định rủi ro từ xa phản hồi |
| Yêu cầu trùng lặp B (Duplicate B) | Đứng chờ khóa dòng của P-42, nhưng lại chiếm giữ một kết nối khác |
| Các yêu cầu C..N | Khóa các đơn hàng khác nhau rồi cùng rủ nhau đứng chờ hệ thống từ xa |
| Yêu cầu không liên quan U (Unrelated request U) | Cần một kết nối cho câu truy vấn cực ngắn, nhưng hồ chứa kết nối đã đầy |
| HikariCP | Quản lý việc cấp phát/thu hồi một lượng hữu hạn (finite) các kết nối cơ sở dữ liệu |
| PostgreSQL | Giữ các khóa dòng (row locks) cho đến tận khi giao dịch được commit hoặc rollback |
| Hệ thống rủi ro từ xa (Remote risk API) | Một miền phụ thuộc độc lập với cơ sở dữ liệu, có độ trễ và rủi ro sập riêng (failure domain) |

## Trạng thái dùng chung và ranh giới giao dịch (Shared state và transaction boundary)

```text
payment_order:
  id = P-42
  status = RISK_PENDING
  amount = 500
  business_version = 12
```

Ranh giới thiết kế sai (Broken boundary):

```text
BEGIN (Bắt đầu giao dịch)
  -> SELECT ... FOR UPDATE (Khóa dữ liệu)
  -> gọi hệ thống từ xa / đứng chờ phản hồi (future wait)
  -> UPDATE APPROVED (Cập nhật trạng thái)
COMMIT (Kết thúc giao dịch)
```

Ranh giới được khuyến nghị (Recommended boundary):

```text
đọc nhanh một bản ảnh (short snapshot read) -> đóng kết nối
gọi hệ thống rủi ro từ xa với thời hạn / vách ngăn (deadline/bulkhead) rõ ràng
BEGIN short commit transaction (Mở một giao dịch commit cực ngắn)
  -> khóa/tải lại dữ liệu (lock/reload)
  -> thẩm định lại phiên bản/trạng thái/đối tượng quyết định (revalidate)
  -> cập nhật hoặc từ chối nếu quyết định đã lỗi thời (reject stale decision)
COMMIT
```

## Điểm tranh chấp (Contention point)

Hệ thống có hai loại tài nguyên hữu hạn (finite resources):

1. Khóa dòng (row lock) trên bảng `payment_order`;
2. Kết nối cơ sở dữ liệu (database connection) trong application pool.

Nhiều yêu cầu truy cập vào cùng một đơn hàng sẽ tạo ra một hàng đợi khóa (lock queue). Trong khi đó, các yêu cầu truy cập vào những đơn hàng khác nhau tuy không tranh chấp khóa dòng (row conflict), nhưng chúng vẫn có khả năng "hút cạn" toàn bộ số lượng kết nối trong hồ chứa khi đồng loạt chờ đợi hệ thống từ xa phản hồi. Do đó, hiện tượng chết đói kết nối (pool starvation) hoàn toàn có thể xảy ra mà chẳng cần một bế tắc cơ sở dữ liệu (database deadlock) nào.

## Kỳ vọng và Thực tế

| | Thời gian giao dịch (Transaction duration) | Hồ chứa kết nối (Pool) | Truy vấn không liên quan |
| --- | --- | --- | --- |
| Điều mong đợi | Chỉ bao quanh các tác vụ DB cực ngắn | Vẫn còn dư dả (headroom) | Hoàn tất bình thường |
| Lỗi thực tế | Bằng thời gian xử lý DB + thời gian chờ hệ thống từ xa/luồng thực thi | Đang dùng = tối đa, số lượng chờ mượn tăng cao | Hết thời gian chờ cấp kết nối (Acquisition timeout) |

Việc truy vấn U bị quá hạn chờ lấy kết nối (Pool acquisition timeout) chỉ là triệu chứng ngọn ngành (downstream symptom); tăng kích thước hồ chứa (pool size) thực chất chỉ là dời điểm sụp đổ hệ thống lùi lại một chút, và tệ hơn là có thể chuyển gánh nặng quá tải sang hẳn PostgreSQL.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| Connection pool | Hồ chứa kết nối: Một tập hợp hữu hạn các kết nối JDBC được tái sử dụng |
| Pool starvation | Chết đói hồ chứa kết nối: Có công việc cần kết nối nhưng mọi kết nối đều đã bị chiếm giữ |
| Transaction duration | Thời gian giao dịch: Khoảng thời gian từ khi bắt đầu (begin) hoặc mượn kết nối (first acquisition) đến khi commit/rollback |
| Lock duration | Thời gian giữ khóa: Khoảng thời gian từ lúc chiếm được khóa (lock acquisition) cho đến khi giao dịch kết thúc |
| Pending acquisition | Hàng chờ mượn: Luồng (thread) đang đứng chờ để mượn kết nối từ hồ chứa |
| Backpressure | Áp lực ngược: Khả năng giới hạn/đẩy ngược tải trọng trước khi tài nguyên sụp đổ |
| Bulkhead | Vách ngăn: Cơ chế giới hạn số lượng xử lý đồng thời của một hệ thống phụ thuộc nhằm cô lập lỗi |
| Timeout budget | Ngân sách thời gian: Phân bổ tổng thời hạn (deadline) cho pool, lock, tác vụ từ xa và khâu dọn dẹp |
| `idle in transaction` | Rảnh rỗi trong giao dịch: Trạng thái cơ sở dữ liệu không chạy câu truy vấn nào nhưng giao dịch thì vẫn mở toang |

## Hướng dẫn điều hướng

- [Giao dịch kéo dài bị lỗi (Broken long transaction)](broken-code.md)
- [Phân tích hiện tượng chết đói kết nối (Pool-starvation analysis)](analysis.md)
- [Giải pháp giao dịch ngắn và áp lực ngược (Short transaction and backpressure solutions)](solutions.md)
- [Các thí nghiệm với sự quá tải có giới hạn (Bounded saturation experiments)](experiments.md)
- [Khóa trong PostgreSQL (PostgreSQL locks)](../../concepts/postgresql-locks.md)
- [Ranh giới giao dịch trong Spring](../../concepts/spring-transaction-boundaries.md)
- [Kiểm thử các vấn đề đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production nếu làm sai

- Các lỗi hết thời gian chờ mượn JDBC (JDBC acquisition timeouts) lan tràn sang cả những API hoàn toàn không liên quan.
- Hàng đợi khóa dòng (Row lock queues) làm tăng tuổi thọ giao dịch (transaction age) và độ trễ đuôi (tail latency).
- Các luồng xử lý yêu cầu (Request threads/executors) bị giam cầm, dẫn đến tình trạng chết đói luồng thứ cấp (secondary thread starvation).
- Các dịch vụ kiểm tra sức khỏe/độ sẵn sàng (Readiness/health dependencies) có thể báo lỗi và châm ngòi cho hàng loạt chu kỳ khởi động lại/thêm bớt máy chủ vô bổ (restart/scale churn).
- Việc mở rộng quy mô (Scale-out) làm nhân lên tổng số kết nối mạng, khiến PostgreSQL quá tải nhanh hơn.
- Việc nhắm mắt nhắm mũi thử lại (Timeout retry mù) càng làm tăng tốc độ gửi yêu cầu (arrival rate) và thổi phồng sự cố lên gấp bội.

## Hướng sửa chữa (Khuyến nghị)

Hãy bóc tách triệt để việc chờ đợi hệ thống từ xa (remote wait) hoặc luồng thực thi (executor wait) ra khỏi phạm vi của một giao dịch (transaction). Tiến hành lấy một bản ảnh bất biến (immutable snapshot), nhả ngay kết nối (release connection), rồi mới gọi dịch vụ từ xa với một thời hạn (deadline) hoặc vách ngăn (bulkhead) đàng hoàng. Sau đó, mở một giao dịch mới thật ngắn (short transaction) để khóa dữ liệu và thẩm định lại tính hợp lệ (revalidate) trước khi chốt commit.

Đối với những quy trình kéo dài hoặc khi hệ thống từ xa có độ tin cậy thấp, hãy cân nhắc sử dụng cỗ máy trạng thái bền vững (durable state machine) hoặc mẫu thiết kế hộp thư đi (outbox) thay cho các giao dịch đồng bộ.

## Phạm vi

Bài toán này tập trung phân tích tình trạng cạn kiệt tài nguyên (resource starvation) và hiện tượng quá hạn chờ dây chuyền (cascading timeout), chứ không đi sâu vào cơ chế dò tìm bế tắc (deadlock detection) hay xử lý những hậu quả không rõ ràng từ các hệ thống phụ thuộc (unknown outcome của remote side effect).
