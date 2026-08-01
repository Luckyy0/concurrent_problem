# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: định hướng tất định (deterministic total order)

```java
package com.example.transfer;

public final class OrderedLocalTransferService {

    public void transfer(LocalAccount source, LocalAccount destination, long amount) {
        if (source == destination) {
            return;
        }
        if (amount <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }

        LocalAccount first = source.accountId().compareTo(destination.accountId()) < 0
                ? source : destination;
        LocalAccount second = first == source ? destination : source;

        first.lock().lock();
        try {
            second.lock().lock();
            try {
                if (source.balance() < amount) {
                    throw new IllegalStateException("insufficient balance");
                }
                source.debit(amount);
                destination.credit(amount);
            } finally {
                second.lock().unlock();
            }
        } finally {
            first.lock().unlock();
        }
    }
}
```

Mẹo của các bậc thầy: So sánh chuỗi ID để xếp hạng. Thằng ID nhỏ luôn bị khóa trước. Làm thế thì dù chuyển tiền qua hay lại, lộ trình đi qua lock đều giống nhau tăm tắp, chả bao giờ kẹt được! Cập nhật dữ liệu thì cứ đợi khi cầm chắc cả 2 lock rồi hẵng làm. Nhớ unlock theo trình tự ngược lại lúc lock.

> **Nói ngắn gọn:** Đặt quy tắc xếp hàng cố định sẽ xoá sổ tình trạng kẹt xe.

## Giải pháp 2: định hướng kết hợp với acquire có thể ngắt và giới hạn thời gian

```java
public boolean transferWithin(
        LocalAccount source,
        LocalAccount destination,
        long amount,
        Duration timeout
) {
    if (timeout.isZero() || timeout.isNegative()) {
        throw new IllegalArgumentException("timeout must be positive");
    }
    LocalAccount first = minByAccountId(source, destination);
    LocalAccount second = first == source ? destination : source;
    long deadline = System.nanoTime() + timeout.toNanos();
    boolean firstHeld = false;
    boolean secondHeld = false;

    try {
        firstHeld = first.lock().tryLock(timeout.toNanos(), TimeUnit.NANOSECONDS);
        if (!firstHeld) return false;

        long remaining = deadline - System.nanoTime();
        if (remaining <= 0) return false;
        secondHeld = second.lock().tryLock(remaining, TimeUnit.NANOSECONDS);
        if (!secondHeld) return false;

        moveAfterValidation(source, destination, amount);
        return true;
    } catch (InterruptedException exception) {
        Thread.currentThread().interrupt();
        throw new TransferInterruptedException(exception);
    } finally {
        if (secondHeld) second.lock().unlock();
        if (firstHeld) first.lock().unlock();
    }
}
```

Vẫn xài ID để xếp hàng, nhưng ta "lên đồ" thêm time out (thời gian giới hạn chờ). Nếu ngâm quá lâu thì nhả cờ rồi làm ván khác. Đảm bảo an toàn không để client phải mòn mỏi.

## Phương án 3: một lock chung (coarse lock)

Dùng một cái lock to đùng khóa luôn cả sổ. Cách này không lo kẹt xe nhưng hiệu năng bèo bọt vì làm tuần tự từng đơn hàng một. Khuyên dùng cho trường hợp bé xíu xiu, ít tranh chấp.

## Phương án 4: không giữ nhiều lock

Xài hàng đợi (queue) hoặc Actor. Thay vì chia lẻ thì bắt một anh điều phối xử lý lần lượt. Sẽ phải đập đi xây lại cấu trúc, nhưng khỏi lo hold-and-wait.

## So sánh đánh đổi

| Phương án | Tránh deadlock (Deadlock prevention) | Độ trễ (Latency) | Thông lượng (Throughput) | Độ phức tạp | Multi-node |
| --- | --- | --- | --- | --- | --- |
| Total order + `lock()` | Chặn đứng kẹt xe | Lúc đông thì hơi lâu | Xử lý các tài khoản rời thì mượt | Thấp | Không |
| Order + timed `tryLock` | Chặn + có mốc thời gian | Tự huỷ khi quá lâu | Tốn chi phí thử lại | Vừa | Không |
| Coarse lock | Một lock cân tất | Tắc đường cục bộ dài | Tốc độ chậm | Siêu dễ | Không |
| Actor/single owner | Khỏi xin xỏ nhiều | Phụ thuộc vào Hàng đợi | Dựa vô thiết kế cấu trúc | Khá khoai | Phải có giao thức riêng |

## Thử lại và chính sách trên production

- Rớt đài thì nhớ dọn dẹp trước khi thử lại. Giới hạn số lần thôi.
- Nhớ chờ một khoảng ngẫu nhiên (jitter) rồi hẵng chui vào lại.
- Lỗi hết tiền hay nhập sai thì đừng có thử lại làm gì cho mệt xác.
- Thao tác thì phải có khóa lũy đẳng (idempotency key) cho chắc.
- Metric đầy đủ: chờ bao lâu, rớt bao lần...
- Chết thì phải có thread dump kẹp theo.
- Lên production bằng Database thì đừng chơi chiêu này nhé, Database nó lo được.
