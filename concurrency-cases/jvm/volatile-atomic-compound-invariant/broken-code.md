# Cách triển khai bị lỗi

## Đoạn code dùng AtomicInteger nhưng vẫn phá invariant

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
`if (...) pending.incrementAndGet()` không phải một atomic operation, và
`pending.decrementAndGet(); active.incrementAndGet();` cũng không phải một
transition nguyên tử trên cặp counter.

## Broken variant dùng volatile

Một field `volatile` cũng không sửa check-then-increment:

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

`used++` là read-modify-write. Volatile read và volatile write có visibility,
nhưng cặp operation không có mutual exclusion; hai thread có thể đọc cùng giá
trị rồi ghi cùng kết quả.

> **Nói ngắn gọn:** `volatile` làm giá trị dễ nhìn thấy hơn; nó không biến dấu
> `++` hoặc chuỗi “check rồi increment” thành một operation không thể chen ngang.

## Vì sao code trông có vẻ hợp lý

- field không còn là plain `int` mà đã dùng class có chữ `Atomic`;
- `get`, `incrementAndGet` và `decrementAndGet` đều thread-safe riêng lẻ;
- health endpoint thường hiển thị các số có vẻ hợp lý;
- concurrency bug chỉ xuất hiện gần limit hoặc đúng lúc transition;
- test tuần tự luôn thấy check và increment nối tiếp nhau.

Sai lầm nằm ở ranh giới invariant: capacity phụ thuộc tổng hai field và cả
transition, không phụ thuộc một method call đơn lẻ.

## Hai cửa sổ tranh chấp

### Check rồi reserve

Khi `active = 9`, `pending = 0`, `limit = 10`, hai thread cùng đọc tổng bằng 9.
Cả hai cùng increment `pending`, khiến state thành `active = 9`, `pending = 2`.

### Chuyển pending thành active

Khi `active = 9`, `pending = 1`, tổng đã đầy 10. Thread T1 decrement pending xuống
0 trước khi increment active. T2 chen vào, thấy tổng bằng 9 và reserve thêm một
pending slot. Sau đó T1 increment active lên 10; tổng trở thành 11.

## Điều kiện để lỗi xuất hiện

1. nhiều thread dùng cùng budget instance;
2. state ở gần capacity;
3. check và update không có chung linearization point;
4. transition chạm nhiều counter bằng các atomic operation tách rời;
5. callback failure/close có thể chạy lặp hoặc sai lifecycle.

## Những cách sửa tưởng đúng nhưng chưa đủ

### Thay get cộng increment bằng updateAndGet trên pending

Lambda của `pending.updateAndGet(...)` chỉ atomic đối với `pending`. Nếu lambda
đọc `active.get()`, tổng hai counter vẫn không phải một snapshot/transaction.

### Dùng incrementAndGet rồi rollback khi vượt limit

```java
int nowPending = pending.incrementAndGet();
if (active.get() + nowPending > limit) {
    pending.decrementAndGet();
    return false;
}
```

State vượt limit đã được công bố giữa increment và rollback. Thread khác có thể
đọc state này hoặc thực hiện transition. Rollback còn cạnh tranh với các update
khác và không tạo một atomic capacity decision.

### Đánh dấu AtomicInteger field là volatile

Reference `AtomicInteger` không được thay đổi; thêm `volatile` cho reference không
mở rộng atomicity của state bên trong và không nối hai instance lại với nhau.

### Dùng LongAdder cho gate

`LongAdder` phù hợp metric có contention cao. `sum()` không phải atomic snapshot
đối với concurrent updates, nên không phù hợp để quyết định có còn capacity hay
không.

### Chỉ thêm Transactional

`@Transactional` điều phối database transaction, không khóa counter trong heap.
Rollback database không tự rollback `AtomicInteger`.

### Chỉ synchronize từng method update

Nếu `tryReserveCreation` được synchronize nhưng `creationSucceeded`,
`creationFailed`, `connectionClosed` hoặc `view` không dùng cùng monitor, invariant
vẫn có đường truy cập không được bảo vệ. Lock scope phải bao trọn mọi transition
liên quan.
