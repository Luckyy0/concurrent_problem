# Phản Mẫu Thiết Kế (Anti-Patterns): Ảo Tưởng Về Ranh Giới Đồng Bộ

## 1. Khởi Tạo ReentrantLock Mới Trong Mỗi Lần Triệu Gọi

Anh em nhìn đoạn code dưới đây, cú pháp thì chuẩn không cần chỉnh nhưng bản chất thì sai bét nhè. Mỗi lần gọi hàm lại đẻ ra một cái khóa mới tinh:

```java
package com.example.settlement;

import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

@Service
public class BrokenSettlementArtifactService {

    private final ArtifactStore artifactStore;
    private final SettlementRenderer renderer;

    public BrokenSettlementArtifactService(
            ArtifactStore artifactStore,
            SettlementRenderer renderer
    ) {
        this.artifactStore = artifactStore;
        this.renderer = renderer;
    }

    public ArtifactResult generate(String artifactKey, Duration lockTimeout) {
        // LỖI CHÍNH: Mỗi Request tạo ra một cá thể Lock hoàn toàn mới
        ReentrantLock lock = new ReentrantLock();
        boolean acquired = false;
        try {
            // Mọi Yêu Cầu đều khóa thành công vì chúng ôm những Ổ Khóa độc lập
            acquired = lock.tryLock(
                    lockTimeout.toMillis(),
                    TimeUnit.MILLISECONDS
            );
            if (!acquired) {
                throw new ArtifactBusyException(artifactKey);
            }

            if (artifactStore.exists(artifactKey)) {
                return ArtifactResult.alreadyExists(artifactKey);
            }

            byte[] content = renderer.render(artifactKey);
            artifactStore.put(artifactKey, content);
            return ArtifactResult.created(artifactKey);
        } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
            throw new ArtifactGenerationException("Interrupted", exception);
        } finally {
            if (acquired) {
                lock.unlock(); // Khóa được giải phóng hoàn hảo, nhưng vô giá trị
            }
        }
    }
}
```

Cấu trúc `try/finally` và bắt lỗi `Interrupt` rất xịn, nhưng ổ khóa thì toang. Mỗi luồng tạo một cái `ReentrantLock` riêng biệt nên chả ai tranh giành với ai, luồng nào cũng chạy tuốt luốt qua cổng. "Vùng tới hạn" ở đây chỉ là trò đùa!

## 2. Từ Khóa `synchronized` Định Vị Sai Mục Tiêu Khóa

```java
public ArtifactResult generate(String artifactKey) {
    // LỖI CHÍNH: Tham số String trở thành Monitor bảo vệ
    synchronized (artifactKey) {
        return generateInsideCriticalSection(artifactKey);
    }
}
```

Nhiều người nghĩ truyền `artifactKey` vào là xong. Nhưng đời không như mơ, hai request dù truyền vào chuỗi "settlement/day-1" giống y hệt nhau, nhưng trong bộ nhớ chúng là **hai object String khác nhau**. Thế là mỗi luồng lại ôm một cái ổ khóa riêng, chạy song song bình thường.
Đừng bao giờ xài `artifactKey.intern()` để ép chúng thành một nhé, làm thế rác bộ nhớ lắm và dễ dính chưởng xung đột khóa toàn hệ thống.

> **Nguyên tắc kỹ thuật:** Lệnh `synchronized` hoạt động dựa trên địa chỉ bộ nhớ (Reference), chứ nó không thèm quan tâm hai chuỗi có giá trị giống nhau (kiểu `equals`) hay không đâu.

## 3. Khóa Cục Bộ `this` Trên Một Dịch Vụ Không Phải Là Singleton Độc Tôn

```java
public synchronized ArtifactResult generate(String artifactKey) {
    return generateInsideCriticalSection(artifactKey);
}
```

Viết thế này chỉ ngon khi Class của bạn là Singleton (có đúng 1 bản duy nhất). Nếu ai đó lỡ tay `new` ra một object mới, hoặc config Spring kiểu Prototype, thì cái khóa này vứt đi. 
Thêm nữa, khóa kiểu này thì **tất cả** các request (dù là file khác nhau) đều phải xếp hàng chờ nhau. Vừa chậm vừa bất tiện.

## 4. Ranh Giới Vùng Tới Hạn Quá Hẹp (Narrow Critical Section)

```java
byte[] content;
synchronized (monitor) {
    if (artifactStore.exists(artifactKey)) {
        return ArtifactResult.alreadyExists(artifactKey);
    }
    // LỖI CHÍNH: Giài Phóng Khóa Quá Sớm
}

// Bãi Chiến Trường Ngoài Vùng Bảo Vệ
content = renderer.render(artifactKey);
artifactStore.put(artifactKey, content);
```

Anh em khóa ngắn quá! Khóa xong lúc hỏi "có file chưa?", luồng 1 và 2 đều thấy "chưa", thế là nhả khóa ra. Kết quả: cả hai anh cùng thi nhau tạo file và ghi đè lộn xộn lên Store. 
Khóa là phải khóa trọn vẹn cụm hành động "Check -> Làm -> Lưu".

## 5. Bản Đồ Khóa Cục Bộ Bị Tháo Gỡ Quá Sớm

```java
ReentrantLock lock = locks.computeIfAbsent(artifactKey, ignored ->
        new ReentrantLock());
lock.lock();
try {
    return generateInsideCriticalSection(artifactKey);
} finally {
    lock.unlock();
    // LỖI CHÍNH: Trảm Khóa mà không đo đếm Vòng Đời
    locks.remove(artifactKey); 
}
```

Chuyện kinh dị như sau:
1. Luồng 1 đang giữ khóa.
2. Luồng 2 tới, lấy khóa từ Map và đứng chờ.
3. Luồng 1 xong việc, mở khóa rồi... xóa cmn khóa khỏi Map.
4. Luồng 3 lao tới, gọi Map thì thấy trống trơn nên đẻ ra một cái khóa MỚI TINH.
Hậu quả: Luồng 2 cầm khóa cũ, Luồng 3 cầm khóa mới. Cả hai cùng chạy vào vùng tới hạn. Rất nguy hiểm! Nếu muốn xóa khóa thì phải quản lý số lượng luồng đang đợi (Reference counting) đàng hoàng.

## 6. Khóa Nội Bộ Đứng Trước Mạng Lưới Đa Nút

Giả sử bạn có 2 server A và B:

```text
node A → locks[stripe] = Lock-A
node B → locks[stripe] = Lock-B
```

Bọn nó có chung một mã `artifactKey`, nhưng Server A xài khóa của A, Server B xài khóa của B. Đừng ảo tưởng rằng khóa trên một máy (local lock) có thể làm cảnh sát giao thông cho tất cả các máy khác khi ghi lên một cái Storage chung. Không bao giờ!

## 7. Các Phương Án Sửa Lỗi Tạm Bợ Cần Tránh (Insufficient Fixes)

- Đổi `synchronized` sang `ReentrantLock` nhưng vẫn tạo mới trong hàm. (Vô dụng)
- Khóa luôn cái DTO (data object) được truyền từ ngoài vào. (Nguy hiểm)
- Đổi khóa thành biến tĩnh `static` nghĩ là khóa được nhiều server. (Static thì cũng chỉ nằm trên 1 máy thôi nha).
- Khóa mỗi cái vòng `exists()` rồi thả cửa cho hàm `render/put`. (Lỗi cày đè như ở phần 4).
- Tôn sùng `ConcurrentHashMap` nhưng lại xóa khóa bậy bạ.
- Xài annotation `@Transactional`. Cái này chả có tác dụng gì với vòng khóa của JVM và Object store đâu.
- Xài khóa công bằng (Fair Lock). Nó chỉ chống "kẹt hàng" chứ không sửa được lỗi sai ổ khóa.
