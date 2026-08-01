# JVM-007 — Deadlock do acquire lock ngược thứ tự

## Tóm tắt

Hai thread chuyển giá trị giữa hai in-memory account. Mỗi thao tác chuyển khóa source rồi destination. T1 chuyển A→B nên giữ Lock-A và chờ Lock-B; T2 chuyển B→A nên giữ Lock-B và chờ Lock-A. Hai thread tạo một wait-for cycle và không tiến tiếp.

Tình huống bảo vệ các bất biến:

```text
Mọi thao tác cần nhiều account lock phải acquire theo cùng một total order.
Không thread nào được chờ vô hạn ngoài ràng buộc độ trễ (latency) và thời hạn (deadline).
Khi acquire thất bại hoặc bị ngắt (interrupt), mọi lock đã giữ phải được release.
Thao tác chuyển hoàn tất phải bảo toàn tổng balance trong phạm vi in-memory của tình huống.
```

> **Nói ngắn gọn:** “khóa cả hai account” vẫn có thể sai; thứ tự khóa phải giống nhau cho mọi hướng chuyển.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| deadlock | Nhiều thread chờ nhau theo vòng kín và không thread nào tiến tiếp |
| wait-for cycle | Chu trình T1 chờ T2, T2 lại chờ T1 |
| lock ordering | Quy tắc total order quyết định lock nào luôn được acquire trước |
| intrinsic lock | Monitor dùng bởi `synchronized` |
| ownable synchronizer | Lock như `ReentrantLock` mà JVM diagnostics biết owner |
| interruptible acquisition | Chờ lock có thể dừng khi thread bị ngắt (interrupt) |
| timed acquisition | `tryLock` với giới hạn thời gian chờ |
| livelock | Tác nhân liên tục thử lại (retry) hoặc nhường nhau nhưng vẫn không hoàn tất |

## Bối cảnh và contention point

T1 thực hiện `transfer(A, B, 10)`, T2 thực hiện `transfer(B, A, 20)`. Account được lưu trong một local registry và có `ReentrantLock` ổn định.

| Thành phần | Giá trị |
| --- | --- |
| Shared state | Hai `LocalAccount` và balance |
| Tác nhân | Hai luồng xử lý hoặc thread yêu cầu |
| Broken order | Khóa source trước, khóa destination sau |
| Cycle | T1 giữ A chờ B; T2 giữ B chờ A |
| Transaction | Không có database transaction |
| Scope | Một JVM; PostgreSQL deadlock là `DB-008` |

## Điều hướng

- [Cách triển khai bị lỗi](broken-code.md)
- [Wait-for cycle và nguyên nhân](analysis.md)
- [Code đã sửa và lựa chọn timeout](solutions.md)
- [Cách tái hiện và phát hiện deadlock](experiments.md)
- [Deadlock và thử lại an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả production

- Các yêu cầu bị treo, thread pool và connection pool dần cạn kiệt;
- Quá thời gian chờ (timeout) ở phía gọi không tự giải phóng thread đang chờ intrinsic lock;
- Việc thử lại từ phía client tạo thêm khối lượng công việc và chu trình (cycle);
- Health endpoint có thể vẫn hoạt động bình thường trong khi lưu lượng nghiệp vụ bị đình trệ;
- Việc khởi động lại process sẽ phá deadlock nhưng làm mất dữ liệu trên bộ nhớ (in-memory).

## Hướng sửa khuyến nghị

Sử dụng khóa định danh account (account key) ổn định để tạo một total order: account ID nhỏ hơn luôn khóa trước, không phụ thuộc vào source hay destination. Thực hiện acquire có thể ngắt (interruptibly) hoặc có thời hạn (deadline) khi ràng buộc độ trễ yêu cầu; release lock thứ hai rồi lock thứ nhất trong `finally`.

Việc giới hạn thời gian acquire (timed acquisition) không thay thế cho lock ordering. Nó là một cơ chế an toàn (safety net) giúp tránh chờ vô hạn; việc thử lại cần giới hạn số lần (bounded attempts), có khoảng lùi (backoff) và chỉ chạy sau khi dọn dẹp (cleanup).

## Phạm vi

Tình huống chỉ xử lý JVM lock và in-memory balance để minh họa. Không dùng local lock để bảo vệ dữ liệu row của account giữa nhiều node. Việc phát hiện, chọn nạn nhân (victim), và rollback của PostgreSQL thuộc về `DB-008`; thao tác chuyển tiền bền vững thuộc các tình huống ngân hàng (banking cases).

## Khi nào dùng giải pháp nào

- Định hướng tất định (Deterministic order): mặc định khi phải giữ nhiều lock cùng lúc.
- Coarse single lock: trạng thái nhỏ, contention thấp, ưu tiên dễ chứng minh.
- `tryLock` với deadline: yêu cầu có ngân sách độ trễ (latency budget), chấp nhận thất bại hoặc thử lại có kiểm soát.
- Thiết kế không giữ hai lock: có thể thay đổi mô hình dữ liệu, quyền sở hữu (ownership), hoặc truyền thông điệp (message passing).
- Database transaction: trạng thái xác thực (authoritative state) là database row và có nhiều node cùng cập nhật.
