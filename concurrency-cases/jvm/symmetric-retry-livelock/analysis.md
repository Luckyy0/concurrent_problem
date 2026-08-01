# Phân tích progress failure và retry symmetry

## Trạng thái ban đầu

Lock-A và Lock-B đều tự do. T1 muốn `A → B`, T2 muốn `B → A`. Ownership chưa bị thay đổi vì mutation chỉ chạy sau khi lấy đủ hai lock.

## Interleaving tạo livelock

| Phase | T1 | T2 | Progress |
| --- | --- | --- | --- |
| 1 | acquire A | acquire B | mỗi actor giữ một lock |
| 2 | fail `tryLock(B)` | fail `tryLock(A)` | không thực hiện hoán đổi (swap) |
| 3 | release A | release B | cả hai vẫn active, giữ nguyên state cũ |
| 4 | fixed backoff | fixed backoff | cùng nhịp |
| 5 | acquire A | acquire B | chu kỳ (phase) lặp lại |

Không có wait-for cycle nào tồn tại lâu: actor release lock và tiếp tục chạy. Vì vậy thread dump có thể hiển thị trạng thái RUNNABLE hoặc TIMED_WAITING thay vì BLOCKED, nhưng số lượng completed operation vẫn bằng 0.

> **Nói ngắn gọn:** deadlock thất bại vì không actor nào hành động được; livelock thất bại vì các actor hành động quá giống nhau.

## Kết quả mong đợi và thực tế

| Khía cạnh | Mong đợi | Broken loop |
| --- | --- | --- |
| Kết quả cuối cùng (Terminal outcome) | Thành công (Success) hoặc lỗi (failure) trước deadline | Có thể retry vô hạn |
| Mức sử dụng CPU và thread | Tỷ lệ thuận với công việc hữu ích (useful work) | Bị tiêu tốn cho conflict và retry |
| Huỷ (Cancellation) | Interrupt dừng operation | `parkNanos` trả về nhưng loop tiếp tục |
| Đột biến (Mutation) | Chỉ có một thao tác hoán đổi (swap) hoàn tất | Không có mutation nhưng cũng không có progress |
| Khả năng quan sát (Observability) | Conflict có budget và outcome rõ ràng | Log retry tăng lên, mức độ hoàn thành (completion) đứng yên |

## Nguyên nhân gốc rễ (Root cause)

`tryLock` bảo vệ tính an toàn (safety) và tránh các blocking cycle, nhưng không tạo ra fairness hoặc progress guarantee. Hai cỗ máy trạng thái tất định (deterministic state machine) nhận cùng một signal, dùng cùng một fixed delay và đưa ra cùng một quyết định nên tính đối xứng (symmetry) được duy trì.

Backoff chỉ hữu ích khi làm giảm hoặc xáo trộn sự chồng chéo (phase overlap). Jitter tạo ra xác suất một actor đến trước; trong khi đó, deadline và giới hạn attempt (attempt cap) mới bảo đảm method có terminal outcome.

## Livelock, deadlock và starvation

- Deadlock: các actor giữ và chờ resource theo vòng (cycle), thường không tiếp tục chạy.
- Livelock: các actor liên tục phản ứng và retry nhưng tổng thể công việc (global work) không hoàn tất.
- Starvation: một actor cụ thể không tạo ra tiến triển, trong khi các actor khác có thể hoàn tất.

Một hệ thống randomized retry có thể tránh global livelock nhưng vẫn không bảo đảm fairness cho từng actor. Nếu fairness là một invariant, hãy dùng queue, owner hoặc fair lock có chính sách rõ ràng thay vì chỉ dùng jitter.

## Ordering là kỹ thuật symmetry breaking mạnh hơn

Stable total order dựa trên `channelId` khiến cả T1 và T2 yêu cầu (acquire) cùng một lock trước. Một actor sẽ giành được lock đầu tiên; actor còn lại không thể đồng thời giữ lock thứ hai để chặn winner. Ordering loại bỏ hoàn toàn cấu trúc gây ra conflict thay vì hy vọng vào sự khác biệt trong timing.

Comparator phải đảm bảo tính total và stable. Nếu hai object khác nhau có ID giống nhau thì cần bị từ chối hoặc có một tie-breaker duy nhất; không thực hiện order dựa trên các thuộc tính có thể thay đổi (mutable owner hay current role).

## Retry budget và deadline

Retry policy cần đồng thời có các cơ chế sau:

- operation deadline tính cả thời gian chờ lock, backoff và xử lý nghiệp vụ (business work);
- số lượng max attempts để phòng tránh lỗi do clock hoặc config;
- giới hạn mũ (exponential cap) để khoảng thời gian delay không bị overflow;
- full jitter hoặc equal jitter phù hợp với workload;
- kiểm tra interrupt hoặc cancellation trước và sau backoff;
- trả về outcome `CONTENDED` hoặc `TIMEOUT` ở trạng thái cuối cho hàm gọi (caller).

Không reset deadline sau mỗi attempt. Không log từng collision ở mức chi tiết cao; thay vào đó, hãy tổng hợp histogram của số lần attempt và đếm số lần sử dụng hết budget (exhausted count).

## Failure, side effect và cleanup

Mỗi attempt phải release mọi lock đã acquire bên trong khối `finally`. Business mutation chỉ chạy sau khi đã lấy đủ lock, nên attempt thất bại không cần rollback. Nếu một side effect bên ngoài đã chạy, cơ chế retry cần tính idempotent hoặc reconciliation và khi đó, nó không còn là một local lock loop đơn giản nữa.

`LockSupport.parkNanos` không ném ra ngoại lệ `InterruptedException`; mã nguồn phải tự kiểm tra `Thread.currentThread().isInterrupted()` và kết thúc, thường là giữ nguyên cờ interrupt (flag).

## Multi-instance và scope

Mỗi JVM có channel và lock riêng biệt; local retry không phối hợp authoritative state giữa các node. Các cơ chế database serialization hoặc deadlock retry có transaction rollback, phân loại lỗi (error classification) và cơ chế idempotency riêng được trình bày tại `LOCK-002` hoặc `DB-009`.

## Hậu quả và khả năng quan sát (observability)

Theo dõi tỷ lệ attempt trên completion, số lần exhausted retry, operation deadline, lock conflict, khoảng thời gian backoff, mức sử dụng CPU và runnable threads. Tín hiệu livelock điển hình: số lần retry tăng nhanh, số lần success gần như bằng 0, state version không thay đổi. Việc lấy thread dump theo thời gian sẽ hữu ích hơn so với một ảnh chụp (snapshot) đơn lẻ.
