# Phản Mẫu Thiết Kế (Anti-Patterns): Bẫy Chờ Đợi Phân Cấp

## 1. Tác Vụ Cha Chờ Đợi Tác Vụ Con Trên Chung Một Executor

Đoạn code dưới đây cho thấy sai lầm điển hình khi làm việc với luồng:

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

Ở phía gọi (Caller), người ta lại bọc thêm một lớp nữa:

```java
// Tác vụ Cha chiếm dụng Luồng Công Nhân đầu tiên
Future<EnrichedOrder> result = enrichmentExecutor.submit(
        () -> enrichmentService.enrich(order));
```

Cùng xem cấu hình Pool thường gặp:

```java
new ThreadPoolExecutor(
        2, 2, 0, TimeUnit.MILLISECONDS, // Chỉ 2 Luồng Công Nhân
        new ArrayBlockingQueue<>(100), // Hàng đợi chứa tới 100 tác vụ
        new ThreadPoolExecutor.AbortPolicy()
);
```

**Kịch bản sập nguồn:** Hai request Cha vào giành mất 2 luồng công nhân. Cả 2 ném tiếp 2 việc Con xuống hàng đợi. Rồi 2 anh Cha gọi `get()` và đứng im. Hàng đợi có 100 chỗ chỉ làm hệ thống báo lỗi chậm hơn thôi, chứ không tự đẻ ra luồng mới để chạy mấy việc Con kia được.

> **Mẹo nhỏ:** Hàng đợi giới hạn (Bounded queue) giúp chống tràn RAM, nhưng chả có tác dụng gì nếu luồng đang chạy lại đi ngóng luồng đang xếp hàng.

## 2. Các Phương Án Sửa Lỗi Tạm Bợ Cần Tránh (Insufficient Fixes)

- **Tăng Pool/Queue mù quáng:** Khi tải tăng lên, hệ thống vẫn sẽ kẹt như cũ, chỉ tốn thêm RAM vô ích thôi.
- **Dùng hàng đợi vô hạn (Unbounded Queue):** Giấu đi lỗi từ chối, nhưng hệ thống sẽ càng lúc càng chậm và dễ văng lỗi Out of Memory.
- **Thêm Timeout cho `get()` (`get(timeout)`):** Đỡ kẹt cứng hơn, nhưng vẫn phí tài nguyên luồng và gây ra bão Retry làm rối hệ thống.
- **Dùng `CallerRunsPolicy`:** Thỉnh thoảng may mắn ép được luồng hiện tại chạy luôn tác vụ Con, giúp phá bế tắc, nhưng nó làm rối loạn hoàn toàn thời gian phản hồi. Đừng dùng nó như cách sửa lỗi tận gốc.
- **Dùng Luồng Ảo (Virtual Thread) bừa bãi:** Luồng ảo rẻ thật, nhưng request ra ngoài (ví dụ gọi DB) vẫn bị giới hạn quota kết nối.
- **Tách Pool nhưng không hãm phanh (Backpressure):** Pool Cha đẩy việc quá nhanh mà Pool Con xử lý không kịp thì hàng đợi Pool Con vẫn chết ngập.

## 3. Tiền Đề Kích Hoạt Lỗ Hổng (Conditions for Replication)

Lỗi này sẽ nổ khi hội tụ đủ 4 yếu tố:
1. Số lượng tác vụ Cha đang chạy lấp đầy số luồng tối đa của Pool.
2. Cha ném tác vụ Con vào chung cái Pool đó.
3. Cha gọi hàm để ngồi chờ Con hoàn thành.
4. Không có luồng nào rảnh rỗi, và cũng chẳng có Timeout hay Cancel nhảy vào cứu bồ.
