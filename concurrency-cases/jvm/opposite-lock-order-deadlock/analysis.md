# Wait-for cycle và nguyên nhân gốc

## Trạng thái ban đầu

A và B có lock tự do. T1 chuyển A→B; T2 chuyển B→A.

## Interleaving tạo deadlock

| Bước | T1 | T2 | Wait-for graph |
| --- | --- | --- | --- |
| 1 | acquire Lock-A | | T1 giữ A |
| 2 | | acquire Lock-B | T2 giữ B |
| 3 | xin Lock-B và block | | T1 → T2 |
| 4 | | xin Lock-A và block | T1 → T2 → T1 |

Cycle không tự biến mất vì không thread nào release lock đầu tiên trước khi lấy
được lock thứ hai.

> **Nói ngắn gọn:** mỗi thread giữ thứ actor kia cần và chỉ release sau khi nhận
> được thứ chính nó đang chờ.

## Expected và actual

| Khía cạnh | Mong đợi | Thực tế |
| --- | --- | --- |
| Progress | Hai transfer hoàn tất tuần tự | Cả hai đứng vô hạn |
| Balance | Tổng được bảo toàn sau operation | State chưa đổi nhưng resource/thread bị giữ |
| Failure | Deadline trả lỗi có cleanup | `lock()`/monitor không có deadline |
| Diagnostics | Conflict thành exception | JVM không tự chọn victim cho application |

## Nguyên nhân theo từng lớp

Bốn điều kiện cùng tồn tại: mutual exclusion, hold-and-wait, no preemption và
circular wait. Broken code trực tiếp tạo circular wait vì order phụ thuộc hướng
transfer.

`ReentrantLock`/monitor tạo happens-before khi cùng lock được release/acquire,
nhưng guarantee đó không giúp progress khi cycle không có release. Spring
transaction không tham gia; đây là heap state. Database deadlock có detector và
victim rollback khác hẳn case này.

## Deterministic ordering phá cycle

Định nghĩa total order theo stable unique `accountId`. Cả A→B và B→A đều acquire
min-ID trước, max-ID sau. Nếu T1 giữ first lock, T2 chưa thể giữ second lock để
tạo chiều ngược của cycle; circular wait bị loại bỏ.

Không order bằng vai trò source/destination. Nếu comparator có thể trả bằng nhau
cho hai resource khác nhau, cần tie-breaker duy nhất hoặc một tie lock.

## Timed acquisition và retry

`tryLock`/`lockInterruptibly` giới hạn wait và cho phép cleanup, nhưng không phải
proof không deadlock. Nếu acquire lock hai thất bại, release lock một trong
`finally`, rồi retry ngoài critical section với deadline, attempt cap và jitter.

Không mutate balance hoặc gọi external side effect trước khi đủ mọi lock. Nếu
attempt đã tạo side effect, retry cần idempotency/reconciliation.

## Failure, interruption và crash

- Khôi phục interrupt status khi chuyển `InterruptedException` thành domain error.
- Chỉ unlock lock đã acquire và unlock theo thứ tự ngược.
- Validation diễn ra sau khi có đủ lock nếu nó phụ thuộc mutable balance.
- Runtime exception vẫn release cả hai lock qua `finally`.
- Process crash phá local deadlock nhưng mất in-memory operation; không có durable
  rollback/retry record.

## Multi-instance và PostgreSQL boundary

Mỗi node có account object/lock riêng, nên local order không bảo vệ shared database
rows. PostgreSQL có thể lock thêm index/row theo execution path và phát hiện cycle
riêng. `DB-008` phải xử lý SQLSTATE, victim rollback và transaction retry.

## Hậu quả

- request latency không giới hạn, thread/executor starvation dây chuyền;
- connection/request context bị giữ;
- retry storm từ upstream;
- health check không chạm lock vẫn báo healthy;
- restart tạm giải phóng nhưng không sửa lock order.

## Quan sát

Thu thread dump và `ThreadMXBean.findDeadlockedThreads`; log account IDs theo
canonical order, lock wait duration và operation deadline. Alert khi detector trả
thread IDs hoặc acquisition timeout tăng. Nội dung chung ở
[Deadlock và retry an toàn](../../concepts/deadlocks-and-retries.md).
