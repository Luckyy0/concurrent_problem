# Cách triển khai bị lỗi

## Code khóa source rồi destination

```java
package com.example.transfer;

import java.util.concurrent.locks.ReentrantLock;

public final class LocalAccount {
    private final String accountId;
    private final ReentrantLock lock = new ReentrantLock();
    private long balance;

    public LocalAccount(String accountId, long balance) {
        this.accountId = accountId;
        this.balance = balance;
    }

    String accountId() { return accountId; }
    ReentrantLock lock() { return lock; }
    long balance() { return balance; }
    void debit(long amount) { balance -= amount; }
    void credit(long amount) { balance += amount; }
}
```

```java
public final class BrokenLocalTransferService {

    public void transfer(LocalAccount source, LocalAccount destination, long amount) {
        source.lock().lock();
        try {
            destination.lock().lock();
            try {
                if (source.balance() < amount) {
                    throw new IllegalStateException("insufficient balance");
                }
                source.debit(amount);
                destination.credit(amount);
            } finally {
                destination.lock().unlock();
            }
        } finally {
            source.lock().unlock();
        }
    }
}
```

Unlock logic đúng khi method tiến qua lock thứ hai. Lỗi là order phụ thuộc vai trò
source/destination, nên hai transfer ngược hướng acquire cùng hai lock theo thứ tự
ngược nhau.

## Intrinsic-lock variant

```java
synchronized (source) {
    synchronized (destination) {
        moveBalance(source, destination, amount);
    }
}
```

Nó tạo cùng wait-for cycle. Intrinsic monitor acquisition không interruptible;
request timeout/cancel không buộc thread đang chờ monitor thoát ra.

## Điều kiện tái hiện

1. account locks có identity ổn định và thật sự được chia sẻ;
2. T1 giữ A trước khi xin B;
3. T2 giữ B trước khi xin A;
4. hai thread cùng giữ lock đầu tiên trước khi acquire lock thứ hai;
5. không có timeout/victim phá cycle.

## Những cách sửa chưa đủ

- Chỉ đổi `synchronized` thành `ReentrantLock` nhưng vẫn khóa source trước.
- Thêm timeout phía HTTP client; server thread vẫn có thể bị kẹt.
- Catch exception; deadlock không ném exception cho intrinsic/`lock()` wait.
- Retry ngay theo cùng order; có thể tái tạo cycle hoặc livelock.
- Dùng `Thread.stop`; không an toàn và có thể để state hỏng.
- Khóa theo `hashCode` mà không xử lý collision/tie; không tạo total order chắc chắn.
- Dùng local ordering cho database rows rồi tuyên bố đã xử lý PostgreSQL deadlock;
  query plan và các lock khác vẫn thuộc layer database.

> **Nói ngắn gọn:** `finally` chỉ chạy sau khi flow thoát khỏi chờ; nó không tự
> phá một wait-for cycle đang tồn tại.
