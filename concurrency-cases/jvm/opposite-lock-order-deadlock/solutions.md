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

`accountId` phải là duy nhất, bất biến và dùng cùng comparator ở mọi nơi gọi (call site). Thứ tự acquire không còn phụ thuộc hướng chuyển. Việc thay đổi (mutation) balance chỉ chạy sau khi có cả hai lock; tiến hành release theo thứ tự ngược.

> **Nói ngắn gọn:** thứ tự chuẩn biến hai hướng chuyển thành cùng một đường đi qua đồ thị lock, nên không thể tạo cạnh chờ ngược.

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

Các hàm hỗ trợ (helper) chọn thứ tự ID và thay đổi trạng thái giống giải pháp 1; exception là một lỗi nghiệp vụ (domain exception) kế thừa `RuntimeException`. Thời hạn chung ngăn tổng thời gian chờ bị nhân đôi. Yêu cầu vượt thời gian chờ sẽ release lock đã giữ và phía gọi quyết định thử lại; không thử lại bên trong khoảng giữ lock.

## Phương án 3: một lock chung (coarse lock)

Một private final monitor cho toàn registry sẽ loại bỏ chu trình nhiều lock và dễ chứng minh, nhưng nó chạy tuần tự các thao tác chuyển trên các account không liên quan. Phù hợp với trạng thái nhỏ, contention thấp; không mở rộng ra khỏi một JVM.

## Phương án 4: không giữ nhiều lock

Phân chia account theo single owner, gửi lệnh qua hàng đợi hoặc tác nhân (actor), hoặc đổi mô hình trạng thái để một coordinator thực hiện thay đổi. Cách này loại bỏ tình trạng hold-and-wait nhưng sẽ thay đổi kiến trúc, độ trễ và ngữ nghĩa khi có lỗi (failure semantics).

## So sánh đánh đổi

| Phương án | Tránh deadlock (Deadlock prevention) | Độ trễ (Latency) | Thông lượng (Throughput) | Độ phức tạp | Multi-node |
| --- | --- | --- | --- | --- | --- |
| Total order + `lock()` | Phá chu trình chờ vòng tròn (circular wait) | Có thể chờ khi contention cao | Các account độc lập có thể chạy song song | Thấp | Không |
| Order + timed `tryLock` | Phòng ngừa + giới hạn thời gian chờ | Có thời hạn và tự chối | Có chi phí thử lại (retry overhead) | Vừa | Không |
| Coarse lock | Chỉ dùng một lock | Gây chặn ở hàng đợi (Head-of-line blocking) | Thấp hơn | Rất thấp | Không |
| Actor/single owner | Không giữ nhiều lock | Phụ thuộc vào hàng đợi (Queueing) | Dựa vào số phân vùng (partition) | Cao | Cần giao thức riêng |

## Thử lại và chính sách trên production

- Chỉ thử lại sau khi đã dọn dẹp, sử dụng thời hạn thao tác và giới hạn số lượt thử.
- Khoảng lùi (Backoff) cần có độ trễ ngẫu nhiên (jitter) để tránh hai tác nhân tái va chạm.
- Không thử lại các lỗi do không đủ balance hoặc validation thất bại.
- External side effect cần khóa lũy đẳng (idempotency key).
- Số liệu: thời gian chờ lock, số timeout, số lượt thử lại, số lượng detector và khoảng thời gian chạy critical-section.
- Thread dump phải lưu stack của owner và waiter khi có sự cố.
- Không dùng giải pháp local cho việc chuyển tiền trên database; xem tình huống `DB-008` và các trường hợp ngân hàng.
