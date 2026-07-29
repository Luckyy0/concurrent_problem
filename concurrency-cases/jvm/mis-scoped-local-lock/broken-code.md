# Cách triển khai bị lỗi

## ReentrantLock mới trong mỗi lần gọi

```java
package com.example.settlement;

import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

@Service
public class BrokenSettlementArtifactService {

    private final ArtifactStore artifactStore;
    private final SettlementRenderer renderer;

    public BrokenSettlementArtifactService(
            ArtifactStore artifactStore,
            SettlementRenderer renderer
    ) {
        this.artifactStore = artifactStore;
        this.renderer = renderer;
    }

    public ArtifactResult generate(String artifactKey, Duration lockTimeout) {
        ReentrantLock lock = new ReentrantLock();
        boolean acquired = false;
        try {
            acquired = lock.tryLock(
                    lockTimeout.toMillis(),
                    TimeUnit.MILLISECONDS
            );
            if (!acquired) {
                throw new ArtifactBusyException(artifactKey);
            }

            if (artifactStore.exists(artifactKey)) {
                return ArtifactResult.alreadyExists(artifactKey);
            }

            byte[] content = renderer.render(artifactKey);
            artifactStore.put(artifactKey, content);
            return ArtifactResult.created(artifactKey);
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new ArtifactGenerationException("interrupted", exception);
        } finally {
            if (acquired) {
                lock.unlock();
            }
        }
    }
}
```

`try/finally` và interrupt handling đều đúng, nhưng lock identity sai. Mỗi call
tạo một `ReentrantLock`, nên mọi caller đều acquire lock riêng ngay lập tức.
Critical section chỉ tồn tại trên giấy.

Các dependency tối thiểu:

```java
public interface ArtifactStore {
    boolean exists(String artifactKey);
    void put(String artifactKey, byte[] content);
}

public interface SettlementRenderer {
    byte[] render(String artifactKey);
}

public record ArtifactResult(String artifactKey, Status status) {
    enum Status { CREATED, ALREADY_EXISTS }

    static ArtifactResult created(String key) {
        return new ArtifactResult(key, Status.CREATED);
    }

    static ArtifactResult alreadyExists(String key) {
        return new ArtifactResult(key, Status.ALREADY_EXISTS);
    }
}
```

## synchronized trên object sai identity

```java
public ArtifactResult generate(String artifactKey) {
    synchronized (artifactKey) {
        return generateInsideCriticalSection(artifactKey);
    }
}
```

Hai request có thể mang hai `String` object khác reference nhưng cùng content.
`new String("settlement/day-1")` và một String deserialized từ HTTP có thể
`equals()` nhau, nhưng monitor khác nhau.

Dùng `artifactKey.intern()` để ép identity còn tạo global string-pool coupling,
memory retention và lock interference với code khác trong JVM. Không dùng object
do caller sở hữu làm monitor công khai.

> **Nói ngắn gọn:** `synchronized` khóa object reference, không khóa “giá trị
> chuỗi” theo nghĩa nghiệp vụ.

## synchronized trên this nhưng service không thật sự singleton

```java
public synchronized ArtifactResult generate(String artifactKey) {
    return generateInsideCriticalSection(artifactKey);
}
```

Cách này đúng nếu mọi actor dùng đúng cùng service instance. Nó thất bại khi code
manual `new`, bean dùng prototype scope, có nhiều application context, hoặc test
vô tình tạo hai instance cùng truy cập shared store.

Ngay cả với Spring singleton chuẩn, `this` chỉ serialize trong một node và còn
serialize mọi artifact key không liên quan.

## Critical section quá hẹp

```java
byte[] content;
synchronized (monitor) {
    if (artifactStore.exists(artifactKey)) {
        return ArtifactResult.alreadyExists(artifactKey);
    }
}

content = renderer.render(artifactKey);
artifactStore.put(artifactKey, content);
```

Lock chỉ bảo vệ check. Hai thread lần lượt thấy “chưa có”, rồi cùng render/put sau
khi đã release monitor. Coordination boundary phải bao trọn compound action hoặc
conflict phải được authoritative store phát hiện tại write.

## Per-key lock map bị remove quá sớm

```java
ReentrantLock lock = locks.computeIfAbsent(artifactKey, ignored ->
        new ReentrantLock());
lock.lock();
try {
    return generateInsideCriticalSection(artifactKey);
} finally {
    lock.unlock();
    locks.remove(artifactKey);
}
```

T1 có lock, T2 đã lấy reference cùng lock và đang chờ. T1 unlock rồi remove entry;
T3 tạo lock mới và đi vào critical section song song với T2 trên lock cũ. Remove
cần reference counting/lifecycle protocol; remove mù sau unlock là sai.

Không remove thì đúng identity nhưng map có thể tăng vô hạn với key không bounded.
Striped locks tránh cả race remove lẫn unbounded key map bằng một tập lock cố định.

## Local lock trong deployment nhiều node

Node A và node B có heap riêng:

```text
node A → locks[stripe] = Lock-A
node B → locks[stripe] = Lock-B
```

Cùng key vẫn acquire hai lock khác nhau. Local locking không thể là proof cho
uniqueness trên shared object store.

## Điều kiện để lỗi xuất hiện

1. hai actor xử lý cùng logical `artifactKey`;
2. lock identity khác nhau hoặc critical section không bao trọn operation;
3. service có nhiều instance hoặc deployment có nhiều node;
4. store dùng overwrite/last-writer-wins thay vì conditional create;
5. test chỉ chạy một thread, một bean hoặc một node.

## Những cách sửa tưởng đúng nhưng chưa đủ

- Đổi `synchronized` sang `ReentrantLock` nhưng vẫn tạo lock trong method.
- Khóa request DTO/key object do caller cung cấp.
- Dùng static lock để “hỗ trợ cluster”; static vẫn chỉ thuộc một classloader/JVM.
- Chỉ khóa `exists()` rồi release trước render/put.
- Dùng `ConcurrentHashMap` cho lock registry nhưng remove entry sai lifecycle.
- Thêm `@Transactional`; nó không khóa object store và local monitor không mở rộng
  qua transaction/node.
- Dùng fair lock mặc định để chữa duplicate; fairness không sửa lock identity.
