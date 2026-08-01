# Môi Trường Thực Nghiệm: Phân Tích Và Đo Lường Cấu Trúc Khóa Đồng Thời

## 1. Chiến Lược Kiểm Thử Cốt Lõi (Core Strategy)

Quá trình thẩm định bắt buộc trả lời tường tận 4 khía cạnh:
1. Xác minh Hai Khóa (Lock objects) Phân Biệt có Cự Tuyệt Hai Tác Nhân (Actors) cùng đột nhập hay không.
2. Thẩm Định Hai Khóa Logic (Logical Keys) Bằng Trị Số nhưng Lệch Tham Chiếu có bị ép bám chung Một Ổ Khóa (Monitor) hay không.
3. Khảo Sát Tính Năng Khóa Phân Dải (Striped Lock) bảo Vệ Mã Khóa Nội Bộ (Service Instance).
4. Vạch Trần Giới Hạn của Khóa Nội Bộ khi Đa Nút Máy Chủ (Service Instances) chạm trán Kho Lưu Trữ Chung.

Triển khai Rào chắn Giả lập (Fake Store Latch) ngay sau lệnh `exists` nhằm ép 100% Actors vào thế Quan Sát Báo Cáo "Tài Khoản Rỗng" (Chưa Tồn Tại). Loại bỏ hoàn toàn Dấu Vết của `Thread.sleep`; Đóng Mọi Rào Chắn Latch/Future Bằng Bộ Giờ Đếm Ngược Timeout.

## 2. Thí Nghiệm 1: Trình Diễn Bế Tắc Khóa Lỗi Nhịp Mỗi Lần Gọi (Per-Call Locks)

```java
@Test
void perCallLocksAllowDuplicateGeneration() throws Exception {
    BarrierStore store = new BarrierStore(2); // Dựng Rào 2 Lớp
    AtomicInteger renders = new AtomicInteger();
    SettlementRenderer renderer = key -> {
        renders.incrementAndGet();
        return key.getBytes(StandardCharsets.UTF_8);
    };
    BrokenSettlementArtifactService service =
            new BrokenSettlementArtifactService(store, renderer);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<ArtifactResult> first = executor.submit(() ->
                service.generate("settlement/day-1.csv", Duration.ofSeconds(5)));
        Future<ArtifactResult> second = executor.submit(() ->
                service.generate("settlement/day-1.csv", Duration.ofSeconds(5)));

        first.get(5, TimeUnit.SECONDS);
        second.get(5, TimeUnit.SECONDS);
    }

    // CHỨNG MINH RỦI RO: Dữ liệu Đã Bị Cày Nát Nhân Bản
    assertEquals(2, renders.get()); 
    assertEquals(2, store.putCount());
}
```

Rào Chắn (Barrier) Cố Đinh: Chỉ vỡ khi Cả 2 Actors Chạm Mốc `exists`. NẾU hệ thống Tuân Thủ Một Khóa Chuẩn Mực, Bài Test Bắt Buộc Đứt Gãy Timeout Do Nghẽn Cổ Chai. Trái Lại, Với Khóa Sinh-Mới-Mỗi-Lần, Cả 2 Lướt Qua Rào và Sinh Ra Hành Vi Nhân Bản (Duplicate Writes).

> **Nguyên tắc kỹ thuật:** Phép Thử không chọc ngoáy bóc tách Bản Thể Khóa (Lock Object) qua Reflection; Nó Tra Khảo Hành Vi Nghiệp Vụ Sinh Tử — Bắt Buộc Chỉ Có Thể Kích Hoạt Một Chặng Sinh Khóa (Generation Workflow) Cho Một Mã Độc Nhất.

## 3. Thí Nghiệm 2: Trị Số Chuỗi Khớp (Equal) KHÔNG Bằng Đồng Nhất Tham Chiếu Khóa (Monitor)

Mô Phỏng Trực Quan Ảo Giác Khóa:

```java
@Test
void equalStringsWithDifferentIdentityUseDifferentMonitors() throws Exception {
    BarrierStore store = new BarrierStore(2);
    AtomicInteger renders = new AtomicInteger();
    KeyMonitorGenerator generator = new KeyMonitorGenerator(
            store,
            key -> {
                renders.incrementAndGet();
                return new byte[] {1};
            }
    );

    // CHỨNG MINH LỖ HỔNG: Tham Chiếu Tách Rời Bất Chấp Cùng Giá Trị
    String firstKey = new String("settlement/day-1.csv");
    String secondKey = new String("settlement/day-1.csv");
    assertEquals(firstKey, secondKey);
    assertNotSame(firstKey, secondKey);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<?> first = executor.submit(() -> generator.generate(firstKey));
        Future<?> second = executor.submit(() -> generator.generate(secondKey));
        first.get(5, TimeUnit.SECONDS);
        second.get(5, TimeUnit.SECONDS);
    }

    // HỆ QUẢ: Cấu Trúc Khóa Không Cầm Chân Nhau
    assertEquals(2, renders.get());
    assertEquals(2, store.putCount());
}
```

## 4. Thí Nghiệm 3: Năng Lực Trấn Thủ Của Khóa Phân Dải (Striped Lock)

Lệnh Triệu Tập Thứ 2 (Second Request) Bị Ép Nghẽn Trên Dải Khóa. Sau khi Quân Chủ Lực (Request 1) Kéo Quân Xuất Bản, Kẻ Tới Sau Phải Nuốt Phản Hồi "Đã Tồn Tại" Thay Vì Tự Tiện Rẽ Hướng Cày Đè.

```java
@Test
void stableStripeAllowsOnlyOneRenderForTheSameKey() throws Exception {
    InMemoryStore store = new InMemoryStore();
    CountDownLatch firstRenderEntered = new CountDownLatch(1);
    CountDownLatch allowFirstRender = new CountDownLatch(1);
    CountDownLatch secondTaskStarted = new CountDownLatch(1);
    AtomicInteger renders = new AtomicInteger();

    SettlementRenderer renderer = key -> {
        if (renders.incrementAndGet() == 1) {
            firstRenderEntered.countDown();
            awaitOrFail(allowFirstRender); // Ép Đọng Chờ
        }
        return key.getBytes(StandardCharsets.UTF_8);
    };
    SettlementArtifactService service =
            new SettlementArtifactService(store, renderer);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<ArtifactResult> first = executor.submit(() ->
                service.generate("settlement/day-1.csv", Duration.ofSeconds(5)));
        assertTrue(firstRenderEntered.await(5, TimeUnit.SECONDS));

        Future<ArtifactResult> second = executor.submit(() -> {
            secondTaskStarted.countDown();
            return service.generate(
                    "settlement/day-1.csv",
                    Duration.ofSeconds(5)
            );
        });
        assertTrue(secondTaskStarted.await(5, TimeUnit.SECONDS));
        allowFirstRender.countDown(); // Buông Tay Khối 1

        // QUY TẮC BẤT BIẾN LÊN NGÔI
        assertEquals(ArtifactResult.Status.CREATED,
                first.get(5, TimeUnit.SECONDS).status());
        assertEquals(ArtifactResult.Status.ALREADY_EXISTS,
                second.get(5, TimeUnit.SECONDS).status());
    }

    assertEquals(1, renders.get());
    assertEquals(1, store.putCount());
}
```

Xác thực Quả Ngọt Nghiệp Vụ (Business Outcome), từ chối lệ thuộc Phương thức `lock.isLocked()`. Vỏ Bọc Khóa Có Thể Sửa Đổi Mà Hệ Quả Cấu Trúc Không Rạn Nứt.

## 5. Thí Nghiệm 4: Hai Bản Thể Máy Chủ Phơi Bày Giới Hạn Khóa Nội Bộ

Giả lập Kiến Trúc Hai Nút Máy Chủ (Two Nodes). Mỗi Máy Sở Hữu Kho Khóa Độc Lập `StripedKeyLocks`, nhưng Tranh Cướp Chung Kho Dữ Liệu `BarrierStore`.

```java
@Test
void twoInstancesStillGenerateTheSameArtifactTwice() throws Exception {
    BarrierStore sharedStore = new BarrierStore(2);
    AtomicInteger renders = new AtomicInteger();
    SettlementRenderer renderer = key -> {
        renders.incrementAndGet();
        return new byte[] {1};
    };

    SettlementArtifactService nodeA =
            new SettlementArtifactService(sharedStore, renderer);
    SettlementArtifactService nodeB =
            new SettlementArtifactService(sharedStore, renderer);

    try (ExecutorService executor = Executors.newFixedThreadPool(2)) {
        Future<?> first = executor.submit(() -> nodeA.generate(
                "settlement/day-1.csv",
                Duration.ofSeconds(5)
        ));
        Future<?> second = executor.submit(() -> nodeB.generate(
                "settlement/day-1.csv",
                Duration.ofSeconds(5)
        ));

        first.get(5, TimeUnit.SECONDS);
        second.get(5, TimeUnit.SECONDS);
    }

    // SỰ THẬT: Khóa Nội Bộ Không Cản Nổi Làn Sóng Ghi Chèn Xuyên Máy Chủ
    assertEquals(2, renders.get());
    assertEquals(2, sharedStore.putCount());
}
```

Đây là Thử Thách Tử Thần Buộc Phải Chạy Trước Khi Ảo Mộng Khóa Nội Bộ Được Đặt Phía Trước Ranh Giới Lưu Trữ Chung.

## 6. Thí Nghiệm 5: Lệnh Cập Nhật Kho Khởi Tạo Có Điều Kiện Định Hình Vương Quyền Độc Tôn (Conditional Create)

Bức Tường Kho Lưu Trữ Phán Quyết Vận Mệnh bằng Vũ Khí `putIfAbsent` Của Atomic:

```java
final class ConditionalInMemoryStore {
    private final ConcurrentMap<String, byte[]> data = new ConcurrentHashMap<>();

    PutIfAbsentResult putIfAbsent(String key, byte[] content) {
        return data.putIfAbsent(key, content) == null
                ? PutIfAbsentResult.CREATED
                : PutIfAbsentResult.ALREADY_EXISTS;
    }
//...
}
```

Thử Thách Ép Bộ Lọc Lỗi. Dẫu Hai Máy Cùng Nhấn Lệnh Render (Nướng CPU), Bắt Buộc Kho Lưu Trữ Dập Tắt Một Yêu Cầu (Trả vể Tồn Tại). Lộ Trình Triển Khai Thực Chiến Yêu Cầu Gắn Bó Sâu Hơn Chức Năng Cực Đoan Từ DB Store API Thực Tế.

## 7. Khung Kiểm Định Quản Trị Khóa Mở Rộng: Timeout, Interruption, Tình Trạng Lỗi

Mở Rộng Quản Trị Rủi Ro Cấu Trúc Khóa:
- T2 Rớt Đài Sớm Nhận Lỗi `ArtifactBusyException` Dưới Áp Lực Timeout Trong Khi T1 Giam Dải Khóa.
- Đánh Phá Ngắt Interrupt Luồng T2 Giữa Tâm Bảo `tryLock`; Kiểm Toán Lại Cờ Khôi Phục Ngắt.
- Bộ Render Sập Bẫy Ngoại Lệ; Khóa Có Buông Tay Nhường Cơ Hội Sau Cùng Lại Cho Luồng Đuôi (Tail request).
- Phán Quyết Mù Dưới Áp Lực Store Timeout; Đòi Hỏi Phương Pháp Khắc Phục Đối Soát (Reconcile).
- Khung Dải Băm Góp Mặt Chung 2 Mã Artifact; Áp Lực Tuần Tự (Serialize) Không Đánh Sập Quy Tắc Bất Biến (Correctness).
- Khóa Có Dám Sống Sót Giữa Giao Quyền Giải Phóng Cho Luồng Cấu Trúc Thứ Cấp (Callback thread)? KHÔNG!

## 8. Khung Giám Sát Khai Thác Môi Trường (Production Blueprint)

Đặc Tả Vận Hành (Metric):
- Chỉ số Hao Mòn: Lock Wait Duration, Acquisition Timeout, Lượng In-flight Hiện Hành.
- Trục Cân Bằng: Render Count So Sánh Lệnh Created Artifact Count.
- Thống Kê Giao Chiến Khóa Liên Máy Chủ: Conditional Conflict / Lệnh `ALREADY_EXISTS`.
- Biến Động Hàng Đợi (Queue Length Diagnostic) và Định Danh Điểm Nóng (Hot spot) Dải Khóa (Stripe contention).

Nghiêm Cấm Suy Đoán Quyết Định Core (Correctness signal) Dựa Lên Tham Số Dò Dẫm (Heuristic) Như Lệnh Đo Kích Cỡ Wait Queue Của ReentrantLock.

## 9. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [ ] Phơi Trần Cấu Trúc Khóa Theo Chặng Nhờ Barrier.
- [ ] Vạch Trần Lỗi Khóa Bằng Tham Chiếu Độc Lập Chứ Không Dùng Nội Dung.
- [ ] Khảo Khảo Sát Một Ghi Mới Vận Hành Nhờ Lệnh Render Trấn Áp.
- [ ] Hiện Thực Thử Đo Đạc Khóa Nội Bộ Quá Hẹp (Two-node setup).
- [ ] Truy Bắt Ranh Giới Giao Thức Khởi Tạo Kho Có Điều Kiện.
- [ ] Trục Xuất `Thread.sleep` Khỏi Hệ Thống Logic Phán Quyết.
- [ ] Bó Cứng 100% Khung Latch Bằng Hạn Mức Timeout.
- [ ] Phơi Trần Sức Trụ Vững `try-finally` Trước Gián Đoạn (Interrupt) và Vỡ Ngoại Lệ (Exception).
- [ ] Buộc Đóng Chốt Toàn Bộ Pool (Executor) Trước Hồi Kết.
