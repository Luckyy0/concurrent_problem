# Môi Trường Thực Nghiệm: Thử Nghiệm Lỗi Của ArrayList

## 1. Chiến Lược Kiểm Thử (Testing Strategy)

Không thể kết luận một đoạn code đa luồng là an toàn chỉ bằng cách chạy thử vài lần và đếm số lượng kết quả. Hệ thống kiểm thử của chúng ta được chia thành 3 lớp:

1. **Ép lỗi xảy ra (Deterministic model):** Dùng các chốt chặn (hook/latch) để ép hai luồng cùng đâm vào một chỗ, qua đó chứng minh chắc chắn lỗi "Ghi đè làm mất dữ liệu" (Lost Update) là có thật.
2. **Ép tải liên tục (Stress test):** Bắn hàng loạt tác vụ vào `ArrayList` để xem nó "gãy" như thế nào trong thực tế. Kết quả này sẽ khác nhau tùy vào máy và phiên bản Java (JDK).
3. **Kiểm tra hồi quy (Regression test):** Dùng để kiểm tra các phương pháp đã khắc phục lỗi (như cách 1 ở file trước). Đảm bảo kết quả phải đủ số lượng (Cardinality), không bị lặp (Uniqueness), và đúng thứ tự (Ordering).

Trong tất cả bài test, tuyệt đối không xài `Thread.sleep` để chờ luồng (vì nó lúc đúng lúc sai). Phải dùng các bộ đếm thời gian hoặc rào cản (`Latch`, `Barrier`, `Future`) có gắn kèm thời gian tối đa (Timeout). Tìm hiểu sâu tại **[Kiểm Thử Tương Tranh (Concurrency Testing)](../../concepts/concurrency-testing.md)**.

## 2. Thí Nghiệm 1: Ép Lỗi Thất Thoát Dữ Liệu Chắc Chắn Xảy Ra

Vì ta không thể chèn code bắt lỗi vào sâu bên trong ruột của class `ArrayList` gốc của Java, ta sẽ viết một class `HookedAccumulator` mô phỏng lại y hệt 3 bước `add` của ArrayList: Lấy Size → Ghi vào mảng → Tăng Size. Ta sẽ dùng `CountDownLatch` để ép 2 luồng cùng lấy ra chung một vị trí `index = 0`.

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
        // Latch này chờ cả 2 luồng đều lấy xong vị trí index hiện tại
        CountDownLatch bothCapturedIndex = new CountDownLatch(2);
        // Latch này giữ 2 luồng lại, bắt chờ để cùng ghi đè một lúc
        CountDownLatch allowWrites = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
            Future<?> first = executor.submit(() -> accumulator.add(
                    "result-a", bothCapturedIndex, allowWrites));
            Future<?> second = executor.submit(() -> accumulator.add(
                    "result-b", bothCapturedIndex, allowWrites));

            // Đợi cả 2 luồng lấy xong chung một vị trí index (chắc chắn là 0)
            assertTrue(bothCapturedIndex.await(5, TimeUnit.SECONDS));
            
            // Ra lệnh cho cả 2 luồng cùng nhào vô ghi dữ liệu
            allowWrites.countDown();

            first.get(5, TimeUnit.SECONDS);
            second.get(5, TimeUnit.SECONDS);
        }

        // Bằng chứng: Bơm 2 dữ liệu nhưng mảng chỉ lưu được 1!
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
            captured.countDown(); // Báo cáo: Tao lấy được vị trí index rồi
            awaitOrFail(allowWrites); // Bị chặn lại, đứng chờ lệnh xuất phát
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
                throw new IllegalStateException("Hết hạn chờ Latch");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Luồng bị ngắt ngang", exception);
        }
    }
}
```

Mô hình trên đã chứng minh rõ lỗi cốt lõi (Bản chất Không-Nguyên-Tử của 3 thao tác). Thí nghiệm tiếp theo sẽ quăng tải nặng vào ArrayList thật để xem nó hỏng ra sao.

## 3. Thí Nghiệm 2: Bắn Tải Nặng (Stress Test) Lên ArrayList Thật

Bài test này dựa vào xác suất hên xui. Nếu bạn chạy mà nó xanh (Pass) thì cũng ĐỪNG NGHĨ LÀ CODE BẠN AN TOÀN, vì lên Production nó vẫn có thể sập bất cứ lúc nào. Ta đánh dấu nó là `@Tag("stress")`.

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

    @RepeatedTest(200) // Chạy lại 200 lần để tăng khả năng tóm được lỗi
    void concurrentAddsMustNotBeTrusted() throws Exception {
        int itemCount = 2_000;
        ArrayList<Integer> results = new ArrayList<>(itemCount);
        CountDownLatch start = new CountDownLatch(1);

        try (ExecutorService executor = Executors.newFixedThreadPool(32)) {
            List<Future<?>> futures = IntStream.range(0, itemCount)
                    .mapToObj(value -> executor.submit(() -> {
                        awaitOrFail(start); // Chờ đông đủ mới chạy
                        results.add(value); // Thao tác chết người
                    }))
                    .toList();

            start.countDown(); // Ra hiệu cho 2000 tác vụ lao vào ghi cùng lúc
            for (Future<?> future : futures) {
                future.get(5, TimeUnit.SECONDS);
            }
        }

        // Chắc chắn sẽ có lúc kết quả này báo đỏ vì mảng không đủ 2000 phần tử!
        assertEquals(itemCount, results.size());
        assertEquals(itemCount, results.stream().distinct().count());
    }

    private static void awaitOrFail(CountDownLatch latch) {
        try {
            if (!latch.await(5, TimeUnit.SECONDS)) {
                throw new IllegalStateException("Hết hạn chờ lệnh xuất phát");
            }
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Gián đoạn lúc chờ", exception);
        }
    }
}
```

Khi chạy bài test này, bạn sẽ thỉnh thoảng thấy kích thước mảng sai, mất dữ liệu, hoặc thậm chí văng cả Exception khi mảng đang resize mà bị luồng khác chọc vào. Khuyến khích ghi nhận lại phiên bản JDK khi chạy để đối chiếu.

## 4. Thí Nghiệm 3: Test Tính An Toàn Và Đúng Thứ Tự Của Cách Sửa Lỗi (Regression Test)

Đoạn code sau test lại Phương pháp số 1 (Luồng chính gộp lại). Ta cố tình làm cho Yêu cầu B chạy xong nhanh hơn Yêu cầu A, để xem kết quả cuối cùng có bị đảo lộn không. Kết quả đúng là danh sách phải đúng thứ tự A rồi mới tới B.

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
                awaitOrFail(secondCompleted); // A bị bắt đứng chờ B chạy xong trước
            } else {
                secondCompleted.countDown(); // B chạy xong báo cáo luôn
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

            // Kiểm tra: Mảng có đủ 2 phần tử không?
            assertEquals(2, result.size());
            
            // Kiểm tra: Thứ tự có đúng y như ban đầu (A rồi B) mặc kệ B xong trước không?
            assertEquals(
                    List.of("request-a", "request-b"),
                    result.stream().map(QuoteResult::requestId).toList()
            );
            
            // Kiểm tra: Có bị trùng phần tử nào không?
            assertEquals(2, result.stream()
                    .map(QuoteResult::requestId)
                    .distinct()
                    .count());
        }
    }

    private static void awaitOrFail(CountDownLatch latch) {
        // ... hàm trợ giúp giống hệt ở trên
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

Nhờ test này, ta tự tin khẳng định Code đã thỏa mãn đủ 3 tiêu chí: Đủ số lượng, Độc bản (không lặp) và Đúng thứ tự.

## 5. Thí Nghiệm 4: Bắn Hủy Toàn Lô Nếu Có 1 Lỗi (All-or-Nothing)

Yêu cầu của Batch API là nếu 1 cái tạch thì dẹp hết, không được trả về kết quả nửa mùa (Partial). Lệnh Hủy (Cancel) phải truyền tới được những thằng đang chạy chậm.

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
                // Giả vờ đứng đây chạy rề rề tới tận 30 giây
                neverReleasedNormally.await(30, TimeUnit.SECONDS);
                throw new AssertionError("Tác vụ xử lý chậm phải nhận lệnh Cancel, không được tiếp tục tới đây");
            } catch (InterruptedException exception) {
                // Nhận được trát tử hình (Cancel)
                interrupted.countDown();
                Thread.currentThread().interrupt();
                throw new IllegalStateException("Đã bị bắt ngắt", exception);
            }
        }

        // Tác vụ này cố tình văng lỗi
        awaitOrFail(slowTaskStarted);
        throw new IllegalStateException("Nhà cung cấp ngoại vi báo lỗi");
    };

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        BatchQuoteService service = new BatchQuoteService(client, executor);

        // Chắc chắn hàm sẽ ném ra lỗi của Lô
        assertThrows(BatchQuoteException.class, () -> service.quoteBatch(
                List.of(
                        new QuoteRequest("fail", 100),
                        new QuoteRequest("slow", 200)
                ),
                Duration.ofSeconds(5)
        ));

        // Kiểm tra xem thằng chạy chậm đã nhận được lệnh Hủy (Cancel) chưa
        assertTrue(interrupted.await(5, TimeUnit.SECONDS)); 
    }
}
```

Nhắc lại: Lệnh Cancel trong Java chỉ là một tín hiệu đề nghị ngưng chạy. Muốn ngưng hẳn việc gọi qua mạng thì bạn phải setup thêm Timeout bên trong HTTP Client của mình, và xử lý kỹ thuật Idempotency ở server đầu nhận.

## 6. Giám Sát Khi Có Người Xem Tiến Độ (Progress Snapshot)

Nếu ứng dụng dùng `ConcurrentLinkedQueue` để hiển thị tiến độ thời gian thực, bộ test cần đo:
- Bất kỳ mảnh dữ liệu nào đọc ra lúc đang chạy cũng phải hợp lệ (không dính rác, null).
- Không được phép có bản ghi trùng nhau.
- Số lượng bản ghi tiến độ luôn <= Tổng yêu cầu.
- Mảng cuối cùng khi chạy xong (Final Snapshot) phải trọn vẹn 100%.

Do Queue dùng cơ chế Nhất quán yếu, nên nếu lúc đang xem tiến độ mà dữ liệu cập nhật chưa lên hết thì cũng không sao, miễn là mảng cuối cùng khi chốt lô phải đầy đủ.

## 7. Cắm Sensor Theo Dõi Trên Production (Metrics)

Phải luôn đo lường các thông số này để truy vết nếu có sự cố:
- Biến đếm: `input_count`, `completed_count`, `failed_count`, `cancelled_count`.
- Theo dõi xem số lượng gửi đi và số lượng gom về có khớp nhau không.
- Ghi lại các lô bị quá thời gian (Timeout).
- Theo dõi hàng đợi Executor xem có bị kẹt (Saturation) và quá tải CPU không.
- Theo dõi độ trễ (Latency) của từng tác vụ lẻ so với tổng thời gian của lô.
- Cảnh báo ngay nếu Lô đã báo Cancel mà các luồng phụ vẫn cứ tiếp tục chạy ngầm ngốn RAM.

## 8. Bảng Tiêu Chuẩn Phải Chấm (Checklist)

- [ ] Cấu trúc mô phỏng (Deterministic test) ép luồng va chạm vào đúng Index.
- [ ] Bật chế độ chạy độc lập cho test chịu tải (Stress test).
- [ ] KHÔNG XÀI `Thread.sleep`, dùng `CountDownLatch` để chặn.
- [ ] Bất kỳ chỗ nào bị chặn chờ đều phải gắn Timeout.
- [ ] Có Test chứng minh: Đủ số lượng, Không lặp, Đúng thứ tự.
- [ ] Có Test chứng minh: Thằng nào hỏng thì kéo cả lũ bị Cancel theo.
- [ ] Sau khi ngắt luồng, trạng thái Interrupt phải được khôi phục.
- [ ] Test xong nhớ dọn dẹp sạch sẽ (Executor shutdown).
- [ ] Cắm Metrics đo lường đúng chuẩn.
- [ ] Assert (Kiểm chứng) toàn vẹn luôn chạy TRƯỚC khi báo cáo ra bên ngoài.
