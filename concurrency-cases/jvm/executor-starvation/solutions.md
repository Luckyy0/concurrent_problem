# Giải pháp và capacity policy

## Giải pháp 1: bỏ nested submission

Nếu parent đã chạy trên executor, gọi dependency trực tiếp trên worker đó:

```java
public EnrichedOrder enrich(Order order) {
    Price price = pricingClient.loadPrice(order.productId());
    return new EnrichedOrder(order.orderId(), price);
}
```

Không child queue/future, nên không có self-starvation. Remote client phải có
timeout; executor concurrency phải nhỏ hơn/equal resource budget phía dưới.

## Giải pháp 2: orchestrator submit leaf task

Request/coordinator ngoài worker pool submit leaf tasks, chờ bằng deadline chung
và không chạy một parent wrapper trong cùng pool. Khi fail/timeout, cancel mọi
future chưa xong, khôi phục interrupt và không công bố partial result. Composition
chi tiết thuộc `JVM-009`.

## Giải pháp 3: tách executor theo dependency

Parent pool và child pool tách biệt phá self-dependency, nhưng phải sizing cả hai
cùng bounded queue/admission. Parent concurrency không được tạo child demand vượt
child/remote capacity; nếu không chỉ chuyển starvation thành queueing overload.

## ThreadPoolExecutor production baseline

```java
new ThreadPoolExecutor(
        coreSize,
        maxSize,
        30, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(queueCapacity),
        threadFactory,
        new ThreadPoolExecutor.AbortPolicy()
);
```

Catch `RejectedExecutionException` tại admission boundary và map thành overload
response; không swallow. `CallerRunsPolicy` chỉ dùng khi caller slowdown và
thread-affinity side effect đã được chấp nhận.

## Deadline và cancellation

Dùng deadline chung, `future.get(remaining, unit)`, `cancel(true)` khi failure và
restore interrupt. Remote/JDBC client cần timeout riêng; interrupt không phải
network cancellation guarantee.

## Virtual threads

Phù hợp blocking I/O và làm thread dump/task model đơn giản hơn, nhưng vẫn đặt
semaphore/admission limit theo connection/remote quota. Không dùng số virtual
thread vô hạn làm capacity policy.

## So sánh

| Phương án | Progress | Overload | Complexity |
| --- | --- | --- | --- |
| Direct dependency call | Loại self-starvation | Bounded executor/admission | Thấp |
| Orchestrated leaf tasks | Không parent giữ worker | Cần deadline/cancel | Vừa |
| Separate pools | Phá cùng-pool cycle | Cần coordinated sizing | Vừa-cao |
| Virtual threads + permits | Nhiều blocking task nhẹ | Permit bảo vệ dependency | Vừa |
| Tăng queue/pool mù | Không chứng minh | Dời điểm sụp đổ | Thấp nhưng sai |

## Production checklist

Bound queue; explicit rejection; operation deadline; downstream timeout; cancel
propagation; no nested same-pool wait; graceful shutdown deadline; metrics cho
queue wait/rejection/progress; load test vượt admission capacity.
