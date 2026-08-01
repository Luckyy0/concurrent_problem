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

Chu trình (cycle) không tự biến mất vì không thread nào release lock đầu tiên trước khi lấy được lock thứ hai.

> **Nói ngắn gọn:** mỗi thread giữ thứ tác nhân kia cần và chỉ release sau khi nhận được thứ chính nó đang chờ.

## Expected và actual

| Khía cạnh | Mong đợi | Thực tế |
| --- | --- | --- |
| Progress | Hai thao tác chuyển hoàn tất tuần tự | Cả hai đứng vô hạn |
| Balance | Tổng được bảo toàn sau thao tác | Trạng thái chưa đổi nhưng resource và thread bị giữ |
| Failure | Deadline trả lỗi và có dọn dẹp (cleanup) | `lock()` và monitor không có deadline |
| Diagnostics | Xung đột tạo ra exception | JVM không tự chọn nạn nhân cho ứng dụng |

## Nguyên nhân theo từng lớp

Bốn điều kiện cùng tồn tại: mutual exclusion, hold-and-wait, no preemption và circular wait. Code lỗi trực tiếp tạo circular wait vì thứ tự phụ thuộc hướng chuyển.

`ReentrantLock` và monitor tạo happens-before khi cùng lock được release và acquire, nhưng đảm bảo đó không giúp tiến triển (progress) khi chu trình không có release. Spring transaction không tham gia; đây là trạng thái trên heap. Database deadlock có bộ phát hiện (detector) và cơ chế rollback nạn nhân khác hẳn tình huống này.

## Định hướng tất định (Deterministic ordering) phá cycle

Định nghĩa total order theo `accountId` duy nhất và ổn định. Cả A→B và B→A đều acquire ID nhỏ trước, ID lớn sau. Nếu T1 giữ lock đầu tiên, T2 chưa thể giữ lock thứ hai để tạo chiều ngược của chu trình; circular wait bị loại bỏ.

Không sắp xếp thứ tự bằng vai trò source hay destination. Nếu bộ so sánh (comparator) có thể trả bằng nhau cho hai tài nguyên khác nhau, cần tiêu chí phân định duy nhất (tie-breaker) hoặc một tie lock.

## Giới hạn thời gian acquire và thử lại

`tryLock` hoặc `lockInterruptibly` giới hạn việc chờ và cho phép dọn dẹp, nhưng không phải là bằng chứng không có deadlock. Nếu acquire lock thứ hai thất bại, hãy release lock thứ nhất trong `finally`, rồi thử lại (retry) ngoài critical section với deadline, giới hạn số lần thử (attempt cap) và độ trễ ngẫu nhiên (jitter).

Không thay đổi (mutate) balance hoặc gọi external side effect trước khi có đủ mọi lock. Nếu một lượt thử đã tạo side effect, việc thử lại cần tính lũy đẳng (idempotency) hoặc đối soát (reconciliation).

## Lỗi, gián đoạn và sự cố hệ thống

- Khôi phục trạng thái ngắt (interrupt status) khi chuyển `InterruptedException` thành lỗi nghiệp vụ (domain error).
- Chỉ unlock những lock đã acquire và unlock theo thứ tự ngược.
- Việc xác thực (validation) diễn ra sau khi có đủ lock nếu nó phụ thuộc vào balance có thể thay đổi.
- Runtime exception vẫn phải release cả hai lock qua `finally`.
- Sự cố tiến trình (Process crash) phá local deadlock nhưng mất thao tác trên bộ nhớ; không có dữ liệu lưu trữ bền vững để rollback hay thử lại.

## Multi-instance và biên giới PostgreSQL

Mỗi node có account object và lock riêng, nên thứ tự local không bảo vệ các database row được chia sẻ. PostgreSQL có thể lock thêm index hoặc row theo execution path và phát hiện chu trình riêng. `DB-008` phải xử lý SQLSTATE, rollback nạn nhân và thử lại transaction.

## Hậu quả

- Độ trễ yêu cầu (request latency) không giới hạn, thread và executor bị cạn kiệt dây chuyền;
- Connection và request context bị giữ;
- Bão thử lại (retry storm) từ upstream;
- Kiểm tra sức khỏe (health check) không chạm tới lock vẫn báo khỏe (healthy);
- Khởi động lại tạm giải phóng nhưng không sửa thứ tự lock.

## Quan sát

Thu thập thread dump và gọi `ThreadMXBean.findDeadlockedThreads`; log account ID theo thứ tự chuẩn, thời gian chờ lock và thời hạn thao tác. Cảnh báo (alert) khi detector trả về thread ID hoặc thời gian chờ acquire tăng. Nội dung chung ở [Deadlock và thử lại an toàn](../../concepts/deadlocks-and-retries.md).
