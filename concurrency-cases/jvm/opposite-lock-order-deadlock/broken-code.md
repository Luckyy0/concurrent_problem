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

Logic unlock đúng khi hàm tiến qua lock thứ hai. Lỗi ở đây là thứ tự phụ thuộc vai trò source và destination, nên hai thao tác chuyển ngược hướng sẽ acquire cùng hai lock theo thứ tự ngược nhau.

## Biến thể Intrinsic-lock

```java
synchronized (source) {
    synchronized (destination) {
        moveBalance(source, destination, amount);
    }
}
```

Nó tạo ra cùng chu trình chờ. Việc acquire intrinsic monitor không thể bị ngắt (interruptible); thời gian chờ hoặc lệnh hủy yêu cầu (request timeout/cancel) không buộc thread đang chờ monitor thoát ra.

## Điều kiện tái hiện

1. Các account lock có định danh ổn định và thật sự được chia sẻ;
2. T1 giữ A trước khi xin B;
3. T2 giữ B trước khi xin A;
4. Hai thread cùng giữ lock đầu tiên trước khi acquire lock thứ hai;
5. Không có timeout hoặc nạn nhân (victim) để phá chu trình.

## Những cách sửa chưa đủ

- Chỉ đổi `synchronized` thành `ReentrantLock` nhưng vẫn khóa source trước.
- Thêm timeout phía HTTP client; server thread vẫn có thể bị kẹt.
- Bắt (Catch) exception; deadlock không ném ra exception cho khoảng chờ intrinsic hoặc `lock()`.
- Thử lại ngay theo cùng thứ tự; có thể tái tạo chu trình hoặc livelock.
- Dùng `Thread.stop`; không an toàn và có thể làm hỏng trạng thái.
- Khóa theo `hashCode` mà không xử lý va chạm (collision) hoặc phân định (tie); không tạo total order chắc chắn.
- Dùng local ordering cho các database row rồi tuyên bố đã xử lý PostgreSQL deadlock; query plan và các lock khác vẫn thuộc tầng database.

> **Nói ngắn gọn:** `finally` chỉ chạy sau khi luồng thực thi (flow) thoát khỏi trạng thái chờ; nó không tự phá một wait-for cycle đang tồn tại.
