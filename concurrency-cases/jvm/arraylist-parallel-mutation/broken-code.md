# Phản Mẫu Thiết Kế (Anti-Patterns): Nguy Cơ Tranh Chấp Tài Nguyên Nội Tại Của ArrayList

## 1. Cấu Trúc Mã Nguồn Dùng Chung ArrayList

Đoạn mã cấu trúc một Service quản lý vòng lặp tác vụ thông qua Executor của hệ sinh thái Spring. Mỗi phân luồng thi hành độc lập truy vấn báo giá và chèn chung (append) vào một cá thể `ArrayList`. Đồng thời, đặc tả thiết kế cho phép phương thức giám sát `progressSnapshot()` kích hoạt song song nhằm khảo sát tiến độ.

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
                // VỊ TRÍ LỖI: Cập nhật song song vào cấu trúc không an toàn
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
        // VỊ TRÍ LỖI: Duyệt danh sách trong lúc cấu trúc đang bị biến đổi ngầm
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

Mô hình đối tượng truyền dẫn cơ bản:

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

Cấu trúc `running` (dựa trên `ConcurrentHashMap`) chỉ giới hạn chức năng bảo vệ bảng đăng ký theo dõi Lô xử lý. Nó **KHÔNG** truyền dẫn tính toàn vẹn (Thread-safety) lan xuống các lớp biến `ArrayList` chứa bên trong cấu trúc `RunningBatch`.

## 2. Phân Rã Hai Nguy Cơ Tranh Chấp Độc Lập

### 2.1 Cạnh Tranh Ghi Đè Trên Cùng Cấu Trúc Nội Tại

Về bản chất, hàm `ArrayList.add` thực thi chuỗi hành vi rời rạc: định vị biến `size` làm chỉ mục, nạp phần tử vào mảng cấp thấp (backing array), và tăng cấp trị số `size`. Trạng thái này thiếu hụt tính Nguyên Tử. Hậu quả: Hai phân luồng tác vụ có rủi ro sử dụng trùng một vị trí chỉ mục, dẫn đến một kết quả hoàn toàn bị xóa sổ bởi kết quả chậm hơn.

Hành động cấp phát bộ nhớ trước `new ArrayList<>(expectedSize)` chỉ cắt giảm tần suất dịch chuyển vùng nhớ (resize array). Nó không cấp phép nguyên tử hóa cho các thao tác đọc/ghi thông số vòng lặp ngầm.

### 2.2 Biến Biến Dạng Cấu Trúc Tại Luồng Điền Danh Sách Chờ

Phương thức `start()` công bố định danh `batchId` tức thời sau khi kết thúc chuỗi gửi tiến trình (submit). Nếu các tác vụ nội tại khởi chạy hệ thống (event trigger) hoặc API gọi `awaitResult()` can thiệp sớm, việc chia sẻ biến `batch.futures` ngay lập tức trở thành nguy cơ phân cực trạng thái bất hợp pháp (Shared mutable state).

Thiết kế này giữ nguyên điểm công bố trạng thái (publish point) tại điểm cuối hàm `start()`, nhưng cần nhận thức rõ đặc tính rủi ro nếu quá trình nâng cấp mã nguồn vô tình phá vỡ hợp đồng luồng này.

### 2.3 Hiện Tượng Ngắt Snapshot Tiến Độ Không Định Hình

Giao thức `List.copyOf(batch.results)` bắt buộc hệ thống duyệt xuất qua toàn bộ mảng. Khi kết hợp với các luồng thay đổi cấu trúc mảng ngầm định, phương thức này phá vỡ hợp đồng quản trị an toàn luồng, gây phản ứng trả về dữ liệu rác (Partial view) hoặc ném lỗi ngưng trệ ứng dụng.

> **Nguyên tắc kỹ thuật:** Đóng gói một danh sách thay đổi (Mutable object) vào bên trong cấu trúc lưu trữ An Toàn Luồng (Concurrent map) KHÔNG cấp phép an toàn tự động cho danh sách đó.

## 3. Bản Chất Xung Đột Dễ Bị Lầm Tưởng

Cấu trúc trên thường đánh lừa tư duy an toàn qua các cảm quan:
- Bộ nhớ lưu trữ `RunningBatch` độc quyền trên mỗi yêu cầu (Request).
- Không gian bộ nhớ mảng đã được phân bổ định mức đầu vào.
- Tác vụ xử lý đối tượng truyền dẫn mới mẻ (`QuoteResult`).
- Phán quyết trả kết quả được phong tỏa chờ luồng hoàn tất (`Future.get`).
- Chuỗi kiểm thử cục bộ (Unit Test) khó bộc lộ lỗ hổng đan xen luồng.

Điểm đứt gãy chí mạng: Tập trung toàn bộ tài nguyên Ghi vào cấu trúc Chia Sẻ (Shared Accumulator) trong điều kiện Vắng Bóng Rào Cản Thẩm Định (Lock / Ownership Protocol).

## 4. Các Tác Nhân Kích Hoạt Lỗi Hệ Thống

1. Lô xử lý khởi phát tối thiểu hai tiến trình chéo (Overlap tasks).
2. Chuỗi luồng thực thi đồng thời kích hoạt `results.add`.
3. Tác vụ trích xuất tiến độ (Progress reader) khởi động trước ranh giới phong tỏa luồng.
4. Triệt tiêu cơ chế Khóa hoặc cấu trúc Tập hợp đồng thời tương xứng.
5. Yêu cầu định hình thứ tự dữ liệu đầu ra không khớp thứ tự hoàn thành.

## 5. Phản Mẫu Kỹ Thuật Khắc Phục Sai Lệch

### Tăng Khối Lượng Phân Bổ (Initial Capacity)
Xử lý Capacity chỉ loại trừ chu trình sao chép mảng mở rộng (Resize). Hiện tượng chồng chéo tham chiếu truy cập thuộc tính `size` nội tại vẫn tồn tại độc lập.

### Che Chắn Bằng Khai Báo Biến Tĩnh (`volatile` / `final`)
Tham số `final` định danh con trỏ vùng nhớ khởi tạo; tham số `volatile` định hình sự nhìn nhận thay đổi của luồng lên vùng nhớ. Cả hai công cụ này hoàn toàn không tái tạo tính Nguyên Tử cho phương thức cấp thấp `ArrayList.add`.

### Ảo Giác An Toàn Qua Mảng Đồng Thời (Parallel Stream)

```java
List<QuoteResult> results = new ArrayList<>();
requests.parallelStream()
        .map(quoteClient::fetch)
        .forEach(results::add);
```
Kiến trúc này duy trì một danh sách tích lũy ngoại vi. Framework Stream không cung cấp cơ chế khóa tự động cho các cấu trúc ngoại sinh nằm ngoài luồng thực thi.

### Áp Dụng Giới Hạn Monitor (`Collections.synchronizedList`)

Bảo chứng tuần tự tại điểm nạp giá trị, nhưng phân tán quyền kiểm soát trên các tiến trình kết hợp:

```java
synchronized (results) {
    return List.copyOf(results);
}
```
Lỗ hổng xuất hiện khi API Giám sát bỏ sót khai báo Monitor này, dẫn đến vỡ cấu trúc hợp đồng an toàn. Mô hình này không xử lý bài toán sắp xếp trạng thái hay quản trị vòng đời ngoại lệ.

### Hao Tổn Cấu Trúc Với (`CopyOnWriteArrayList`)

Kiến tạo Mảng sao lưu (Copy backing array) trên từng thao tác Ghi. Ổn định ở điểm Truyệt Xuất, nhưng thiêu rụi bộ nhớ qua vô số lần phân bổ không cần thiết (Allocation) cho các quy trình tập trung Ghi Dữ Liệu (Write-heavy).

### Trực Tiến Quản Trị Ngoại Lệ Bằng Thử Lại (Catch & Retry)

Giao thức đánh ngắt nhanh (Fail-fast iterator) hoạt động dựa trên phương thức tối đa (Best-effort). Sự vắng mặt của ngoại lệ không đại diện cho Snapshot tinh sạch. Kích hoạt Retry hoàn toàn vô nghĩa trong việc khôi phục các dữ liệu đã bị xóa sổ bởi luồng đan chéo.
