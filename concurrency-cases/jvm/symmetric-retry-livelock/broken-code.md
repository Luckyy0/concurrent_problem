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

T1 gọi `swap(A, B)`, T2 gọi `swap(B, A)`. `tryLock` ngăn hai thread bị block vô hạn, nhưng vòng lặp không có deadline hay attempt limit. Việc sử dụng fixed backoff giữ các luồng ở cùng một chu kỳ (phase) nên có thể liên tục tái tạo conflict.

## Lý do đoạn code trông có vẻ hợp lý

- không có thread nào giữ lock rồi bị block vô hạn;
- mọi attempt thất bại đều thực hiện unlock trong khối `finally`;
- operation chỉ thực hiện mutation sau khi có đủ hai lock;
- backoff có vẻ giúp giảm contention;
- unit test với một worker luôn pass.

> **Nói ngắn gọn:** đoạn code đã bảo vệ được safety nhưng chưa bảo đảm được liveness hay progress.

## Điều kiện tái hiện

1. hai actor chọn lock đầu tiên đối nghịch nhau;
2. cả hai acquire thành công lock đầu tiên trong cùng một chu kỳ (phase);
3. cả hai đều không lấy được lock thứ hai và tiến hành release gần như đồng thời;
4. cơ chế backoff hoặc CPU scheduling giữ chúng lặp lại cùng nhịp;
5. cơ chế retry không có điểm dừng (terminal budget).

## Các cách sửa chữa không triệt để

- Chỉ đổi `lock()` thành `tryLock()`; khi đó deadlock có thể biến thành livelock.
- Sử dụng fixed backoff giống hệt nhau cho mọi actor.
- Sử dụng random jitter nhưng vẫn cho phép retry vô hạn.
- Tăng số lượng thread; contention trên cùng hai resource vẫn không biến mất.
- Ghi log cho mỗi lần retry ở mức WARN; điều này có thể tạo ra logging storm.
- Catch interrupt rồi vẫn tiếp tục vòng lặp; khi đó việc huỷ request (cancellation) sẽ mất hiệu lực.
- Thực hiện mutate một phần trước khi lấy lock thứ hai rồi mới rollback; khi đó các side effect của retry trở nên khó chứng minh tính đúng đắn và có thể để lộ trạng thái trung gian (intermediate state).
