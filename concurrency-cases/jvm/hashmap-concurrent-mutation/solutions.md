# Giải Pháp Kiến Trúc Tối Ưu và Ma Trận Đánh Đổi

## 1. Phương Án Lõi: Bản Chụp Bất Biến Qua AtomicReference (Immutable Snapshot)

Đây là tuyệt chiêu tối thượng cho hệ thống đọc liên tục (Read-heavy) và yêu cầu cập nhật nguyên cả bảng dữ liệu. Chúng ta gom cả phiên bản (Generation) và dữ liệu Map thành một khối không thể tách rời (Khối Nguyên Tử).

```java
package com.example.routing;

import java.util.Map;

public record RoutingSnapshot(
        long generation,
        Map<String, PaymentRoute> routes
) {
    public RoutingSnapshot {
        // Tái tạo Bản sao Phòng thủ Bất Biến (Defensive Copy)
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
    // Điểm Hiệu Lực Trung Tâm (Linearization Point) - Nơi chốt sổ
    private final AtomicReference<RoutingSnapshot> current =
            new AtomicReference<>(RoutingSnapshot.empty());

    public PaymentRoutingRegistry(RoutingConfigClient configClient) {
        this.configClient = configClient;
    }

    public Optional<PaymentRoute> selectRoute(String merchantId) {
        // Bào đọc: Lấy Tham chiếu Duy nhất, Phục vụ Toàn Hành Trình
        RoutingSnapshot snapshot = current.get();
        return Optional.ofNullable(snapshot.routes().get(merchantId));
    }

    public RoutingSnapshot snapshot() {
        return current.get();
    }

    @Scheduled(fixedDelayString = "${routing.refresh-delay:PT30S}")
    public void refresh() {
        RoutingSnapshot loaded = configClient.loadSnapshot();
        validate(loaded); // Tường Lửa Thẩm định Dữ liệu Rác
        publishIfNewer(loaded);
    }

    boolean publishIfNewer(RoutingSnapshot loaded) {
        while (true) {
            RoutingSnapshot observed = current.get();
            // Từ chối thẳng thừng nếu phiên bản cũ hơn
            if (loaded.generation() <= observed.generation()) {
                return false;
            }
            // Mệnh lệnh Xuất Bản Nguyên Tử (chỉ ai nhanh chân mới được)
            if (current.compareAndSet(observed, loaded)) {
                return true;
            }
        }
    }

    private void validate(RoutingSnapshot snapshot) {
        if (snapshot.routes().isEmpty()) {
            throw new IllegalArgumentException("Khước từ Bản Chụp Trống Rỗng");
        }
        boolean mixedGeneration = snapshot.routes().values().stream()
                .anyMatch(route -> route.generation() != snapshot.generation());
        if (mixedGeneration) {
            throw new IllegalArgumentException("Khước từ Thế Hệ Lai Tạp");
        }
    }
}
```

### Tại sao khối bất biến lại vô đối?
- **Sức mạnh của `current.get()`:** Luồng đọc chỉ bốc đúng 1 lần duy nhất, lấy trọn gói dữ liệu để xài xuyên suốt, vĩnh viễn không lo ai đó sờ vào làm rác dữ liệu.
- **Xuất bản cực an toàn:** Việc đổi tham chiếu biến mọi thứ trở nên minh bạch và an toàn ngay lập tức.
- **Dữ liệu "Cứng như đá":** Một khi đã xuất bản thì Luồng ghi hết cửa sửa đổi.
- **Lưới bảo vệ `compare-and-set` (CAS):** Nếu có nhiều luồng ghi tranh nhau cập nhật, lệnh CAS sẽ tát văng mấy luồng mang dữ liệu lỗi thời.
- **Luồng đọc phi ầm ầm:** Hệ thống không bao giờ bị đóng băng khi luồng đọc lao vào, bất chấp lúc đó tải đang cao cỡ nào.

> **Nguyên tắc kỹ thuật:** Cứ tưởng tượng nhà thầu (Writer) xây nguyên một cái nhà xưởng mới ở nơi khác. Khi xong xuôi, họ chỉ tốn 1 giây để tráo biển số nhà (CAS). Khách (Request) cứ thế mà vào nhà xưởng, không mảy may hít phải hạt bụi nào từ lúc thi công.

## 2. Phương Án Trọng Tài Độc Nhất: Bản Chụp Bất Biến Volatile

Nếu bạn thề thốt là hệ thống của bạn luôn luôn chỉ có ĐÚNG 1 luồng ghi cập nhật dữ liệu, thì hãy xài cách này cho gọn nhẹ, khỏi cần CAS loằng ngoằng.

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
        current = validated; // Lệnh Xuất Bản Đơn Tốc bằng Volatile
    }
// ...
}
```

Dùng `volatile` giúp luồng đọc luôn thấy được đồ tươi mới nhất. Đổi lại, chỉ cần xuất hiện thêm một luồng ghi thứ 2 chen ngang, hệ thống của bạn sẽ sụp đổ cái một!

## 3. Chấp Nhận Khuyết Tật Tập Thể: Mô Hình ConcurrentHashMap

Cách này dành cho lúc bạn chỉ quan tâm cấu hình từng đối tác lẻ tẻ (Per-Merchant) và "buông xuôi" cái luật bắt buộc MỌI quy tắc phải cùng 1 Thế Hệ.

```java
@Service
public class PerMerchantRoutingRegistry {

    private final ConcurrentMap<String, PaymentRoute> routes =
            new ConcurrentHashMap<>();

    public Optional<PaymentRoute> selectRoute(String merchantId) {
        return Optional.ofNullable(routes.get(merchantId));
    }

    public void upsert(String merchantId, PaymentRoute route) {
        routes.put(merchantId, route); // Đóng Cọc Từng Mũi Độc Lập
    }

    public List<String> enabledMerchantsApproximation() {
        return routes.entrySet().stream()
                .filter(entry -> entry.getValue().enabled())
                .map(Map.Entry::getKey)
                .toList(); // Cảnh báo: Kết quả Tạp Nham (Weakly Consistent)
    }
}
```

Nó giúp Map không bị sập, nhưng đổi lại khi liệt kê toàn bộ, kết quả sẽ là một nồi lẩu thập cẩm cũ mới lẫn lộn (Approximation). Bạn nên dán nhãn rõ để mọi người dùng cái hàm này biết mà liệu.

## 4. Xích Cổ Khóa Trọng Lực: ReentrantReadWriteLock

Đây là đòn "bất chấp tất cả" để bảo vệ an toàn: Bắt cả luồng đọc và luồng ghi dùng chung 1 cái khóa.

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

    public void replaceAll(Map<String, PaymentRoute> loaded) {
        Map<String, PaymentRoute> validated = Map.copyOf(loaded); // Xác Thực Ngoài Vùng Khóa
        lock.writeLock().lock(); // Xích Chặt Toàn Cục Tuyến Đọc
        try {
            routes.clear();
            routes.putAll(validated);
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

An toàn thì vô đối, nhưng luồng đọc phải "nằm chờ" dài cổ mỗi khi luồng ghi làm việc. Dưới áp lực hệ thống cao, cách này sẽ bóp nghẹt hiệu năng của bạn.

## 5. Ma Trận Đánh Đổi Chiến Lược (Trade-offs)

| Giải Pháp Kỹ Thuật | Đặc Tính Trọng Tâm | Tác Động Tuyến Đọc | Tác Động Tuyến Ghi/Nghẽn | Phạm Vi Khai Thác |
| --- | --- | --- | --- | --- |
| `AtomicReference` (Khuyên Dùng) | Tạo Bản Chụp Bất Biến cho Toàn Bảng, Thế Hệ luôn tăng dần | Nhanh như chớp (1 Lệnh) | Tốn chút xíu chi phí copy tạo mới lúc Ghi | Nội Bộ 1 Máy Ảo JVM |
| `volatile` Snapshot | Tương tự bản chụp nhưng Không chịu nổi Đa Ghi | Nhanh như chớp | Toang ngay nếu có Đa Tuyến Ghi | Nội Bộ 1 Máy Ảo JVM |
| `ConcurrentHashMap` | Thay đổi lẻ tẻ từng Khóa, Bỏ luật "Thế Hệ đồng nhất" | Thấy lờ mờ, Đọc trộn cũ mới | Chống nghẽn cực ngon khi update lẻ tẻ | Nội Bộ 1 Máy Ảo JVM |
| `ReadWriteLock` | Khóa chết trạng thái để ép buộc an toàn | Chịu cảnh xếp hàng đợi Khóa Ghi | Bóp nghẹt hệ thống lúc tải lớn | Nội Bộ 1 Máy Ảo JVM |
| Kiến Trúc Phân Tán (Database/Protocol) | Dập tắt khác biệt trên Toàn mạng lưới | Tốn nhiều tài nguyên mạng | Quản lý được giao dịch cực kỳ phức tạp | Cả hệ thống Microservices |

## 6. Sách Lược Vận Hành (Production Imperatives)

Khi đưa lên chạy thật (Production), nhớ kỹ:
- Tránh vỡ RAM bằng cách đặt giới hạn độ to của Map ở chỗ `Map.copyOf`.
- Đừng cố in log nguyên cái Map to đùng; chỉ in Thế Hệ, số lượng mục hoặc Checksum thôi.
- Đặt biểu đồ theo dõi các bản cập nhật quá cũ bị từ chối (`stale_publish_rejected_total`).
- Gắn cờ Cảnh báo (Alert) réo ầm lên nếu quá trình Tải Dữ Liệu thất bại liên tục.
- Khóa tay mọi cố gắng đục khoét (Mutate) cái cấu trúc `PaymentRoute` sau khi nó đã lên sóng.
- Ở tầm mức hệ thống phân tán, nên làm biểu đồ xem có máy chủ nào rớt nhịp, không theo kịp Thế hệ hiện hành hay không.
