# Phân tích progress failure và retry symmetry

## Trạng thái ban đầu

Lock-A và Lock-B đều tự do. T1 muốn `A → B`, T2 muốn `B → A`. Ownership chưa bị
thay đổi vì mutation chỉ chạy sau khi có đủ hai lock.

## Interleaving tạo livelock

| Phase | T1 | T2 | Progress |
| --- | --- | --- | --- |
| 1 | acquire A | acquire B | mỗi actor giữ một lock |
| 2 | fail `tryLock(B)` | fail `tryLock(A)` | không swap |
| 3 | release A | release B | cả hai active, state cũ |
| 4 | fixed backoff | fixed backoff | cùng nhịp |
| 5 | acquire A | acquire B | phase lặp lại |

Không có wait-for cycle tồn tại lâu: actor release lock và tiếp tục chạy. Vì vậy
thread dump có thể cho thấy RUNNABLE/TIMED_WAITING thay vì BLOCKED, nhưng completed
operation vẫn bằng 0.

> **Nói ngắn gọn:** deadlock thất bại vì không actor nào hành động được; livelock
>thất bại vì actor hành động quá giống nhau.

## Expected và actual

| Khía cạnh | Mong đợi | Broken loop |
| --- | --- | --- |
| Terminal outcome | Success hoặc failure trước deadline | Có thể retry vô hạn |
| CPU/thread use | Tỷ lệ với useful work | Tốn cho conflict/retry |
| Cancellation | Interrupt dừng operation | `parkNanos` trở về nhưng loop tiếp tục |
| Mutation | Chỉ một complete swap | Không mutation nhưng cũng không progress |
| Observability | Conflict có budget/outcome | Retry log tăng, completion đứng yên |

## Root cause

`tryLock` bảo vệ safety và tránh blocking cycle, nhưng không tạo fairness hoặc
progress guarantee. Hai deterministic state machine nhận cùng signal, dùng cùng
fixed delay và đưa ra cùng quyết định nên symmetry được duy trì.

Backoff chỉ hữu ích khi làm giảm/xáo trộn phase overlap. Jitter tạo xác suất một
actor đến trước; deadline/attempt cap mới bảo đảm method có terminal outcome.

## Livelock, deadlock và starvation

- Deadlock: actor giữ/chờ resource theo cycle, thường không chạy tiếp.
- Livelock: actor liên tục phản ứng/retry nhưng global work không hoàn tất.
- Starvation: một actor cụ thể không tiến, trong khi actor khác có thể hoàn tất.

Một randomized retry system có thể tránh global livelock nhưng vẫn không bảo đảm
fairness cho từng actor. Nếu fairness là invariant, dùng queue/owner/fair lock có
policy rõ thay vì chỉ jitter.

## Ordering là symmetry breaking mạnh hơn

Stable total order theo `channelId` khiến cả T1 và T2 xin cùng lock trước. Một
actor thắng lock đầu; actor kia không thể đồng thời giữ lock thứ hai để chặn
winner. Ordering loại cấu trúc conflict thay vì hy vọng timing khác nhau.

Comparator phải total và stable. Equal ID cho hai object khác nhau cần bị từ chối
hoặc có unique tie-breaker; không order bằng mutable owner/current role.

## Retry budget và deadline

Retry policy cần đồng thời:

- operation deadline tính cả lock wait, backoff và business work;
- max attempts chống bug clock/config;
- exponential cap để delay không overflow;
- full/equal jitter phù hợp workload;
- interrupt/cancellation check trước và sau backoff;
- terminal `CONTENDED/TIMEOUT` outcome cho caller.

Không reset deadline sau mỗi attempt. Không log từng collision ở mức cao; aggregate
attempt histogram và exhausted count.

## Failure, side effect và cleanup

Mỗi attempt release mọi acquired lock trong `finally`. Business mutation chỉ chạy
sau khi có đủ lock, nên attempt thua không cần rollback. Nếu external side effect
đã chạy, retry cần idempotency/reconciliation và không còn là local lock loop đơn
giản.

`LockSupport.parkNanos` không ném `InterruptedException`; code phải kiểm tra
`Thread.currentThread().isInterrupted()` và kết thúc, thường giữ nguyên flag.

## Multi-instance và scope

Mỗi JVM có channel/lock riêng; local retry không phối hợp authoritative state giữa
node. Database serialization/deadlock retry có transaction rollback, error
classification và idempotency riêng tại `LOCK-002`/`DB-009`.

## Hậu quả và observability

Theo dõi attempt/completion ratio, exhausted retry, operation deadline, lock
conflict, backoff duration, CPU và runnable threads. Livelock signal điển hình:
retry tăng nhanh, success gần 0, state version không đổi. Thread dump theo thời
gian hữu ích hơn một snapshot đơn.
