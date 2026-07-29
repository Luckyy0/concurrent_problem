# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: immutable snapshot qua AtomicReference

Đây là lựa chọn khuyến nghị khi request đọc thường xuyên còn refresh thay toàn bộ
bảng rule. Snapshot gói cả generation và map để hai phần được công bố cùng nhau.

```java
package com.example.routing;

import java.util.Map;

public record RoutingSnapshot(
        long generation,
        Map<String, PaymentRoute> routes
) {
    public RoutingSnapshot {
        routes = Map.copyOf(routes);
    }

    public static RoutingSnapshot empty() {
        return new RoutingSnapshot(0, Map.of());
    }
}
```

```java
package com.example.routing;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.Optional;
import java.util.concurrent.atomic.AtomicReference;

@Service
public class PaymentRoutingRegistry {

    private final RoutingConfigClient configClient;
    private final AtomicReference<RoutingSnapshot> current =
            new AtomicReference<>(RoutingSnapshot.empty());

    public PaymentRoutingRegistry(RoutingConfigClient configClient) {
        this.configClient = configClient;
    }

    public Optional<PaymentRoute> selectRoute(String merchantId) {
        RoutingSnapshot snapshot = current.get();
        return Optional.ofNullable(snapshot.routes().get(merchantId));
    }

    public RoutingSnapshot snapshot() {
        return current.get();
    }

    @Scheduled(fixedDelayString = "${routing.refresh-delay:PT30S}")
    public void refresh() {
        RoutingSnapshot loaded = configClient.loadSnapshot();
        validate(loaded);
        publishIfNewer(loaded);
    }

    boolean publishIfNewer(RoutingSnapshot loaded) {
        while (true) {
            RoutingSnapshot observed = current.get();
            if (loaded.generation() <= observed.generation()) {
                return false;
            }
            if (current.compareAndSet(observed, loaded)) {
                return true;
            }
        }
    }

    private void validate(RoutingSnapshot snapshot) {
        if (snapshot.routes().isEmpty()) {
            throw new IllegalArgumentException("routing snapshot must not be empty");
        }
        boolean mixedGeneration = snapshot.routes().values().stream()
                .anyMatch(route -> route.generation() != snapshot.generation());
        if (mixedGeneration) {
            throw new IllegalArgumentException("route generation mismatch");
        }
    }
}
```

`RoutingConfigClient.loadSnapshot()` phải trả object đã dựng đầy đủ. Constructor
của `RoutingSnapshot` tạo defensive copy bằng `Map.copyOf`; `PaymentRoute` là
immutable `record` nên không có mutation phía sau snapshot.

### Vì sao invariant được bảo vệ

- Reader gọi `current.get()` đúng một lần cho mỗi operation.
- Atomic read/write tạo visibility và safe publication.
- Writer không mutate snapshot đã công bố.
- Compare-and-set là nơi phát hiện conflict giữa hai writer.
- Writer có generation cũ hơn thua ngay và không retry vô hạn.
- Reader không block trong lúc config service đang tải dữ liệu.

> **Nói ngắn gọn:** writer chuẩn bị cả bảng ở hậu trường rồi đổi biển chỉ dẫn một
> lần; request không bao giờ nhìn thấy quá trình lắp ráp.

### Xử lý lỗi và timeout

Config client phải có connect/read timeout. Nếu load hoặc validate thất bại,
`publishIfNewer` chưa chạy nên snapshot cũ vẫn phục vụ request. Retry thuộc refresh
workflow và cần backoff; không retry ngay trong request path.

Nếu nhiều writer cùng tải, compare-and-set có thể thất bại vì writer khác đã
publish. Vòng lặp đọc lại generation: snapshot cũ hơn dừng, snapshot mới hơn thử
CAS lần nữa.

## Giải pháp 2: volatile immutable snapshot

Khi chỉ có một writer hoặc không cần CAS để bảo vệ generation tăng đơn điệu, một
field `volatile` đơn giản hơn:

```java
@Service
public class VolatileSnapshotRoutingRegistry {

    private volatile RoutingSnapshot current = RoutingSnapshot.empty();

    public Optional<PaymentRoute> selectRoute(String merchantId) {
        RoutingSnapshot snapshot = current;
        return Optional.ofNullable(snapshot.routes().get(merchantId));
    }

    public void refresh(RoutingSnapshot loaded) {
        RoutingSnapshot validated = validateAndCopy(loaded);
        current = validated;
    }

    private RoutingSnapshot validateAndCopy(RoutingSnapshot loaded) {
        if (loaded.routes().isEmpty()) {
            throw new IllegalArgumentException("routing snapshot must not be empty");
        }
        return new RoutingSnapshot(loaded.generation(), loaded.routes());
    }
}
```

Volatile write công bố toàn bộ object graph đã được dựng trước đó, còn volatile
read lấy snapshot mới nhất theo memory model. Tuy nhiên, check generation rồi gán
là một compound action; nếu có nhiều writer, cần lock hoặc `AtomicReference` CAS.

## Giải pháp 3: ConcurrentHashMap cho update độc lập theo key

Nếu mỗi merchant có vòng đời độc lập và nghiệp vụ chấp nhận các key được cập
nhật ở thời điểm khác nhau, dùng `ConcurrentHashMap`:

```java
@Service
public class PerMerchantRoutingRegistry {

    private final ConcurrentMap<String, PaymentRoute> routes =
            new ConcurrentHashMap<>();

    public Optional<PaymentRoute> selectRoute(String merchantId) {
        return Optional.ofNullable(routes.get(merchantId));
    }

    public void upsert(String merchantId, PaymentRoute route) {
        routes.put(merchantId, route);
    }

    public void remove(String merchantId) {
        routes.remove(merchantId);
    }

    public List<String> enabledMerchantsApproximation() {
        return routes.entrySet().stream()
                .filter(entry -> entry.getValue().enabled())
                .map(Map.Entry::getKey)
                .toList();
    }
}
```

Code này bảo vệ cấu trúc map và từng operation theo key. Iterator không ném
`ConcurrentModificationException`, nhưng là weakly consistent: kết quả có thể
chứa một số update mới và bỏ qua update khác. Vì vậy method đặt tên
`enabledMerchantsApproximation` để contract không giả vờ trả snapshot chính xác.

Không dùng phương án này cho invariant “toàn bộ bảng cùng generation”, trừ khi
thiết kế thêm versioning và reader protocol tương ứng.

## Giải pháp 4: ReentrantReadWriteLock cho mutable state

Khi cần mutate nhiều cấu trúc liên quan và reader cần một view nhất quán, tất cả
đường truy cập có thể dùng cùng read-write lock:

```java
@Service
public class LockedRoutingRegistry {

    private final Map<String, PaymentRoute> routes = new HashMap<>();
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public Optional<PaymentRoute> selectRoute(String merchantId) {
        lock.readLock().lock();
        try {
            return Optional.ofNullable(routes.get(merchantId));
        } finally {
            lock.readLock().unlock();
        }
    }

    public Map<String, PaymentRoute> snapshot() {
        lock.readLock().lock();
        try {
            return Map.copyOf(routes);
        } finally {
            lock.readLock().unlock();
        }
    }

    public void replaceAll(Map<String, PaymentRoute> loaded) {
        Map<String, PaymentRoute> validated = Map.copyOf(loaded);
        lock.writeLock().lock();
        try {
            routes.clear();
            routes.putAll(validated);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

Load remote data và validate diễn ra trước khi lấy write lock. Writer chỉ giữ
lock trong đoạn thay state ngắn. Reader block trong lúc replace, nhưng không thấy
state trung gian. Mọi method, kể cả diagnostic và test helper, phải tuân thủ cùng
lock.

## So sánh các đánh đổi

| Phương án | Correctness phù hợp | Reader | Writer/contention | Multi-instance | Độ phức tạp vận hành |
| --- | --- | --- | --- | --- | --- |
| `AtomicReference` + immutable snapshot | Snapshot toàn bảng, generation tăng đơn điệu với CAS | Không block, một atomic read | Tốn copy khi refresh; writer conflict được phát hiện | Chỉ bảo vệ từng JVM | Thấp đến vừa |
| `volatile` + immutable snapshot | Snapshot toàn bảng với single writer | Không block | Không tự bảo vệ check-then-set giữa nhiều writer | Chỉ bảo vệ từng JVM | Thấp |
| `ConcurrentHashMap` | Từng key độc lập | Concurrent, iterator weakly consistent | Contention phân tán theo key | Chỉ bảo vệ từng JVM | Thấp |
| `ReentrantReadWriteLock` | Compound state và iteration nhất quán | Có thể block | Writer chờ mọi reader; giữ lock sai làm tăng latency | Chỉ bảo vệ từng JVM | Vừa |
| `synchronized` | Đúng nếu mọi access cùng monitor | Block theo một lock chung | Đơn giản nhưng serialize mạnh | Chỉ bảo vệ từng JVM | Thấp |

Không có con số throughput hoặc latency chung cho mọi workload. Cần đo với tỷ lệ
read/write, kích thước snapshot và thời gian copy thực tế.

## Khi nào nên dùng

- Chọn `AtomicReference` snapshot cho bảng cấu hình read-heavy, replace-all và
  cần loại bỏ stale writer.
- Chọn `volatile` snapshot khi publisher thật sự là single writer.
- Chọn `ConcurrentHashMap` khi semantics là per-key, không phải snapshot.
- Chọn read-write lock khi state mutable phức tạp hoặc nhiều collection phải đổi
  cùng nhau.
- Chuyển invariant sang database/configuration protocol khi nhiều node phải
  thống nhất cùng generation.

## Lưu ý khi áp dụng thực tế

- Đặt giới hạn kích thước snapshot trước `Map.copyOf` để tránh memory spike.
- Không log toàn bộ route table; log generation, số entry, checksum và thời gian
  load/validate/publish.
- Theo dõi `current_generation`, `refresh_success_total`,
  `refresh_failure_total`, `stale_publish_rejected_total` và tuổi snapshot.
- Alert khi snapshot quá cũ hoặc liên tiếp bị refresh failure.
- Không mutate `PaymentRoute` sau publish; nếu value chứa list/map, tạo defensive
  copy cho các cấu trúc lồng nhau.
- Shutdown không cần rollback snapshot local, nhưng config client/executor tự tạo
  phải được đóng theo lifecycle của Spring.
- Canary và nhiều node cần hiển thị generation theo instance để phát hiện lệch
  cấu hình giữa các node.
