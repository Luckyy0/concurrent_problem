# Giải Pháp Kiến Trúc Tối Ưu và Ma Trận Đánh Đổi

## 1. Phương Án Lõi: Bản Chụp Bất Biến Qua AtomicReference (Immutable Snapshot)

Chiến thuật thượng tôn cho hệ thống Bào Đọc (Read-heavy) và Yêu cầu Tái thiết Toàn Bảng. Cấu trúc đóng gói đồng nhất Metadata (Generation) và Dữ liệu Map làm một Khối Nguyên Tử.

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
    // Điểm Hiệu Lực Trung Tâm (Linearization Point)
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
            // Cự Tuyệt Tàn Nhẫn Kẻ Lỗi Thời
            if (loaded.generation() <= observed.generation()) {
                return false;
            }
            // Mệnh lệnh Xuất Bản Nguyên Tử
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

### Cơ Trí Tối Thượng Của Khối Bất Biến
- Quyền Năng `current.get()`: Đoạt tham chiếu đúng 1 lần cho mọi Đọc/Truy Xuất, Cắt đứt hoàn toàn nguy cơ dính dáng Dữ Liệu Thay Đổi.
- Giao Điểm Nguyên Tử kiến tạo Tường Lửa Khả Kiến (Safe Publication) tuyệt đối.
- Tuyến Ghi bị tước bỏ khả năng Phá Hủy (Mutate) Dữ liệu đã Lên Sóng.
- Lệnh So-Sánh-Và-Gán (CAS) là Lưới Đánh Chặn xung đột của Đa Tuyến Ghi. Kẻ Mang Thế Hệ Cũ Bị Tiêu Diệt Ngay Tức Khắc.
- Tuyến Đọc lướt qua Hàng Đợi, Khước từ Áp lực Nghẽn Đóng Băng trong lúc Client Gánh Dữ Liệu.

> **Nguyên tắc kỹ thuật:** Nhà thầu (Writer) đúc bê tông toàn bộ Công trình ở Phân xưởng, rồi thay biển Hiệu ngay trong Đêm (CAS). Người dân (Request) vĩnh viễn không bao giờ phải chịu trận Bụi bặm của quá trình Xây Dựng Cấu Trúc.

## 2. Phương Án Trọng Tài Độc Nhất: Bản Chụp Bất Biến Volatile

Rút gọn cấu trúc khi Hệ thống Tuyên thệ chỉ Duy trì Độc Tôn 1 Tuyến Ghi, Triệt tiêu Cấu trúc CAS phức tạp.

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
        current = validated; // Lệnh Xuất Bản Đơn Tốc Volatile
    }
// ...
}
```

Phép Ghi `volatile` phát tán Tính Toàn Vẹn Cấu Trúc Khả Kiến Mạng Lưới; Phép Đọc bảo chứng Tươi Mới. Rủi ro: Khối lệnh Khảo sát Thế Hệ đan xen Khối Gán biến đổi thành Hành Vi Phức Hợp (Compound Action); Nếu có Đa Tuyến Ghi (Multi-writer), Khung Sườn Vỡ Vụn Ngay Lập Tức.

## 3. Chấp Nhận Khuyết Tật Tập Thể: Mô Hình ConcurrentHashMap

Áp dụng cho Mô hình Quản Trị Khóa Độc Lập (Per-Merchant) - Buông bỏ Luật "Đồng Nhất Thế Hệ":

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

Bảo toàn Cấu Trúc, Tránh Lỗi Ngoại Lệ, Bù lại Hệ Thống Trả về Khối Dữ Liệu Lai Tạp. Định danh Rạch Ròi Khái Niệm `Approximation` (Ước Lượng) để Caller Không Bị Đánh Tráo Khái Niệm.

## 4. Xích Cổ Khóa Trọng Lực: ReentrantReadWriteLock

Lựa chọn Cực Đoan bảo vệ Tính Khả Biến Cấu Trúc (Mutable State) Bắt buộc: Đóng đinh Tuyến Đọc/Tuyến Ghi vào Chung Lõi Khóa.

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

Tuyến Đọc Phải Hiến Tế Độ Trễ (Block) để đổi lấy Không Gian Nhất Quán Tuyệt Đối. Sự xuất hiện của Khóa làm tê liệt Khả năng Xử lý Đỉnh Tải (Peak).

## 5. Ma Trận Đánh Đổi Chiến Lược (Trade-offs)

| Giải Pháp Kỹ Thuật | Đặc Tính Trọng Tâm | Tác Động Tuyến Đọc | Tác Động Tuyến Ghi/Nghẽn | Phạm Vi Khai Thác |
| --- | --- | --- | --- | --- |
| `AtomicReference` (Khuyên Dùng) | Bản Chụp Bất Biến Toàn Bảng, Thế Hệ Tiến Đơn Điệu | Tốc Độ Quang Học (1 Lệnh) | Trả Giá Chi Phí Copy Gắn Rời | Nội Bộ 1 JVM |
| `volatile` Snapshot | Kế Thừa Sức Mạnh Chụp, Hủy Chống Đa Ghi | Tốc Độ Quang Học | Sụp Đổ Trước Đa Tuyến Ghi | Nội Bộ 1 JVM |
| `ConcurrentHashMap` | Khóa Độc Lập Biến Động, Khước Từ Tính Thế Hệ | Khả Kiến Lỏng, Đọc Trộn Thế Hệ | Kháng Nghẽn Cực Cao Theo Key | Nội Bộ 1 JVM |
| `ReadWriteLock` | Khóa Chết Trạng Thái Khả Biến Đa Lớp | Chịu Trận Đóng Băng Khóa Ghi | Bức Tử Hệ Thống Đọc Dưới Tải Lớn | Nội Bộ 1 JVM |
| Kiến Trúc Phân Tán (Database/Protocol) | Dập Tắt Biến Thiên Toàn Mạng Lưới Cluster | Ngân Sách Network (Cao) | Kiểm Soát Giao Dịch Phức Hợp | Kiến Trúc Microservices Toàn Cục |

## 6. Sách Lược Vận Hành (Production Imperatives)

- Ngăn Chặn Khủng Hoảng RAM Bằng Giới Hạn Kích Thước (Size Limit) tại ngưỡng Cửa `Map.copyOf`.
- Khước từ Tẩy Rác Log Toàn Bộ Dữ Liệu; Khoanh Vùng Checksum, Thế Hệ và Đếm Mục Lục.
- Quét Hệ Thống `stale_publish_rejected_total` và Thời Hạn Tuổi Đời Snapshot Lỗi.
- Kích Hoạt Tín Hiệu Cảnh Báo Đỏ (Alert) khi Vòng Đời Tải Dữ Liệu Liên Tiếp Thất Bại Dưới Cờ Fail.
- Phế Truất Mọi Hoạt Động Cày Xới `PaymentRoute` (Mutate Value) Hậu Xuất Bản.
- Cấu Hình Giám Sát Phân Tán: Phơi Trần Thế Hệ (Generation Dashboard) Phân Loại Theo Định Danh Node Để Tầm Soát Lệch Cấu Hình.
