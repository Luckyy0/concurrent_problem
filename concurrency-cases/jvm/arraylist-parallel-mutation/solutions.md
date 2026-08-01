# Giải Pháp: Cô Lập Trạng Thái Và Quản Trị Hệ Thống Tương Tranh

## 1. Phương Pháp 1: Mô Hình Uỷ Quyền (Tác vụ trả kết quả, Luồng chính gộp lại)

Đây là **cách được khuyến nghị dùng làm tiêu chuẩn (Default choice)**. Theo cách này, các tác vụ con (Worker) hoàn toàn không biết đến sự tồn tại của mảng dùng chung. Luồng chính (Coordinator) sẽ tạo ra danh sách các `Future` theo đúng thứ tự ban đầu, đặt một thời gian chờ chung (timeout) cho toàn lô, rồi sau đó đi gom kết quả từng cái một.

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
                // Tác vụ chỉ lo lấy dữ liệu và bọc trong Future, không được đụng vào mảng chung
                futures.add(quoteExecutor.submit(
                        () -> quoteClient.fetch(request)));
            }
        } catch (RuntimeException submissionFailure) {
            cancelAll(futures); // Hủy các tác vụ lỡ gửi nếu có lỗi giữa chừng
            throw new BatchQuoteException(
                    "Không thể đẩy hết toàn bộ tác vụ của lô vào Executor",
                    submissionFailure
            );
        }

        ArrayList<QuoteResult> results = new ArrayList<>(input.size());
        boolean completed = false;

        try {
            for (Future<QuoteResult> future : futures) {
                long remaining = deadline - System.nanoTime();
                if (remaining <= 0) {
                    throw new TimeoutException("Vượt quá thời gian cho phép của lô");
                }
                // Đợi và lấy kết quả đúng theo thứ tự các Future đã được xếp hàng
                results.add(future.get(remaining, TimeUnit.NANOSECONDS));
            }
            completed = true;
            return List.copyOf(results);
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new BatchQuoteException("Luồng chính bị ngắt ngang", exception);
        } catch (ExecutionException exception) {
            throw new BatchQuoteException(
                    "Một tác vụ xử lý báo giá bị lỗi",
                    exception.getCause()
            );
        } catch (TimeoutException | CancellationException exception) {
            throw new BatchQuoteException("Lô xử lý thất bại do quá hạn hoặc bị hủy", exception);
        } finally {
            if (!completed) {
                cancelAll(futures); // Dọn dẹp tài nguyên treo nếu có lỗi
            }
        }
    }

    private static void cancelAll(List<? extends Future<?>> futures) {
        futures.forEach(future -> future.cancel(true));
    }
}
```

Lớp Exception dùng riêng cho bài toán này:

```java
package com.example.quote;

public final class BatchQuoteException extends RuntimeException {
    public BatchQuoteException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Tại sao cách này an toàn tuyệt đối?

- **Quyền Sở Hữu (Ownership):** Tác vụ (Worker) tự tạo `QuoteResult` và chuyển giao nó thông qua `Future`.
- **Chỉ 1 luồng được ghi:** Duy nhất luồng chính (Coordinator) mới có quyền gọi `.add()` vào mảng `results`.
- **Rào Cản Hiện Thị (Visibility):** Việc gọi `Future.get` giống như một điểm chốt chặn an toàn (Safe publication), đảm bảo luồng chính sẽ thấy được kết quả mới nhất mà luồng phụ đã làm.
- **Giữ nguyên thứ tự:** Mảng `futures` được tạo dựa trên thứ tự ban đầu. Nhờ đó, kết quả trả ra cũng y hệt thứ tự đó, mặc kệ luồng nào chạy xong nhanh hơn.
- **Không trả về mảng rác:** Lệnh `List.copyOf` chỉ chạy khi toàn bộ lô đã thành công 100%. Nếu không, mảng sẽ bị vứt đi.
- **Dọn dẹp triệt để:** Áp dụng luật "Tất-cả-hoặc-không-có-gì". Nếu bị lỗi hay quá hạn, toàn bộ `Future` sẽ bị hủy, không có chuyện ứng dụng trả về lỗi nhưng luồng vẫn chạy ngầm tốn RAM.

> **Nguyên tắc kỹ thuật:** Việc ngăn chặn luồng phụ đụng vào biến dùng chung là giải pháp sạch sẽ và đáng tin cậy hơn bất cứ trò khóa luồng (Lock) hay cấu trúc Collection đa luồng nào.

### Cảnh báo thêm

- Lệnh `cancel(true)` chỉ là một tín hiệu đề nghị ngưng chạy. Muốn ngưng hẳn thì bên trong `QuoteClient` phải có cơ chế cấu hình Timeout khi gọi mạng (I/O).
- Lệnh gọi mạng (Remote Call) có nguy cơ gây ảnh hưởng tới dữ liệu ở server đối tác ngay cả khi đã bị cancel. Cần truyền theo Khóa Chống Trùng (Idempotency Key) để an toàn khi thử lại.
- Tránh sập nguồn (Overload): Cần thiết lập giới hạn số lượng tác vụ trong Thread Pool để hệ thống không bị "nghẽn cổ chai" khi gửi lô quá to.

## 2. Phương Pháp 2: Chia Trước Vị Trí Cố Định (Indexed Slots)

Cách này dùng khi ta biết chính xác mảng đầu vào có bao nhiêu phần tử. Ta dùng `AtomicReferenceArray` và phân công cho mỗi tác vụ một vị trí cố định (Slot) để ghi.

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
        // Mỗi Thread được phân quyền ghi vào một vị trí index duy nhất của nó
        futures.add(quoteExecutor.submit(() ->
                slots.set(ownedIndex, quoteClient.fetch(request))));
    }

    awaitAllOrCancel(futures, timeout);

    ArrayList<QuoteResult> ordered = new ArrayList<>(requests.size());
    for (int index = 0; index < slots.length(); index++) {
        QuoteResult result = slots.get(index);
        if (result == null) {
            throw new BatchQuoteException(
                    "Thất thoát dữ liệu tại vị trí " + index,
                    null
            );
        }
        ordered.add(result);
    }
    return List.copyOf(ordered);
}
```
*(Code này bỏ bớt hàm `awaitAllOrCancel` vì nó giống hệt giải pháp 1)*. 

Mảng `AtomicReferenceArray` cho phép API giám sát đọc mảng một cách an toàn mà không sợ quăng lỗi, nhưng người xem tiến độ có thể thấy một mảng lở dở với nhiều chỗ còn là `null` (Partial State).

## 3. Phương Pháp 3: Dùng Sức Mạnh Của Framework (Parallel Stream)

Nếu các tác vụ của bạn chủ yếu là tính toán nặng bằng CPU (như mã hóa, giải thuật) chứ không phải đi gọi API qua mạng (chờ I/O), thì đây là cách ngắn gọn và thanh lịch nhất:

```java
public List<QuoteResult> calculateLocally(List<QuoteRequest> requests) {
    return requests.parallelStream()
            .map(localQuoteCalculator::calculate)
            .toList();
}
```
Framework Java Stream sẽ tự động lo việc chia nhỏ mảng, giao cho các luồng xử lý, rồi gộp lại đúng thứ tự ban đầu (`Encounter order`). Bạn không phải viết code quản lý gì cả.

**Lưu ý:** `parallelStream()` hoàn toàn khác với việc xài `Concurrent collection`. Stream xài bộ nhớ riêng của nó, không đưa mảng chia sẻ ra ngoài cho các luồng thi nhau xé.
Tuy nhiên, cẩn thận không dùng cái này nếu tác vụ phải gọi qua mạng (I/O). Java dùng chung một cái `ForkJoinPool` ngầm cho mọi Stream. Gây kẹt mạng ở đây sẽ làm tê liệt toàn bộ các chỗ khác xài parallelStream trong ứng dụng.

## 4. Phương Pháp 4: Hàng Đợi (ConcurrentLinkedQueue)

Cách này chỉ nên dùng khi bạn muốn lấy ngay kết quả của những thằng chạy xong sớm (không quan tâm thằng nào vào trước).

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

Vì Queue này an toàn nên các luồng gọi `add()` thoải mái. Tuy nhiên, nếu bạn vừa add vừa lấy list ra đọc (iterator), kết quả trả về đôi khi không đầy đủ vì tính chất của nó là "Nhất quán yếu" (Weakly consistent). Nếu bạn muốn kết quả trả về đúng thứ tự như lúc vào, bạn phải nhét cả `Index` vào trong biến `QuoteResult` rồi xếp lại.

## 5. Phương Pháp 5: Bọc Khóa Bằng Collections.synchronizedList

Đừng dùng cách này, trừ khi bí bách:

```java
List<QuoteResult> results =
        Collections.synchronizedList(new ArrayList<>());

// Luồng Tác Vụ thi nhau gọi hàm này
results.add(result);

// Thiết Lập Đọc Giám Sát (phải nhớ tự bọc lock)
List<QuoteResult> snapshot;
synchronized (results) {
    snapshot = List.copyOf(results);
}
```

Tuy hàm `add` đã được bọc lại cho an toàn, nhưng khi duyệt danh sách thì framework không tự bọc cho mình, bắt buộc bạn phải viết thêm `synchronized (results)`. Hệ quả là kết quả lộn xộn theo thời gian hoàn thành (không đúng thứ tự ban đầu) và các luồng chen lấn chờ ghi (Monitor contention) làm chậm ứng dụng. Rất dễ sinh lỗi nếu một ông dev khác ở file khác lỡ gọi danh sách này mà quên bọc block `synchronized`.

## 6. So Sánh Các Giải Pháp

| Giải Pháp | Tính Đúng Đắn | Thứ Tự | Xung Đột Khóa | Quản Lý Lỗi/Timeout | Dùng Khi Nào |
| --- | --- | --- | --- | --- | --- |
| Điều phối Tổng hợp `Future.get` (Cách #1) | Rất an toàn (All-or-Nothing) | Chuẩn 100% như lúc gửi | Không có | Rất dễ hủy và gom lỗi | Tiêu chuẩn cho hầu hết dự án |
| Vị trí cố định (Indexed slots) | An toàn | Chuẩn 100% | Không có | Phải tự chốt chặn đồng bộ | Khối lượng input luôn cố định |
| Parallel stream | Độc quyền xử lý CPU | Chuẩn theo Encounter order | Framework lo | Hơi khó bắt lỗi linh hoạt | Chỉ tính toán (CPU bound), không xài API ngoài |
| `ConcurrentLinkedQueue` | Giám sát tác vụ thời gian thực | Xếp lộn xộn theo ai xong trước | Rất ít (Non-blocking) | Buộc tạo snapshot mới lúc xong | Cần xem tiến độ realtime |
| `synchronizedList` | Dễ bị quên bọc synchronized | Lộn xộn | Gây kẹt luồng (Cổ chai) | Thiếu tương tác Lô | Hạn chế xài |

## 7. Khuyên Dùng Thế Nào Cho Hợp Lý?

- Luôn ưu tiên dùng **Cách Số 1** (Luồng chính đi thu gom) vì nó đáp ứng đầy đủ yêu cầu giữ đúng thứ tự và xử lý lỗi ngon lành.
- Dùng **Parallel stream** nếu code của bạn không có kết nối ra ngoài (chỉ xử lý dữ liệu nặng trong bộ nhớ RAM/CPU).
- Dùng **Concurrent Queue** nếu bạn đang xây dựng một ứng dụng kiểu Streaming, cần thấy kết quả nẩy lên màn hình liên tục khi có thằng chạy xong.

## 8. Các Lưu Ý Khi Đưa Lên Production

- Giới hạn số lượng `Future` bay qua lại (In-flight). Dù bạn có xài Virtual Thread thì RAM vẫn có giới hạn.
- Đặt thời gian hết hạn của luồng chờ (Deadline lô xử lý) lớn hơn một chút so với thời gian hết hạn mạng (Read timeout) để luồng chính không bị sập trước khi luồng phụ báo kết nối mạng thất bại.
- Luôn cắm các loại theo dõi (Metrics): Đếm số đầu vào, số lượng thành công, thất bại, và tỷ lệ lấp đầy Queue.
- Luôn kiểm định lại cuối cùng: Check số lượng records, loại bỏ `null`, xem dữ liệu có bị lặp hay không trước khi trả về cho khách.
- Khi tác vụ phải đụng đến server người khác hoặc ghi Database (Side-effect), bạn phải chắc chắn gửi kèm một Khóa định danh (Idempotency Key) để phòng trường hợp ứng dụng phải Retry.
