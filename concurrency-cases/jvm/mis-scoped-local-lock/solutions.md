# Giải Pháp Kiến Trúc Tối Ưu Và Ma Trận Đánh Đổi

## 1. Phương Án Thiết Kế Khóa Phân Dải (Striped ReentrantLock) Ổn Định Nội Bộ (JVM)

Hệ thống cung cấp sẵn một lượng khóa cố định (ví dụ 256 khóa). Kiểu này siêu an toàn vì anh em không sợ rò rỉ bộ nhớ (leak) hay lỗi xóa nhầm khóa khỏi Map. Các mã khóa (key) sẽ được băm (hash) và chia đều vào các dải này.

```java
package com.example.settlement;

import java.util.concurrent.locks.ReentrantLock;

public final class StripedKeyLocks {

    private final ReentrantLock[] stripes;

    public StripedKeyLocks(int stripeCount) {
        if (stripeCount <= 0) {
            throw new IllegalArgumentException("Số lượng dải khóa phải lớn hơn 0");
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
            throw new ArtifactGenerationException("Khóa Bị Ngắt", exception);
        } finally {
            if (acquired) {
                lock.unlock(); // Buông Khóa chuẩn chỉ trong mọi trường hợp
            }
        }
    }
}
```

Các Exception như `ArtifactBusyException` chỉ là lớp vỏ bọc bên ngoài báo lỗi nghẽn hàng đợi hoặc sập hệ thống thôi nhé.

### Bức Tường Thành Ràng Buộc (Invariant) Nội Bộ Bất Khả Xâm Phạm
- Dàn khóa `keyLocks` là duy nhất (Singleton).
- Nếu bị trùng băm (Hash collision) thì mấy request khác nhau phải xếp hàng chung 1 dải, hơi chậm tí nhưng an toàn tuyệt đối, không rớt dữ liệu.
- Vòng lặp Kiểm tra -> Tạo -> Lưu được khóa chặt chẽ.
- Hàm `tryLock` giúp giới hạn thời gian chờ.
- Lưỡi gươm `finally` luôn đảm bảo khóa được nhả ra sạch sẽ.

> **Nguyên tắc kỹ thuật:** Khóa Phân Dải (Striped Lock) hi sinh một tí tốc độ cực đại để đổi lấy sự an toàn vững như bàn thạch, không bao giờ nhầm khóa hay tràn RAM.

### Giới Hạn Kiến Trúc Phân Dải
Khóa bị đóng băng suốt quá trình render và I/O mạng. Nhớ là cái khóa này KHÔNG bảo vệ được việc chạy trùng file trên 2 server khác nhau đâu nhé!

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

Kiểu này dùng 1 khóa duy nhất chặn mọi ngóc ngách. Rất an toàn, nhưng lại giết chết luôn khả năng chạy song song. Hàng đợi lúc này sẽ dài loằng ngoằng. Chỉ xài khi file ít thôi nhé.

## 3. Cấu Trúc Bản Đồ Ổn Định Phân Mã (Stable Per-Key Map) Dành Cho Hệ Khóa Nhỏ (Bounded)

```java
private final ConcurrentMap<String, ReentrantLock> locks =
        new ConcurrentHashMap<>();

private ReentrantLock lockFor(String key) {
    return locks.computeIfAbsent(key, ignored -> new ReentrantLock());
}
```

Chạy tuyệt vời, khóa rất linh hoạt, chạy song song cực bốc. NHƯNG, không bao giờ được phép xóa khóa bừa bãi. Mà không xóa thì cái Map cứ to dần lên theo thời gian. Nên cách này chỉ áp dụng khi số lượng mã file của bạn ít và giới hạn thôi. Dữ liệu lớn thì quay về cách số 1 nha.

## 4. Chuyển Giao Quyền Sinh Sát Cho Kho Dữ Liệu Uy Quyền (Authoritative Store)

Cho Database hoặc Kho lưu trữ đứng ra phân xử luôn, dùng chiêu "Chỉ lưu nếu chưa có" (Atomic Write):

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
    byte[] content = renderer.render(key); // Rủi Ro Tính toán dư thừa (Waste) 
    PutIfAbsentResult result = artifactStore.putIfAbsent(key, content); // Đập Chạm Giao Thức DB

    return switch (result) {
        case CREATED -> ArtifactResult.created(key);
        case ALREADY_EXISTS -> ArtifactResult.alreadyExists(key);
    };
}
```

Cách này quá chuẩn cho chạy đa máy chủ! Các máy cứ cắm đầu tính toán, kệ chúng nó. Lúc đẩy lên Kho, chỉ có một thằng đầu tiên thành công, mấy thằng sau bị Kho đuổi về (báo Đã tồn tại). Tất nhiên, đổi lại là máy chủ của bạn hơi tốn sức tính toán dư thừa.

## 5. Ràng Buộc Độc Nhất Cơ Sở Dữ Liệu Và Điều Phối Cụm (DB Uniqueness/Distributed Coordination)

Bạn setup cột `artifact_key` trong DB là Unique (duy nhất). Ai ghi trước thì ăn, ai chậm chân dính lỗi Constraint Violation, ráng chịu. Đánh đổi hoàn hảo trong môi trường đa máy chủ.

Hoặc xịn hơn là chơi Giao thức Thuê Khóa (Lease Protocol), giúp các máy từ chối việc tính toán dư thừa luôn (đỡ tốn CPU như cách số 4). Đợi xem bài `DIST-001` nhé.

## 6. Bảng Phân Phối Đánh Đổi Hiệu Năng Trấn Áp (Trade-offs)

| Chiến Thuật Phân Giải | Phạm Vi An Toàn | Chạy Song Song | Nhược điểm khi Lỗi | Ram / Vòng Đời | Hỗ Trợ Đa Máy Chủ |
| --- | --- | --- | --- | --- | --- |
| Khóa Diện Rộng (1 cục) | Nội bộ 1 JVM | Chậm rì, xếp hàng | Dễ bị nghẽn toàn hệ thống | Ổn định tuyệt đối | Không |
| Striped `ReentrantLock` | Nội bộ 1 JVM | Khá bốc, rải đều tải | Hơi khó bắt bệnh Timeout | Cố định, không sợ phình RAM | Không |
| Map Khóa Linh Hoạt | Nội bộ 1 JVM | Max bốc, chạy mướt | Xóa bậy là tạch luôn | Dễ tràn RAM | Không |
| Lưu "Có điều kiện" | Toàn bộ cụm Server| Chạy mượt | Phân định lỗi nghẽn Store hơi căng| Chả tốn RAM mấy | Rất xịn |
| Set Unique trong DB | Toàn bộ cụm Server| Bùng nổ tốc độ | Phải xử lý ngoại lệ DB mệt mỏi | Khỏe re | Rất xịn |

## 7. Giải Pháp Phân Luồng Thua Cuộc & Thử Lại (Loser & Retry)

- Ai dính lỗi Timeout khi đợi khóa cục bộ: Ném ngay lỗi 202 gọn gàng. Cấm đứng lỳ chờ (spin) ngốn CPU vô ích.
- Kẻ thắng lúc vô kiểm tra thấy "Có rồi": Thôi nhẹ nhàng trả data cũ về, cấm giãy nảy.
- Kẻ thua lúc đẩy data lên Kho Lưu Trữ: Đứt ruột bỏ cái file vừa cày cuốc đi, kéo file của người thắng về xài. 

## 8. Khuyến Nghị Phân Phối Triển Khai Thực Chiến (Implementation Playbook)

- Dùng Khóa phân dải (Striped Lock) để chặn bớt luồng cày cuốc (Render) trùng lặp. Vừa đủ xài, vừa an toàn.
- Nếu chạy cụm (Cluster), BẮT BUỘC dùng `putIfAbsent` trên Kho hoặc set Unique Database để chống ghi đè dữ liệu cuối.
- Phải gom số liệu (Metrics): đếm thời gian chờ khóa, số lượng file tạo dư... để biết đường tối ưu. Cấm rò rỉ dữ liệu nhạy cảm ra Log nhé.
- CẤM Tù Khóa Ngủ Đông Xuyên Tuyến Cục Bộ Bất Đồng Bộ (Async Thread Boundary). CẤM Khóa Khi Chặn Dừng Nguồn Máy Chủ (Shutdown Wait Limit).
