# Cách triển khai bị lỗi

## Đoạn code dùng AtomicInteger nhưng vẫn phá vỡ invariant

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

Mỗi method của `AtomicInteger` là atomic đối với chính counter đó. Toàn bộ
phép kiểm tra `if (...) pending.incrementAndGet()` không phải một atomic operation, và
quá trình `pending.decrementAndGet(); active.incrementAndGet();` cũng không phải một
transition nguyên tử trên cặp counter.

## Phiên bản lỗi (Broken variant) dùng volatile

Một field `volatile` cũng không sửa được lỗi kiểm tra-rồi-increment (check-then-increment):

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

`used++` là một thao tác đọc-sửa-ghi (read-modify-write). Volatile read và volatile write có visibility,
nhưng cặp operation không có sự loại trừ lẫn nhau (mutual exclusion); hai thread có thể đọc cùng giá
trị rồi ghi cùng kết quả.

> **Nói ngắn gọn:** `volatile` làm giá trị dễ nhìn thấy hơn; nó không biến dấu
> `++` hoặc chuỗi “kiểm tra rồi increment” thành một operation không thể chen ngang.

## Vì sao code trông có vẻ hợp lý

- field không còn là plain `int` mà đã dùng class có chữ `Atomic`;
- `get`, `incrementAndGet` và `decrementAndGet` đều an toàn luồng (thread-safe) khi đứng riêng lẻ;
- health endpoint thường hiển thị các con số có vẻ hợp lý;
- concurrency bug chỉ xuất hiện khi gần đạt limit hoặc đúng lúc diễn ra transition;
- kiểm thử tuần tự luôn thấy bước kiểm tra và increment nối tiếp nhau.

Sai lầm nằm ở ranh giới của invariant: capacity phụ thuộc vào tổng của hai field và cả
quá trình transition, không phụ thuộc vào một method call đơn lẻ.

## Hai cửa sổ tranh chấp

### Kiểm tra rồi giữ chỗ (reserve)

Khi `active = 9`, `pending = 0`, `limit = 10`, hai thread cùng đọc tổng bằng 9.
Cả hai cùng increment `pending`, khiến state thành `active = 9`, `pending = 2`.

### Chuyển pending thành active

Khi `active = 9`, `pending = 1`, tổng đã đầy 10. Thread T1 decrement pending xuống
0 trước khi increment active. T2 chen vào, thấy tổng bằng 9 và reserve thêm một
pending slot. Sau đó T1 increment active lên 10; tổng trở thành 11.

## Điều kiện để lỗi xuất hiện

1. nhiều thread dùng chung một budget instance;
2. state ở gần capacity;
3. bước kiểm tra và update không có chung một linearization point;
4. transition chạm vào nhiều counter bằng các atomic operation tách rời;
5. callback xử lý failure/close có thể chạy lặp lại hoặc sai lifecycle.

## Những cách sửa tưởng đúng nhưng chưa đủ

### Thay get cộng increment bằng updateAndGet trên pending

Lambda của `pending.updateAndGet(...)` chỉ atomic đối với `pending`. Nếu lambda
đọc `active.get()`, tổng hai counter vẫn không phải là một snapshot/transaction.

### Dùng incrementAndGet rồi rollback khi vượt limit

```java
int nowPending = pending.incrementAndGet();
if (active.get() + nowPending > limit) {
    pending.decrementAndGet();
    return false;
}
```

State vượt limit đã được công bố ra ngoài giữa quá trình increment và rollback. Thread khác có thể
đọc state này hoặc thực hiện transition. Việc rollback còn cạnh tranh với các update
khác và không tạo ra một quyết định capacity nguyên tử.

### Đánh dấu AtomicInteger field là volatile

Reference `AtomicInteger` không được thay đổi; thêm `volatile` cho reference không
mở rộng tính atomicity của state bên trong và không nối hai instance lại với nhau.

### Dùng LongAdder cho chốt chặn (gate)

`LongAdder` phù hợp với số liệu (metric) có contention cao. `sum()` không phải là một atomic snapshot
đối với các concurrent update, nên không phù hợp để quyết định xem có còn capacity hay
không.

### Chỉ thêm Transactional

`@Transactional` điều phối database transaction, không khóa counter trong heap.
Rollback database không tự động rollback `AtomicInteger`.

### Chỉ synchronize từng method update

Nếu `tryReserveCreation` được synchronize nhưng `creationSucceeded`,
`creationFailed`, `connectionClosed` hoặc `view` không dùng chung một monitor, invariant
vẫn có đường truy cập không được bảo vệ. Phạm vi lock phải bao trọn mọi transition
liên quan.
