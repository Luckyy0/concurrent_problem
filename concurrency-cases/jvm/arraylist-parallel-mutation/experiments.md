# Môi Trường Thực Nghiệm: Xác Minh Tính Nguyên Tử Trên Cấu Trúc ArrayList

## 1. Chiến Lược Kiểm Thử (Testing Strategy)

Không thể kết luận tính an toàn của hệ thống khi chạy thử nghiệm `parallelStream()` qua loa với số lượng phần tử trả về ngẫu nhiên. Cấu trúc thực nghiệm bao gồm ba tầng:

1. Thiết kế mô hình đan xen luồng (Interleaving model) với các cơ chế kiểm soát chốt chặn (hook) nhằm phơi bày lỗ hổng "thất thoát ghi đè" (Lost Update) mang tính chất phân định logic (Deterministic).
2. Kiểm tra giới hạn chịu tải (Stress test) trực diện trên `ArrayList` nhằm theo dõi sự cố sai lệch tùy biến của từng bộ phân bản JDK/Runtime.
3. Triển khai cấu trúc kiểm định thoái lui (Regression test) dành riêng cho phương pháp khắc phục; đo lường trực diện các biến Cardinality, tính Độc bản (Uniqueness), và trật tự Đầu vào (Ordering).

Toàn bộ các tín hiệu rào cản (`Latch`, `Barrier`, `Future`) bắt buộc đính kèm giới hạn thời gian (Timeout). Cấm sử dụng phương thức hoãn luồng vật lý (`Thread.sleep`) làm hệ quy chiếu quyết định Pass/Fail. Tìm hiểu sâu tại **[Kiểm Thử Tương Tranh (Concurrency Testing)](../../concepts/concurrency-testing.md)**.

## 2. Thí Nghiệm 1: Định Hình Cơ Chế Thất Thoát (Deterministic Lost Update)

Không thể cài đặt Latch vào bên trong các lệnh cấp thấp thuộc quyền riêng tư của JVM trên `ArrayList.add`. Hệ thống kiểm thử này mô phỏng trọn vẹn chuỗi mệnh lệnh "truy xuất Size → Điền thông tin Khe → Cập nhật Size" và thiết lập cưỡng ép hai tác vụ trỏ vào cùng vị trí Index 0.

```java
package com.example.quote;

import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class LostUpdateInterleavingTest {

    @Test
    void twoWritersCanOverwriteTheSameLogicalSlot() throws Exception {
        HookedAccumulator accumulator = new HookedAccumulator(2);
        CountDownLatch bothCapturedIndex = new CountDownLatch(2);
        CountDownLatch allowWrites = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            Future<?> first = executor.submit(() -> accumulator.add(
                    "result-a", bothCapturedIndex, allowWrites));
            Future<?> second = executor.submit(() -> accumulator.add(
                    "result-b", bothCapturedIndex, allowWrites));

            assertTrue(bothCapturedIndex.await(5, TimeUnit.SECONDS));
            allowWrites.countDown();

            first.get(5, TimeUnit.SECONDS);
            second.get(5, TimeUnit.SECONDS);
        }

        // Bằng chứng thất thoát: Mảng trả về 1 nhưng thực thể sinh ra 2!
        assertEquals(1, accumulator.size());
        assertEquals(1, accumulator.values().size());
    }

    private static final class HookedAccumulator {
        private final Object[] elements;
        private int size;

        private HookedAccumulator(int capacity) {
            this.elements = new Object[capacity];
        }

        void add(
                String value,
                CountDownLatch captured,
                CountDownLatch allowWrites
        ) {
            int index = size;
            captured.countDown(); // Thông báo đã đánh cắp chỉ mục
            awaitOrFail(allowWrites); // Đợi lệnh ghi đè đồng loạt
            elements[index] = value;
            size = index + 1;
        }

        int size() {
            return size;
        }

        List<Object> values() {
            return java.util.Arrays.stream(elements, 0, size).toList();
        }
    }

    private static void awaitOrFail(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("Hết hạn chờ đồng bộ Latch");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Phát sinh gián đoạn luồng", exception);
        }
    }
}
```

Kiến trúc đại diện cho tính Không-Nguyên-Tử của đối tượng, đóng vai trò bản mẫu kiểm chứng độc lập hoàn toàn khỏi hệ thống phân luồng máy tính (Scheduler).

> **Nguyên tắc kỹ thuật:** Bằng chứng Deterministic chứng minh lý thuyết cốt lõi; Khảo sát Stress Test sau đây sẽ đánh giá hệ lụy phát sinh của cấu trúc `ArrayList` thực.

## 3. Thí Nghiệm 2: Giới Hạn Tải Vật Lý Trên ArrayList

Bài kiểm thử mang tính xác suất, yêu cầu khởi chạy trong chuỗi kiểm thử độc lập `@Tag("stress")`. Việc hệ thống chạy ngang qua mà không bộc lộ lỗi không phủ nhận mã nguồn Production hoàn toàn có khả năng sụp đổ do thiếu hụt hợp đồng bảo hộ (Thread-safety).

```java
package com.example.quote;

import org.junit.jupiter.api.RepeatedTest;
import org.junit.jupiter.api.Tag;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
import java.util.stream.IntStream;

import static org.junit.jupiter.api.Assertions.assertEquals;

@Tag("stress")
class ArrayListConcurrentAddStressTest {

    @RepeatedTest(200) // Đẩy cao tần suất phát sinh đụng độ
    void concurrentAddsMustNotBeTrusted() throws Exception {
        int itemCount = 2_000;
        ArrayList<Integer> results = new ArrayList<>(itemCount);
        CountDownLatch start = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newFixedThreadPool(32)) {
            List<Future<?>> futures = IntStream.range(0, itemCount)
                    .mapToObj(value -> executor.submit(() -> {
                        awaitOrFail(start);
                        results.add(value); // Thao tác nguy hiểm
                    }))
                    .toList();

            start.countDown();
            for (Future<?> future : futures) {
                future.get(5, TimeUnit.SECONDS);
            }
        }

        // Lỗi Thất Thoát Dữ Liệu Chắc Chắn Xảy Ra!
        assertEquals(itemCount, results.size());
        assertEquals(itemCount, results.stream().distinct().count());
    }

    private static void awaitOrFail(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("Hết hạn chờ lệnh kích hoạt đồng bộ");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Gián đoạn trong quá trình đồng bộ", exception);
        }
    }
}
```

Kiểm định lỗi qua sai số kích thước (size mismatch), trùng lặp dữ liệu, hoặc ngoại lệ đính kèm vào khối Giao dịch. Không sử dụng kết quả bài đo lường này làm tiêu chuẩn độ tin cậy duy nhất. Khuyến nghị ghi lại phiên bản JDK, tài nguyên cấp phát khi vận hành các bộ Stress test diện rộng.

## 4. Thí Nghiệm 3: Bảo Toàn Tính Quy Mô Vị Trí (Regression Test)

Mô phỏng máy chủ phản hồi bất đối xứng: Yêu cầu B hoàn thành tốc độ nhanh hơn Yêu cầu A. Kiến trúc tối ưu bắt buộc tái lập nguyên trạng quy mô tham số đầu vào.

```java
package com.example.quote;

import org.junit.jupiter.api.Test;

import java.time.Duration;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import static org.junit.jupiter.api.Assertions.assertEquals;

class BatchQuoteServiceTest {

    @Test
    void returnsExactlyOneResultPerInputInInputOrder() {
        CountDownLatch secondCompleted = new CountDownLatch(1);

        QuoteClient client = request -> {
            if (request.requestId().equals("request-a")) {
                awaitOrFail(secondCompleted); // Chờ B hoàn thành trước
            } else {
                secondCompleted.countDown(); // B hoàn thành báo cáo sớm
            }
            return new QuoteResult(
                    request.requestId(),
                    request.amountInMinorUnits() + 10
            );
        };

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            BatchQuoteService service = new BatchQuoteService(client, executor);
            List<QuoteRequest> input = List.of(
                    new QuoteRequest("request-a", 100),
                    new QuoteRequest("request-b", 200)
            );

            List<QuoteResult> result = service.quoteBatch(
                    input,
                    Duration.ofSeconds(5)
            );

            // Xác minh trình tự trả về phải tuyệt đối ăn khớp với cấu trúc Mảng Input ban đầu
            assertEquals(2, result.size());
            assertEquals(
                    List.of("request-a", "request-b"),
                    result.stream().map(QuoteResult::requestId).toList()
            );
            // Xác thực độ độc bản
            assertEquals(2, result.stream()
                    .map(QuoteResult::requestId)
                    .distinct()
                    .count());
        }
    }

    private static void awaitOrFail(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("Hết hạn chờ đồng bộ");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Luồng kiểm thử gián đoạn", exception);
        }
    }
}
```

Lấy trục đối chiếu là định danh `requestId`. Bỏ qua chuỗi thứ tự (order) nếu API không quy định, nhưng nguyên tắc Toàn Vẹn Quy Mô (Cardinality) và Độc bản (Uniqueness) luôn phải thi hành nghiêm ngặt.

## 5. Thí Nghiệm 4: Chính Sách Khử Toàn Cục Nếu Lỗi (All-or-Nothing Failure)

Một Lô xử lý khi đổ vỡ phải điều hướng tín hiệu Khử (Interrupt) truy kích các tiến trình xử lý chậm hơn (Unfinished task).

```java
@Test
void failureCancelsUnfinishedTaskAndPublishesNoPartialResult() throws Exception {
    CountDownLatch slowTaskStarted = new CountDownLatch(1);
    CountDownLatch interrupted = new CountDownLatch(1);
    CountDownLatch neverReleasedNormally = new CountDownLatch(1);

    QuoteClient client = request -> {
        if (request.requestId().equals("slow")) {
            slowTaskStarted.countDown();
            try {
                neverReleasedNormally.await(30, TimeUnit.SECONDS);
                throw new AssertionError("Tác vụ xử lý chậm phải nhận lệnh Cancel, không được tiếp tục");
            } catch (InterruptedException exception) {
                interrupted.countDown();
                Thread.currentThread().interrupt();
                throw new IllegalStateException("Xác nhận tiến trình chạm ngưỡng ngắt", exception);
            }
        }

        awaitOrFail(slowTaskStarted);
        throw new IllegalStateException("Nhà cung cấp ngoại vi văng lỗi");
    };

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        BatchQuoteService service = new BatchQuoteService(client, executor);

        assertThrows(BatchQuoteException.class, () -> service.quoteBatch(
                List.of(
                        new QuoteRequest("fail", 100),
                        new QuoteRequest("slow", 200)
                ),
                Duration.ofSeconds(5)
        ));

        assertTrue(interrupted.await(5, TimeUnit.SECONDS)); // Đảm bảo luồng chậm đã tiếp thu lệnh Cancel
    }
}
```

Kiểm định phản hồi Hủy từ Client (Cancellation signal), không mang tính cam kết về quá trình hệ thống bên ngoài tự động Hoàn Tác (Rollback). Ràng buộc bảo mật mạng và cơ chế chống lặp (Idempotency) là hai tuyến bảo mật song song cần thiết trên cấp độ Tích hợp.

## 6. Mô Phỏng Hợp Đồng Đo Lường Tiến Độ

Khi khai thác cấu trúc `ConcurrentLinkedQueue`, quy chuẩn dữ liệu đối với người truy xuất Tiến độ (Progress reader):

- Xác nhận mọi phân mảnh dữ liệu (Item) hợp pháp hóa (Valid).
- Loại trừ bản sao `requestId` khi phân luồng phân phát một chiều.
- Khống chế dung lượng Mảng Tiến độ <= Dung lượng Yêu cầu.
- Snapshot Mảng Cuối (Final Snapshot) được công nhận tại điểm chốt Rào cản (Barrier).

Không ép buộc Snapshot Mảng Trung Gian (Progress Snapshot) hiển thị tuyệt đối mọi thuộc tính nếu Iterator chạy trên cơ chế Đảm Bảo Đứt Quãng (Weakly consistent).

## 7. Giám Sát Môi Trường Khai Thác (Production Metrics)

Các thông số cần đo lường liên tục:

- Biến đếm: `input_count`, `completed_count`, `failed_count`, `cancelled_count`.
- Dấu hiệu mất cân xứng tập lệnh (Cardinality mismatch), lặp thông số truy xuất.
- Thời gian Lô hết hạn và Lỗi Timeout cục bộ (Client).
- Mức độ sử dụng Executor queue, luồng xử lý tồn đọng (Saturation).
- Độ phân tán hiệu suất (Latency distribution) giữa các luồng con và Tác vụ Lô.
- Phát hiện rò rỉ tác vụ (Task leak) bám trụ hệ thống dù Lô đã bị Cancel.

Khống chế khai báo nhật ký nhạy cảm (PII). Xây dựng cấu trúc tham số chuẩn (JDK, Processor count, Iteration) nhằm tái định vị hệ thống lúc gặp lỗi (Stress failure).

## 8. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [ ] Cấu trúc mô phỏng Deterministic khẳng định đặc tả dùng chung vị trí Index.
- [ ] Phân vùng bài Stress test vào quy chuẩn đánh giá độc lập.
- [ ] Triệt tiêu cấu trúc `Thread.sleep` thay thế bằng Rào cản Đồng Bộ (CountDownLatch).
- [ ] 100% Cấu trúc chặn giới hạn đính kèm Thời hạn (Timeout).
- [ ] Kiểm chứng các trục Tọa độ Hệ thống: Quy Mô, Độc Bản, và Trình Tự.
- [ ] Đánh giá khả năng Cắt đứt nguồn tài nguyên đối với Tiến trình ngoại lệ.
- [ ] Trả nguyên hiện trạng Cờ Báo Gián Đoạn (Interrupt status).
- [ ] Kết thúc sạch môi trường thực thi bộ đệm sau Test (Executor shutdown).
- [ ] Hợp nhất các tham chiếu đo lường theo đúng đặc tả của Dữ liệu Cấu trúc.
- [ ] Xác nhận toàn vẹn (Invariant assertion) bắt buộc chạy trước tiến trình Trả kết quả Phân luồng.
