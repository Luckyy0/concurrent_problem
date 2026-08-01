# Giải Pháp Kiến Trúc Tối Ưu: Cơ Chế Check-Then-Act An Toàn

## 1. Phương Pháp Số 1: Trao Quyền Cho Khối Nguyên Tử (computeIfAbsent)

Cách này bao xịn nếu cái hàm tạo (Factory) của bạn chạy nhanh, không dính tới network I/O, và không gọi đệ quy ngược lại vào cái Map. Mình xài luôn hàm hỗ trợ nguyên tử (atomic) của `ConcurrentHashMap`:

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

        // Giao luôn việc "Check & Tạo" cho thằng Map nó tự xử
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

Hàm `computeIfAbsent(...)` nó tự động chặn các luồng khác và gom chung bước kiểm tra và gán vào làm một. Kịch bản thế này:

1. Luồng T1 đang nai lưng ra chạy hàm tạo đối tượng cho `tenant-a`.
2. Luồng T2 nhảy vào đòi `tenant-a`, nhưng bị hàm `computeIfAbsent` chặn lại luôn chứ không cho tách ra 3 bước (check-open-put) như hồi nãy.
3. Luồng T2 đứng chờ hoặc tự động xài ké kết quả ngon lành mà T1 trả về.
4. Đảm bảo 100% các luồng đều lấy về đúng 1 object `ManagedResource` duy nhất.

Trường hợp mà hàm tạo bị quăng lỗi (Exception) hay trả về `null`, Map sẽ tự động huỷ kèo, không ghi gì hết, mấy luồng tới sau tha hồ mà làm lại.

> **Nguyên tắc kỹ thuật:** Đừng dại tự code tay 3 bước (check-open-put). Hãy để hàm API có sẵn của Map nó lo trọn gói việc "vắng mặt thì nạp vào".

### Ranh Giới Chống Chỉ Định Của `computeIfAbsent`

Cấm tuyệt đối dùng cách này nếu việc khởi tạo của bạn dính tới mấy món sau:

- Gọi API qua mạng chậm rì;
- Giữ cái khoá (Lock) quá lâu;
- Viết hàm đệ quy tự gọi lại cái Map đó;
- Dính vòng lặp phụ thuộc vòng quanh;
- Hàm tính toán không có điểm dừng (không có Timeout).

Lý do là `ConcurrentHashMap` nó khóa ngầm cái slot đó trong Map lúc đang chạy. Bạn mà gọi lồng vào chính cái Key đó là Java nó đập thẳng vào mặt lỗi `IllegalStateException`.

## 2. Phương Pháp Số 2: Định Vị Chờ Tín Hiệu (FutureTask Placeholder)

Nếu việc tạo tài nguyên tốn thời gian cực kỳ (Heavy-lifting), thì dùng cách "cắm mỏ neo" (Placeholder) là chuẩn bài. Anh nào nhanh chân "cắm" được cái mỏ neo vào Map thì ảnh có quyền tạo tài nguyên. Các luồng đến sau thấy có mỏ neo rồi thì vui vẻ đứng chờ. Nhớ là phải tự quản lý Timeout nhé.

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
                // Tạo cái mỏ neo (chưa chạy hàm thật)
                FutureTask<ManagedResource> candidate = new FutureTask<>(
                        () -> resourceFactory.open(resourceKey)
                );

                // Thi nhau cắm mỏ neo, ai win thì đi tiếp
                future = registrations.putIfAbsent(resourceKey, candidate);

                if (future == null) {
                    future = candidate;
                    candidate.run(); // Anh Win mới được chạy tạo tài nguyên
                }
            }

            try {
                return future.get(); // Mấy ông đến sau thì bu lại xếp hàng ở đây chờ
            } catch (CancellationException exception) {
                registrations.remove(resourceKey, future);
            } catch (InterruptedException exception) {
                Thread.currentThread().interrupt();
                throw new ResourceRegistrationException(
                        "Quá trình đăng ký bị gián đoạn: " + resourceKey,
                        exception
                );
            } catch (ExecutionException exception) {
                registrations.remove(resourceKey, future); // Tạo lỗi thì nhổ mỏ neo vứt đi
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

- Nhờ `putIfAbsent` mà chỉ 1 cái `FutureTask` duy nhất được lọt vào.
- Chỉ thằng thắng mới được gọi `candidate.run()`.
- Hàm `get()` đảm bảo toàn bộ các ông đứng đợi đều lấy chung 1 kết quả về.
- Hàm Factory lúc này chạy bên ngoài luồng xử lý của Map, nên Map chả lo bị khóa cục bộ như cái `computeIfAbsent`.
- Nếu có lỗi xảy ra, mỏ neo bị "nhổ" bỏ, sạch sẽ để luồng sau bay vô làm lại từ đầu.

Cảnh báo: Viết vòng lặp `while (true)` thì nhớ có kiểm soát. Nếu lỗi liên tục là lặp vô tận đấy, nên có Retry giới hạn, thêm tý thời gian giãn cách (Backoff), và dọn dẹp sạch sẽ trước khi quăng Exception.

## 3. Phương Án 3: Thí Điểm Khởi Tạo & Hủy Tài Nguyên Dư Thừa

Nếu hàm tạo của bạn chạy đi chạy lại cũng không chết ai, và tài nguyên thừa thãi có thể đóng được liền tay, thì bạn xài cách này:

```java
public ManagedResource register(String resourceKey) {
    ManagedResource created = resourceFactory.open(resourceKey);
    ManagedResource existing = resources.putIfAbsent(
            resourceKey,
            created
    );

    if (existing != null) {
        created.close(); // Hàng hớ thì vứt luôn
        return existing;
    }

    return created;
}
```

Cách này map vẫn chỉ lưu 1 object chuẩn, các anh em cũng chỉ dùng 1 cái chung. Nhưng nhược điểm là Factory có thể bị gọi nhiều lần! Và nhớ phải `close()` hàng thải cho đàng hoàng, code không kỹ chỗ này là rò rỉ rác nha.

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

Dùng `synchronized` thế này thì đúng là chắc cú 100% trên 1 server, nhưng tất cả các request sẽ phải đứng xếp hàng dài đằng đẵng. Đứa gọi `tenant-a` bị nghẽn thì đứa gọi `tenant-b` cũng phải đứng chờ. Cách này không dùng được nếu bạn muốn hiệu suất tốt, và hoàn toàn vô dụng trên cụm nhiều server (Multi-instance).

## 5. Ma Trận Đánh Đổi Hiệu Suất Hệ Thống

| Giải Pháp Kỹ Thuật | Đặc Điểm | Tốc độ/Độ Trễ | Xử Lý Lỗi/Retry | Nguy cơ Deadlock | Độ khó | Xài cho Đa Server? |
| --- | --- | --- | --- | --- | --- | --- |
| `computeIfAbsent` | Rất chuẩn trên 1 JVM | Cực nhanh (Nếu tạo nhẹ) | Tự chờ; Luồng sau Retry thoải mái | Thấp | Dễ | Không |
| Placeholder `FutureTask` | Chuẩn JVM; Đảm bảo 1 luồng tạo | Chạy tốt nhiều Key; Cùng Key thì chung chờ | Tự dọn mỏ neo hỏng & Retry | Có rủi ro nếu Factory dính Dependency lằng nhằng | Vừa | Không |
| `putIfAbsent` + Dọn rác | Chỉ lưu 1, nhưng có thể sinh thừa | Siêu mượt (đổi lại tốn ram lúc đầu) | Dọn dẹp đồ bỏ đi mệt mỏi | Rất thấp | Vừa | Không |
| `synchronized` | Khóa siêu chặt trên JVM | Chậm rì, thắt cổ chai | Nghẽn nguyên dàn | Tuỳ hàm Factory có chờ Lock khác không | Quá dễ | Không |
| Khoá Database / Distributed Lock | Xuyên server vẫn chuẩn | Trễ cao, dễ kẹt ở DB | Phụ thuộc vô Database retry | Dễ bị deadlock nếu đảo thứ tự khoá | Khó | Có |

## 6. Khuyến Nghị Phân Phối Khai Thác

- Chọn `computeIfAbsent` nếu làm trên 1 máy, hàm chạy lẹ, không đụng network.
- Chọn `Placeholder Future` (Mỏ neo) nếu tạo cực lâu và muốn ép mấy anh em khác ngồi chờ chung mâm kết quả.
- Chọn `putIfAbsent` + dọn dẹp nếu hệ thống bạn đủ mạnh để sinh rác tạm thời thay vì bắt chờ nhau.
- `synchronized` là kèo khá chuối, chỉ xài cho mấy ứng dụng tí hon, không thèm quan tâm tốc độ.
- Tránh xa Distributed Lock nếu cái bạn cần chỉ là giải một bài toán bé tẹo ở Registry local.

## 7. Chuẩn Phân Cấp Triển Khai Thực Tế

- Nhớ ép cái Timeout cứng vào Factory khi đụng tới Network hay File.
- Factory phải biết tự dọn dẹp hiện trường sạch sẽ rồi mới được quăng Exception.
- Nên gắn thêm công cụ giám sát: Số lần Factory chạy, Size của Map, Tài nguyên Active, Lỗi lúc dọn dẹp.
- Không tự lấy đá đập chân bằng cách viết vòng lặp gọi lại cái Map trong hàm khởi tạo.
- Lúc gỡ (Unregister) thì xài `remove(key, object_đang_có)` để khỏi xoá nhầm cái vừa được thay thế.
- Muốn quản lý đồ thuê (lease) hay đếm số người dùng, thì viết cái quản lý Vòng Đời riêng chứ đừng nhét hết vô Registry.
- Bài toán duy nhất (Uniqueness) mà xuyên cả cụm máy chủ thì lo mang xuống Database hoặc làm Giao thức Distributed Lock.
