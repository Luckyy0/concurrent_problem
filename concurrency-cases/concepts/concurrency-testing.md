# Kiểm thử đồng thời (Concurrency Testing)

## Mục tiêu

Kiểm thử đồng thời không chỉ là việc bạn tạo ra nhiều luồng (thread) chạy lung tung rồi hy vọng nó lỗi. Một bài test chuẩn phải điều khiển được các luồng chạy xen kẽ (đan xen) nhau có ý đồ, và cuối cùng phải kiểm tra xem các quy tắc nghiệp vụ (business rules) có còn đúng sau khi mọi luồng đã chạy xong hay không.

Khi làm hệ thống nhiều luồng, bạn cần phân biệt rõ hai nhóm mục tiêu sau:
- **Tính an toàn (Safety):** Chắc chắn rằng những điều sai trái KHÔNG BAO GIỜ được phép xảy ra. Ví dụ: không được tạo trùng ID, không được để số dư tài khoản bị âm, hay không được xếp hai người vào cùng một ghế máy bay.
- **Khả năng tiếp tục chạy (Liveness/Progress):** Hệ thống phải có khả năng nhích tới phía trước, xử lý xong công việc. Nó không được phép bị đứng hình (deadlock), không được bỏ đói luồng (starvation), hoặc thử lại (retry) không điểm dừng.

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| `interleaving` (chạy đan xen) | Tình huống các câu lệnh của nhiều luồng khác nhau chạy xen kẽ, nhào lộn vào nhau chứ không chạy tuần tự từng luồng một. |
| `deterministic test` (test có kịch bản) | Test chủ động ép các luồng dừng/chạy đúng theo một kịch bản định sẵn để chắc chắn bắt được lỗi. |
| `stress test` (test nhồi tải) | Test xả một lượng lớn công việc cùng lúc để dò xem có vô tình dính lỗi đồng thời không. |
| `safety` (an toàn) | Quy tắc bảo vệ dữ liệu (sai sót không được phép xảy ra). |
| `liveness` (sống còn) | Đảm bảo hệ thống vẫn tiếp tục chạy được, không bị kẹt. |
| `barrier` (rào chắn/điểm hẹn) | Một điểm mốc trong code mà các luồng tới đó phải đứng đợi nhau, khi nào đủ mặt mới được đi tiếp. |
| `test invariant` (bất biến) | Quy tắc hoặc trạng thái cuối cùng bắt buộc phải luôn đúng sau khi bài test kết thúc. |

## Phải điều khiển luồng một cách khoa học, đừng dùng `Thread.sleep()` đoán mò

Tuyệt đối không dùng `Thread.sleep(...)` để canh thời gian cho các luồng chạy. Tốc độ máy tính của bạn, máy server (CI) hoàn toàn khác nhau. Dùng sleep vừa làm test chạy chậm, vừa sinh ra những lỗi lúc đậu lúc rớt (flaky test).

Hãy ưu tiên dùng các bộ công cụ xịn của Java:
- `CountDownLatch`: Dùng làm "còi báo danh" hoặc "súng lệnh xuất phát" để tất cả cùng chạy.
- `CyclicBarrier` hoặc `Phaser`: Bắt các luồng dừng lại đúng một dòng code để đợi nhau.
- `Future.get(timeout)`: Giới hạn thời gian chờ để không bị treo test mãi mãi.
- `ExecutorService`: Quản lý một nhóm luồng làm việc (worker pool).
- Tự viết các "móc" (hook) dành riêng cho test để ép luồng dừng lại chính xác ngay giữa bước đọc (SELECT) và bước ghi (UPDATE).

Mẫu code điển hình để tạo 32 luồng cùng xuất phát một lúc:

```java
int actors = 32; // Số luồng giả lập
var ready = new CountDownLatch(actors); // Đếm ngược chờ tất cả vào vị trí
var start = new CountDownLatch(1);      // Súng lệnh xuất phát
var done = new CountDownLatch(actors);  // Đếm ngược xem ai đã chạy xong
var executor = Executors.newFixedThreadPool(actors);

try {
    for (int i = 0; i < actors; i++) {
        executor.submit(() -> {
            ready.countDown(); // Báo cáo: "Tôi đã sẵn sàng!"
            try {
                start.await(); // Đợi tiếng súng xuất phát...
                
                // >>> THỰC THI CODE NGHIỆP VỤ CỦA BẠN Ở ĐÂY <<<
                
            } catch (Exception e) {
                // Xử lý lỗi (nếu cần)
            } finally {
                done.countDown(); // Báo cáo: "Tôi đã chạy xong!"
            }
            return null;
        });
    }

    assertTrue(ready.await(5, TimeUnit.SECONDS)); // Chờ max 5s để mọi người vào vị trí
    start.countDown(); // BẮN SÚNG LỆNH CHO TẤT CẢ CÙNG CHẠY!
    assertTrue(done.await(10, TimeUnit.SECONDS)); // Chờ max 10s để mọi người hoàn thành
} finally {
    executor.shutdownNow(); // Nhớ dọn dẹp tắt pool
}
```

**Lưu ý:** Mọi hàm chờ (await/get) bắt buộc phải có thời gian giới hạn (timeout), phòng trường hợp code bạn bị kẹt (deadlock) thì bài test cũng sẽ tự ngắt chứ không treo mãi mãi.

## Hai cách kiểm thử cần biết

### 1. Tái hiện lỗi có kịch bản (Deterministic Test)

Bạn can thiệp để ép hệ thống đi theo kịch bản này:
```text
Luồng 1 (T1) đọc dữ liệu từ DB (kho còn 1 món)
Luồng 2 (T2) cũng đọc dữ liệu (kho cũng thấy còn 1 món)
-- Bắt hai luồng đợi nhau ở đây --
Cho phép cả hai cùng trừ kho và lưu lại!
T1 lưu thành công.
T2 cũng lách luật lưu thành công (và kho bị âm).
```
Cách này chỉ ra rõ ràng "kẻ hở" nằm ở đâu. Đây nên là bài test chính thức để bảo vệ code của bạn. (Lưu ý: Các rào chắn chờ đợi này chỉ được nằm trong code Test, tuyệt đối không được mang vào code chạy thật Production).

### 2. Kiểm thử nhồi tải (Stress Test)

Bạn quăng lượng lớn yêu cầu cùng lúc xem hệ thống có lỗi không. Nó rất tốt để test code thực tế, nhưng nhớ rằng: Chạy pass 1 lần không có nghĩa là code bạn an toàn 100% (có thể lần đó xui chưa kẹt).
Một bài test tải tốt cần:
- Cuối cùng vẫn phải chốt lại dữ liệu có chuẩn không (tiền không được âm).
- Lưu lại cấu hình chạy nếu có yếu tố ngẫu nhiên (randomness).
- Không được viết kiểu "lặp vô cực đến khi nào ra lỗi" để che giấu việc test chạy không ổn định.
- Hãy dùng chung với kiểu test kịch bản, chứ đừng dùng thay thế.

> **Nói ngắn gọn:** Test kịch bản (Deterministic) giúp giải thích vì sao lỗi; Test tải (Stress) giúp khẳng định trên thực tế môi trường như thật có bị dính lỗi đó không.

## Kiểm tra kết quả (Quy tắc nghiệp vụ)

Đừng chỉ kiểm tra xem code có ném ra cái lỗi (`Exception`) nào không là xong (`assertThrows`). Ví dụ với bài toán tạo ID, bạn phải kiểm tra:

```java
// Tổng số kết quả trả về phải bằng số lượng luồng yêu cầu
assertEquals(requestCount, results.size());

// Số lượng ID DUY NHẤT sinh ra phải đúng bằng số yêu cầu (không bị trùng ID)
assertEquals(
        requestCount,
        results.stream().map(Result::id).distinct().count()
);

// Trạng thái các ID có hợp lệ hay không
assertTrue(results.stream().allMatch(r -> r.customerId().equals(r.inputId())));
```

Nếu một luồng bị rớt vì lý do đồng thời (conflict), bạn phải đo lường:
- Có bao nhiêu luồng thành công?
- Có bao nhiêu luồng bị chặn lại / phải thử lại?
- Trạng thái dòng dữ liệu sau cùng là gì?
- Có dữ liệu rác (chạy dở dang) nào bị sinh ra không?
- Lỗi kỹ thuật của hệ thống có được chuyển đổi báo ra thành lỗi nghiệp vụ đàng hoàng cho người dùng hiểu không?

## Khi nào cần dùng Database xịn (Real dependency)

Nếu bạn chỉ test code logic thuần trong RAM (Java heap), JUnit thường là đủ.
Nhưng nếu bạn test các tính năng dính tới Cơ sở dữ liệu thì PHẢI ưu tiên dùng PostgreSQL thật (thông qua Testcontainers). Ví dụ:
- Cách ly giao dịch (Isolation level / MVCC).
- Khóa dòng, khóa bảng (Row lock / Table lock).
- Câu lệnh khóa bi quan (`FOR UPDATE`, `SKIP LOCKED`).
- Cách phát hiện hệ thống kẹt khóa (Deadlock detection).
- Hibernate báo lỗi văng khóa lạc quan (`@Version`).
- Bị hủy giao dịch vì mức cách ly quá ngặt nghèo (`SERIALIZABLE`).

**Tuyệt đối không dùng cơ sở dữ liệu ảo H2 để suy diễn hành vi của PostgreSQL!**

Tương tự, với Kafka hay Redis, bạn phải chú thích rõ bài test này đang test tính năng gì: tự động gửi lại tin (redelivery), xử lý tuần tự (partition ordering), cơ chế khóa (Lua atomicity) hay vòng đời bộ nhớ (TTL, expiry)...

## Kinh nghiệm dọn dẹp và thu thập lỗi

- Luôn nhớ đóng `ExecutorService` trong khối `finally`.
- Khi hứng lỗi đứt ngắt (`InterruptedException`), hãy nhớ phục hồi cờ báo ngắt (`Thread.currentThread().interrupt()`).
- Luôn đặt tên rõ ràng cho các luồng (Thread name) để khi xảy ra chuyện, mở nhật ký (thread dump) ra dễ tìm bằng chứng hơn.
- Khi bị báo lỗi TimeOut (hết giờ chờ), hãy tìm cách lấy `thread dump`, xem trạng thái chờ khóa (lock wait) hoặc Database activity chứ đừng chỉ đối phó bằng cách tăng số giây Timeout lên.
- Phân biệt thật rõ: Lỗi hạ tầng kết nối (infrastructure) khác hoàn toàn với Lỗi do vi phạm quy tắc nghiệp vụ.

## Liên kết tài liệu tham khảo

- Nền tảng JMM: [Java Memory Model và tính nguyên tử](java-memory-model-and-atomicity.md)
- Case áp dụng đầu tiên: [JVM-001](../jvm/spring-singleton-mutable-state/experiments.md)
- Atomic-claim database experiments: [DB-006](../postgresql/unique-constraint-concurrency/experiments.md)
- Pessimistic row-lock experiments: [LOCK-003](../locking/pessimistic-write-for-update/experiments.md)
- Conditional atomic-update experiments: [LOCK-004](../locking/conditional-atomic-update/experiments.md)
- Các case database trong [catalog](../CONCURRENCY_CASE_LIBRARY.md) sẽ mở rộng
- Work-claiming database experiments: [DB-010](../postgresql/skip-locked-work-queue/experiments.md) pattern này bằng PostgreSQL Testcontainers.
