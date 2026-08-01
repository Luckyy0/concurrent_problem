# Phản Mẫu Thiết Kế (Anti-Patterns): Cấu Trúc Khuyết Tật

## 1. Biến Dạng HashMap Cục Bộ (In-place Mutation)

Đoạn code dưới đây nhìn thì có vẻ rất chuẩn chỉnh trong Spring. Cái `HashMap` được sinh ra cùng với Bean, nhưng ngay sau đó nó lại trở thành "bãi chiến trường" giữa hai phe: Tuyến Yêu Cầu (đọc) và Tuyến Cập Nhật (ghi).

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

Đây là các kiểu dữ liệu phụ trợ:
```java
public record PaymentRoute(
        String provider,
        boolean enabled,
        long generation
) {}
```

Chữ `final` khi khai báo `routes` chỉ giúp bảo vệ cái hộp (con trỏ tham chiếu) không bị tráo đổi sang cái hộp khác. Nó hoàn toàn vô dụng trong việc ngăn chặn người ta đục khoét, thay đổi đồ đạc bên trong cái `HashMap` đó.

## 2. Ảo Giác An Toàn Chết Người

- **Cảm giác một mình một chợ:** Vì Spring tạo Bean kiểu Singleton (chỉ có 1 bản duy nhất), anh em hay ảo tưởng là chỉ có 1 luồng xài nó. Nhưng không, Một Đối Tượng không có nghĩa là chỉ có Một Luồng chạy qua!
- **Sự đánh lừa của `loadAll()`:** Tải dữ liệu xong xuôi rồi mới gọi `clear()`, nghe có vẻ êm xuôi và an toàn đấy, nhưng thực chất khe hở chết người vẫn nằm ở giữa `clear()` và `putAll()`.
- **Lỗi khó bắt:** Vì thời gian tải lại dữ liệu (30s) thưa hơn rất nhiều so với thời gian xử lý yêu cầu, nên khi test ở máy dev (chạy tuần tự) bạn sẽ thấy nó trơn tru hoàn hảo. Lỗi này gần như "tàng hình" cho đến khi lên production.

> **Nguyên tắc kỹ thuật:** Biến `final HashMap` bản chất vẫn là một nạn nhân dễ vỡ nằm giữa làn đạn đa luồng. Spring không tự động mặc áo giáp (Lock) cho các hàm của Bean đâu nhé.

## 3. Hoán Đổi Tham Chiếu Không An Toàn (Unsafe Publication)

Nhiều người nhận ra lỗi trên nên nảy ra sáng kiến: Tội gì sửa trực tiếp, cứ tạo một cái Map mới rồi tráo nó với cái cũ là xong!

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

Bản chụp (Snapshot) mới tạo ra đúng là đã được bảo vệ (dùng `Map.copyOf()`), nhưng việc Đọc/Ghi biến `routes` lại là một cuộc đua dữ liệu (Data race). Vì bạn không dùng `volatile`, Lock hay Atomic, JVM không hề ép buộc Luồng đọc phải nhìn thấy cái bản chụp mới này. Nó có thể cứ dùng mãi cái bản cũ. Đây gọi là lỗi **Công bố không an toàn (Unsafe publication)**.

## 4. Tiền Đề Kích Hoạt Lỗ Hổng (Conditions for Replication)

Lỗ hổng này sẽ phát nổ khi có đủ các yếu tố sau:
1. Có một kho dữ liệu dùng chung (Registry) được gọi bởi nhiều luồng.
2. Có ít nhất một Luồng ghi (Writer) hoạt động xen kẽ giữa các Luồng đọc (Reader).
3. Luồng ghi xông thẳng vào sửa Map cũ, hoặc gán Map mới vào một biến bình thường (không đồng bộ).
4. Luồng đọc vô tư gọi `get` hoặc duyệt Map mà không thèm xin phép hay có cơ chế đồng bộ nào.

## 5. Các Phương Án Sửa Lỗi Tạm Bợ Cần Tránh (Insufficient Fixes)

Đừng cố vá víu bằng những cách này, vì chúng không giải quyết tận gốc:

### Lạm Dụng Từ Khóa `final`
Chữ `final` chỉ khóa được con trỏ lúc khởi tạo thôi. Khi bạn gọi `clear()` hay `putAll()` bên trong Map, nó vẫn tan nát như thường.

### Bơm `volatile` Cho HashMap Khả Biến
Nếu bạn dùng `volatile HashMap`, thì người khác sẽ thấy ngay khi bạn gán một Map mới. Nhưng nếu bạn vẫn dùng cái Map cũ và gọi `clear()` rồi `putAll()`, thì `volatile` hoàn toàn vô dụng trước sự vỡ vụn của dữ liệu bên trong.

### Dịch Chuyển Sang `Collections.synchronizedMap`
Cái này giúp khóa (lock) từng thao tác một. Nhưng ngặt nỗi `clear()` và `putAll()` là hai thao tác tách biệt. Ở giữa hai thao tác đó, dữ liệu vẫn trống trơn. Thêm nữa, khi duyệt Map (iterate), bạn phải tự mình viết code khóa lại, lỡ quên một chỗ là toang cả hệ thống.

### Ký Thác Vào `ConcurrentHashMap`
Nó giúp Map không bị sập (lỗi) khi có nhiều luồng chọc vào cùng lúc. NHƯNG, gọi `clear()` rồi đắp 100 lần `put()` không gom thành 1 cục (atomic). Luồng đọc vẫn có thể bắt trúng lúc dữ liệu mới tải được một nửa (tức là đọc dữ liệu lai tạp). Cơ chế duyệt của nó cũng là "Nhất quán yếu", không đảm bảo Snapshot toàn vẹn.

### Áo Choàng `@Transactional`
Cái này là bảo bối của Database, nó dựa vào kết nối (Connection). Nó hoàn toàn bó tay trong việc bảo vệ dữ liệu nằm trên bộ nhớ RAM (JVM Memory) của bạn.

### Bắt Ngoại Lệ `ConcurrentModificationException` Rồi Kêu Gọi Thử Lại
Lỗi này sinh ra để "chết sớm bớt đau khổ", nhắc nhở bạn code đang có mùi, chứ không phải để làm cơ chế đồng bộ. Việc lờ đi và châm chước (Retry) chỉ làm ứng dụng lại lao đầu vào đạn của Luồng ghi thêm lần nữa mà thôi.
