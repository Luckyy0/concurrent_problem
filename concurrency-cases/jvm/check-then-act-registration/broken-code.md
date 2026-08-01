# Phản Mẫu Thiết Kế (Anti-Patterns): Lỗ Hổng Cơ Chế Kiểm Tra Rồi Hành Động

## 1. Cấu Trúc Mã Nguồn Dùng Chung ArrayList

Mã code dưới đây xài `ConcurrentHashMap` để làm Registry. Nghe qua thì có vẻ xịn, nhưng thay vì tận dụng sức mạnh của nó, code lại tự chia quy trình ra thành mấy bước rời rạc:

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

        // BƯỚC 1: KIỂM TRA (Check) - Dễ bị luồng khác chen ngang vô đây nè
        ManagedResource existing = resources.get(resourceKey);
        if (existing != null) {
            return existing;
        }

        // BƯỚC 2: HÀNH ĐỘNG (Act) - Chỗ nguy hiểm nếu nhiều luồng cùng phi vào
        ManagedResource created = resourceFactory.open(resourceKey);
        
        // BƯỚC 3: CẬP NHẬT (Update) - Đè bẹp hết kết quả của mấy luồng khác
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

Đây là phần interface đơn giản của Factory và Resource:

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

## 2. Ảo Giác An Toàn Chết Người

Nhìn qua đoạn code trên, review code tĩnh rất dễ bỏ sót lỗi này vì một số lý do:
- Hàm `ConcurrentHashMap.get(...)` lúc đọc nó an toàn lắm.
- Hàm `ConcurrentHashMap.put(...)` ghi vào cũng rất rạch ròi, chẳng có vấn đề gì.
- Chạy Unit Test (chạy 1 luồng) thì xanh lè xanh lẹt, pass hết.
- Cuối cùng thì trong Map cũng chỉ có 1 cái value cho 1 key, nên ta dễ bị lừa là mọi thứ đang ổn mà không biết là trong Heap đang có cả rổ tài nguyên bị thừa mứa.

Điểm chết chóc ở đây là: Lỗi nó nằm ở **cả một chuỗi logic kết hợp lại** (`compound action`). Dù từng hàm `get` hay `put` có xịn sò cỡ nào thì ghép lại hở hang vẫn hoàn hở hang!

## 3. Các Tác Nhân Kích Hoạt Lỗi Hệ Thống

1. Cái Registry của mình là Singleton, tức là mọi con đường đều dẫn về một mối.
2. Từ hai luồng trở lên cùng nhảy vào xin cấp chung 1 Key ngay lúc Key đó đang chưa có.
3. Cái hàm Factory chạy đủ chậm để luồng số 2 kịp lướt qua cái chốt `get` lúc luồng 1 chưa kịp `put`.
4. Việc mở tài nguyên sinh ra side effect và bắt buộc phải có cơ chế dọn dẹp đàng hoàng.

> **Nguyên tắc kỹ thuật:** Bất cứ khe hở thời gian nào cũng là miếng mồi ngon cho luồng khác xen vào. Luồng T2 check xong nhảy vào khởi tạo cùng lúc với T1, rồi cả 2 anh đua nhau `put` đè dữ liệu vào Map.

## 4. Phản Mẫu Kỹ Thuật Khắc Phục Sai Lệch

### Ảo Tưởng Cải Biến Collection Bề Mặt
Cấu trúc `ConcurrentHashMap` sinh ra chỉ bảo vệ được *Từng thao tác đơn lẻ*. Nó không thể hiểu được ý định của bạn muốn biến nguyên cái chuỗi `get → open → put` thành một cục nguyên khối (atomic) đâu.

### Thay Thế Bằng `containsKey`
Nhiều bạn nghĩ ra cách này:
```java
if (!resources.containsKey(resourceKey)) {
    resources.put(resourceKey, resourceFactory.open(resourceKey));
}
```
Trời ơi, `containsKey` và `put` nó vẫn là hai bước tách rời! Cái khe hở giữa 2 hàm nó vẫn chình ình ra đó, chẳng giải quyết được tí nào.

### Biện Pháp Bán Thời `putIfAbsent` Sau Cấp Phát
Có người sửa thế này:
```java
ManagedResource created = resourceFactory.open(resourceKey);
ManagedResource existing = resources.putIfAbsent(resourceKey, created);
return existing != null ? existing : created;
```
Cách này đảm bảo Map giữ đúng 1 giá trị và tất cả các luồng sẽ nhận chung một cái tham chiếu. NHƯNG, Factory sẽ bị gọi đi gọi lại vô số lần! Những cái tài nguyên sinh ra mà thua cuộc (không được put vào map) phải được dọn dẹp (`close`). Quên dọn dẹp là lại dính rò rỉ bộ nhớ. Đã thế lại còn phá luôn quy tắc "Factory chỉ được gọi 1 lần".

### Che Chắn Bằng Khai Báo `@Transactional`
Nhiều bác đâm đầu vào dùng Annotation `@Transactional` của Database. Xin thưa là cái này sinh ra để xử lý db transaction, nó không hề "khoá" đối tượng Map trên Java lại đâu. Và nó cũng chả có chức năng Rollback để tự động huỷ mấy cái kết nối hay luồng đã lỡ tạo ra trên máy ảo. Đừng lạm dụng nó sai mục đích nhé!
