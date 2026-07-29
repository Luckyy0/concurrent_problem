# JVM-007 — Deadlock do acquire lock ngược thứ tự

## Tóm tắt

Hai thread chuyển giá trị giữa hai in-memory account. Mỗi transfer khóa source
rồi destination. T1 chuyển A→B nên giữ Lock-A và chờ Lock-B; T2 chuyển B→A nên
giữ Lock-B và chờ Lock-A. Hai thread tạo một wait-for cycle và không tiến tiếp.

Case bảo vệ các invariant:

```text
Mọi operation cần nhiều account lock phải acquire theo cùng một total order.
Không thread nào được chờ vô hạn ngoài latency/deadline contract.
Khi acquire thất bại hoặc bị interrupt, mọi lock đã giữ phải được release.
Transfer hoàn tất phải bảo toàn tổng balance trong scope in-memory của case.
```

> **Nói ngắn gọn:** “khóa cả hai account” vẫn có thể sai; thứ tự khóa phải giống
> nhau cho mọi hướng transfer.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| deadlock | Nhiều thread chờ nhau theo vòng kín và không thread nào tiến tiếp |
| wait-for cycle | Chu trình T1 chờ T2, T2 lại chờ T1 |
| lock ordering | Quy tắc total order quyết định lock nào luôn được acquire trước |
| intrinsic lock | Monitor dùng bởi `synchronized` |
| ownable synchronizer | Lock như `ReentrantLock` mà JVM diagnostics biết owner |
| interruptible acquisition | Chờ lock có thể dừng khi thread bị interrupt |
| timed acquisition | `tryLock` với timeout giới hạn thời gian chờ |
| livelock | Actor liên tục retry/nhường nhau nhưng vẫn không hoàn tất |

## Bối cảnh và contention point

T1 thực hiện `transfer(A, B, 10)`, T2 thực hiện `transfer(B, A, 20)`. Account được
lưu trong một local registry và có `ReentrantLock` ổn định.

| Thành phần | Giá trị |
| --- | --- |
| Shared state | Hai `LocalAccount` và balance |
| Actor | Hai request/worker thread |
| Broken order | source lock trước, destination lock sau |
| Cycle | T1 giữ A chờ B; T2 giữ B chờ A |
| Transaction | Không có database transaction |
| Scope | Một JVM; PostgreSQL deadlock là `DB-008` |

## Điều hướng

- [Cách triển khai bị lỗi](broken-code.md)
- [Wait-for cycle và nguyên nhân](analysis.md)
- [Code đã sửa và lựa chọn timeout](solutions.md)
- [Cách tái hiện và phát hiện deadlock](experiments.md)
- [Deadlock và retry an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả production

- request treo, thread pool/pool connection dần cạn;
- timeout phía caller không tự giải phóng thread đang chờ intrinsic lock;
- retry phía client tạo thêm work và cycle;
- health endpoint có thể vẫn sống trong khi business traffic đứng;
- restart process phá deadlock nhưng làm mất in-memory work.

## Hướng sửa khuyến nghị

Dùng stable account key tạo một total order: account ID nhỏ hơn luôn khóa trước,
không phụ thuộc source/destination. Acquire interruptibly hoặc có deadline khi
latency contract cần; release lock thứ hai rồi lock thứ nhất trong `finally`.

Timed acquisition không thay thế ordering. Nó là safety net giúp tránh chờ vô
hạn; retry cần bounded attempts/backoff và chỉ chạy sau cleanup.

## Phạm vi

Case chỉ xử lý JVM locks và in-memory balance để minh họa. Không dùng local lock
để bảo vệ account row giữa nhiều node. PostgreSQL detection/victim/rollback thuộc
`DB-008`; transfer tiền bền vững thuộc các banking cases.

## Khi nào dùng giải pháp nào

- Deterministic order: mặc định khi phải giữ nhiều lock cùng lúc.
- Coarse single lock: state nhỏ, contention thấp, ưu tiên dễ chứng minh.
- `tryLock` deadline: request có latency budget, chấp nhận fail/retry có kiểm soát.
- Thiết kế không giữ hai lock: có thể thay data model/ownership/message passing.
- Database transaction: authoritative state là database row và nhiều node cùng
  cập nhật.
