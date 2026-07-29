# Broken retry loop

## Code tránh deadlock nhưng tạo livelock

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

T1 gọi `swap(A, B)`, T2 gọi `swap(B, A)`. `tryLock` ngăn hai thread block vô hạn,
nhưng loop không có deadline/attempt limit. Fixed backoff giữ cùng phase nên có
thể tái tạo conflict.

## Vì sao code trông hợp lý

- không thread nào giữ lock rồi block vô hạn;
- mọi attempt thua đều unlock trong `finally`;
- operation chỉ mutate sau khi có đủ hai lock;
- backoff có vẻ giảm contention;
- unit test một worker luôn pass.

> **Nói ngắn gọn:** code đã bảo vệ safety nhưng chưa bảo vệ liveness/progress.

## Điều kiện tái hiện

1. hai actor chọn first lock đối nghịch;
2. cả hai acquire first lock trong cùng phase;
3. cả hai fail second lock, release gần đồng thời;
4. backoff/CPU scheduling giữ chúng cùng nhịp;
5. retry không có terminal budget.

## Các cách sửa chưa đủ

- Chỉ đổi `lock()` thành `tryLock()`; deadlock có thể thành livelock.
- Fixed backoff giống nhau cho mọi actor.
- Random jitter nhưng vẫn retry vô hạn.
- Tăng thread count; contention trên cùng hai resource không biến mất.
- Log mỗi retry ở mức WARN; có thể tạo logging storm.
- Catch interrupt rồi tiếp tục; request cancellation mất hiệu lực.
- Mutate một phần trước khi lấy lock thứ hai rồi rollback; retry side effect trở
  nên khó chứng minh và có thể lộ intermediate state.
