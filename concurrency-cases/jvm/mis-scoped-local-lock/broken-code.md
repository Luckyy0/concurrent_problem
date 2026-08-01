# Phản Mẫu Thiết Kế (Anti-Patterns): Ảo Tưởng Về Ranh Giới Đồng Bộ

## 1. Khởi Tạo ReentrantLock Mới Trong Mỗi Lần Triệu Gọi

Quan sát đoạn mã dưới đây, cấu trúc xử lý Khóa hoàn toàn đúng về mặt cú pháp nhưng sai lệch toàn bộ về Bản Chất Đồng Bộ:

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

Kiến trúc `try/finally` và Quản trị Ngắt (Interrupt) không có điểm mù, nhưng Định danh Khóa (Lock identity) bị xé nát. Từng cuộc gọi sinh ra một `ReentrantLock` tách biệt. Lẽ dĩ nhiên, 100% các Luồng đều qua cửa mà không gặp bất kỳ kháng cự nào. Khái niệm Vùng Tới Hạn (Critical section) ở đây chỉ là Lý Thuyết Suông.

## 2. Từ Khóa `synchronized` Định Vị Sai Mục Tiêu Khóa

```java
public ArtifactResult generate(String artifactKey) {
    // LỖI CHÍNH: Tham số String trở thành Monitor bảo vệ
    synchronized (artifactKey) {
        return generateInsideCriticalSection(artifactKey);
    }
}
```

Hai Yêu Cầu Độc lập hoàn toàn có thể mang tới hai đối tượng `String` hoàn toàn khác nhau về Tham chiếu dẫu sở hữu Nội Dung Đỉnh Khớp. Khai báo `new String("settlement/day-1")` và một Giá trị String tái tạo (deserialized) từ HTTP có đặc quyền Trùng Lặp Bằng Giá Trị (`equals() == true`), Nhưng Bản thể Monitor là Hai Cá Thể Độc Lập.

Sử dụng `artifactKey.intern()` để Ép Buộc Đồng Nhất Định Danh là một thảm họa, Gây Ô Nhiễm Không Gian Bộ Nhớ Chuỗi Toàn Cục (Global string-pool), Giam Giữ Rác Nhớ, Và Xung Đột Khóa Với Mọi Cấu Trúc Độc Lập Trong Máy Ảo. CẤM sử dụng Đối Tượng do Caller khởi tạo làm Monitor Trấn Phái.

> **Nguyên tắc kỹ thuật:** Khối `synchronized` niêm phong dựa trên Con Trỏ Tham Chiếu (Reference Identity), Không Chấp Nhận So Sánh Giá Trị Chuỗi dưới mọi hình thức.

## 3. Khóa Cục Bộ `this` Trên Một Dịch Vụ Không Phải Là Singleton Độc Tôn

```java
public synchronized ArtifactResult generate(String artifactKey) {
    return generateInsideCriticalSection(artifactKey);
}
```

Chiến lược này chỉ thành công nếu mọi Tác nhân Tôn sùng Một Bản thể Dịch Vụ Duy Nhất. Nó sẽ vỡ nát khi Bất kỳ đoạn mã nào tự triệu hồi lệnh `new`, Cấu hình Bean dưới dạng Prototype, Trùng lắp Đa Ngữ cảnh Spring (Application context), Hoặc Khối Test vô tình thả xích Hai Cá Thể.
Bất cập hơn: Ngay cả khi Độc Tôn Bản Thể (Chuẩn Spring Singleton), Khóa `this` tự động Cầm Chân (Serialize) MỌI Mã Artifact. Hai Khóa Không Liên Quan cũng phải xếp hàng chờ đợi nhau.

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

Chiếc Khóa chỉ bảo vệ Lệnh Hỏi Mật Khẩu (Check). T1 và T2 tuần tự chứng kiến "Chưa có", Và cùng lúc Nhào Nặn Dữ Liệu rồi Thi Nhau Đẩy Lên Máy Chủ ngay sau khi Buông Khóa. 100% Cụm Hành Vi (Compound action) buộc phải bị nhốt Trong Một Vành Đai Phán Xử, Hoặc Việc Đụng Độ phải được Giao Phó Cho Kho Lưu Trữ Uy Quyền (Authoritative store).

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

Dòng Chảy Tai Họa: T1 Đang Giữ Khóa. T2 Móc Lấy Tham Chiếu Khóa Từ Map Và Xếp Hàng Khóa. T1 Xong Việc, Buông Khóa Rồi Tàn Nhẫn Xóa Trắng Map. T3 Châm Ngòi Lệnh Khởi Tạo Sinh Ra Ổ Khóa Mới Toanh Và Đi Vào Khóa. Hậu Quả: T2 Bám Khóa Cũ, T3 Cầm Khóa Mới. Cả Hai Đồng Thời Đi Vào Vùng Tới Hạn. Hành vi Xóa cần Giao Thức Quản Trị Đếm Tham Chiếu (Reference counting) Đàng Hoàng.

## 6. Khóa Nội Bộ Đứng Trước Mạng Lưới Đa Nút

Máy A và Máy B Sở Hữu Hai Phân Vùng Heap Biệt Lập:

```text
node A → locks[stripe] = Lock-A
node B → locks[stripe] = Lock-B
```

Cùng Khóa vẫn triệu tập Hai Ổ Khóa Bất Đồng. Đừng Ảo Tưởng Khóa Nội Bộ có Thẩm Quyền Phán Quyết Tính Độc Nhất Toàn Cầu trên Cấu trúc Lưu Trữ Phân Tán (Shared Object Store).

## 7. Các Phương Án Sửa Lỗi Tạm Bợ Cần Tránh (Insufficient Fixes)

- Đổi `synchronized` sang `ReentrantLock` nhưng vẫn Mù Quáng tạo trong Nội Hàm.
- Gắn Mác Khóa Lên Tham Số Cấu Trúc Yêu Cầu (DTO/Key Object) Của Caller.
- Biến Khóa Thành Khối Tĩnh (Static) để Hô Hào Hỗ Trợ Cụm (Cluster); Static cũng chỉ Tĩnh Nội Bộ JVM.
- Chỉ Khóa Vòng `exists()` Rồi Buông Tay Mở Lối `render/put`.
- Tôn Vinh `ConcurrentHashMap` Làm Sổ Đăng Ký Nhưng Trảm Dữ Liệu Sai Vòng Đời.
- Chắp Vá Bằng `@Transactional`; Nó Hoàn Toàn Bất Lực Trước JVM Memory Barrier và Kho Đối Tượng (Object store).
- Viện Trợ Cờ Công Bằng (Fair Lock) Chống Lặp; Nó Tuyệt Đối Không Thể Nắn Lại Định Danh Khóa Bị Sai.
