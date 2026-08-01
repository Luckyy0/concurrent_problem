# Phản Mẫu Thiết Kế (Anti-Patterns): Bẫy Chờ Đợi Phân Cấp

## 1. Tác Vụ Cha Chờ Đợi Tác Vụ Con Trên Chung Một Executor

Đoạn mã bộc lộ rõ tư duy thiết kế luồng sai lệch:

```java
@Service
public class BrokenEnrichmentService {
    private final ExecutorService enrichmentExecutor;
    private final PricingClient pricingClient;

    public EnrichedOrder enrich(Order order) {
        // LỖI CHÍNH: Tác vụ Cha nhồi Tác vụ Con vào chính hệ thống Executor đang vận hành nó
        Future<Price> child = enrichmentExecutor.submit(
                () -> pricingClient.loadPrice(order.productId()));
        try {
            // Đóng băng Luồng Công Nhân để chờ kết quả từ hàng đợi
            Price price = child.get(); 
            return new EnrichedOrder(order.orderId(), price);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new EnrichmentException(e);
        } catch (ExecutionException e) {
            throw new EnrichmentException(e.getCause());
        }
    }
}
```

Và tại thượng tầng (Caller), yêu cầu tiếp tục bị đóng gói đệ trình:

```java
// Tác vụ Cha chiếm dụng Luồng Công Nhân đầu tiên
Future<EnrichedOrder> result = enrichmentExecutor.submit(
        () -> enrichmentService.enrich(order));
```

Khảo sát cấu hình Cấu trúc Pool/Hàng đợi phổ biến:

```java
new ThreadPoolExecutor(
        2, 2, 0, TimeUnit.MILLISECONDS, // Chỉ 2 Luồng Công Nhân
        new ArrayBlockingQueue<>(100), // Hàng đợi chứa tới 100 tác vụ
        new ThreadPoolExecutor.AbortPolicy()
);
```

Kịch bản sụp đổ: Hai Tác vụ Cha lao vào chiếm giữ hoàn toàn hai Luồng Công Nhân. Cả hai nhả hai Tác vụ Con vào Hàng đợi. Lập tức, cả hai Tác vụ Cha tự phong tỏa bằng lệnh `get()` vô thời hạn. Sức chứa khổng lồ 100 của Hàng đợi chỉ làm chậm thời điểm phát lệnh Từ chối (Rejection); Nó tuyệt đối không có năng lực tự sinh ra Luồng Công Nhân mới để cứu vớt các Tác vụ Con.

> **Nguyên tắc kỹ thuật:** Hàng đợi có giới hạn (Bounded queue) là lá chắn bảo vệ Bộ nhớ (Memory), nhưng nó hoàn toàn vô hại đối với một Đồ thị Phụ thuộc (Dependency graph) khuyết tật, nơi mà một tác vụ đang chạy lại treo sinh mạng vào một tác vụ bị xếp vế sau chính nó.

## 2. Các Phương Án Sửa Lỗi Tạm Bợ Cần Tránh (Insufficient Fixes)

- **Tăng Quy Mô Pool/Queue Tùy Hứng:** Dưới áp lực tải cao, hệ thống rốt cuộc cũng sẽ va phải bức tường bế tắc cũ, chỉ là tốn thêm tài nguyên vô ích.
- **Xóa Bỏ Giới Hạn Hàng Đợi (Unbounded Queue):** Che giấu triệu chứng Từ chối, đổi lấy thảm họa Phình to Độ trễ (Latency) và Sập Bộ nhớ (Memory).
- **Trang Bị Hạn Mức Cho Khối Chờ (`get(timeout)`):** Cô lập sự cố tốt hơn, song vẫn phung phí Luồng Công Nhân và sinh ra chuỗi biến động Timeout/Retry hỗn loạn (Churn).
- **Áp Dụng Chính Sách `CallerRunsPolicy` Mù Quáng:** Có thể ngẫu nhiên ép Tác vụ Con chạy nối tiếp (Inline) và phá vỡ cấu trúc bế tắc trong một vài nhịp độ (Timing) may mắn, nhưng gây nhiễu loạn hoàn toàn Độ trễ và Đặc tính bám luồng (Thread-affinity); Đây không phải là một chiến lược thiết kế nền tảng (Design proof).
- **Lạm Dụng Luồng Ảo (Virtual Thread) Không Kiểm Soát:** Nên nhớ rằng, bất chấp số lượng Luồng Ảo, các cấu trúc Tương tác Ngoại vi (External dependency) vẫn bị trói buộc bởi Hạn Mức Tải (Quota).
- **Tách Cấu Trúc Pool Nhưng Thiếu Giảm Áp (Backpressure):** Nếu cho phép Pool của Tác vụ Cha phình to hơn năng lực xử lý của Tác vụ Con mà không có cơ chế hãm phanh, Hàng đợi của Tác vụ Con sẽ nhanh chóng rơi vào trạng thái Ách Tắc (Bão hòa).

## 3. Tiền Đề Kích Hoạt Lỗ Hổng (Conditions for Replication)

1. Số lượng Tác vụ Cha đang thi hành đạt tới giới hạn tổng số Luồng Công Nhân.
2. Mỗi Tác vụ Cha tự tay ném ít nhất một Tác vụ Con vào chính Executor đang giam giữ nó.
3. Tác vụ Cha tự khóa bản thân chờ Tác vụ Con.
4. Không có bất kỳ Luồng Công Nhân nào thoát vòng lặp, hoặc không có cơ chế Timeout/Cancel can thiệp.
