# Cách triển khai bị lỗi

## Đoạn code gây điều kiện tranh chấp

Registry dùng `ConcurrentHashMap`, nhưng việc tìm và tạo resource được tách thành
nhiều thao tác:

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

        ManagedResource existing = resources.get(resourceKey);
        if (existing != null) {
            return existing;
        }

        ManagedResource created = resourceFactory.open(resourceKey);
        resources.put(resourceKey, created);
        return created;
    }

    public ManagedResource find(String resourceKey) {
        return resources.get(resourceKey);
    }

    public int size() {
        return resources.size();
    }
}
```

Factory và resource có contract tối thiểu:

```java
package com.example.registry;

@FunctionalInterface
public interface ManagedResourceFactory {

    ManagedResource open(String resourceKey);
}
```

```java
package com.example.registry;

public interface ManagedResource extends AutoCloseable {

    String id();

    @Override
    void close();
}
```

## Vì sao đoạn code trông có vẻ hợp lý

- `ConcurrentHashMap.get(...)` an toàn khi nhiều thread gọi đồng thời.
- `ConcurrentHashMap.put(...)` cũng an toàn khi nhiều thread gọi đồng thời.
- Unit test tuần tự luôn tìm thấy resource sau lần đăng ký đầu tiên.
- Map cuối cùng chỉ có một value cho mỗi key, nên lỗi tạo dư resource dễ bị che
  giấu.

Vấn đề nằm ở **chuỗi nhiều bước** (`compound action`), không nằm ở một lần `get`
hay `put` riêng lẻ.

## Điều kiện để lỗi xuất hiện

1. Registry là singleton hoặc được nhiều thread chia sẻ.
2. Hai actor đăng ký cùng một key khi key chưa tồn tại.
3. Factory mất đủ thời gian để hai actor cùng đi qua bước `get`.
4. Việc mở resource có side effect hoặc cần cleanup.

> **Nói ngắn gọn:** thread T2 có thể chạy trọn bước kiểm tra trong lúc T1 đang
> mở resource nhưng chưa kịp đưa nó vào map.

## Các cách sửa tưởng đúng nhưng chưa đủ

### Đổi HashMap thành ConcurrentHashMap

Code lỗi đã dùng `ConcurrentHashMap`. Collection an toàn cho nhiều luồng chỉ bảo
vệ từng operation; nó không tự gộp `get → open → put` thành một operation.

### Chỉ dùng containsKey

```java
if (!resources.containsKey(resourceKey)) {
    resources.put(resourceKey, resourceFactory.open(resourceKey));
}
```

`containsKey` và `put` vẫn là hai thao tác riêng. Cửa sổ tranh chấp thậm chí rõ
hơn nhưng bản chất không thay đổi.

### Chỉ dùng putIfAbsent sau khi tạo

```java
ManagedResource created = resourceFactory.open(resourceKey);
ManagedResource existing = resources.putIfAbsent(resourceKey, created);
return existing != null ? existing : created;
```

Map chỉ giữ một value và mọi caller có thể nhận cùng value, nhưng factory vẫn có
thể chạy hai lần. Resource thua cuộc phải được đóng; nếu không, lỗi rò rỉ vẫn
còn. Cách này không bảo vệ invariant “factory chỉ chạy một lần”.

### Chỉ thêm @Transactional

`@Transactional` quản lý database transaction. Nó không biến chuỗi thao tác trên
Java map thành một thao tác nguyên tử và không khóa Java object.
