# Phản Mẫu Thiết Kế (Anti-Patterns): Cấu Trúc Khuyết Tật

## 1. Biến Dạng HashMap Cục Bộ (In-place Mutation)

Trường hợp dưới đây bóc trần một mã nguồn tưởng chừng rất hợp lý trong kiến trúc Spring. Cá thể `HashMap` sinh ra cùng Spring Bean, nhưng lập tức bị đưa lên giàn hỏa thiêu bởi sự càn quét đồng thời của Tuyến Yêu Cầu (Request) và Tuyến Cập Nhật (Refresh).

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
    // LỖI: Cấu trúc lõi Không An Toàn Luồng bị thao túng trực tiếp
    private final Map<String, PaymentRoute> routes = new HashMap<>();

    public PaymentRoutingRegistry(RoutingConfigClient configClient) {
        this.configClient = configClient;
    }

    public PaymentRoute selectRoute(String merchantId) {
        return routes.get(merchantId); // Xung đột Đọc trong lúc Dữ liệu bị cày xới
    }

    public List<String> enabledMerchants() {
        return routes.entrySet().stream()
                .filter(entry -> entry.getValue().enabled())
                .map(Map.Entry::getKey)
                .toList(); // Hiểm họa văng ConcurrentModificationException
    }

    @Scheduled(fixedDelayString = "${routing.refresh-delay:PT30S}")
    public void refresh() {
        Map<String, PaymentRoute> loaded = configClient.loadAll();

        // LỖI NGHIÊM TRỌNG: Hành vi xóa trắng bản đồ tạo khe hở chết người cho luồng Yêu Cầu
        routes.clear();
        routes.putAll(loaded);
    }
}
```

Các kiểu dữ liệu phụ trợ:
```java
public record PaymentRoute(
        String provider,
        boolean enabled,
        long generation
) {}
```

Từ khóa `final` định danh trên `routes` chỉ mang ý nghĩa bảo vệ Con trỏ Tham chiếu (Reference) không bị hoán đổi. Nó hoàn toàn vô dụng trong việc tạo màng bảo vệ Bất biến (Immutable) hay An Toàn Luồng (Thread-safe) cho cấu trúc nội tạng của `HashMap`.

## 2. Ảo Giác An Toàn Chết Người

- Cảm giác về Sở Hữu (Ownership): Do Spring khởi tạo Singleton, giới lập trình ngộ nhận Trạng Thái chỉ có Một Chủ Nhân. Tuy nhiên, Một Chủ Nhân (Object) KHÔNG đồng nghĩa với Một Tuyến Vận Hành (Thread).
- Sự hoàn hảo giả tạo của `loadAll()`: Chạy trọn vẹn trước `clear()`, ru ngủ suy nghĩ về tính toàn vẹn dữ liệu.
- Tần suất Lỗi ngụy tạo: Hành vi Cập nhật thưa thớt hơn Yêu cầu, nên Lỗi gần như tàng hình trên môi trường Local Development, và các Khối Unit Test Tuần Tự (Sequential) luôn hoàn thành trót lọt.

> **Nguyên tắc kỹ thuật:** Biến `final HashMap` bản chất vẫn là một Nạn Nhân Khả Biến (Mutable object) nằm giữa làn đạn đa luồng; Spring không cung cấp áo giáp (Lock) cho các Phương thức của Singleton.

## 3. Hoán Đổi Tham Chiếu Không An Toàn (Unsafe Publication)

Sáng kiến loại bỏ Mutate tại chỗ (In-place) bằng cách tạo Map mới rồi dán nhãn Tham chiếu:

```java
@Service
public class UnsafeSnapshotRoutingRegistry {

    private Map<String, PaymentRoute> routes = Map.of();

    public PaymentRoute selectRoute(String merchantId) {
        return routes.get(merchantId);
    }

    public void refresh(Map<String, PaymentRoute> loaded) {
        // Sao chép an toàn, nhưng...
        Map<String, PaymentRoute> next = Map.copyOf(loaded);
        // LỖI: Gán Tham chiếu không được bảo chứng Khả kiến (Visibility)
        routes = next; 
    }
}
```

Bản chụp (Snapshot) mới được bảo vệ, nhưng tiến trình Đọc/Ghi trên thuộc tính `routes` bản chất vẫn là một Cuộc Đua Dữ Liệu (Data race). Sự vắng bóng của `volatile`, Lock hay Atomic Barrier đồng nghĩa Tuyến Yêu cầu không có nghĩa vụ hay bị ép buộc phải chiêm ngưỡng Bản chụp Mới. Đây chính là lỗ hổng **Công bố không an toàn (Unsafe publication)**; khác biệt hoàn toàn với khái niệm Khởi tạo an toàn (Safe startup) của Spring Bean.

## 4. Tiền Đề Kích Hoạt Lỗ Hổng (Conditions for Replication)

1. Chung một Hạch Tâm (Registry instance) cung cấp dịch vụ cho Đa Luồng.
2. Tồn tại ít nhất một Tuyến Ghi (Writer) khởi động giữa vòng vây của Tuyến Đọc (Reader).
3. Tuyến Ghi thay đổi bề mặt Map tại chỗ, hoặc gán Tham chiếu Bản chụp vào một Thuộc tính Vô danh (Không đồng bộ).
4. Tuyến Đọc tiến hành `get` hoặc duyệt Bản chụp Giám sát mà không tuân thủ Giao thức Cân bằng Đồng bộ (Coordination).

## 5. Các Phương Án Sửa Lỗi Tạm Bợ Cần Tránh (Insufficient Fixes)

### Lạm Dụng Từ Khóa `final`
Chỉ bảo vệ mỏ neo tại thời khắc Khởi tạo (Constructor). Hoàn toàn vô tri trước các cơn bão `clear()` và `putAll()`.

### Bơm `volatile` Cho HashMap Khả Biến
Truyền tải tính Khả kiến (Visibility) cho Tham chiếu Khóa, nhưng không có năng lực ngăn cản các vết nứt Cấu trúc nội bộ của `HashMap`. Nếu Tuyến Ghi châm ngòi `clear()` rồi `putAll()`, Tuyến Đọc vẫn hứng chịu một Bản Đồ phân mảnh.

### Dịch Chuyển Sang `Collections.synchronizedMap`
Phong tỏa từng lệnh đơn. Nhưng `clear()` và `putAll()` vẫn bị đứt gãy làm 2 nhịp tách biệt. Hành vi Duyệt (Iterate) ép buộc hệ Caller phải tự giăng Khóa thủ công. Chỉ cần một ngả Đọc quên luật, toàn bộ Khung Giao Ước sụp đổ.

### Ký Thác Vào `ConcurrentHashMap`
Giáp bảo vệ Cấu Trúc và hỗ trợ Đọc/Ghi đan xen. NHƯNG, `clear()` + N lần `put()` không hóa thân thành Cụm Nguyên Tử. Tuyến Đọc vẫn có thể bốc trúng một Thế hệ lai tạp; Bộ Duyệt mang đặc tính Nhất quán Yếu (Weakly consistent), hoàn toàn từ chối Đặc Quyền Snapshot.

### Áo Choàng `@Transactional`
Mảnh đất của Database (Thread-bound connection). Hoàn toàn bất lực trong việc phong tỏa JVM Memory Barrier.

### Bắt Ngoại Lệ `ConcurrentModificationException` Rồi Kêu Gọi Thử Lại
Ngắt Nhanh (Fail-fast) chỉ là Hàng rào Cảnh báo, tuyệt đối không phải Hợp Đồng Đồng Bộ. Vượt ải Exception không bảo chứng tính Nhất Quán; Vòng Thử lại (Retry) lại một lần nữa lao đầu vào họng súng của Tuyến Ghi.
