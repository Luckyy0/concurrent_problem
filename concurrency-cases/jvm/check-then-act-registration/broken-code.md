# Phản Mẫu Thiết Kế (Anti-Patterns): Lỗ Hổng Cơ Chế Kiểm Tra Rồi Hành Động

## 1. Cấu Trúc Mã Nguồn Dùng Chung ArrayList

Mô hình thiết lập Registry dựa trên hạ tầng `ConcurrentHashMap`, nhưng thay vì khai thác triệt để các đặc tính nguyên tử nội tại, người phát triển lại phân mảnh quy trình tìm kiếm và cấp phát thành các thao tác rời rạc:

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

        // BƯỚC 1: KIỂM TRA (Check) - Phơi bày lỗ hổng cho luồng khác xen ngang
        ManagedResource existing = resources.get(resourceKey);
        if (existing != null) {
            return existing;
        }

        // BƯỚC 2: HÀNH ĐỘNG (Act) - Vùng nguy hiểm khi đa luồng cùng tiến vào
        ManagedResource created = resourceFactory.open(resourceKey);
        
        // BƯỚC 3: CẬP NHẬT (Update) - Đè bẹp trạng thái của các luồng song song
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

Thiết lập chuẩn giao tiếp tối thiểu giữa Factory và Tài nguyên:

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

Đoạn mã rất dễ vượt qua vòng thẩm định tĩnh (Code Review) nhờ vào các cảm quan sau:
- Phương thức `ConcurrentHashMap.get(...)` về cơ bản đáp ứng tính an toàn cao khi đa luồng truy xuất.
- Phương thức `ConcurrentHashMap.put(...)` hoàn toàn đảm bảo phân định rõ các tác vụ cập nhật.
- Các bộ kiểm thử Unit Test (tuần tự) hoàn toàn pass vì luôn phát hiện tài nguyên ở những yêu cầu gọi sau.
- Trạng thái chung cuộc của Map chỉ bảo lưu một Giá trị (Value) ứng với một Khóa (Key), ngụy trang thành công hệ lụy tạo thừa tài nguyên bên trong vùng nhớ Heap.

Điểm đứt gãy chí mạng: Gốc rễ vấn đề nằm ở một **Chuỗi phức hợp nhiều lệnh** (`compound action`), hoàn toàn không nằm ở năng lực chống chịu của một lệnh `get` hay `put` nguyên thủy lẻ tẻ.

## 3. Các Tác Nhân Kích Hoạt Lỗi Hệ Thống

1. Mô hình Registry phân bổ dưới dạng Singleton, trở thành tâm điểm hội tụ của mọi ngả đường truy cập.
2. Hai hay nhiều chủ thể (Actor) đồng khởi yêu cầu đăng ký cùng một Key tại thời điểm Key hoàn toàn vắng bóng.
3. Hàm kiến tạo (Factory) sở hữu thời gian thực thi (Latency) vừa đủ lớn để luồng thứ hai thoải mái lướt qua chốt chặn `get`.
4. Chu trình mở khóa tài nguyên sinh ra phản ứng phụ (Side Effect) và yêu cầu phải có tiến trình hoàn trả khắt khe (Cleanup).

> **Nguyên tắc kỹ thuật:** Bất kỳ khe hở thời gian nào cũng là một lời mời gọi đối với luồng T2 can thiệp. T2 hoàn tất công đoạn "Kiểm tra" trong lúc T1 vẫn đang loay hoay "Khởi tạo", để rồi cả 2 cùng xô đẩy nhau chèn dữ liệu vào Map.

## 4. Phản Mẫu Kỹ Thuật Khắc Phục Sai Lệch

### Ảo Tưởng Cải Biến Collection Bề Mặt
Mã nguồn nguyên thủy vốn dĩ đã triển khai `ConcurrentHashMap`. Loại cấu trúc An Toàn Luồng (Thread-safe) này chỉ thiết lập Rào cản lên *Từng thao tác đơn*; Không có cơ chế tự định hình chuỗi `get → open → put` thành một khối nguyên khối (Atomic operation).

### Thay Thế Bằng `containsKey`
Một số đề xuất biến tướng:
```java
if (!resources.containsKey(resourceKey)) {
    resources.put(resourceKey, resourceFactory.open(resourceKey));
}
```
Lệnh `containsKey` và `put` bản chất vẫn là hai phân đoạn cách ly hoàn toàn. Cửa sổ thời gian tranh chấp thậm chí không mảy may xê dịch.

### Biện Pháp Bán Thời `putIfAbsent` Sau Cấp Phát
```java
ManagedResource created = resourceFactory.open(resourceKey);
ManagedResource existing = resources.putIfAbsent(resourceKey, created);
return existing != null ? existing : created;
```
Bảo chứng cho kết quả rằng Map chỉ giữ lại 1 Value và các Caller đồng nhất tham chiếu, tuy nhiên tiến trình Factory vẫn chạy loạn xạ nhiều lần. Các tài nguyên bị loại (thua cuộc) buộc phải thi hành lệnh đóng. Nếu bỏ quên cơ chế này, vết rò rỉ vẫn tàn phá hệ thống. Quan trọng hơn, không thể bảo toàn đặc tả "Factory chỉ gọi 1 lần".

### Che Chắn Bằng Khai Báo `@Transactional`
Khung `Transactional` phân xử tranh chấp mức độ Database. Nó không có chức năng "Khóa khối" cho đối tượng Java Map và càng không có năng lực Rollback các tài nguyên vật lý đã bị phân bổ trên bộ nhớ máy ảo.
