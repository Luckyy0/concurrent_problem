# JVM-008 — Executor saturation, starvation và nested blocking

## Tóm tắt

Một bounded executor có 2 worker. Hai parent task chiếm cả hai worker, mỗi task
submit child task vào chính executor rồi block ở `Future.get()`. Child nằm trong
queue nhưng không worker nào rảnh để chạy: hệ thống bị **starvation** dù không có
lock cycle.

Invariant:

```text
Accepted work phải hoàn tất, fail hoặc bị cancel trong operation deadline.
Worker không được block chờ child chỉ có thể chạy trên cùng exhausted executor.
Queue phải bounded và overload phải có rejection/backpressure policy rõ.
Task cancellation phải được truyền xuống dependency đang chờ.
```

> **Nói ngắn gọn:** queue còn chỗ không có nghĩa hệ thống còn khả năng tiến; worker
>đang giữ chỗ có thể chính là actor chờ work trong queue.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| saturation | Mọi worker bận và capacity nhận thêm work đã cạn |
| starvation | Task không được cấp worker/resource để tiến tiếp |
| nested blocking | Worker submit child rồi chờ đồng bộ child |
| backpressure | Buộc producer chậm/reject thay vì tích queue vô hạn |
| rejection policy | Hành vi khi pool/queue không nhận thêm task |
| admission control | Kiểm soát số operation được phép vào hệ thống |
| queueing delay | Thời gian task nằm trong queue trước khi chạy |
| latency budget | Deadline tổng của request, không phải timeout riêng từng bước |

## Bối cảnh và contention point

Batch enrichment request chạy parent task trên `enrichmentExecutor`. Parent lại
submit pricing/profile child tasks vào cùng pool và gọi `get`. Với pool size 2,
hai request đồng thời đủ chiếm toàn bộ worker.

| Thành phần | Giá trị |
| --- | --- |
| Shared resource | Worker slots và bounded queue |
| Parent | Đang RUNNING nhưng block ở child future |
| Child | QUEUED, cần worker của cùng pool |
| Cycle tiến triển | Parent giữ worker chờ child; child chờ worker |
| Scope | Một JVM; JDBC connection pool thuộc `SPR-007` |

## Điều hướng

- [Broken code](broken-code.md)
- [Phân tích starvation](analysis.md)
- [Giải pháp và capacity policy](solutions.md)
- [Deterministic experiments](experiments.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả production

Request timeout, queue tăng, rejection muộn, thread/request context bị giữ và
retry storm. CPU có thể thấp nên dashboard “CPU bình thường” che mất outage.

## Hướng sửa

Ưu tiên bỏ nested submission: parent thực hiện dependency call trực tiếp hoặc
orchestrator submit các leaf task rồi compose ngoài worker pool. Tách executor chỉ
khi dependency/resource budget thật sự độc lập. Dùng bounded queue, explicit
rejection/backpressure, deadline chung và metric active/queue/wait/rejection.

Virtual thread giảm chi phí thread bị block nhưng không tăng JDBC connection,
remote quota hoặc heap; vẫn cần admission control.
