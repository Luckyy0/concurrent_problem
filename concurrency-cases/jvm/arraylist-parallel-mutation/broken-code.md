# Phản Mẫu Thiết Kế (Anti-Patterns): Nguy Cơ Khi Dùng Chung ArrayList

## 1. Mã Nguồn Lỗi Dùng Chung ArrayList

Đoạn mã dưới đây minh họa một Service quản lý xử lý tác vụ theo lô (batch) dùng Executor của Spring. Mỗi luồng (thread) chạy song song sẽ tự lấy báo giá và thêm (append) vào chung một `ArrayList`. Đồng thời, phương thức giám sát `progressSnapshot()` có thể được gọi bất cứ lúc nào để xem tiến độ hiện tại.

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
                // LỖI Ở ĐÂY: Nhiều luồng cùng gọi add() vào danh sách không an toàn
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
        // LỖI Ở ĐÂY: Đọc/Copy danh sách trong lúc các luồng khác vẫn đang ngầm add()
        return List.copyOf(requireBatch(batchId).results);
    }

    private RunningBatch requireBatch(UUID batchId) {
        RunningBatch batch = running.get(batchId);
        if (batch == null) {
            throw new IllegalArgumentException("Định danh lô không hợp lệ: " + batchId);
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

Các record/class cơ bản được sử dụng:

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

**Lưu ý:** Việc bọc `RunningBatch` bên trong `ConcurrentHashMap` (biến `running`) chỉ giúp an toàn khi thêm/xóa mã lô (`batchId`). Nó **KHÔNG** làm cho danh sách `ArrayList` bên trong `RunningBatch` trở nên an toàn khi bị đa luồng truy cập (thread-safe).

## 2. Phân Tích Các Lỗi Tranh Chấp

### 2.1 Ghi Đè Lên Nhau Trong ArrayList
Hàm `ArrayList.add` thực ra gồm nhiều bước nhỏ bên trong: đọc biến `size` để tìm vị trí, ghi dữ liệu vào mảng (backing array) ở vị trí đó, rồi tăng biến `size++`. Vì quá trình này không phải là một cục liền khối không thể chia cắt (không nguyên tử - not atomic), nên hai luồng có thể cùng đọc được một vị trí `size`, rồi cùng ghi dữ liệu vào vị trí đó. Kết quả là dữ liệu của luồng ghi sau sẽ đè lên luồng trước, gây mất dữ liệu.

Việc khai báo trước kích thước `new ArrayList<>(expectedSize)` chỉ giúp tránh việc mảng phải liên tục mở rộng bộ nhớ (resize), chứ không giúp giải quyết xung đột khi cộng biến đếm `size`.

### 2.2 Sửa Đổi Trạng Thái Dùng Chung
Phương thức `start()` trả về `batchId` ngay sau khi ném hết tác vụ vào thread pool. Lúc này, cả danh sách `batch.futures` lẫn `batch.results` đã ở trạng thái chia sẻ (Shared mutable state) và có thể bị các hàm khác đọc giữa chừng.

### 2.3 Lỗi Khi Xem Tiến Độ Cùng Lúc (Snapshot)
Lệnh `List.copyOf(batch.results)` đòi hỏi duyệt qua toàn bộ mảng. Nếu lúc nó đang duyệt mà có luồng khác đang gọi `add()` thì cấu trúc mảng bị thay đổi giữa chừng. Hệ quả là nó có thể ném ra lỗi `ConcurrentModificationException` làm treo hàm đọc, hoặc tệ hơn là trả về mảng có chứa các giá trị rác.

> **Nguyên tắc kỹ thuật:** Việc đặt một danh sách `ArrayList` (Mutable object) vào trong một `ConcurrentHashMap` (Thread-safe map) KHÔNG hề giúp danh sách đó trở nên an toàn trước đa luồng.

## 3. Vì Sao Code Dễ Đánh Lừa Lập Trình Viên?

Lập trình viên thường nghĩ mã này an toàn vì:
- Mỗi `RunningBatch` đều được cấp phát riêng cho từng request mới.
- Mảng `ArrayList` đã được khai báo sẵn kích thước.
- Dữ liệu `QuoteResult` là dữ liệu mới tạo riêng biệt.
- Hàm `awaitResult` có dùng `Future.get` để chờ tất cả xong mới lấy kết quả.
- Khi chạy thử nội bộ (Unit Test) một mình, lỗi đa luồng rất hiếm khi xảy ra.

Điểm chết người ở đây là: Tất cả các luồng ghi đều trút dữ liệu vào cùng một mảng dùng chung (Shared Accumulator) nhưng lại thiếu cơ chế bảo vệ như Khóa (Lock).

## 4. Các Điều Kiện Kích Hoạt Lỗi

Lỗi sẽ xảy ra khi hội tụ đủ các yếu tố:
1. Có từ hai tác vụ trở lên chạy song song trong cùng một lô (Overlap tasks).
2. Các luồng này tình cờ gọi `results.add` cùng lúc.
3. Người dùng vô tình gọi API xem tiến độ (Progress reader) trước khi lô chạy xong.
4. Không hề có Lock (Synchronized) hay dùng Collection hỗ trợ đa luồng.
5. Nghiệp vụ yêu cầu danh sách kết quả phải đúng thứ tự, nhưng các luồng lại add() lộn xộn.

## 5. Những Cách Sửa Sai Lầm (Phản Mẫu - Anti-patterns)

Dưới đây là những cách sửa mà lập trình viên hay dùng nhưng thực chất **không giải quyết được vấn đề**:

### Truyền Khối Lượng Khởi Tạo Lớn (Initial Capacity)
Như đã nói ở trên, việc `new ArrayList<>(size)` chỉ giúp mảng đỡ phải mở rộng kích thước, hoàn toàn không chặn được việc 2 luồng cùng ghi đè lên chung một vị trí `size`.

### Dùng từ khóa `volatile` hoặc `final`
Từ khóa `final` chỉ khóa không cho trỏ biến đó sang một danh sách khác, `volatile` thì giúp luồng khác thấy danh sách được cập nhật. Cả hai từ khóa này không liên quan gì đến việc bảo vệ cơ chế `add()` của mảng khỏi xung đột đa luồng.

### Dùng Parallel Stream Sai Cách

```java
List<QuoteResult> results = new ArrayList<>();
requests.parallelStream()
        .map(quoteClient::fetch)
        .forEach(results::add); // LỖI Y HỆT
```
Vòng lặp bên trong chia luồng chạy song song, nhưng vẫn trút dữ liệu vào một biến `results` dùng chung bên ngoài (external accumulator). Framework Stream không tự động sinh ra Lock để bảo vệ biến đó.

### Bọc Khóa Bằng Collections.synchronizedList

Cách này có thể tạm ổn khi `add`, nhưng lại rất dễ dính lỗi ở những chỗ khác:

```java
// Khi gọi progressSnapshot
synchronized (results) {
    return List.copyOf(results);
}
```
Lỗ hổng xảy ra nếu ở hàm nào đó lập trình viên quên dùng `synchronized (results)` khi duyệt mảng. Ngoài ra, cách này làm các luồng phải xếp hàng chờ nhau để `add()`, gây chậm, và nó không đảm bảo thứ tự kết quả như ban đầu.

### Dùng CopyOnWriteArrayList
Mỗi khi gọi `add()`, class này sẽ tạo bản sao (copy) của toàn bộ mảng. Nếu gọi `add()` hàng trăm lần, nó sẽ phung phí và ngốn bộ nhớ kinh khủng (Write-heavy). Class này chỉ phù hợp nếu ta ít khi thêm/xóa mà chủ yếu là đọc.

### Dùng Khối Catch Bắt Lỗi Rồi Chạy Lại (Catch & Retry)
Bắt lỗi `ConcurrentModificationException` rồi bỏ qua không giải quyết được việc các luồng đã lỡ đè dữ liệu lên nhau. Ngoài ra, Collection đôi khi bị hỏng cấu trúc ngầm mà không ném lỗi ngay. Việc Retry cũng không khôi phục được dữ liệu đã bị ghi đè mất.
