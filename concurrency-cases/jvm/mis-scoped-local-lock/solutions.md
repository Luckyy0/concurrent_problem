# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: striped ReentrantLock ổn định trong một JVM

Một tập lock cố định tránh lock-map leak và remove race. Cùng key luôn hash vào
cùng stripe trong service instance.

```java
package com.example.settlement;

import java.util.concurrent.locks.ReentrantLock;

public final class StripedKeyLocks {

    private final ReentrantLock[] stripes;

    public StripedKeyLocks(int stripeCount) {
        if (stripeCount <= 0) {
            throw new IllegalArgumentException("stripeCount must be positive");
        }
        this.stripes = new ReentrantLock[stripeCount];
        for (int index = 0; index < stripeCount; index++) {
            stripes[index] = new ReentrantLock();
        }
    }

    public ReentrantLock lockFor(String key) {
        int hash = key.hashCode();
        int spread = hash ^ (hash >>> 16);
        return stripes[Math.floorMod(spread, stripes.length)];
    }
}
```

```java
package com.example.settlement;

import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

@Service
public class SettlementArtifactService {

    private final ArtifactStore artifactStore;
    private final SettlementRenderer renderer;
    private final StripedKeyLocks keyLocks = new StripedKeyLocks(256);

    public SettlementArtifactService(
            ArtifactStore artifactStore,
            SettlementRenderer renderer
    ) {
        this.artifactStore = artifactStore;
        this.renderer = renderer;
    }

    public ArtifactResult generate(String artifactKey, Duration timeout) {
        if (timeout.isZero() || timeout.isNegative()) {
            throw new IllegalArgumentException("timeout must be positive");
        }

        ReentrantLock lock = keyLocks.lockFor(artifactKey);
        boolean acquired = false;
        try {
            acquired = lock.tryLock(timeout.toNanos(), TimeUnit.NANOSECONDS);
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

`ArtifactBusyException` và `ArtifactGenerationException` là các
`RuntimeException` domain mỏng, lần lượt biểu diễn lock timeout và technical
failure. Constructor nhận `artifactKey` hoặc `message/cause`; phần khai báo được
lược để tập trung vào lock protocol.

### Vì sao invariant local được bảo vệ

- `keyLocks` là field ổn định của singleton.
- Cùng key luôn nhận cùng stripe; hash collision chỉ serialize thêm key khác.
- Check, render và put nằm trong cùng critical section.
- `tryLock` có timeout; interrupt status được khôi phục.
- `finally` release lock trên mọi return/exception path.
- Không có per-key entry cần remove.

> **Nói ngắn gọn:** striped lock đổi một chút concurrency giữa các key để lấy
> lock identity ổn định và lifecycle đơn giản.

### Giới hạn

Lock bị giữ trong render và remote write vì invariant local yêu cầu single
workflow. Đây có thể là critical section dài. Nếu duplicate render được chấp
nhận, có thể render ngoài lock và dựa vào conditional create tại store cho
correctness, giảm lock hold time nhưng tăng wasted work.

Service này chỉ đúng trong một JVM. Không ghi trong API/documentation rằng nó bảo
vệ global uniqueness.

## Giải pháp 2: private final monitor cho coarse-grained locking

```java
@Service
public class CoarseSettlementArtifactService {

    private final Object generationMonitor = new Object();
    private final ArtifactStore artifactStore;
    private final SettlementRenderer renderer;

    public ArtifactResult generate(String key) {
        synchronized (generationMonitor) {
            if (artifactStore.exists(key)) {
                return ArtifactResult.alreadyExists(key);
            }
            byte[] content = renderer.render(key);
            artifactStore.put(key, content);
            return ArtifactResult.created(key);
        }
    }
}
```

Private final monitor không bị caller giữ hoặc thay identity. Phương án dễ review
nhưng mọi key serialize; không có bounded/interruptible lock acquisition như
`ReentrantLock`.

## Giải pháp 3: stable per-key map cho bounded key set

```java
private final ConcurrentMap<String, ReentrantLock> locks =
        new ConcurrentHashMap<>();

private ReentrantLock lockFor(String key) {
    return locks.computeIfAbsent(key, ignored -> new ReentrantLock());
}
```

Đúng nếu entry sống ít nhất bằng mọi waiter/reference và key set bounded. Không
`remove(key)` mù sau unlock. Với key không bounded, dùng striped locks hoặc một
lock registry/library có reference-count eviction đã được kiểm chứng.

Per-key map cho concurrency tối đa giữa key nhưng tốn memory theo cardinality và
lifecycle phức tạp hơn.

## Giải pháp 4: conditional create tại authoritative store

Đổi contract store để conflict được quyết định atomically:

```java
public interface ArtifactStore {
    PutIfAbsentResult putIfAbsent(String key, byte[] content);
}

public enum PutIfAbsentResult {
    CREATED,
    ALREADY_EXISTS
}
```

```java
public ArtifactResult generateGloballySafe(String key) {
    byte[] content = renderer.render(key);
    PutIfAbsentResult result = artifactStore.putIfAbsent(key, content);

    return switch (result) {
        case CREATED -> ArtifactResult.created(key);
        case ALREADY_EXISTS -> ArtifactResult.alreadyExists(key);
    };
}
```

Store implementation phải dùng conditional request thật sự, không tự triển khai
`exists` rồi `put` trong client. Hai node có thể cùng render, nhưng chỉ một write
được authoritative store chấp nhận.

Nếu put timeout và outcome không rõ, lookup key/idempotency metadata trước retry.
Content nên deterministic hoặc có checksum/version để phát hiện conflict khác dữ
liệu.

## Giải pháp 5: database uniqueness hoặc distributed coordination

Một database idempotency/artifact row với unique `artifact_key` có thể chọn
winner giữa node. Transaction semantics, trạng thái `GENERATING/READY/FAILED` và
recovery phải được thiết kế riêng; không giữ DB transaction mở quanh render dài
một cách mặc định.

Distributed lease chỉ dùng khi authoritative conditional operation không đủ và
cần giảm duplicate work dài. Lease phải có expiry/renewal và fencing token được
resource kiểm tra. Chi tiết thuộc `DIST-001`.

## So sánh các đánh đổi

| Phương án | Correctness scope | Concurrency | Timeout/failure | Memory/lifecycle | Multi-node |
| --- | --- | --- | --- | --- | --- |
| Private final monitor | Một service instance/JVM | Mọi key serialize | Exception-safe với block | Đơn giản | Không |
| Striped `ReentrantLock` | Một service instance/JVM | Song song theo stripe | Bounded/interruptible acquire | Cố định | Không |
| Stable per-key lock map | Một service instance/JVM | Song song theo key | Cần unlock/lifecycle đúng | Tăng theo key | Không |
| Store conditional create | Authoritative store | Các node có thể render song song | Phải xử lý ambiguous timeout | Không lock map | Có |
| DB unique state row | Database boundary | Conflict tại unique key/state | Transaction/recovery phức tạp | Durable rows | Có |
| Lease + fencing | Resource kiểm tra token | Theo lease key | Expiry/partition/renewal khó | Operational state | Có |

## Loser và retry behavior

- Local `tryLock` loser timeout: fail-fast/202 retry theo API; không spin.
- Local winner thấy artifact tồn tại: no-op và trả existing result.
- Conditional-create loser: discard rendered bytes, không overwrite winner.
- Ambiguous store timeout: read/reconcile trước retry.
- Renderer failure: unlock; lần gọi sau có thể thử lại.
- Không retry toàn critical section vô hạn trong request thread.

## Khi nào nên dùng

- Dùng coarse lock cho workload nhỏ và correctness đơn giản.
- Dùng striped lock cho local per-key suppression với key space lớn.
- Dùng stable map khi key bounded và cần concurrency per-key chính xác.
- Dùng conditional create/unique constraint khi nhiều node chia shared resource.
- Dùng lease + fencing khi cần single-flight dài qua node và store không cung cấp
  primitive phù hợp.

## Lưu ý khi áp dụng thực tế

- Metric: lock wait/timeout, render duration, conditional conflict, duplicate
  render, store ambiguous timeout và stripe contention.
- Không log content nhạy cảm; log key hash, correlation ID, node ID và checksum.
- Thread dump/lock owner diagnostics khi wait tăng.
- Không giữ lock qua async thread boundary.
- Shutdown phải ngừng nhận generation mới và chờ bounded in-flight work.
- Test ít nhất hai service instance dùng chung fake store để không nhầm local
  correctness với global correctness.
