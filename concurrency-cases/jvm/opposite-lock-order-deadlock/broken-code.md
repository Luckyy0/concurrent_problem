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

Code viết `finally` để unlock thì rất chuẩn bài rồi, nhưng chết ở chỗ thứ tự lock bị phụ thuộc vào biến `source` và `destination`. A chuyển sang B thì lock A trước, B chuyển sang A thì lại lock B trước. Thế là xong đời!

## Biến thể Intrinsic-lock

```java
synchronized (source) {
    synchronized (destination) {
        moveBalance(source, destination, amount);
    }
}
```

Dùng `synchronized` cũng dính chưởng y chang. Đã thế dùng thằng này còn không ngắt (interrupt) được, thread cứ chờ mãi mãi thôi.

## Điều kiện tái hiện

Deadlock sẽ nổ ra nếu hội đủ mấy điều kiện sau:
1. Các lock đều là thật và dùng chung.
2. Thread 1 khóa A rồi xin B.
3. Thread 2 khóa B rồi xin A.
4. Hai ông cùng giữ lock đầu tiên đúng cùng một lúc.
5. Không có timeout để dọn dẹp hiện trường.

## Những cách sửa chưa đủ

Nhiều người hay fix sai lầm kiểu này:
- Chuyển `synchronized` qua `ReentrantLock` nhưng vẫn giữ nguyên thứ tự khóa. Chả giải quyết gì!
- Chặn timeout ở phía gọi API (HTTP client): Server thì vẫn đang treo cơ mà.
- Đặt `try-catch`: Hàm `lock()` thường nó không ném lỗi timeout để mà bắt đâu.
- Chết máy thì thử lại ngay: Bạn có thể gây ra livelock.
- Xài `Thread.stop`: Cái này nguy hiểm, dễ làm hỏng cả trạng thái dữ liệu.
- Khoá theo `hashCode`: Lỡ hai cái trùng hash thì sao? Chết chắc!
- Nghĩ rằng khoá trong Code thì DB sẽ an toàn: Ở DB nó lại có luật chơi riêng nhé.

> **Nói ngắn gọn:** `finally` chỉ giúp nhả lock khi luồng code chạy qua được thôi, nếu nó kẹt cứng lúc đang xin lock thì `finally` cũng đành bất lực.
