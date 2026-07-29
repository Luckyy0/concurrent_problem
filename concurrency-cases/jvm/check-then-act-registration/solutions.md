# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp 1: computeIfAbsent cho factory nhanh

Khi factory chạy nhanh, không thực hiện remote I/O và không gọi ngược vào cùng
map, dùng atomic API có sẵn của `ConcurrentHashMap`:

```java
package com.example.registry;

import java.util.Objects;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import org.springframework.stereotype.Service;

@Service
public final class ManagedResourceRegistry {

    private final ConcurrentMap<String, ManagedResource> resources =
            new ConcurrentHashMap<>();
    private final ManagedResourceFactory resourceFactory;

    public ManagedResourceRegistry(ManagedResourceFactory resourceFactory) {
        this.resourceFactory = resourceFactory;
    }

    public ManagedResource register(String resourceKey) {
        Objects.requireNonNull(resourceKey, "resourceKey");

        return resources.computeIfAbsent(
                resourceKey,
                resourceFactory::open
        );
    }

    public ManagedResource find(String resourceKey) {
        return resources.get(resourceKey);
    }

    public int size() {
        return resources.size();
    }
}
```

## Tại sao computeIfAbsent hoạt động

`ConcurrentHashMap.computeIfAbsent(...)` thực hiện việc kiểm tra key và công bố
value như một atomic map operation. Với một key đang vắng mặt:

1. T1 bắt đầu tính value cho `tenant-a`;
2. T2 đăng ký cùng key không tự chạy một chuỗi `get → open → put` khác;
3. T2 chờ hoặc nhận value đã được T1 công bố;
4. cả hai caller nhận cùng `ManagedResource`.

Mapping function được áp dụng tối đa một lần cho key trong operation thành công.
Nếu function throw hoặc trả `null`, map không ghi mapping và caller sau có thể
thử lại.

> **Nói ngắn gọn:** thay vì để application tự ghép ba bước, ta giao toàn bộ
> quyết định “chưa có thì tạo” cho một atomic API của map.

## Giới hạn của computeIfAbsent

Không đặt công việc chậm hoặc không kiểm soát được vào mapping function:

- remote network call;
- chờ lock khác trong thời gian dài;
- callback cập nhật lại cùng map;
- factory có dependency cycle;
- operation không có timeout.

`ConcurrentHashMap` có thể block các update cạnh tranh trong lúc mapping function
chạy. Function cũng không được cập nhật đệ quy cùng key; Java có thể phát hiện và
ném `IllegalStateException`.

## Giải pháp 2: FutureTask placeholder cho factory chậm

Nếu mở resource tốn thời gian, đưa một placeholder vào map trước. Chỉ actor đưa
placeholder thành công mới chạy factory; actor khác chờ cùng kết quả. Factory
phải tự cấu hình timeout cho network/file operation của nó.

```java
package com.example.registry;

import java.util.Objects;
import java.util.concurrent.CancellationException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.FutureTask;
import org.springframework.stereotype.Service;

@Service
public final class FutureManagedResourceRegistry {

    private final ConcurrentMap<String, Future<ManagedResource>> registrations =
            new ConcurrentHashMap<>();
    private final ManagedResourceFactory resourceFactory;

    public FutureManagedResourceRegistry(
            ManagedResourceFactory resourceFactory
    ) {
        this.resourceFactory = resourceFactory;
    }

    public ManagedResource register(String resourceKey) {
        Objects.requireNonNull(resourceKey, "resourceKey");

        while (true) {
            Future<ManagedResource> future = registrations.get(resourceKey);

            if (future == null) {
                FutureTask<ManagedResource> candidate = new FutureTask<>(
                        () -> resourceFactory.open(resourceKey)
                );

                future = registrations.putIfAbsent(resourceKey, candidate);

                if (future == null) {
                    future = candidate;
                    candidate.run();
                }
            }

            try {
                return future.get();
            } catch (CancellationException exception) {
                registrations.remove(resourceKey, future);
            } catch (InterruptedException exception) {
                Thread.currentThread().interrupt();
                throw new ResourceRegistrationException(
                        "Interrupted while registering " + resourceKey,
                        exception
                );
            } catch (ExecutionException exception) {
                registrations.remove(resourceKey, future);
                throw new ResourceRegistrationException(
                        "Failed to register " + resourceKey,
                        exception.getCause()
                );
            }
        }
    }
}
```

```java
package com.example.registry;

public final class ResourceRegistrationException extends RuntimeException {

    public ResourceRegistrationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Tại sao placeholder hoạt động

- `putIfAbsent` chọn đúng một `FutureTask` cho key.
- Chỉ thread thắng gọi `candidate.run()`.
- Các thread khác gọi `get()` trên cùng future và nhận cùng resource.
- Factory chạy ngoài internal map computation nên không giữ vùng tính toán của
  `computeIfAbsent` trong suốt remote operation.
- Nếu factory thất bại, entry được remove có điều kiện; lần gọi sau có thể tạo
  placeholder mới và retry toàn bộ operation.

Không dùng retry vô hạn tại registry. Retry policy phải giới hạn số lần, có
backoff và chỉ retry lỗi phù hợp. Factory phải đóng phần resource đã tạo dở dang
trước khi throw.

## Phương án 3: putIfAbsent và đóng resource thua cuộc

Nếu factory chạy nhiều lần vẫn an toàn và resource có thể đóng ngay:

```java
public ManagedResource register(String resourceKey) {
    ManagedResource created = resourceFactory.open(resourceKey);
    ManagedResource existing = resources.putIfAbsent(
            resourceKey,
            created
    );

    if (existing != null) {
        created.close();
        return existing;
    }

    return created;
}
```

Cách này bảo đảm map chỉ quản lý một resource và mọi caller trả về resource thắng.
Nó không bảo đảm factory chỉ chạy một lần. Cleanup phải an toàn ngay cả khi
`close()` gặp lỗi; production code cần ghi nhận và xử lý cleanup failure.

## Phương án 4: synchronized

```java
public synchronized ManagedResource register(String resourceKey) {
    ManagedResource existing = resources.get(resourceKey);
    if (existing != null) {
        return existing;
    }

    ManagedResource created = resourceFactory.open(resourceKey);
    resources.put(resourceKey, created);
    return created;
}
```

Cách này đúng trong một JVM nếu mọi access dùng cùng monitor, nhưng mọi key bị
serialize chung. Một factory chậm cho `tenant-a` sẽ chặn đăng ký độc lập của
`tenant-b`. Lock cũng không có hiệu lực ở application instance khác.

## So sánh các đánh đổi

| Giải pháp | Mức bảo đảm | Thông lượng/độ trễ | Tranh chấp và retry | Nguy cơ deadlock | Độ phức tạp | Mở rộng nhiều node |
| --- | --- | --- | --- | --- | --- | --- |
| `computeIfAbsent` | Mạnh trong một JVM; factory tối đa một lần cho atomic computation thành công | Tốt nếu factory ngắn; latency tăng nếu factory chậm | Actor cùng key có thể chờ; lỗi cho phép lần sau retry | Thấp nếu function không gọi map/lock khác | Thấp | Không bảo vệ giữa node |
| `FutureTask` placeholder | Mạnh trong một JVM; một factory execution cho placeholder thắng | Tốt cho key khác nhau; caller cùng key chờ cùng future | Có failure removal và bounded retry bên ngoài | Có thể xảy ra nếu factory tạo dependency cycle | Trung bình | Không bảo vệ giữa node |
| `putIfAbsent` + close | Map unique nhưng factory có thể chạy nhiều lần | Tốt, đổi lại tạo resource dư | Không application retry; cần cleanup loser | Thấp | Thấp/trung bình | Không bảo vệ giữa node |
| `synchronized` | Mạnh trong một JVM | Mọi key chạy lần lượt, latency cao hơn | Thread khác block; không retry | Thấp với một monitor, tăng khi factory dùng lock khác | Thấp | Không bảo vệ giữa node |
| Database constraint/coordination | Có thể bảo vệ invariant dùng chung | Thêm I/O và contention tại shared boundary | Phụ thuộc transaction/retry policy | Phụ thuộc lock order | Trung bình/cao | Phù hợp cho invariant toàn hệ thống |

## Khi nào nên dùng

- Chọn `computeIfAbsent` cho object cục bộ, khởi tạo nhanh và không có I/O dài.
- Chọn placeholder future khi khởi tạo đắt và mọi caller phải chia sẻ cùng một
  kết quả.
- Chọn `putIfAbsent` + cleanup khi tạo dư an toàn hơn việc giữ lock trong lúc mở
  resource.
- Chỉ chọn `synchronized` khi registry nhỏ, contention thấp và global lock không
  ảnh hưởng các key độc lập.
- Không chọn distributed lock chỉ để sửa một local registry.

## Lưu ý khi áp dụng thực tế

- Đặt timeout tại factory cho network, file hoặc process operation.
- Factory phải cleanup resource đã mở một phần trước khi throw.
- Theo dõi số lần factory chạy theo key, registry size, active resource count và
  cleanup failure.
- Không dùng mapping function cập nhật đệ quy cùng map.
- Unregister nên dùng conditional remove như `remove(key, expectedResource)` để
  không xóa nhầm resource mới hơn.
- Nếu resource cần reference counting hoặc lease lifecycle, tách thành case
  riêng thay vì nhồi thêm state vào registry này.
- Nếu uniqueness là invariant giữa nhiều node, chuyển enforcement tới database
  hoặc protocol phân tán phù hợp.
