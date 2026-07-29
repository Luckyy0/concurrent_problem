# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: task trả value, coordinator sở hữu output list

Đây là lựa chọn mặc định. Worker không nhận reference tới accumulator. Coordinator
lưu future theo input order, chờ bằng một deadline chung rồi tự append tuần tự.

```java
package com.example.quote;

import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CancellationException;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

@Service
public class BatchQuoteService {

    private final QuoteClient quoteClient;
    private final ExecutorService quoteExecutor;

    public BatchQuoteService(
            QuoteClient quoteClient,
            ExecutorService quoteExecutor
    ) {
        this.quoteClient = quoteClient;
        this.quoteExecutor = quoteExecutor;
    }

    public List<QuoteResult> quoteBatch(
            List<QuoteRequest> requests,
            Duration timeout
    ) {
        if (timeout.isZero() || timeout.isNegative()) {
            throw new IllegalArgumentException("timeout must be positive");
        }

        List<QuoteRequest> input = List.copyOf(requests);
        long deadline = System.nanoTime() + timeout.toNanos();
        ArrayList<Future<QuoteResult>> futures =
                new ArrayList<>(input.size());

        try {
            for (QuoteRequest request : input) {
                futures.add(quoteExecutor.submit(
                        () -> quoteClient.fetch(request)));
            }
        } catch (RuntimeException submissionFailure) {
            cancelAll(futures);
            throw new BatchQuoteException(
                    "could not submit complete batch",
                    submissionFailure
            );
        }

        ArrayList<QuoteResult> results = new ArrayList<>(input.size());
        boolean completed = false;

        try {
            for (Future<QuoteResult> future : futures) {
                long remaining = deadline - System.nanoTime();
                if (remaining <= 0) {
                    throw new TimeoutException("batch deadline exceeded");
                }
                results.add(future.get(remaining, TimeUnit.NANOSECONDS));
            }
            completed = true;
            return List.copyOf(results);
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new BatchQuoteException("batch interrupted", exception);
        } catch (ExecutionException exception) {
            throw new BatchQuoteException(
                    "quote task failed",
                    exception.getCause()
            );
        } catch (TimeoutException | CancellationException exception) {
            throw new BatchQuoteException("batch did not complete", exception);
        } finally {
            if (!completed) {
                cancelAll(futures);
            }
        }
    }

    private static void cancelAll(List<? extends Future<?>> futures) {
        futures.forEach(future -> future.cancel(true));
    }
}
```

Exception nghiệp vụ tối thiểu:

```java
package com.example.quote;

public final class BatchQuoteException extends RuntimeException {
    public BatchQuoteException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Vì sao invariant được bảo vệ

- Worker chỉ sở hữu local `QuoteResult` và trả value qua `Future`.
- Chỉ request/coordinator thread mutate `results`.
- `Future.get` tạo completion barrier và safe publication cho result.
- Futures được lưu theo input order nên final list giữ input order, không phụ
  thuộc completion order.
- Chỉ `List.copyOf` sau khi mọi future thành công mới công bố final result.
- Khi một task thua do failure hoặc deadline, batch fail và cancel phần còn lại;
  không có partial list bị trả như final result.

> **Nói ngắn gọn:** bỏ shared writer thường đơn giản và chắc chắn hơn thay
> `ArrayList` bằng một collection có nhiều lock hơn.

### Giới hạn cần ghi rõ

- `cancel(true)` mang tính cooperative; `QuoteClient` phải có timeout riêng.
- Task đã gọi remote system có thể đã tạo side effect trước khi bị cancel; retry
  cần idempotency key nếu operation không chỉ là read.
- Submit toàn bộ input cùng lúc có thể gây overload; production cần batch-size
  limit, bounded admission hoặc executor concurrency limit.
- Code minh họa basic fan-out/fan-in, không thay thế policy phức tạp của JVM-009.

## Giải pháp 2: mỗi task sở hữu một index cố định

Khi số input cố định và cần progress đọc an toàn theo index, dùng
`AtomicReferenceArray`. Mỗi task chỉ ghi slot của chính nó; coordinator kiểm tra
mọi slot sau completion.

```java
public List<QuoteResult> quoteIntoIndexedSlots(
        List<QuoteRequest> requests,
        Duration timeout
) {
    AtomicReferenceArray<QuoteResult> slots =
            new AtomicReferenceArray<>(requests.size());
    List<Future<?>> futures = new ArrayList<>(requests.size());

    for (int index = 0; index < requests.size(); index++) {
        int ownedIndex = index;
        QuoteRequest request = requests.get(index);
        futures.add(quoteExecutor.submit(() ->
                slots.set(ownedIndex, quoteClient.fetch(request))));
    }

    awaitAllOrCancel(futures, timeout);

    ArrayList<QuoteResult> ordered = new ArrayList<>(requests.size());
    for (int index = 0; index < slots.length(); index++) {
        QuoteResult result = slots.get(index);
        if (result == null) {
            throw new BatchQuoteException(
                    "missing result at index " + index,
                    null
            );
        }
        ordered.add(result);
    }
    return List.copyOf(ordered);
}
```

`awaitAllOrCancel` dùng cùng deadline, interrupt restoration và cancellation như
giải pháp 1; phần helper được lược để không lặp lại code. `AtomicReferenceArray`
cho progress reader visibility theo từng slot, nhưng progress vẫn là partial
state và không được gọi là final result.

Nếu không có concurrent progress reader, một plain array với ownership riêng cho
mỗi index và `Future.get` trước khi coordinator đọc cũng đủ; atomic array làm
contract đọc tiến độ rõ hơn.

## Giải pháp 3: framework-managed collection trong parallel stream

Với transformation CPU-bound, không blocking I/O và không cần executor policy
riêng:

```java
public List<QuoteResult> calculateLocally(List<QuoteRequest> requests) {
    return requests.parallelStream()
            .map(localQuoteCalculator::calculate)
            .toList();
}
```

Pipeline không có side effect lên external collection. Framework tạo partition,
tích lũy và merge kết quả. Với ordered source như `List`, terminal operation này
bảo toàn encounter order.

Không nhầm “collector chạy được với parallel stream” với collector có
`Collector.Characteristics.CONCURRENT`. Một concurrent collector cho phép nhiều
thread gọi accumulator trên cùng result container và container đó phải thật sự
thread-safe. `Collectors.toList()` vẫn an toàn trong parallel pipeline vì stream
framework dùng container riêng rồi combine; nó không hứa trả một concurrent
list.

Không dùng common `ForkJoinPool` mặc định cho remote blocking calls nếu chưa đánh
giá starvation, timeout và isolation. Lựa chọn executor là một phần của capacity
planning.

## Giải pháp 4: ConcurrentLinkedQueue cho completion-order progress

Khi nhiều producer cần publish result ngay lúc hoàn tất và order được định nghĩa
là completion order:

```java
ConcurrentLinkedQueue<QuoteResult> completed =
        new ConcurrentLinkedQueue<>();

List<Future<?>> futures = requests.stream()
        .map(request -> quoteExecutor.submit(() ->
                completed.add(quoteClient.fetch(request))))
        .toList();

awaitAllOrCancel(futures, timeout);
List<QuoteResult> finalSnapshot = List.copyOf(completed);
```

Queue bảo vệ concurrent `add`; iterator weakly consistent nên progress snapshot
có thể chưa chứa result vừa hoàn tất. Final snapshot chỉ tạo sau completion
barrier. Nếu cần input order, enqueue `IndexedQuoteResult` rồi sort theo index
hoặc ưu tiên giải pháp 1.

## Phương án 5: Collections.synchronizedList

Có thể đúng nếu mọi access dùng contract của wrapper:

```java
List<QuoteResult> results =
        Collections.synchronizedList(new ArrayList<>());

// Writer
results.add(result);

// Reader tạo snapshot
List<QuoteResult> snapshot;
synchronized (results) {
    snapshot = List.copyOf(results);
}
```

Từng `add` được serialize, nhưng iteration cần external synchronization. Phương
án này vẫn tạo completion-order output, tăng contention ở một monitor và dễ bị
vi phạm khi reference lan qua nhiều layer. Ownership/merge thường dễ review hơn.

## So sánh các đánh đổi

| Phương án | Correctness phù hợp | Ordering | Contention | Failure/timeout | Horizontal scalability |
| --- | --- | --- | --- | --- | --- |
| Future trả value + coordinator merge | Final result đầy đủ, all-or-nothing | Input order tự nhiên | Không lock accumulator | Policy tập trung, dễ cancel | Local cho từng batch |
| Indexed slots | Mỗi task một vị trí | Input order theo index | Không tranh cùng slot | Cần await/cancel chung | Local cho từng batch |
| Parallel stream `map().toList()` | Pure CPU transformation | Giữ encounter order | Framework partition/merge | Exception policy ít linh hoạt hơn | Local JVM/common pool |
| `ConcurrentLinkedQueue` | Multi-producer progress | Completion order | Non-blocking, có allocation node | Cần barrier trước final snapshot | Local JVM |
| `synchronizedList` | Shared append nếu mọi access đúng monitor | Completion order | Một monitor chung | Không tự cung cấp batch policy | Local JVM |
| `CopyOnWriteArrayList` | Read-mostly, rất ít write | Completion order | Mỗi write copy array | Không tự cung cấp batch policy | Local JVM |

## Khi nào nên dùng

- Chọn coordinator merge cho API trả final batch result có order và failure
  contract rõ ràng.
- Chọn indexed slots khi input có index ổn định hoặc cần progress theo vị trí.
- Chọn parallel pipeline cho computation thuần CPU, không dùng external mutable
  state.
- Chọn concurrent queue khi progress completion-order là requirement thực sự.
- Không chọn collection chỉ vì tên có chữ “concurrent”; chọn theo snapshot,
  ordering và failure semantics cần bảo vệ.

## Lưu ý khi áp dụng thực tế

- Giới hạn batch size và số task in-flight; virtual thread không loại bỏ giới hạn
  connection, rate limit hoặc heap.
- Dùng deadline chung cho batch và timeout riêng cho remote client.
- Ghi nhận `input_count`, `success_count`, `failure_count`, `cancelled_count`,
  batch duration và executor queue/saturation.
- Assertion runtime trước publish: số result đúng, `requestId` không trùng, không
  có `null`, và order khớp input nếu contract yêu cầu.
- Không trả mutable list hoặc live view cho controller; trả immutable snapshot.
- Dọn job registry sau success/failure với retention phù hợp, tránh memory leak.
- Nếu task có side effect, truyền idempotency key theo item trước khi bật retry.
