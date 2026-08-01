# Giải Pháp Kiến Trúc Tối Ưu và Chiến Lược Hoạch Định Sức Chứa

## 1. Phương Án 1: Xóa Sổ Cơ Chế Đệ Trình Lồng Nhau (Drop Nested Submission)

Nếu Tác vụ Cha đã yên vị trên Executor, hãy thẳng tay thực thi lệnh gọi Cấu trúc phụ thuộc (Dependency) ngay trên chính Luồng Công Nhân đó:

```java
public EnrichedOrder enrich(Order order) {
    Price price = pricingClient.loadPrice(order.productId()); // Gọi trực tiếp đồng bộ
    return new EnrichedOrder(order.orderId(), price);
}
```

Bứt rễ hoàn toàn hệ sinh thái Hàng Đợi/Future của Tác vụ Con, triệt tiêu tận gốc mầm mống Nạn Đói Nội Bộ (Self-starvation). Điểm lưu ý: Client Viễn trình (Remote client) bắt buộc tự trang bị Timeout; Sức chứa đồng thời của Executor phải thấp hơn/bằng Tổng Ngân Sách Tài Nguyên (Resource budget) phân bổ dưới tầng sâu.

## 2. Phương Án 2: Bộ Điều Phối Kích Hoạt Tác Vụ Lá (Orchestrator Submits Leaf Tasks)

Yêu cầu/Bộ Điều Phối đứng chênh vênh bên ngoài Worker Pool, trực tiếp nã đạn (submit) các Tác vụ Lá (Leaf tasks), neo mình tại Hạn mức thời gian (Deadline chung), và cấm tiệt hành vi nhúng vỏ bọc Tác vụ Cha vào chung một Pool. Khi hệ thống đổ vỡ/Quá hạn, kích hoạt Hủy mọi Future còn tồn đọng, khôi phục Cờ Ngắt (Interrupt) và chặn đứng hành vi phát tán Mảnh dữ liệu rác (Partial result). Đi sâu cấu trúc Hợp nhất tại chuyên đề `JVM-009`.

## 3. Phương Án 3: Phân Ly Executor Theo Khối Phụ Thuộc (Separate Pools)

Tách bạch Pool của Cha và Con chém đứt vòng lặp tự sinh tự diệt, song đòi hỏi kỹ nghệ Hoạch định cấu trúc (Sizing) song song bao hàm cả Hàng đợi Giới hạn và Trạm kiểm soát Đầu vào. Khối lượng công việc đồng thời của Cha KHÔNG được phép thổi bùng Nhu cầu (Demand) của Tác vụ Con vượt khỏi Ngưỡng Sức Chứa của Tầng Con/Viễn Trình; Nhu nhược ở khâu này chỉ là việc dời Nạn Đói (Starvation) sang thảm họa Quá tải Hàng đợi (Queueing overload).

## 4. Định Chuẩn Khai Thác ThreadPoolExecutor (Production Baseline)

```java
new ThreadPoolExecutor(
        coreSize,
        maxSize,
        30, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(queueCapacity),
        threadFactory,
        new ThreadPoolExecutor.AbortPolicy() // Rắn rỏi Khước từ
);
```

Thiết lập lưới đánh chặn `RejectedExecutionException` ngay tại Cửa Ngõ (Admission boundary) và tái cấu trúc nó thành Phản hồi Quá Tải (Overload response); Tuyệt đối cấm hành vi Nuốt Lỗi (Swallow). Chính sách `CallerRunsPolicy` chỉ phát huy hiệu quả khi hệ Caller tự giảm tốc độ (Slowdown) và hệ lụy Bám Luồng (Thread-affinity side effect) đã được bảo chứng chấp thuận.

## 5. Kỷ Luật Hạn Mức Và Khống Chế Hủy Bỏ (Deadline & Cancellation)

Khai thác Hạn mức chung, lệnh `future.get(remaining, unit)`, lệnh `cancel(true)` khi sụp đổ và Khôi phục Cờ Ngắt. Hệ Client Viễn Trình/JDBC luôn cần một rào cản Timeout độc quyền; Cờ Interrupt chưa bao giờ là một Bảo chứng Hủy Mạng lưới (Network cancellation guarantee) đáng tin cậy.

## 6. Tranh Luận Về Luồng Ảo (Virtual Threads)

Công cụ thần thánh đối phó I/O Tắc Nghẽn (Blocking I/O), biến hóa Thread Dump/Mô hình Tác vụ thành tuyệt tác tinh giản. NHƯNG, bắt buộc giăng rào Semaphore/Giới hạn Đầu vào bám sát Hạn Mức Tải Kết Nối/Viễn Trình (Connection/Remote quota). Khước từ tư duy lạm dụng Đội Quân Luồng Ảo Vô Hạn làm phao cứu sinh cho Hoạch Định Sức Chứa (Capacity policy).

## 7. Ma Trận Đánh Đổi Thiết Kế (Trade-offs)

| Giải Pháp Kỹ Thuật | Đặc Tính Tiến Trình (Progress) | Áp Lực Quá Tải (Overload) | Độ Phức Tạp |
| --- | --- | --- | --- |
| Truy Vấn Trực Tiếp (Direct Call) | Tiêu diệt Tận Gốc Self-starvation | Bảo vệ bằng Bounded Executor / Cổng Kiểm Soát | Thấp |
| Điều Phối Tác Vụ Lá (Leaf Tasks) | Cha từ bỏ Quyền Giam Luồng | Ràng buộc bởi Deadline / Cờ Hủy | Vừa Phải |
| Phân Ly Pool (Separate Pools) | Xé rách Vòng Lặp Chung-Pool | Đòi hỏi Tính Toán Sức Chứa Đồng Bộ | Vừa-Cao |
| Luồng Ảo + Cấp Phép (Permits) | Chịu Tải Tác vụ Đóng Băng Khổng Lồ | Cấp Phép che chắn Khối Phụ Thuộc | Vừa Phải |
| Bơm Mù Hàng Đợi/Pool | Phi Chứng Minh | Trì hoãn Thời Khắc Tử Thần | Thấp nhưng Khuyết Tật |

## 8. Khung Giám Sát Khai Thác (Production Checklist)

Quy định cấu hình: Hàng đợi Giới hạn; Khước từ Rắn rỏi; Hạn mức Chiến dịch (Operation deadline); Hạn mức Tầng Thấp (Downstream timeout); Truyền dẫn Cờ Hủy (Cancel propagation); Tuyên chiến Vòng lặp chờ Chung-Pool; Hạn mức Đóng Cửa (Graceful shutdown deadline); Triển khai Hệ thống Đo lường (Metrics) Độ trễ Hàng Đợi / Khước từ / Tiến Trình; Áp dụng Bài Cào Tải (Load test) bức tử Cổng Kiểm Soát.
