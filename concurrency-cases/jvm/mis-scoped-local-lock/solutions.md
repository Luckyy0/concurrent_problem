# Giải Pháp Kiến Trúc Tối Ưu Và Ma Trận Đánh Đổi

## 1. Phương Án Thiết Kế Khóa Phân Dải (Striped ReentrantLock) Ổn Định Nội Bộ (JVM)

Hệ thống cung cấp một Lượng Khóa Cố Định Nhất Định nhằm né tránh Rò rỉ Bản đồ Khóa (Lock-map leak) và rủi ro Xóa Sai (Remove race). Chung Mã Khóa (Key) luôn được băm cứng vào Một Dải Cố Định trong Hệ Thống Local.

```java
package com.example.settlement;

import java.util.concurrent.locks.ReentrantLock;

public final class StripedKeyLocks {

    private final ReentrantLock[] stripes;

    public StripedKeyLocks(int stripeCount) {
        if (stripeCount <= 0) {
            throw new IllegalArgumentException("Khung Dải Băm bắt buộc phải Dương");
        }
        this.stripes = new ReentrantLock[stripeCount];
        for (int index = 0; index < stripeCount; index++) {
            stripes[index] = new ReentrantLock(); // Khởi tạo Ổ Khóa Nhất Định
        }
    }

    public ReentrantLock lockFor(String key) {
        int hash = key.hashCode();
        int spread = hash ^ (hash >>> 16);
        return stripes[Math.floorMod(spread, stripes.length)]; // Băm Chọn Dải Tối Ưu
    }
}
```

```java
package com.example.settlement;

import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

@Service
public class SettlementArtifactService {

    private final ArtifactStore artifactStore;
    private final SettlementRenderer renderer;
    private final StripedKeyLocks keyLocks = new StripedKeyLocks(256); // Khung 256 Dải Băm 

    public SettlementArtifactService(
            ArtifactStore artifactStore,
            SettlementRenderer renderer
    ) {
        this.artifactStore = artifactStore;
        this.renderer = renderer;
    }

    public ArtifactResult generate(String artifactKey, Duration timeout) {
        if (timeout.isZero() || timeout.isNegative()) {
            throw new IllegalArgumentException("Hạn mức Timeout Vô Nghĩa");
        }

        ReentrantLock lock = keyLocks.lockFor(artifactKey); // Trích Xuất Chốt Khóa Cố Định 
        boolean acquired = false;
        try {
            // Nghiêm Ngặt Phân Bổ Hạn Mức Khóa
            acquired = lock.tryLock(timeout.toNanos(), TimeUnit.NANOSECONDS);
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
            throw new ArtifactGenerationException("Khóa Bị Ngắt Rẽ Nhánh", exception);
        } finally {
            if (acquired) {
                lock.unlock(); // Buông Khóa Rắn Rỏi Dưới Mọi Sát Na
            }
        }
    }
}
```

Các Lớp Ngoại Lệ `ArtifactBusyException` và `ArtifactGenerationException` cấu thành từ Vỏ Bọc `RuntimeException` Mỏng Tầng Nghiệp Vụ, phản ánh Tắc Nghẽn Hàng Đợi Hoặc Sập Gãy Kỹ Thuật Đa Tầng.

### Bức Tường Thành Ràng Buộc (Invariant) Nội Bộ Bất Khả Xâm Phạm
- Dàn Khóa `keyLocks` Thống Trị Trạng Thái Ổn Định Singleton.
- Va Chạm Băm (Hash Collision) Có Thể Tự Ép Cấu Trúc Khóa Tuần Tự Nhưng Bất Khả Sinh Ra Đứt Gãy.
- Tam Giác Quản Trị (Check-Render-Put) Vĩnh Viễn Không Thể Bị Thâm Nhập Song Song.
- Phép Giải Cứu `tryLock` Trang Bị Giới Hạn Cứng. Cờ Đứt Gãy Được Khôi Phục Đúng Trọng Tâm Tầng Ứng Dụng.
- Lưỡi Gươm Phán Xét `finally` Đảm Bảo Dập Tắt Độc Tôn Khóa Nội Bộ Ở Mọi Hồi Kết Đóng Khép.

> **Nguyên tắc kỹ thuật:** Khóa Phân Dải (Striped Lock) hi sinh Đặc Quyền Băng Thông Đỉnh Cục Bộ Để Chặn Đứng Tai Họa Nhận Diện Sai Khóa và Chống Thất Thoát Dữ Liệu Rác Xuyên Tuyến Hệ Thống.

### Giới Hạn Kiến Trúc Phân Dải
Khóa bị đóng băng giam lỏng xuyên suốt Tiến Trình (Render / Remote I/O) Bởi Yêu Cầu Chặn Áp Tuyệt Đối Luồng Cập Nhật. Sự phung phí Không Gian Khóa Giam Xuyên Vành Đai có thể Trầm Trọng Hoá Nút Cổ Chai (Long critical section). Tuyết Đối Không Ban Thừa Đặc Quyền Global Uniqueness Bảo Kê Mạng Phân Tán Cho Thiết Kế Khóa Này.

## 2. Phương Án Monitor Diện Rộng Tối Cổ Hủ (Coarse-Grained Monitor)

```java
@Service
public class CoarseSettlementArtifactService {

    private final Object generationMonitor = new Object(); // Ổ Khóa Tuyệt Mật 
    private final ArtifactStore artifactStore;
    private final SettlementRenderer renderer;

    public ArtifactResult generate(String key) {
        synchronized (generationMonitor) {
            if (artifactStore.exists(key)) {
                return ArtifactResult.alreadyExists(key);
            }
            byte[] content = renderer.render(key);
            artifactStore.put(key, content);
            return ArtifactResult.created(key);
        }
    }
}
```

Monitor Cá Thể Ẩn Tuyệt Đối Không Bị Áp Chế Dưới Cờ Caller Nhầm Khóa (Lock Identity Safe). Phương Pháp Tranh Đoạt Quyền Kiểm Duyệt Hoàn Hảo Nhưng Đạp Đổ Toàn Diện Đặc Tính Song Song Luồng Khác Key (Toàn Bộ Khóa Chịu Trận Một Hàng Đợi).

## 3. Cấu Trúc Bản Đồ Ổn Định Phân Mã (Stable Per-Key Map) Dành Cho Hệ Khóa Nhỏ (Bounded)

```java
private final ConcurrentMap<String, ReentrantLock> locks =
        new ConcurrentHashMap<>();

private ReentrantLock lockFor(String key) {
    return locks.computeIfAbsent(key, ignored -> new ReentrantLock());
}
```

Đúng Tuyệt Đối Nhưng Yêu Cầu Vòng Đời Entry Rắn Như Đá (Không Xóa Bừa Khỏi Map). Không Cố Chấp Khai Thác Bản Đồ (Map) Khi Tầm Vóc Băm Mã Rộng Lớn Đòi Hỏi Đảo Chiều Trấn Áp Bằng Trình Tự Khóa Phân Dải (Striped Lock) Chống Sập Ram.

## 4. Chuyển Qiao Sinh Sát Khởi Tạo Có Điều Kiện Cho Kho Dữ Liệu Uy Quyền (Authoritative Store)

Ép Hợp Đồng Thống Nhất Quy Tắc Cạnh Tranh (Atomic Write Conflict Decision):

```java
public interface ArtifactStore {
    PutIfAbsentResult putIfAbsent(String key, byte[] content);
}

public enum PutIfAbsentResult {
    CREATED,
    ALREADY_EXISTS
}
```

```java
public ArtifactResult generateGloballySafe(String key) {
    byte[] content = renderer.render(key); // Rủi Ro Khấu Hao Kép Render (Waste) 
    PutIfAbsentResult result = artifactStore.putIfAbsent(key, content); // Đập Chạm Giao Thức DB

    return switch (result) {
        case CREATED -> ArtifactResult.created(key);
        case ALREADY_EXISTS -> ArtifactResult.alreadyExists(key);
    };
}
```

Quyền Trượng Khởi Tạo Phải Gắn Với Yêu Cầu Ghi Đè Database Ngầm. Đa Máy Chủ Đồng Cày (Render) Chấp Nhận Hệ Số Đốt Lãng Phí (Wasted Render) Cục Bộ Để Dập Ngay Một Kẻ Xấu Số Khi Ép Dữ Liệu Lên Đám Mây Mạng Lưới Chung.

## 5. Ràng Buộc Độc Nhất Cơ Sở Dữ Liệu Và Điều Phối Cụm Giao Thức (DB Uniqueness/Distributed Coordination)

Row Hệ Số Artifact Mang Mã `artifact_key` Độc Quyền (Unique) Bẻ Gãy Ranh Giới Giao Đấu Kép Của Mạng Lưới Đa Nút Máy Chủ. Quyết Định Thắng Thua Định Đoạt Bởi `Index Conflict/Commit Rules`. 

Thiết Lập Chặng Giao Thức Thuê (Lease Protocol): Nhắm Trúng Đích Đạn Ngăn Ngừa Gánh Nặng Kết Xuất Rác Nặng Nề Xuyên Lục Địa (Long duplicate work). Hàng Loạt Điều Phối Đỉnh Cao Hơn Chờ Tại `DIST-001`.

## 6. Bảng Phân Phối Đánh Đổi Hiệu Năng Trấn Áp (Trade-offs)

| Chiến Thuật Phân Giải | Vành Đai Tính Đúng Đắn | Khả Năng Mở Rộng Đồng Thời | Định Danh Tắc Nghẽn Thất Bại (Failure) | Cân Bằng Ram/Vòng Đời | Áp Chế Đa Nút (Multi-node) |
| --- | --- | --- | --- | --- | --- |
| Monitor Diện Rộng Kín (Private Final) | Khối Instance/JVM Duy Nhất | Tê Liệt Hàng Loạt (Khóa Gộp Mọi Key) | Cưỡng Bước Gắn Rủi Ro Chặn Cứng | Ổn Định Nhất | Khước Từ |
| Striped `ReentrantLock` | Khối Instance/JVM Duy Nhất | Băm Đều Năng Lực Giải Phóng Hẹp | Giới Hạn Kín Hãm Tắc Mạng (Timeout) | Bất Biến (Tĩnh) | Khước Từ |
| Bản Đồ Stable Per-Key Map | Khối Instance/JVM Duy Nhất | Chạy Tẹt Ga Năng Lực Trùng Trọng Tâm | Tái Đóng Cọc Quản Trị Giải Phóng (Lifecycle) | Phình Kích Cỡ Khóa Nguy Hiểm | Khước Từ |
| Khởi Tạo Điều Kiện Kho (Authoritative) | Tuyên Cáo Uy Quyền Mạng Lưới Kho (DB) | Mở Rộng Bùng Nổ Render Chéo Máy A/B | Mò Lỗi Tắc Đường Trực Tiếp (Ambiguous Timeout) | Xóa Bản Đồ Khóa Sạch Sẽ | Hỗ Trợ Đỉnh |
| Thẻ Bài Độc Tôn Trạng Thái DB | Quyết Định Tại Chốt Transaction Database | Phá Rào Trùng Kép Index (Chỉ Mục Độc Quyền) | Yêu Cầu Cỗ Máy Fix Lỗi Giao Dịch Đồ Sộ | Cố Định Ghi Sổ DB Kính (Rows) | Hỗ Trợ Đỉnh |

## 7. Giải Pháp Phân Luồng Thua Cuộc & Thử Lại (Loser & Retry)

- Kẻ Xấu Số Gãy `tryLock` Cục Bộ: Móc Lỗi API 202 Nhanh Gọn (Fail-fast); Cấm Tư Duy Spin Vô Bổ Tốn Áp Cấp (CPU).
- Kẻ Phất Cờ Thắng Cục Bộ Va Chạm Lệnh "Đã Tồn Tại": Đổi Trạng Thái Trả Dữ Liệu Có Sẵn êm dịu (No-op).
- Kẻ Thua Cược Chặng Cổng Kho Lưu Trữ Uy Quyền (Store Conditional Loser): Dập Tắt Tức Khắc Nỗ Lực Ép Bytes Dữ Liệu Cuối, Nhượng Quyền Vương Miện Rút Dữ Liệu Về Cho Người Thắng Cộc (Read Artifact).

## 8. Khuyến Nghị Phân Phối Triển Khai Thực Chiến (Implementation Playbook)

- Trấn Áp Lệnh Render Dài/Vòng Mạng Khổng Lồ Bằng Cổng Chặn Thép Striped Lock (Hạn Mức Khóa Vừa Độ Căng Ram).
- Phán Quyết Đồng Thuận Mạng Lưới Bắt Buộc Sử Dụng Khởi Tạo Cụm DB / Giao Thức Khởi Tạo Kho Có Điều Kiện.
- Thiết Lập Ngân Hàng Số Liệu Metrics Đánh Sập Lỗi Chìm: Lock Wait/Timeout, Tín Hiệu Khóa Nóng (Stripe contention), Lệnh Duplicate Cấp Phép. Cấm Rò Rỉ Bí Mật (Sensitive Data) Ra Log.
- CẤM Tù Khóa Ngủ Đông Xuyên Tuyến Cục Bộ Bất Đồng Bộ (Async Thread Boundary). CẤM Khóa Khi Chặn Dừng Nguồn Máy Chủ (Shutdown Wait Limit).
