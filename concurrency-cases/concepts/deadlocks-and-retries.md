# Bế tắc (Deadlock) và cách thử lại (Retry) an toàn

## Mục tiêu

Tài liệu này sẽ giúp bạn hiểu rõ các từ vựng chung khi nói về cơ chế khóa (lock) trong Java, khóa trong cơ sở dữ liệu (Database) và cách để "thử lại" (retry) một cách an toàn khi có lỗi xảy ra. 

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Bế tắc (`deadlock`) | Tình trạng hai hay nhiều luồng (thread) hoặc giao dịch (transaction) đang ôm khư khư tài nguyên của mình và đứng chờ tài nguyên của đối phương. Kết quả là tạo thành một vòng luẩn quẩn không ai nhúc nhích được. |
| Đồ thị chờ (`wait-for graph`) | Một sơ đồ để vẽ ra xem luồng nào đang chờ tài nguyên của luồng nào. |
| Chu trình chờ (`wait-for cycle`) | Khi nhìn vào sơ đồ trên mà thấy một vòng tròn khép kín (A chờ B, B chờ C, C lại chờ A) thì đó chính là dấu hiệu chắc chắn của Deadlock. |
| Thứ tự khóa thống nhất (`deterministic lock ordering`) | Một nguyên tắc vàng: Ép tất cả các luồng phải lấy khóa (lock) theo đúng một thứ tự giống hệt nhau (ví dụ: luôn khóa dòng có ID nhỏ trước, ID lớn sau). |
| Nạn nhân (`victim`) | Luồng hoặc giao dịch bị hệ thống tự động "giết" (hủy bỏ) để phá vỡ vòng luẩn quẩn, cứu các luồng khác. |
| Giới hạn thời gian chờ (`bounded acquisition`) | Đặt thời gian tối đa (timeout) khi đứng chờ khóa, thay vì chờ đợi vô tận. |
| Phạm vi thử lại (`retry boundary`) | Xác định rõ đoạn code nào sẽ được phép chạy lại từ đầu nếu bị lỗi xung đột. |

## Bốn yếu tố tạo nên Deadlock

Để một deadlock kinh điển xảy ra, phải hội đủ 4 yếu tố:
1. **Độc quyền:** Một tài nguyên chỉ cho một người dùng.
2. **Giữ và chờ:** Đang cầm tài nguyên này nhưng lại đứng chờ tài nguyên khác.
3. **Không thể tước đoạt:** Không ai có quyền giật tài nguyên khỏi tay người đang giữ nó.
4. **Chờ đợi vòng tròn:** Tạo thành một vòng khép kín.

**Cách giải quyết dễ nhất:** Phá vỡ yếu tố thứ 4 (chờ đợi vòng tròn) bằng cách bắt mọi người lấy khóa theo **cùng một thứ tự**.
Lưu ý: Việc đặt thời gian chờ (timeout) không giải quyết tận gốc nguyên nhân gây deadlock. Nó chỉ giúp luồng không bị kẹt mãi mãi, nếu luồng bị timeout, nó phải chủ động nhả các khóa đang giữ ra. Nếu bạn code cho nó "thử lại" (retry) ngay lập tức mà không chịu nhường nhịn hay lùi thời gian lại (backoff), hệ thống của bạn có thể chuyển từ Deadlock sang Livelock (cứ liên tục đụng nhau rồi cùng lùi lại mãi mãi không thể hoàn thành).

## Điểm khác biệt giữa Java (JVM) và PostgreSQL

- **Trong Java (JVM):** Nếu dùng từ khóa khóa cơ bản (`synchronized`), Java rất "ngốc". Nó không biết tự tìm "nạn nhân" để hủy. Nếu dính deadlock, các luồng đó sẽ đứng chết trân mãi mãi cho đến khi bạn khởi động lại (restart) ứng dụng. Bắt buộc phải dùng `ReentrantLock.tryLock()` để tự cài thời gian chờ (timeout) để tự thoát ra.
- **Trong PostgreSQL:** Database rất thông minh. Nó có bộ phận dò tìm deadlock. Nếu thấy vòng luẩn quẩn, nó sẽ tự chọn một giao dịch làm "nạn nhân", hủy giao dịch đó (báo lỗi) để giải cứu các giao dịch còn lại. 

> **Nói ngắn gọn:** Cả hai đều có thể bị vòng luẩn quẩn chờ đợi. Nhưng Java thì treo luôn không biết làm gì, còn Database thì sẽ báo lỗi và giết một luồng. Do đó, code Java của bạn phải hứng lỗi này, hoàn tác dữ liệu (rollback) và tự quyết định xem có nên thử lại (retry) hay không.

## 6 quy tắc vàng để phòng ngừa Deadlock

1. **Giữ ít khóa nhất có thể:** Càng ít khóa cùng lúc, càng ít nguy cơ dính chùm.
2. **Khóa theo thứ tự:** Sắp xếp các khóa theo một thứ tự chuẩn (ví dụ: luôn khóa bảng User trước, rồi mới khóa bảng Order) và mọi luồng phải tuân theo thứ tự đó.
3. **Làm nhanh rút gọn:** Khối code nằm trong vùng khóa phải chạy thật nhanh. Tuyệt đối tránh gọi API bên ngoài (remote I/O) khi đang giữ khóa nếu có thể.
4. **Luôn có Timeout:** Dùng các loại khóa có hỗ trợ thời gian chờ (timeout/interruptible).
5. **Mở khóa ngược chiều:** Khóa cái nào trước thì mở (release) cái đó sau cùng trong khối `finally`.
6. **Không thử lại mù quáng:** Giới hạn số lần retry, có thời gian chờ (deadline), và mỗi lần thử lại nên trễ đi một chút ngẫu nhiên (jitter backoff).

## Thử lại (Retry) an toàn là như thế nào?

- Chỉ được thử lại sau khi đã đảm bảo mọi thứ (trạng thái code, giao dịch DB) của lần thử trước đã được dọn dẹp sạch sẽ (rollback hoàn toàn).
- Nếu trong lúc xử lý bạn có gọi qua các hệ thống bên ngoài (ví dụ gửi Email, gọi API thanh toán), bạn phải có cơ chế kiểm tra chống trùng lặp (idempotency key) trước khi thử lại để tránh gửi Email 2 lần.
- **Phân biệt rõ ràng:** Lỗi do tranh chấp, deadlock thì CÓ THỂ thử lại. Còn lỗi do logic nghiệp vụ (ví dụ: dữ liệu không hợp lệ) thì TUYỆT ĐỐI KHÔNG tự động thử lại.

## Dấu hiệu nhận biết và chẩn đoán (Monitoring)

- **Trong Java:** Dùng lệnh lấy luồng (Thread dump), tìm kiếm bằng JMX `ThreadMXBean.findDeadlockedThreads`, xem ai đang giữ khóa nào và luồng đang chạy tới dòng code nào.
- **Trong PostgreSQL:** Xem mã lỗi SQL (SQLSTATE), đọc log deadlock của database, xem mã tiến trình (PID) nào đang bị chặn.
- **Chỉ số cần theo dõi (Metric):** Số lần bị timeout chờ khóa, số lần bị báo deadlock, số lần phải thử lại (retry attempt) và kết quả của những lần thử đó.

## Liên kết tài liệu tham khảo

- [JVM-007 — Opposite Lock Ordering Deadlock](../jvm/opposite-lock-order-deadlock/README.md)
- [DB-008 — PostgreSQL opposite row order deadlock](../postgresql/opposite-row-order-deadlock/README.md)
- [DB-009 — Serializable abort and safe retry](../postgresql/serializable-retry/README.md)
- [SPR-006 — Retry transaction boundary](../spring/retry-transaction-boundary/README.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
