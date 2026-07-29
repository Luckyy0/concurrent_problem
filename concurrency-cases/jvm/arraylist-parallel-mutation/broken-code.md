# Cách triển khai bị lỗi

## Đoạn code dùng chung ArrayList

Service dưới đây nhận một executor do Spring quản lý. Mỗi task gọi remote client
rồi append vào cùng `ArrayList`. Method `progressSnapshot()` có thể chạy đồng
thời để phục vụ endpoint theo dõi batch.

```java
package com.example.quote;

import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

@Service
public class BrokenBatchQuoteService {

    private final QuoteClient quoteClient;
    private final ExecutorService quoteExecutor;
    private final Map<UUID, RunningBatch> running = new ConcurrentHashMap<>();

    public BrokenBatchQuoteService(
            QuoteClient quoteClient,
            ExecutorService quoteExecutor
    ) {
        this.quoteClient = quoteClient;
        this.quoteExecutor = quoteExecutor;
    }

    public UUID start(List<QuoteRequest> requests) {
        UUID batchId = UUID.randomUUID();
        RunningBatch batch = new RunningBatch(requests.size());

        for (QuoteRequest request : requests) {
            Future<?> future = quoteExecutor.submit(() -> {
                QuoteResult result = quoteClient.fetch(request);
                batch.results.add(result);
            });
            batch.futures.add(future);
        }

        running.put(batchId, batch);
        return batchId;
    }

    public List<QuoteResult> awaitResult(UUID batchId, Duration timeout)
            throws Exception {
        RunningBatch batch = requireBatch(batchId);
        long deadline = System.nanoTime() + timeout.toNanos();

        for (Future<?> future : batch.futures) {
            long remaining = deadline - System.nanoTime();
            future.get(Math.max(remaining, 1), TimeUnit.NANOSECONDS);
        }

        return List.copyOf(batch.results);
    }

    public List<QuoteResult> progressSnapshot(UUID batchId) {
        return List.copyOf(requireBatch(batchId).results);
    }

    private RunningBatch requireBatch(UUID batchId) {
        RunningBatch batch = running.get(batchId);
        if (batch == null) {
            throw new IllegalArgumentException("unknown batch: " + batchId);
        }
        return batch;
    }

    private static final class RunningBatch {
        private final ArrayList<QuoteResult> results;
        private final ArrayList<Future<?>> futures;

        private RunningBatch(int expectedSize) {
            this.results = new ArrayList<>(expectedSize);
            this.futures = new ArrayList<>(expectedSize);
        }
    }
}
```

Các type nghiệp vụ tối thiểu:

```java
package com.example.quote;

public interface QuoteClient {
    QuoteResult fetch(QuoteRequest request);
}

public record QuoteRequest(String requestId, long amountInMinorUnits) {}

public record QuoteResult(
        String requestId,
        long quotedAmountInMinorUnits
) {}
```

`running` là `ConcurrentHashMap`, nhưng nó chỉ bảo vệ registry của batch. Nó
không truyền thread-safety sang hai `ArrayList` nằm bên trong `RunningBatch`.

## Hai race condition độc lập

### Nhiều task cùng append result

`ArrayList.add` về logic cần chọn index theo `size`, ghi vào backing array rồi
tăng `size`. Đây không phải một atomic operation. Hai task có thể cùng chọn một
index và một result ghi đè result còn lại.

Pre-size bằng `new ArrayList<>(expectedSize)` chỉ giảm khả năng resize. Nó không
làm cho việc cập nhật `size` hoặc slot trở nên atomic.

### Coordinator thêm future trong lúc caller bắt đầu chờ

`start()` công bố `batchId` sau vòng submit, nên caller tuần tự nhận return rồi
mới gọi `awaitResult()` sẽ thấy danh sách future đã được dựng. Tuy nhiên, nếu code
sau này phát event hoặc expose batch trước khi vòng submit kết thúc, cả
`batch.futures` cũng trở thành shared mutable state không an toàn.

Case giữ publish point tại cuối `start`, nhưng vẫn nhấn mạnh contract này vì một
refactor “cập nhật progress sớm” có thể mở thêm race.

### Progress endpoint traverse trong lúc writer add

`List.copyOf(batch.results)` phải traverse source list. Nếu task đồng thời thay
đổi cấu trúc, operation không có thread-safety contract và có thể trả partial
view hoặc ném exception.

> **Nói ngắn gọn:** bọc batch bằng một concurrent map không làm cho object mutable
> nằm trong value tự động an toàn.

## Vì sao code trông có vẻ hợp lý

- mỗi request chỉ có một `RunningBatch` riêng;
- list đã pre-size theo số input;
- mỗi task tạo một `QuoteResult` riêng;
- coordinator chờ mọi future trước khi trả final result;
- test nhỏ thường không tạo đủ overlap để làm mất update.

Điểm sai nằm ở accumulator: mọi task vẫn mutate cùng `ArrayList` và không có lock
hoặc ownership protocol.

## Điều kiện để lỗi xuất hiện

1. batch có ít nhất hai task chạy overlap;
2. các task cùng gọi `results.add`;
3. progress reader có thể traverse trước completion barrier;
4. không có lock hoặc concurrent collection bảo vệ access;
5. task completion order khác input order nếu API yêu cầu giữ order.

## Những cách sửa tưởng đúng nhưng chưa đủ

### Chỉ tăng initial capacity

Capacity chỉ liên quan số lần resize. Internal `size` và các lần ghi slot vẫn bị
nhiều thread cạnh tranh.

### Khai báo list là volatile hoặc final

`final` bảo vệ reference khởi tạo; `volatile` bảo vệ việc thay reference. Cả hai
không làm `ArrayList.add` atomic và không bảo vệ backing array.

### Dùng parallelStream với forEach results::add

```java
List<QuoteResult> results = new ArrayList<>();
requests.parallelStream()
        .map(quoteClient::fetch)
        .forEach(results::add);
```

Đây vẫn là external mutable accumulator. Parallel stream không tự phát hiện và
khóa object bên ngoài pipeline.

### Chỉ dùng Collections.synchronizedList

Từng `add` được serialize, nhưng iteration và compound operation phải khóa thủ
công trên đúng list:

```java
synchronized (results) {
    return List.copyOf(results);
}
```

Nếu progress endpoint quên monitor này, contract lại bị phá. Cách này cũng không
tự quyết định input order hoặc batch failure policy.

### Dùng CopyOnWriteArrayList cho batch ghi nhiều

Mỗi write tạo một bản sao backing array. Iterator ổn định, nhưng workload append
nhiều result phải copy liên tục và tạo nhiều allocation. Đây thường là lựa chọn
không phù hợp cho write-heavy aggregation.

### Bắt ConcurrentModificationException rồi retry

Fail-fast iterator là best-effort. Không có exception không chứng minh snapshot
đầy đủ; retry cũng không phục hồi result đã bị mất do hai lần `add` ghi đè nhau.
