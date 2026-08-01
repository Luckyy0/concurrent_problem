# Giải Pháp Kiến Trúc Tối Ưu: Cô Lập Trạng Thái Và Quản Trị Hệ Thống Tương Tranh

## 1. Phương Pháp Số 1: Mô Hình Uỷ Quyền Kết Quả Cho Bộ Điều Phối Lô (Task Returns Value, Coordinator Merges)

Được khuyến nghị làm chuẩn thiết kế nền tảng (Default choice). Hệ thống tác vụ (Worker) không tiếp cận mảng phân bổ bộ nhớ tổng hợp. Cấu trúc Điều phối viên (Coordinator) lưu trữ danh mục `Future` theo trật tự gốc, thiết lập vòng giới hạn chung, và thu thập tuần tự sau hoàn tất.

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
            throw new IllegalArgumentException("Khung thời gian chờ phải lớn hơn 0");
        }

        List<QuoteRequest> input = List.copyOf(requests);
        long deadline = System.nanoTime() + timeout.toNanos();
        ArrayList<Future<QuoteResult>> futures =
                new ArrayList<>(input.size());

        try {
            for (QuoteRequest request : input) {
                // Đóng gói đối tượng phản hồi vào Future, không tham chiếu mảng lưu trữ
                futures.add(quoteExecutor.submit(
                        () -> quoteClient.fetch(request)));
            }
        } catch (RuntimeException submissionFailure) {
            cancelAll(futures); // Đình chỉ luồng hệ thống
            throw new BatchQuoteException(
                    "Sự cố hệ thống không cấp phát toàn bộ cấu trúc lô",
                    submissionFailure
            );
        }

        ArrayList<QuoteResult> results = new ArrayList<>(input.size());
        boolean completed = false;

        try {
            for (Future<QuoteResult> future : futures) {
                long remaining = deadline - System.nanoTime();
                if (remaining <= 0) {
                    throw new TimeoutException("Vượt quá ngân sách thời gian xử lý lô");
                }
                // Đồng bộ hóa việc lấy phần tử theo trật tự Mảng Future đã xếp
                results.add(future.get(remaining, TimeUnit.NANOSECONDS));
            }
            completed = true;
            return List.copyOf(results);
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new BatchQuoteException("Gián đoạn luồng xử lý lõi", exception);
        } catch (ExecutionException exception) {
            throw new BatchQuoteException(
                    "Tác vụ xử lý báo giá thất bại",
                    exception.getCause()
            );
        } catch (TimeoutException | CancellationException exception) {
            throw new BatchQuoteException("Đứt gãy hợp đồng tiến trình Lô xử lý", exception);
        } finally {
            if (!completed) {
                cancelAll(futures); // Dọn dẹp tài nguyên treo
            }
        }
    }

    private static void cancelAll(List<? extends Future<?>> futures) {
        futures.forEach(future -> future.cancel(true));
    }
}
```

Kiến trúc Lớp Ngoại Lệ Nghiệp Vụ:

```java
package com.example.quote;

public final class BatchQuoteException extends RuntimeException {
    public BatchQuoteException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Cơ Sở Đảm Bảo Cấu Trúc Bất Biến (Invariant Safeness)

- **Quyền Sở Hữu (Ownership):** Tác vụ (Worker) đóng gói cục bộ `QuoteResult` và chuyển giao thẩm quyền qua `Future`.
- **Giới Hạn Tác Động:** Duy nhất luồng Điều phối cấu trúc hệ thống (Request Thread) tiến hành tác động `results`.
- **Rào Cản Vận Hành (Visibility):** Cơ chế `Future.get` xây dựng Rào cản Công Bố (Safe publication) chắc chắn cho mọi kết quả.
- **Bảo Toàn Trật Tự:** Khối mảng `Futures` duy trì Cấu trúc Trật tự Hệ Thống; Output không bẻ cong do Trật tự Hoàn Tất (Completion order).
- **Phân Phát Duy Nhất:** Quá trình `List.copyOf` chỉ xuất hiện và chốt sổ sau khi xác lập Vòng đời Lô thành công.
- **Xóa Bỏ Cục Bộ Rác Dữ Liệu:** Một tác vụ Thất bại (Timeout/Error) dẫn đến quá trình Hủy Lô Đơn Môn (All-or-Nothing); ngăn chặn hiện tượng rò rỉ (Partial list).

> **Nguyên tắc kỹ thuật:** Cắt đứt luồng Truy xuất dùng chung (Shared writer) là hệ phương thức hoàn hảo và đáng tin cậy hơn mọi thủ thuật bao bọc Lock/Collection đồng thời nào.

### Cảnh Báo Khai Thác Mở Rộng

- Lệnh `cancel(true)` chỉ ban phát cảnh báo phối hợp (Cooperative); yêu cầu `QuoteClient` xây dựng Cấu trúc Timeout I/O nội bộ.
- Xử lý Remote Call có nguy cơ lưu vết Side-effect dù có bị báo Cancellation; Kỹ thuật Retry bắt buộc phân phối mã Khóa (Idempotency Key).
- Hiện tượng Dội Tải (Overload): Khởi chạy quy mô Lô lớn bắt buộc có lớp Phòng ngự Tải (Admission control) và giới hạn kích cỡ Pool hệ thống.

## 2. Phương Pháp Số 2: Định Tuyến Chỉ Mục Khe Cố Định (Indexed Slots)

Tương tác cấu trúc xác định tổng thể độ dài đầu vào. Áp dụng Mảng tham chiếu Nguyên Tử (`AtomicReferenceArray`) cho từng tác vụ chiếm hữu một Slot cục bộ.

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
        // Mỗi Thread làm chủ quyền Ghi trên một tọa độ độc lập
        futures.add(quoteExecutor.submit(() ->
                slots.set(ownedIndex, quoteClient.fetch(request))));
    }

    awaitAllOrCancel(futures, timeout);

    ArrayList<QuoteResult> ordered = new ArrayList<>(requests.size());
    for (int index = 0; index < slots.length(); index++) {
        QuoteResult result = slots.get(index);
        if (result == null) {
            throw new BatchQuoteException(
                    "Thất thoát điểm dữ liệu tại chỉ mục " + index,
                    null
            );
        }
        ordered.add(result);
    }
    return List.copyOf(ordered);
}
```

Bỏ qua hàm trợ giúp `awaitAllOrCancel` để tránh dư thừa (Kế thừa giải pháp 1). `AtomicReferenceArray` cấp phép khả năng đọc tiến độ, song vẫn cần lưu tâm rủi ro truy xuất Mảnh dữ liệu chưa hoàn chỉnh (Partial State).

## 3. Phương Pháp Số 3: Giao Quyền Quản Trị Bằng Framework Mảng Cấp Cấu (Parallel Stream)

Khuyến nghị sử dụng hệ thống luồng dữ liệu (Transformation) nặng tính toán lõi CPU, vắng bóng I/O độ trễ, loại bỏ sự điều phối thủ công:

```java
public List<QuoteResult> calculateLocally(List<QuoteRequest> requests) {
    return requests.parallelStream()
            .map(localQuoteCalculator::calculate)
            .toList();
}
```

Khung Pipeline cách ly Môi trường cấu trúc ngoài. Hệ Framework chịu toàn quyền phân cụm (Partition), gom nhặt và Hợp nhất. Trình tự cấu trúc đầu ra (`Encounter order`) được bảo tồn hoàn thiện.

Ghi Chú: Phân biệt `parallelStream()` khác biệt với cơ chế `Collector.Characteristics.CONCURRENT`. Khối xử lý luồng (Stream) quản trị bộ nhớ kín, không triển khai cấu trúc Tự-Đồng-Bộ vào không gian bộ nhớ.
Cảnh báo: Hạn chế dùng `ForkJoinPool` ngầm định của Java khi chưa đo đếm các bài toán Hạn mức độ trễ hệ thống.

## 4. Phương Pháp Số 4: Đánh Bắt Tiến Độ Hoàn Tất Thời Gian Thực Bằng Hàng Đợi (ConcurrentLinkedQueue)

Khi môi trường sản xuất nhiều thông tin và cấu trúc trật tự xếp hạng hoàn thành (Completion Order):

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

Cấu trúc bảo bọc thao tác Ghi (`add`); Nhưng tính Nhất Quán Yếu (Weakly consistent) của Iterator sẽ tạo ra phản hồi không đầy đủ trong chặng giữa. Đặc tả Snapshot Đóng Gói (Final snapshot) buộc triển khai ngay sau Chốt Rào Cản. Bắt buộc tạo lập `IndexedQuoteResult` để trả về đúng chuẩn Input Order nếu hệ thống yêu cầu.

## 5. Phương Pháp Số 5: Phân Phối Khoảng Đồng Bộ (Collections.synchronizedList)

Tránh phương pháp này, dù có thể áp dụng đặc tả gói an toàn của hệ Monitor:

```java
List<QuoteResult> results =
        Collections.synchronizedList(new ArrayList<>());

// Luồng Tác Vụ
results.add(result);

// Thiết Lập Đọc Giám Sát
List<QuoteResult> snapshot;
synchronized (results) {
    snapshot = List.copyOf(results);
}
```

Tuần tự hóa tại lệnh Điền `add`, song cấu trúc duyệt bắt buộc thiết lập cơ chế khóa Ngoài (External synchronization). Phá vỡ trật tự cấu trúc gốc (Completion Order) và làm tăng rủi ro Tắc nghẽn nút cổ chai (Monitor contention). Lỗ hổng tiềm tàng nếu tham chiếu trượt qua nhiều Lớp phát triển phần mềm khác.

## 6. Ma Trận Đánh Đổi Hiệu Suất Hệ Thống

| Giải Pháp Kỹ Thuật | Phương Án Correctness | Đặc Điểm Ordering | Áp Lực Contention | Độ Trễ Cục Bộ (Failure) | Khả Năng Mở Rộng |
| --- | --- | --- | --- | --- | --- |
| Điều phối Tổng hợp `Future.get` (Vị Trí #1) | Bền vững hệ thống (All-or-Nothing) | Bảo toàn thứ tự gốc | Zero Lock trên Accumulator | Chính sách hợp nhất, hủy dễ | Cục bộ JVM |
| Chỉ mục Tĩnh (Indexed slots) | Mỗi tác vụ một Khe | Bảo toàn thứ tự Slot | Không giành giật Vị trí Khe | Cần Chốt Hủy Đồng Bộ | Cục bộ JVM |
| Cấu trúc Hệ Thống (Parallel stream) | Độc quyền xử lý tác vụ Lõi CPU | Định dạng tuần tự Encounter | Framework chia sẻ bộ nhớ | Thiếu linh hoạt quy tắc Lỗi | Vùng JVM chung (Pool) |
| Hàng Đợi Hoàn Tất (`ConcurrentLinkedQueue`) | Giám sát tác vụ thời gian thực | Phụ thuộc tốc độ hoàn thành | Mở Rộng Non-blocking | Buộc rào cản Final Snapshot | Cục bộ JVM |
| Trình Đồng Bộ (`synchronizedList`) | Kỹ thuật chắp nối đồng bộ mảng | Phụ thuộc tốc độ hoàn thành | Cổ chai tài nguyên (Lock) | Thiếu tương tác Lô | Cục bộ JVM |

## 7. Khuyến Nghị Phân Phối Khai Thác

- Ưu tiên phương án Số 1 (Bộ điều phối hợp nhất) cho các kết quả Lô hoàn thiện định hình cấu trúc dữ liệu theo khuôn mẫu Trật tự và Khống chế Rủi ro.
- Khai thác Phương án Khe phân tầng tĩnh khi Khối lượng Input không biến thiên.
- Áp dụng cấu trúc Parallel cho hệ tính toán Khối tài nguyên CPU nguyên chất.
- Dùng Concurrent Queue khi cấu trúc Giám sát Mảnh dữ liệu đòi hỏi tốc độ Cập nhật.

## 8. Chuẩn Phân Cấp Triển Khai Thực Tế

- Tối giản số lượng đầu vào lô và số tác vụ bay lơ lửng (In-flight); Virtual Thread không khử giới hạn phần cứng (Heap, Rate limit).
- Phân tách cấu trúc Vòng hết hạn (Deadline chung hệ Lô) với Hạn mức Ngoại vi Remote Client (Read/Timeout).
- Tập trung theo dõi hệ thống Đo lường (Metrics): `input_count`, `success_count`, `failure_count`, Cường độ bão hòa Queue (Saturation).
- Validation Hệ thống: Kiểm chứng Số lượng bản ghi, Định danh không đụng độ, loại bỏ `null`, và đánh giá cấu trúc Đầu Ra có trùng lặp Input.
- Không phát tán Trạng Thái Biến Thiên (Mutable/Live view) tới Tầng Điều Khiển API.
- Quét sạch Danh bạ lưu trữ cấu trúc Task sau kỳ xử lý.
- Khi Remote Task có mang hiệu ứng vật lý, bổ sung đặc tả Khóa Giao Dịch (Idempotency Key).
