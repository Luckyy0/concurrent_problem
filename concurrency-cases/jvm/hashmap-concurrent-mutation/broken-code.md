# Cách triển khai bị lỗi

## Đoạn code mutate HashMap dùng chung

Ví dụ rút gọn dưới đây là code hợp lý thường xuất hiện trong một Spring service.
`HashMap` được tạo cùng bean, nhưng sau đó bị request thread và refresh thread
truy cập đồng thời.

```java
package com.example.routing;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Service
public class PaymentRoutingRegistry {

    private final RoutingConfigClient configClient;
    private final Map<String, PaymentRoute> routes = new HashMap<>();

    public PaymentRoutingRegistry(RoutingConfigClient configClient) {
        this.configClient = configClient;
    }

    public PaymentRoute selectRoute(String merchantId) {
        return routes.get(merchantId);
    }

    public List<String> enabledMerchants() {
        return routes.entrySet().stream()
                .filter(entry -> entry.getValue().enabled())
                .map(Map.Entry::getKey)
                .toList();
    }

    @Scheduled(fixedDelayString = "${routing.refresh-delay:PT30S}")
    public void refresh() {
        Map<String, PaymentRoute> loaded = configClient.loadAll();

        routes.clear();
        routes.putAll(loaded);
    }
}
```

Các type hỗ trợ có thể rất nhỏ:

```java
package com.example.routing;

import java.util.Map;

public interface RoutingConfigClient {
    Map<String, PaymentRoute> loadAll();
}

public record PaymentRoute(
        String provider,
        boolean enabled,
        long generation
) {}
```

`routes` là `final` chỉ có nghĩa reference không được trỏ sang map khác. Nó không
làm nội dung của `HashMap` trở nên immutable hoặc thread-safe.

## Vì sao code trông có vẻ hợp lý

- Spring chỉ tạo một registry nên developer cho rằng state có một owner.
- `loadAll()` chạy xong trước khi `clear()` nên dữ liệu nguồn đã đầy đủ.
- từng dòng `get`, `clear` và `putAll` trông đơn giản;
- refresh ít xảy ra hơn request nên bug khó xuất hiện trên máy cá nhân;
- unit test tuần tự luôn gọi `refresh()` xong rồi mới đọc.

Vấn đề là singleton đồng nghĩa cùng object được nhiều request thread chia sẻ. Một
owner object không đồng nghĩa một owner thread.

> **Nói ngắn gọn:** `final HashMap` vẫn là một mutable object dùng chung; Spring
> không đặt lock quanh method của singleton.

## Một cách sửa khác vẫn bị unsafe publication

Developer có thể tránh mutate tại chỗ bằng cách tạo map mới rồi gán reference:

```java
@Service
public class UnsafeSnapshotRoutingRegistry {

    private Map<String, PaymentRoute> routes = Map.of();

    public PaymentRoute selectRoute(String merchantId) {
        return routes.get(merchantId);
    }

    public void refresh(Map<String, PaymentRoute> loaded) {
        Map<String, PaymentRoute> next = Map.copyOf(loaded);
        routes = next;
    }
}
```

Snapshot mới không bị mutate, nhưng read và write trên field `routes` vẫn là một
data race. Không có `volatile`, lock hoặc atomic variable để tạo quan hệ
happens-before giữa writer và reader. Java Memory Model không bảo đảm request sẽ
nhìn thấy reference mới đúng lúc.

Đây là lỗi **công bố không an toàn** (`unsafe publication`) của mỗi snapshot mới;
nó khác với việc Spring đã công bố bean an toàn lúc startup.

## Điều kiện để lỗi xuất hiện

1. cùng một registry instance phục vụ nhiều thread;
2. ít nhất một writer refresh trong khi có reader;
3. writer mutate map tại chỗ hoặc gán snapshot qua field không được đồng bộ;
4. reader thực hiện `get`, iterate hoặc tạo diagnostic snapshot mà không dùng
   cùng cơ chế coordination;
5. test không điều phối đúng cửa sổ tranh chấp nên lỗi chỉ xuất hiện ngẫu nhiên.

## Những cách sửa tưởng đúng nhưng chưa đủ

### Chỉ thêm final cho field

`final` bảo vệ reference được thiết lập trong constructor. Nó không serialize
các lần `clear()` và `putAll()` sau startup.

### Chỉ thêm volatile cho HashMap mutable

`volatile Map` làm cho việc đọc và ghi reference có visibility, nhưng không làm
các mutation bên trong cùng một `HashMap` an toàn. Nếu writer vẫn gọi `clear()`
và `putAll()` trên object hiện tại, reader vẫn thấy state trung gian.

### Đổi sang Collections.synchronizedMap

Từng method được khóa, nhưng chuỗi `clear()` rồi `putAll()` vẫn là hai operation
riêng. Việc iterate còn yêu cầu caller khóa thủ công trên đúng map trong suốt
vòng lặp. Chỉ một đường đọc quên lock là contract bị phá vỡ.

### Đổi sang ConcurrentHashMap

`ConcurrentHashMap` bảo vệ cấu trúc và cho phép read/write đồng thời. Tuy nhiên,
`clear()` cộng nhiều lần `put()` không trở thành một operation atomic. Reader có
thể thấy generation mới chỉ được cài một phần; iterator của nó có semantics nhất
quán yếu chứ không phải snapshot.

### Chỉ thêm Transactional

`@Transactional` điều phối database transaction qua thread-bound connection.
Nó không đặt monitor hoặc memory barrier cho `HashMap` trong JVM.

### Bắt ConcurrentModificationException rồi retry

Iterator fail-fast chỉ là cơ chế phát hiện best-effort, không phải synchronization
contract. Không có exception không có nghĩa kết quả đã nhất quán; retry cũng có
thể tiếp tục đụng writer hoặc ghép dữ liệu từ hai generation.
