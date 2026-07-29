# Phân tích starvation và progress dependency

## Timeline

| Bước | Worker-1 | Worker-2 | Queue |
| --- | --- | --- | --- |
| 1 | Parent P1 | Parent P2 | rỗng |
| 2 | P1 submit C1 | P2 submit C2 | C1, C2 |
| 3 | P1 chờ C1 | P2 chờ C2 | C1, C2 không chạy |

Không có monitor wait-for cycle; cycle nằm ở capacity: parent giữ worker chờ child,
child chờ worker do parent giữ.

## Expected và actual

Expected: accepted request hoàn tất trong deadline và queue phản ánh work có thể
được phục vụ. Actual: active count luôn bằng pool size, completed count đứng yên,
queue chứa child, CPU có thể thấp.

## Root cause

Executor sizing được quyết định độc lập với dependency graph. Blocking coefficient
hay công thức pool size không cứu một topology mà mọi worker đồng thời chờ task
cùng pool. Nested blocking phá progress invariant.

`Future.get()` tạo happens-before khi child hoàn tất, nhưng child không có cơ hội
hoàn tất. Timeout chỉ phá wait sau deadline; cancellation phải propagate và task
phải tôn trọng interrupt.

## Overload và backpressure

Bounded queue cần rejection policy gắn với API: fail-fast 429/503, bounded caller
wait hoặc admission semaphore. Không retry ngay vào cùng saturated instance.
Deadline phải tính queue delay + execution + dependency latency.

## Failure, shutdown và multi-instance

- Khi parent timeout, cancel child và cleanup context.
- `shutdown()` không nhận task mới; `shutdownNow()` gửi interrupt nhưng không bảo
  đảm remote I/O dừng.
- Mỗi node có pool riêng; load balancer có thể tiếp tục gửi vào saturated node.
- Autoscaling chậm hơn retry storm; cần per-node admission/load-shedding.
- JDBC connection starvation là resource cycle khác, thuộc `SPR-007`.

## Quan sát

Theo dõi active/max, queue size/capacity, oldest queue age, task submitted/started/
completed/rejected, execution time, queue wait, cancellation và deadline. Thread
dump cho thấy worker ở `Future.get` và child chưa chạy.

> **Nói ngắn gọn:** utilization phải được đọc cùng progress; “100% worker bận” có
>thể nghĩa mọi worker đang chờ chứ không phải đang xử lý.
