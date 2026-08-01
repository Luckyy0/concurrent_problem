# Vòng lặp retry bị lỗi (Broken retry loop)

## Đoạn code tránh được deadlock nhưng lại tạo ra livelock

```java
package com.example.channel;

import java.util.concurrent.locks.ReentrantLock;

public final class BrokenChannelSwapService {

    public void swap(Channel firstChoice, Channel secondChoice) {
        while (true) {
            boolean firstHeld = firstChoice.lock().tryLock();
            if (!firstHeld) {
                Thread.onSpinWait();
                continue;
            }

            try {
                if (secondChoice.lock().tryLock()) {
                    try {
                        swapOwnership(firstChoice, secondChoice);
                        return;
                    } finally {
                        secondChoice.lock().unlock();
                    }
                }
            } finally {
                firstChoice.lock().unlock();
            }

            fixedBackoff();
        }
    }

    private void fixedBackoff() {
        java.util.concurrent.locks.LockSupport.parkNanos(1_000_000);
    }

    private void swapOwnership(Channel first, Channel second) {
        String owner = first.owner();
        first.changeOwner(second.owner());
        second.changeOwner(owner);
    }
}
```

```java
public final class Channel {
    private final String channelId;
    private final ReentrantLock lock = new ReentrantLock();
    private String owner;

    public Channel(String channelId, String owner) {
        this.channelId = channelId;
        this.owner = owner;
    }

    String channelId() { return channelId; }
    ReentrantLock lock() { return lock; }
    String owner() { return owner; }
    void changeOwner(String newOwner) { owner = newOwner; }
}
```

Hãy nhìn đoạn code trên: T1 thì gọi `swap(A, B)`, còn T2 gọi `swap(B, A)`. Dùng `tryLock` thì đúng là giúp luồng không bị block cứng đơ ngàn năm (deadlock). Tuy nhiên, cái vòng lặp `while(true)` lại vô tận, chẳng có deadline cũng chẳng đếm số lần thử. Kết hợp với việc chờ một khoảng thời gian cố định `fixedBackoff`, tụi nó cứ chạy song song, đụng nhau rồi lại thử lại theo cùng một nhịp. Bùm, bạn có livelock!

## Lý do đoạn code trông có vẻ hợp lý

Nếu chỉ nhìn lướt qua, bạn sẽ thấy code này "tưởng không lỏ mà lỏ không tưởng", vì:
- Chẳng có luồng nào ôm khư khư khóa rồi ngủ quên (không có deadlock).
- Lỗi hay không thì khóa cũng được nhả ra đàng hoàng trong block `finally`.
- Dữ liệu (mutation) chỉ được đổi khi đã nắm chắc cả 2 khóa trong tay.
- Có chờ (backoff) đàng hoàng để giảm tranh chấp chứ bộ.
- Chạy Unit test với 1 worker thì xanh lè (pass) mượt mà.

> **Nói ngắn gọn:** Code này chạy an toàn không làm hỏng dữ liệu, nhưng xui cái là có thể nó... chẳng thèm làm xong việc (không có liveness hay progress).

## Điều kiện tái hiện

Làm sao để ra được cái lỗi củ chuối này? Cần 5 bước tụ hội:
1. Hai anh worker chọn thứ tự lấy khóa ngược nhau (người A-B, người B-A).
2. Cả hai anh đều nhanh tay túm được cái khóa đầu tiên cùng một lúc.
3. Chợt nhận ra cái khóa thứ hai đã bị tên kia nẫng mất, cả hai cùng bực mình nhả khóa đầu tiên ra.
4. Cơ chế chờ (backoff) hoặc cách hệ điều hành xếp lịch cho CPU làm cho cả hai anh cứ gặp nhau ở cùng một nhịp lặp đi lặp lại.
5. Vòng lặp xui xẻo này lại không có điểm dừng.

## Các cách sửa chữa không triệt để

Nhiều khi vội, anh em hay chắp vá theo mấy cách này, nhưng nó không giải quyết tận gốc đâu nhé:
- Đổi mỗi `lock()` thành `tryLock()`: Chúc mừng, bạn vừa biến deadlock thành livelock!
- Cho thời gian chờ cố định: Mọi người chờ giống nhau thì lại đụng nhau tiếp thôi.
- Random thời gian chờ nhưng... cho thử lại vô hạn: Rủi ro là nó vẫn kẹt lâu ơi là lâu.
- Tăng số thread lên: Càng đông càng tắc đường chứ được gì, tài nguyên vẫn chỉ có 2 cái.
- Bắn log WARN mỗi lần retry: Chờ tới lúc log ngập tràn ổ cứng (logging storm) nhé.
- Bắt lỗi interrupt rồi... chạy tiếp: Thế này thì ai mà huỷ được request khi cần.
- Sửa đại dữ liệu trước khi lấy khóa thứ hai rồi tính đường lùi (rollback) sau: Code sẽ rất phức tạp và nguy cơ lộ dữ liệu dở dang (intermediate state) cực cao.
