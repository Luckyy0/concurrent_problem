# Giải Pháp Kiến Trúc Tối Ưu và Ma Trận Đánh Đổi Hiệu Suất

## 1. Cấu Trúc Hợp Nhất Giá Trị Theo Trật Tự Gốc (Value Composition In Input Order)

```java
List<CompletableFuture<Enrichment>> futures = inputs.stream()
    .map(input -> CompletableFuture.supplyAsync(
        () -> client.load(input), executor))
    .toList(); // Mỗi tác vụ độc lập chứa 1 Value, không dùng chung ArrayList

// Rào cản khép vòng mọi tiến trình
CompletableFuture<Void> all = CompletableFuture.allOf(
    futures.toArray(CompletableFuture[]::new));

// Điều phối viên đứng sau rào cản, đọc lại List Future tĩnh đã giữ sẵn thứ tự đầu vào
return all.thenApply(ignored -> futures.stream()
    .map(CompletableFuture::join)
    .toList());
```

**Cách làm chuẩn:** Cứ để mỗi luồng con tự lấy về data của nó. Chỉ có một anh Điều phối viên ở cuối mới được quyền tạo List. Vì mình đã lưu cái mảng `futures` theo đúng thứ tự lúc đầu rồi, nên kết quả cuối cùng chắc chắn sẽ đúng thứ tự, khỏi lo luồng nào xong trước xong sau.

## 2. Kỷ Luật Xử Lý Ngoại Lệ (Failure Policy)

- **Kiểu "Ăn cả ngã về không" (All-or-nothing):** Hễ hết giờ hay có lỗi là duyệt vòng lặp `futures.forEach(f -> f.cancel(true))` để dẹp hết. Cấm trả về cái mảng dở dang.
- **Kiểu "Sai đâu bỏ đó" (Per-item policy):** Gom từng cái vào một class rõ ràng, ví dụ `Outcome.Success` hoặc `Outcome.Failure`. Input vào bao nhiêu thì Output ra bấy nhiêu cái. Khuyên thật là đừng có xài chữ `null` để lừa hệ thống bảo là lỗi nhé.

## 3. Khống Chế Điểm Cổ Chai Tài Nguyên (Executor & Backpressure)

Nên cấu hình thread pool có giới hạn (Bounded pool) sao cho vừa sức với mấy dịch vụ bên ngoài thôi. Nhớ bắt và xử lý cái lỗi đầy pool (`RejectedExecutionException`). 
Đừng bao giờ chặn (block) luồng ngay trên một cái pool đã cạn sạch. Ở từng client thì tự set Timeout, còn phần tổng hợp thì phải có cái Deadline chung.

## 4. Ma Trận Đánh Đổi Thiết Kế (Trade-offs)

Làm theo kiểu hợp nhất giá trị (Value composition) thì code rõ ràng, dễ hiểu nhưng bù lại phải tốn RAM để giữ hết mảng Future cho đến khi xong xuôi.
Nếu chọn kiểu xử lý từng phần (Per-item outcome) thì biết rõ phần nào tạch phần nào qua, nhưng API nhìn sẽ cồng kềnh hơn.
Sài Concurrent queue (hàng đợi chạy đồng thời) cũng hay nếu bạn chỉ cần ai xong trước trả về trước. Nhưng nếu sếp yêu cầu phải giữ đúng thứ tự ban đầu thì cách này bỏ đi nha.

## 5. Giám Sát Môi Trường Khai Thác (Production Blueprint)

Để hệ thống chạy ngon ở môi trường thật, cần theo chuẩn sau:
- Output lúc trả về không được cho ai sửa nữa (Immutable).
- Lệnh Hủy (Cancel) phải ráng mà chạy cho bằng được (Best-effort).
- Nếu có luồng bị chặn, phải đảm bảo trả lại đúng trạng thái Interrupt.
- Bắt buộc phải có hệ thống đo lường (Metric) số lượng success, failure và cancel của các luồng con.
- Canh me sát sao thời gian phản hồi (Latency), thời gian chờ ở Hàng đợi (Queue wait), và cả mấy task chạy trễ quá giờ (Late completion).
