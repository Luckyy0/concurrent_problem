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

Nguyên tắc triển khai: Mỗi công nhân (Worker) hoàn trả Giá trị (Value); Độc tôn bộ Điều phối trung tâm (Terminal coordinator) mới sở hữu quyền khởi tạo List. Mảng Future duy trì cố định trật tự đầu vào (Input order), qua đó cấu trúc Output triệt tiêu hoàn toàn sự phụ thuộc vào Trật tự hoàn thành luồng (Completion order).

## 2. Kỷ Luật Xử Lý Ngoại Lệ (Failure Policy)

- **Mô Hình Giao Dịch Tuyệt Đối (All-or-nothing):** Tại thời điểm vượt ngưỡng Deadline hoặc phát sinh Ngoại lệ, kích hoạt lệnh quét `futures.forEach(f -> f.cancel(true))`. Cấm tuyệt đối hành vi hoàn trả mảng phân mảnh (Partial list).
- **Mô Hình Giao Dịch Phân Rã (Per-item policy):** Ánh xạ (Map) từng Tác vụ con thành một kết quả Đóng gói (Sealed class) rõ ràng: `Outcome.Success` hoặc `Outcome.Failure`. Bảo chứng quy mô mảng Output (Cardinality) tương đương Input. Tuyệt đối không dùng cờ `null` để đánh tráo khái niệm Lỗi.

## 3. Khống Chế Điểm Cổ Chai Tài Nguyên (Executor & Backpressure)

Thiết lập cơ chế Quản trị Pool (Bounded/Admission-controlled executor) tương thích với sức tải của các liên kết Ngoại vi (Blocking dependency); Triển khai bắt và xử lý ngoại lệ cạn tài nguyên `RejectedExecutionException`. 
Nghiêm cấm hành vi Chặn lồng nhau (Nested-block) ngay trên chính một Pool đã cạn kiệt (Exhausted pool). Khối Client bắt buộc quy định Timeout cục bộ; Khối Aggregate quản trị Deadline Toàn cục.

## 4. Ma Trận Đánh Đổi Thiết Kế (Trade-offs)

Cơ chế Hợp nhất giá trị (Value composition) đơn giản hóa quá trình chứng minh logic hệ thống, tuy nhiên đòi hỏi bộ nhớ duy trì khối Future/Value cho tới điểm cuối của chu trình Lô. 
Mô hình Giao dịch phân rã (Per-item outcome) minh bạch hóa toàn bộ ngữ nghĩa rác (Partial semantics), song khiến giao diện API trở nên nặng nề hơn. 
Cấu trúc Hàng Đợi Đồng Thời (Concurrent queue) là ứng cử viên sáng giá cho hệ quy chiếu Stream theo trình tự hoàn thành (Completion-order), nhưng hoàn toàn lạc quẻ nếu áp đặt vào yêu cầu phản hồi Nguyên tử, Đa trật tự (Ordered atomic response).

## 5. Giám Sát Môi Trường Khai Thác (Production Blueprint)

Đặc tả vận hành:
- Output bắt buộc ở định dạng Bất Biến (Immutable).
- Tín hiệu Hủy (Cancel) thi hành trên mức Tối đa khả năng (Best-effort).
- Khôi phục nguyên trạng Trạng thái Ngắt (Restore interrupt) tại các ranh giới Bị chặn (Blocking boundary).
- Đo lường (Metric) chi tiết các tín hiệu Child success/failure/cancel.
- Giám sát gắt gao độ trễ Khối Aggregate (Latency), thời gian chờ ở Hàng đợi (Queue wait), và độ rò rỉ tác vụ trễ sau ngưỡng Timeout (Late completion).
