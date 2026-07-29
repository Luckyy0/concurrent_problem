# Cách triển khai bị lỗi

## Parent chờ child trên cùng executor

```java
@Service
public class BrokenEnrichmentService {
    private final ExecutorService enrichmentExecutor;
    private final PricingClient pricingClient;

    public EnrichedOrder enrich(Order order) {
        Future<Price> child = enrichmentExecutor.submit(
                () -> pricingClient.loadPrice(order.productId()));
        try {
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

Caller cũng submit parent vào executor:

```java
Future<EnrichedOrder> result = enrichmentExecutor.submit(
        () -> enrichmentService.enrich(order));
```

Pool/queue thường được cấu hình:

```java
new ThreadPoolExecutor(
        2, 2, 0, TimeUnit.MILLISECONDS,
        new ArrayBlockingQueue<>(100),
        new ThreadPoolExecutor.AbortPolicy()
);
```

Hai parent chiếm hai worker, hai child vào queue, cả hai parent `get()` vô hạn.
Queue capacity 100 chỉ trì hoãn rejection; nó không tạo worker cho child.

> **Nói ngắn gọn:** bounded queue bảo vệ memory nhưng không sửa dependency graph
>mà task đang chạy phụ thuộc task bị xếp sau chính nó.

## Các cách sửa chưa đủ

- Tăng pool/queue tùy ý: tải lớn hơn vẫn chạm cùng cycle, tăng resource usage.
- Đổi sang unbounded queue: che rejection rồi tăng latency/memory.
- Chỉ thêm `get(timeout)`: containment tốt hơn nhưng vẫn lãng phí worker và tạo
  timeout/retry churn.
- Dùng `CallerRunsPolicy` mù quáng: có thể chạy child inline và phá cycle trong
  một số timing, nhưng thay đổi latency/thread-affinity và không phải design proof.
- Dùng virtual thread không giới hạn: external dependency vẫn có quota.
- Tách pool nhưng cho parent pool lớn hơn child capacity mà không backpressure:
  child queue vẫn bão hòa.

## Điều kiện tái hiện

Số parent đang chạy bằng worker count; mỗi parent submit ít nhất một child vào
cùng executor; parent chờ child; không worker nào thoát hoặc timeout/cancel.
