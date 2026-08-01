# Cách triển khai bị lỗi

## Đoạn code dùng AtomicInteger nhưng vẫn phá vỡ invariant

Hãy cùng nhìn vào đoạn code mà một lập trình viên nghĩ là "chuẩn bài" dưới đây:

```java
package com.example.connection;

import org.springframework.stereotype.Component;

import java.util.concurrent.atomic.AtomicInteger;

@Component
public class BrokenProviderConnectionBudget {

    private final int limit;
    private final AtomicInteger active = new AtomicInteger();
    private final AtomicInteger pending = new AtomicInteger();

    public BrokenProviderConnectionBudget(ConnectionBudgetProperties properties) {
        this.limit = properties.maxConnections();
    }

    public boolean tryReserveCreation() {
        if (active.get() + pending.get() >= limit) {
            return false;
        }

        pending.incrementAndGet();
        return true;
    }

    public void creationSucceeded() {
        pending.decrementAndGet();
        active.incrementAndGet();
    }

    public void creationFailed() {
        pending.decrementAndGet();
    }

    public void connectionClosed() {
        active.decrementAndGet();
    }

    public BudgetView view() {
        return new BudgetView(active.get(), pending.get(), limit);
    }
}
```

```java
package com.example.connection;

public record ConnectionBudgetProperties(int maxConnections) {}

public record BudgetView(int active, int pending, int limit) {
    public int used() {
        return active + pending;
    }
}
```

Ở đây, bạn thấy mỗi hàm của `AtomicInteger` thì nguyên tử (atomic) thật, nhưng **chỉ với chính cái biến đó thôi**. Cái bước `if (...) pending.incrementAndGet()` không hề được gói chung thành một thao tác duy nhất. Tương tự, hành động `pending.decrementAndGet(); active.incrementAndGet();` cũng không phải là một bước chuyển trạng thái (transition) gộp chung cho hai biến đếm này đâu.

## Phiên bản lỗi (Broken variant) dùng volatile

Nhiều người nghĩ "thế thì dùng `volatile` có khi ngon hơn". Sai nhé, `volatile` cũng không cứu được lỗi kiểm tra-rồi-mới-tăng (check-then-increment) này:

```java
@Component
public class BrokenVolatileLimiter {

    private final int limit = 10;
    private volatile int used;

    public boolean tryAcquire() {
        if (used >= limit) {
            return false;
        }
        used++;
        return true;
    }

    public void release() {
        used--;
    }
}
```

Cái `used++` nhìn gọn vậy thôi chứ đằng sau nó là ba bước: đọc (read), sửa (modify), và ghi (write). Volatile chỉ đảm bảo mọi thread thấy dữ liệu mới nhất (visibility), chứ nó không hề khóa cửa để một thread làm xong hết ba bước trên (mutual exclusion). Kết quả là hai ông thread có thể cùng đọc một giá trị, cộng lên, rồi cùng ghi đè lên nhau.

> **Nói ngắn gọn:** `volatile` giúp nhìn giá trị rõ hơn thôi; nó không hề biến dấu `++` hay chuỗi "kiểm tra rồi tăng" thành một hành động bất khả xâm phạm.

## Vì sao code trông có vẻ hợp lý

- Biến đã thoát kiếp `int` bình thường và khoác áo class `Atomic` nghe rất ngầu.
- Mấy hàm như `get`, `incrementAndGet`, và `decrementAndGet` đứng một mình thì an toàn luồng (thread-safe) tuyệt đối.
- Gọi health endpoint ra xem thì số má lúc nào cũng có vẻ tròn trịa.
- Lỗi đồng thời (concurrency bug) chỉ ngóc đầu lên khi hệ thống chạm nóc (limit) hoặc xui rủi sao đúng ngay lúc dữ liệu đang chuyển trạng thái.
- Nếu test kiểu tuần tự (chạy từng luồng một) thì mọi thứ mượt mà, kiểm tra xong mới tăng.

Cái bẫy nằm ở chỗ này: sức chứa (capacity) của hệ thống phụ thuộc vào **tổng của cả hai biến đếm** và cả **quá trình dịch chuyển** giữa chúng, chứ không phải nằm ở việc một hàm chạy mượt thế nào.

## Hai cửa sổ tranh chấp

### Kiểm tra rồi giữ chỗ (reserve)

Tưởng tượng `active = 9`, `pending = 0`, `limit = 10`. Hai thread cùng nhào vô, đọc thấy tổng mới bằng 9 (vẫn còn chỗ chán). Cả hai sung sướng gọi increment `pending`, và thế là `active = 9`, `pending = 2` — Bùm! Vượt limit.

### Chuyển pending thành active

Lại tưởng tượng `active = 9`, `pending = 1` (đã chạm ngưỡng 10). Thread T1 giảm pending xuống 0, chuẩn bị tăng active lên. Ngay lúc này, Thread T2 nhảy vào chớp nhoáng, thấy tổng mới bằng 9 (do T1 vừa giảm), thế là nhanh nhảu giữ ngay một chỗ pending nữa. T1 hoàn hồn tăng active lên thành 10. Tổng cục giờ thành 11. Toang!

## Điều kiện để lỗi xuất hiện

1. Nhiều thread cùng xúm vào gọi chung một object (budget instance).
2. Tình hình đang căng, sát nút sức chứa (capacity).
3. Bước kiểm tra và bước cập nhật không được chốt chung trong một khoảnh khắc duy nhất.
4. Việc chuyển đổi đụng chạm tới nhiều biến thông qua các bước nhỏ rải rác.
5. Callback dọn dẹp lỗi hoặc đóng kết nối bị gọi lặp lại hoặc trật nhịp.

## Những cách sửa tưởng đúng nhưng chưa đủ

### Thay get cộng increment bằng updateAndGet trên pending

Bạn định dùng lambda `pending.updateAndGet(...)`? Thao tác đó chỉ nguyên tử (atomic) trên chính cái `pending` thôi. Nếu lambda đó thò tay ra đọc `active.get()`, thì tổng hai biến vẫn không phải là một bức ảnh chụp đồng bộ (snapshot) đâu nhé.

### Dùng incrementAndGet rồi rollback khi vượt limit

Thử kiểu "cứ tăng đi rồi tính, lố thì lùi":

```java
int nowPending = pending.incrementAndGet();
if (active.get() + nowPending > limit) {
    pending.decrementAndGet();
    return false;
}
```

Nghe có vẻ khôn, nhưng trạng thái "lố limit" đã lòi ra ngoài cho bàn dân thiên hạ thấy giữa cái lúc bạn tăng và lùi rồi. Thread khác có thể vô tình đọc trúng cái trạng thái lỗi đó, hoặc xử lý sai nghiệp vụ. Cộng thêm việc rollback này còn phải cạnh tranh với mấy update của thread khác, làm rối tung cả lên.

### Đánh dấu AtomicInteger field là volatile

Bản thân class `AtomicInteger` không thay đổi địa chỉ của nó. Bạn thêm `volatile` vào biến trỏ tới nó thì cũng chả tăng thêm miếng atomicity nào bên trong, và cũng chả giúp hai biến đếm liên kết với nhau được.

### Dùng LongAdder cho chốt chặn (gate)

`LongAdder` chỉ xịn khi bạn cần gom số liệu (metric) ở môi trường tranh chấp siêu cao. Hàm `sum()` của nó không phải là một snapshot atomic. Cho nên đừng dại dùng nó làm người gác cổng quyết định xem còn chỗ hay không.

### Chỉ thêm Transactional

Bạn tính quăng `@Transactional` vào cho rảnh nợ? Annotation này dùng để điều khiển database transaction, không phải để khóa các biến đếm trong bộ nhớ heap của Java đâu. Database có rollback thì cái `AtomicInteger` kia cũng chả tự lùi về số cũ.

### Chỉ synchronize từng method update

Bạn định xài `synchronized` cho `tryReserveCreation`? Nếu các hàm khác như `creationSucceeded`, `creationFailed`, `connectionClosed` hoặc `view` không dùng chung cái chìa khóa (monitor) đó, thì kẻ hở vẫn còn nguyên. Muốn xài lock thì phải trùm hết mọi đường đi nước bước liên quan.
