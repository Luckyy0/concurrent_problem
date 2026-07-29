# Deadlock và retry an toàn

## Mục đích

Tài liệu này định nghĩa vocabulary chung cho JVM lock, database lock và
coordination retry. Mỗi case vẫn phải giải thích detector, victim và transaction
boundary của layer cụ thể.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| bế tắc (`deadlock`) | Một nhóm actor chờ lẫn nhau theo vòng kín nên không actor nào tiến tiếp |
| đồ thị chờ (`wait-for graph`) | Đồ thị actor → resource/actor đang bị chờ |
| chu trình chờ (`wait-for cycle`) | Một vòng trong wait-for graph, dấu hiệu cốt lõi của deadlock |
| thứ tự khóa xác định (`deterministic lock ordering`) | Mọi actor acquire nhiều lock theo cùng một total order |
| victim | Actor/transaction bị detector hủy để phá cycle |
| bounded acquisition | Chờ lock có deadline/timeout thay vì vô hạn |
| retry boundary | Phạm vi operation được phép chạy lại sau conflict |

## Bốn điều kiện cần

Deadlock cổ điển cần đồng thời: mutual exclusion, hold-and-wait, không thể cưỡng
chế thu hồi resource (`no preemption`) và circular wait. Phá circular wait bằng
một total lock order thường là giải pháp dễ chứng minh nhất.

Timeout không xóa nguyên nhân; nó chỉ giới hạn thời gian chờ và yêu cầu actor
release những lock đã giữ. Retry không có ordering/backoff có thể biến deadlock
thành livelock hoặc retry storm.

## JVM và PostgreSQL khác nhau

JVM intrinsic lock không có cơ chế tự chọn victim cho application. Deadlocked
platform thread thường đứng cho tới khi process restart; intrinsic monitor wait
không interruptible. `ReentrantLock.lockInterruptibly`/`tryLock` cho phép code tự
contain wait nếu cleanup đúng.

PostgreSQL có deadlock detector, chọn một transaction làm victim và abort với lỗi.
Application phải rollback transaction trước retry. Isolation, row-lock order và
idempotency quyết định retry có an toàn hay không.

> **Nói ngắn gọn:** cùng có wait-for cycle, nhưng JVM và database khác nhau ở
> người phát hiện, cách phá vòng và state phải rollback.

## Cách phòng ngừa

1. Giảm số lock cần giữ đồng thời.
2. Định nghĩa total order trên stable key và acquire theo order đó.
3. Giữ critical section ngắn; không gọi remote I/O khi đang giữ nhiều lock nếu
   có thể tránh.
4. Dùng bounded/interruptible acquisition khi latency contract yêu cầu.
5. Release theo thứ tự ngược trong `finally`.
6. Không retry vô hạn; dùng attempt limit, deadline và backoff có jitter.

## Retry an toàn

Chỉ retry sau khi mọi lock/transaction state của attempt cũ đã được cleanup.
Operation có external side effect cần idempotency key hoặc reconciliation trước
retry. Phân loại rõ deadlock/conflict có thể retry với validation/business failure
không nên retry.

## Quan sát và chẩn đoán

- JVM: thread dump, `ThreadMXBean.findDeadlockedThreads`, lock owner và stack.
- PostgreSQL: SQLSTATE, deadlock log, blocked/blocking PID, transaction age.
- Metric: acquisition timeout, deadlock detection, retry attempt/outcome và
  operation deadline.

## Liên kết

- [JVM-007 — Opposite Lock Ordering Deadlock](../jvm/opposite-lock-order-deadlock/README.md)
- [DB-008 — PostgreSQL opposite row order deadlock](../postgresql/opposite-row-order-deadlock/README.md)
- [DB-009 — Serializable abort and safe retry](../postgresql/serializable-retry/README.md)
- [SPR-006 — Retry transaction boundary](../spring/retry-transaction-boundary/README.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
