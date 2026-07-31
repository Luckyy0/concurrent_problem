# SPR-002 — Async work thoát khỏi transaction của caller

## Tóm tắt

`OrderPlacementService` tạo đơn hàng (order) trong một giao dịch (transaction) rồi gọi đến một bộ xử lý `@Async`.
Hàm bất đồng bộ (async method) này chạy trên một luồng (thread) khác thuộc executor, do đó nó không tham gia vào giao dịch của phương thức gọi (caller). Hậu quả là, nó có thể truy vấn dữ liệu trước khi giao dịch bên ngoài (outer transaction) kịp commit và sẽ không tìm thấy đơn hàng; hoặc nó có thể ghi nhận một tác động bên ngoài (side effect) độc lập rồi sau đó giao dịch bên ngoài bị rollback, để lại một công việc bị mồ côi (orphan work).

Bài toán này giúp bảo vệ các rào chắn tính đúng đắn (invariant) sau:

```text
Công việc bất đồng bộ phụ thuộc vào đơn hàng chỉ được phép điều phối (dispatch) sau khi giao dịch của đơn hàng đã commit thành công.
Việc giao dịch bên ngoài bị rollback không được phép sinh ra các thông báo (notification) hay bản ghi kiểm toán (audit) ngầm hiểu rằng đơn hàng đã hợp lệ.
Tuyệt đối không truyền EntityManager, các thực thể đang được quản lý (managed entity) hoặc bối cảnh giao dịch (transaction context) xuyên qua các luồng.
Yêu cầu về độ bền dữ liệu (durability) phải được xác định rõ ràng: chấp nhận rủi ro mất mát dữ liệu cục bộ sau khi commit (local after-commit) hay phải dùng bảng lưu trữ tạm (transactional outbox) để đảm bảo an toàn tuyệt đối.
```

> **Nói ngắn gọn:** Chuyển mã nguồn sang chạy trên một luồng khác cũng đồng nghĩa với việc đưa nó ra khỏi ranh giới của giao dịch hiện tại; `@Async` không phải là sự nối tiếp của giao dịch cơ sở dữ liệu.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Thread-bound transaction | Giao dịch hoặc tài nguyên được gắn chặt với luồng đang thực thi |
| Async boundary | Ranh giới bất đồng bộ, nơi công việc được giao sang một luồng xử lý khác |
| After-commit work | Công việc chỉ được lên lịch chạy sau khi giao dịch đã commit thành công |
| Orphan side effect | Tác động bên ngoài vẫn tồn tại dù cho khối dữ liệu gốc đã bị rollback |
| Context propagation | Việc truyền các bối cảnh như MDC, bảo mật hoặc truy vết (trace); không đồng nghĩa với việc truyền giao dịch |
| Transactional event listener | Trình lắng nghe sự kiện chạy theo từng giai đoạn (commit/rollback) của giao dịch phát ra sự kiện |
| Local handoff | Giao công việc vào một executor trong cùng tiến trình, có nguy cơ bị mất khi tiến trình gặp sự cố (crash) |
| Transactional outbox | Ghi trạng thái nghiệp vụ và bản ghi sự kiện trong cùng một giao dịch cơ sở dữ liệu |

## Bối cảnh và transaction boundary

| Thành phần | Giá trị |
| --- | --- |
| Caller | Luồng yêu cầu (Request thread), chứa `@Transactional` bên ngoài |
| Async worker | Luồng `orderExecutor` |
| Caller transaction | Lưu đơn hàng, commit/rollback ở cuối hàm gọi |
| Worker transaction | Không có, hoặc là một giao dịch mới hoàn toàn nếu hàm async có gắn annotation |
| Race window | Khoảng thời gian từ sau khi nộp (submit) tác vụ async, đến trước khi giao dịch bên ngoài commit |
| Database | PostgreSQL với mức độ cô lập `READ COMMITTED` |
| Durability boundary | Bộ thực thi cục bộ (Local executor) không đảm bảo tính bền vững của dữ liệu |

## Hướng dẫn điều hướng

- [Code lỗi](broken-code.md)
- [Phân tích luồng và giao dịch](analysis.md)
- [Các giải pháp after-commit và outbox](solutions.md)
- [Các thí nghiệm tích hợp với PostgreSQL](experiments.md)
- [Ranh giới giao dịch trong Spring](../../concepts/spring-transaction-boundaries.md)
- [Kiểm thử các vấn đề đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production nếu làm sai

- Truy vấn bất đồng bộ báo "không tìm thấy đơn hàng" do sai lệch về thời gian (timing).
- Thông báo, kiểm toán hoặc công việc nền được commit cho một đơn hàng đã bị rollback.
- Lỗi tải dữ liệu trễ (lazy-loading exception) khi một thực thể (entity) đi qua luồng khác.
- Tác vụ tương lai (future) bị hủy (cancel) nhưng giao dịch của worker vẫn tiếp tục commit.
- Bộ thực thi (executor) từ chối công việc sau khi đã commit, làm mất sự kiện cục bộ.
- Bối cảnh truy vết (trace) hoặc bảo mật bị sai lệch nếu chính sách truyền tải (propagation policy) không được định nghĩa rõ ràng.

## Hướng sửa chữa (Khuyến nghị)

- Nếu công việc đòi hỏi phải chung ranh giới nguyên tử (atomic boundary): Hãy chạy đồng bộ (synchronous) ngay trong giao dịch của caller.
- Nếu công việc chỉ được bắt đầu sau khi commit và có thể chấp nhận rủi ro mất dữ liệu khi hệ thống sập: Hãy phát hành sự kiện nghiệp vụ (domain event) và xử lý bằng `@TransactionalEventListener(AFTER_COMMIT)`; có thể kết hợp thêm với `@Async`.
- Nếu công việc bắt buộc phải bền vững và thực hiện ít nhất một lần (durable/at-least-once): Sử dụng mẫu transactional outbox (`MSG-007`).
- Nếu hàm async cần ghi dữ liệu riêng: Mở một giao dịch mới trên worker sau khi sự kiện after-commit được điều phối, tuyệt đối không cố gắng truyền giao dịch của caller sang.

## Phạm vi của bài viết

Bài viết này xử lý việc gọi hàm bất đồng bộ cục bộ (local async invocation). Việc phát hành sự kiện bền vững (durable event publication), mô hình inbox/outbox và cơ chế gửi lại tin nhắn (message redelivery) thuộc phạm vi của bài toán `MSG-007`; tình trạng "chết đói" của executor (executor starvation) thuộc phạm vi của bài toán `JVM-008`.
