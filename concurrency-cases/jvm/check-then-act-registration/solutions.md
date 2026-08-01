# Giải Pháp Kiến Trúc Tối Ưu: Cơ Chế Check-Then-Act An Toàn

## 1. Phương Pháp Số 1: Trao Quyền Cho Khối Nguyên Tử (computeIfAbsent)

Phương thức đặc trị khi Tác vụ Khởi tạo (Factory) sở hữu cường độ tính toán thấp, loại trừ các lệnh I/O qua mạng, và không có hiệu ứng gọi ngược (recursive) vào khối Map trung tâm. Triệt để ứng dụng API nguyên tử hóa tích hợp sẵn của hạ tầng `ConcurrentHashMap`:

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

        // Giao phó trọn gói chu trình "Kiểm tra & Cấp phát" cho một lệnh nguyên khối
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

### Cơ Trí Đảm Bảo Nguyên Tử Của `computeIfAbsent`

Hàm `ConcurrentHashMap.computeIfAbsent(...)` phong tỏa và thi hành trọn gói hành vi kiểm tra biến Key cùng với Công bố giá trị. Mô tả kịch bản khi Key chưa thiết lập:

1. Luồng T1 tiến hành khối tính toán khởi tạo (mapping) cho `tenant-a`;
2. Luồng T2, dù đăng ký đồng thời trên cùng Key, không được phép bóc tách quy trình thành `get → open → put` thứ hai;
3. Luồng T2 rơi vào trạng thái chờ hoặc tự động tiếp thu dữ liệu an toàn do T1 công bố;
4. 100% Caller thu thập chính xác cùng 1 thực thể `ManagedResource`.

Hàm tính toán cam kết chỉ thiết lập tối đa 1 lượt. Trường hợp Hàm tung ngoại lệ hoặc hoàn trả định danh `null`, Map tự động phủ quyết cập nhật dữ liệu, mở cơ hội cho hệ Caller tiếp theo thử lại (retry).

> **Nguyên tắc kỹ thuật:** Bác bỏ mọi thao tác tự nối ghép 3 bước (check-open-put) thủ công. Chuyển nhượng quyền phán quyết "Nạp khi Vắng" trực tiếp vào tầng quản trị cấu trúc API nguyên thủy.

### Ranh Giới Chống Chỉ Định Của `computeIfAbsent`

Nghiêm cấm lạm dụng Mapping Function với các chu trình mang tính trễ (Latency) hoặc bộc phát vô hình:

- Yêu cầu xử lý liên kết mạng (Remote network call);
- Chiếm giữ các mã Khóa (Lock) thời gian lớn;
- Vòng lặp cập nhật Callback truy lại cấu trúc Map gốc;
- Vòng lặp đứt gãy phụ thuộc qua lại (Dependency cycle);
- Các tác vụ vô thời hạn (Không có giới hạn Timeout).

`ConcurrentHashMap` kích hoạt cơ chế Đóng Băng Cục Bộ lên các lệnh Cập nhật đan xen trong lúc Mapping Function làm việc. Việc cập nhật đệ quy cùng biến Key bị Java cấm tiệt và tung lỗi `IllegalStateException`.

## 2. Phương Pháp Số 2: Định Vị Chờ Tín Hiệu (FutureTask Placeholder)

Đáp ứng yêu cầu xử lý các tài nguyên nặng (Heavy-lifting). Trọng tâm là nạp mỏ neo giữ chỗ (Placeholder) vào Map. Chỉ định duy nhất Luồng kiến tạo thành công Placeholder được phép triệu tập Factory; Các thành viên còn lại chuyển sang chờ kết quả tại mỏ neo. Yêu cầu tự cấp cơ cấu Timeout cứng cho quá trình.

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
                // Tạo Mỏ neo (Chưa chạy nghiệp vụ vật lý)
                FutureTask<ManagedResource> candidate = new FutureTask<>(
                        () -> resourceFactory.open(resourceKey)
                );

                // Luồng nào chiếm chỗ thành công sẽ thống trị Map
                future = registrations.putIfAbsent(resourceKey, candidate);

                if (future == null) {
                    future = candidate;
                    candidate.run(); // Duy nhất Luồng thắng mới mở Tài nguyên
                }
            }

            try {
                return future.get(); // Đám đông Caller cùng xếp hàng đợi ở Mỏ neo này
            } catch (CancellationException exception) {
                registrations.remove(resourceKey, future);
            } catch (InterruptedException exception) {
                Thread.currentThread().interrupt();
                throw new ResourceRegistrationException(
                        "Quá trình đăng ký bị gián đoạn: " + resourceKey,
                        exception
                );
            } catch (ExecutionException exception) {
                registrations.remove(resourceKey, future); // Giải phóng mỏ neo hỏng
                throw new ResourceRegistrationException(
                        "Thất bại trong tiến trình khởi tạo: " + resourceKey,
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

### Cơ Trí Tối Ưu Của Khối Placeholder

- Lệnh `putIfAbsent` quyết định chỉ một chủ thể `FutureTask` thắng lợi.
- Lệnh `candidate.run()` được độc quyền phát động bởi luồng giành chỗ thành công.
- Tuyệt đối mọi Caller nhận về cùng đối tượng tài nguyên khi kích hoạt hàm `get()`.
- Hàm Factory thi hành ở vị trí nằm ngoài chuỗi tính toán nội tại (internal map computation), loại trừ hiện tượng khóa cục bộ của `computeIfAbsent`.
- Tại thời điểm đứt gãy cấu trúc (Lỗi Factory), Placeholder được xóa sổ, tạo lộ trình sạch cho hệ Caller vòng tới Retry hệ thống.

Cảnh báo: Triệt tiêu cấu trúc Retry vô hạn (Infinite loop) tại Registry. Policy bắt buộc giới hạn dung lượng vòng lặp, ấn định cơ chế giãn cách (Backoff), và tập trung tự hủy (Clean up dở dang) trước khi báo Exception.

## 3. Phương Án 3: Thí Điểm Khởi Tạo & Hủy Tài Nguyên Dư Thừa

Dành cho hệ thống an toàn nếu thi hành Factory lặp lại và tài nguyên rác có tính đàn hồi đóng ngay lập tức:

```java
public ManagedResource register(String resourceKey) {
    ManagedResource created = resourceFactory.open(resourceKey);
    ManagedResource existing = resources.putIfAbsent(
            resourceKey,
            created
    );

    if (existing != null) {
        created.close(); // Thu hồi khẩn cấp mảnh tài nguyên rác
        return existing;
    }

    return created;
}
```

Cấu hình bảo lưu quy tắc 1 Giá trị trong Map, 100% Caller trả chung kết quả của luồng chiếm dụng. Song không cứu chữa được bài toán Khởi chạy đa vòng của Factory. Quy trình dọn dẹp bắt buộc "Chống đạn" (Fail-safe) khi `close()` lỗi, Code môi trường phải ghim vết Cleanup failure.

## 4. Phương Án 4: Phân Quyền Tuần Tự Toàn Cục (synchronized)

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

Hiệu lực 100% đối với 1 Máy ảo, nhưng biến toàn bộ Caller (bất kể Key độc lập) thành cấu trúc Hàng đợi (Serialize). Một Factory nghẽn ở `tenant-a` sẽ lập tức gây ứ đọng trên luồng của `tenant-b`. Vô giá trị trên phạm vi hệ thống Đa Máy Chủ (Multi-instance).

## 5. Ma Trận Đánh Đổi Hiệu Suất Hệ Thống

| Giải Pháp Kỹ Thuật | Đặc Điểm Bảo Đảm | Thông lượng/Độ Trễ | Xử Lý Tranh Chấp/Retry | Rủi Ro Deadlock | Độ Phức Tạp | Phạm Vi Khai Thác |
| --- | --- | --- | --- | --- | --- | --- |
| `computeIfAbsent` | Chuẩn vị trong JVM; Tối đa 1 truy cập thành công | Hiệu năng ưu việt (Nếu Factory nhẹ) | Xử lý đợi; Sẵn sàng cho Caller sau Retry | Thấp (Trừ khi map đan xen khóa) | Dễ Dàng | Phi Hỗ Trợ Đa Node |
| Placeholder `FutureTask` | Chuẩn vị trong JVM; Tuyệt đối 1 tài nguyên thắng cuộc | Tối ưu đa Key; Đồng nhất tiến trình đợi Caller chung Key | Có xóa bỏ thất bại & Retry từ Caller | Phụ thuộc lỗi Factory Dependency | Vừa Phải | Phi Hỗ Trợ Đa Node |
| `putIfAbsent` + Thu dọn | Map duy nhất, Factory không đảm bảo 1 lần | Thông lượng cực cao (Đổi lấy cấp rác tài nguyên) | Bỏ qua Retry Application; Cần dọn kỹ kẻ thua cuộc | Siêu Thấp | Vừa Phải | Phi Hỗ Trợ Đa Node |
| `synchronized` | Khóa Chặt trong JVM | Tuần tự hóa, Thông lượng tê liệt | Block toàn hệ thống | Tăng dần khi Factory xài khóa ngoài | Siêu Thấp | Phi Hỗ Trợ Đa Node |
| Khóa Điều Phối / Database | Bảo lưu Quy Tắc Xuyên Suốt Mọi Node Máy Chủ | Suy giảm độ trễ, Contention nghẽn tại Ranh giới chung | Phụ thuộc chính sách Transaction/Retry của DB | Cảnh báo khi cấu trúc đảo khóa (Lock order) | Phức Tạp | Chuẩn hệ thống |

## 6. Khuyến Nghị Phân Phối Khai Thác

- Chọn `computeIfAbsent` định dạng kiến trúc Cục Bộ, Lõi xử lý nhanh và cách ly I/O nghẽn.
- Chọn `Placeholder Future` (Mỏ neo) đáp ứng cấp phát khổng lồ và ép 100% người tới sau phải bám sát kết quả luồng đi trước.
- Chọn `putIfAbsent` kết hợp dọn rác (Clean up) chỉ định trong trường hợp hệ thống dư thừa tài nguyên còn an toàn hơn đánh đổi Lock thời gian thực.
- Kế hoạch `synchronized` là một nước đi nguy hiểm, chỉ dành cho quy mô vĩ mô nhỏ không quan tâm tới tính cách ly Key.
- Tuyệt đối cấm khai thác cấu trúc Phân Tán (Distributed Lock) chỉ để giải bài toán cỏn con trên Local Registry.

## 7. Chuẩn Phân Cấp Triển Khai Thực Tế

- Cấp phép Timeout cưỡng chế tại Factory cho Network/File operation.
- Hàm Factory mang bổn phận đóng mọi vết rác tài nguyên dang dở trước khi tung lỗi (Throw exception).
- Triển khai Đo lường quy mô lớn: Số lần Factory khởi chạy, Map Size, Số lượng Resource Active, và Lỗi Clean Up.
- Khước từ Mapping Function tự lặp (đệ quy) vào cùng 1 Map chứa nó.
- Lệnh Thu hồi (Unregister) phải dùng cấu trúc gỡ chặn có điều kiện `remove(key, expectedResource)` để né trường hợp thanh trừng nhầm tài nguyên phiên bản mới.
- Khai thác Lease lifecycle hoặc Reference counting phải đưa vào bộ chuyên đề vòng đời riêng thay vì cố nhồi nhét vào Registry.
- Uniqueness tại Cluster bắt buộc chuyển giao nhiệm vụ về Database hoặc Giao thức Phân tán (Distributed Protocol).
