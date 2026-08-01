# Môi Trường Thực Nghiệm: Cấu Trúc Kiểm Định Rủi Ro & Đánh Giá Khai Thác

## 1. Chiến Lược Kiểm Thử Cốt Lõi (Core Strategy)

Ma trận kiểm định tập trung bóc tách 3 luồng rủi ro:
1. Đánh giá Cấu trúc Suy thoái (Broken code) phơi bày mảnh vỡ Dữ liệu Đang làm mới.
2. Chứng minh Cấu trúc Bản Chụp Bất Biến (Immutable snapshot) bảo vệ Tính Toàn Vẹn và buộc Thế hệ (Generation) tăng đơn điệu.
3. Rà soát chuẩn Ngữ nghĩa Duyệt (Iteration semantics) của Khung Tập Hợp (Collection) tương thích Khách hàng (Caller).

Mỏ neo `Latch` điều phối luồng vào chính xác Tọa độ Tranh chấp (Conflict window). Tuyệt đối cấm khai thác `Thread.sleep` làm mốc Phán Quyết vì tính bấp bênh của Bộ lập lịch (Scheduler). Thiết lập Timeout cưỡng chế cho 100% lệnh chốt chặn.

## 2. Thí Nghiệm 1: Trình Diễn Bóc Tách Bản Chụp Phân Mảnh (Partial Snapshot)

Luồng Kiểm Thử hãm phanh Tuyến Ghi (Writer) ngay sau khi Nhập liệu Khóa (Entry) đầu tiên của Thế hệ mới. Tuyến Đọc (Reader) xông vào lúc Tuyến Ghi tạm dừng, tạo tiền đề Lỗi Xác Định (Deterministic) hoàn toàn không dựa vào xác suất.

```java
@Test
void readerCanObserveOnlyPartOfTheNewGeneration() throws Exception {
    HookedBrokenRegistry registry = new HookedBrokenRegistry();
    registry.replaceForSetup(routes(41)); // Nạp Thế hệ cũ

    CountDownLatch firstEntryPublished = new CountDownLatch(1);
    CountDownLatch allowRefreshToFinish = new CountDownLatch(1);

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<?> refresh = executor.submit(() -> registry.refresh(
                routes(42),
                () -> {
                    firstEntryPublished.countDown(); // Tín hiệu: Đã nạp 1 phần
                    awaitOrFail(allowRefreshToFinish, Duration.ofSeconds(5));
                }
        ));

        assertTrue(firstEntryPublished.await(5, TimeUnit.SECONDS));

        // CHỨNG MINH RỦI RO: Đọc cấu trúc, phơi bày dữ liệu dở dang
        Map<String, PaymentRoute> observed = registry.copyForDiagnostic();
        assertEquals(1, observed.size()); // Sập Invariant (Mọi Thế hệ có 2)
        assertTrue(observed.values().stream()
                .allMatch(route -> route.generation() == 42));

        allowRefreshToFinish.countDown(); // Nhả khóa cho Tuyến Ghi chạy tiếp
        refresh.get(5, TimeUnit.SECONDS);

        assertEquals(2, registry.copyForDiagnostic().size());
    }
}
```

Mục tiêu không phải là ép phát sinh `ConcurrentModificationException`. Hệ thống ưu tiên đánh phá Quy tắc Bất Biến (Invariant): Tuyến Đọc phải chứng kiến bằng được cái khung Bản Đồ chỉ chứa 1 mảnh vụn dữ liệu dẫu cả 2 Thế hệ Gốc đều có Khối lượng là 2.

## 3. Thí Nghiệm 2: Tôn Tôn Trọng Bản Chụp Nguyên Tử & Chặn Đứng Tuyến Ghi Chậm Trễ (Stale Writer)

Cuộc đụng độ của 2 Tuyến Làm Mới (Writer). Kịch bản: Tuyến Ghi Thế hệ 42 bị phong tỏa; Thế hệ 43 lướt qua xuất bản trước. Sau khi nhả Khóa, Khối Logic So-Sánh-Và-Gán (Compare-and-set) phải cự tuyệt tàn nhẫn Tuyến Ghi 42 Lỗi thời.

```java
@Test
void snapshotIsCompleteAndOlderWriterCannotOverwriteNewerGeneration() throws Exception {
    PaymentRoutingRegistry registry = registryWithoutRemoteCalls();
    assertTrue(registry.publishIfNewer(snapshot(41)));

    CountDownLatch olderWriterReady = new CountDownLatch(1);
    CountDownLatch releaseOlderWriter = new CountDownLatch(1);

    try (ExecutorService executor = Executors.newSingleThreadExecutor()) {
        Future<Boolean> olderResult = executor.submit(() -> {
            RoutingSnapshot older = snapshot(42); // Chuẩn bị Thế hệ 42
            olderWriterReady.countDown();
            awaitOrFail(releaseOlderWriter, Duration.ofSeconds(5)); // Bị Đóng Băng
            return registry.publishIfNewer(older); // Nỗ lực Xuất bản (Sẽ thất bại)
        });

        assertTrue(olderWriterReady.await(5, TimeUnit.SECONDS));
        assertTrue(registry.publishIfNewer(snapshot(43))); // Thế hệ 43 Vượt lên trước

        RoutingSnapshot whileOlderWriterIsBlocked = registry.snapshot();
        assertCompleteGeneration(whileOlderWriterIsBlocked, 43); // 43 Đã thống trị

        releaseOlderWriter.countDown();
        assertFalse(olderResult.get(5, TimeUnit.SECONDS)); // Bị Từ Chối Thẳng Thừng
        assertCompleteGeneration(registry.snapshot(), 43); // 43 Bất Khả Xâm Phạm
    }
}
```

Kiểm định Quy tắc: Bản chụp vinh danh 2 Khóa trọn vẹn và 100% Giá trị chia sẻ Chung 1 Dấu mộc Thế hệ. Cấm lạm dụng Phép đo Kích thước (Size) hoặc Cờ Ngoại Lệ để thay thế Xác thực Invariant.

## 4. Thí Nghiệm 3: Thẩm Định Năng Lực Trụ Vững Khi Cập Nhật Vỡ Lở (Failure Keeps Last-Known-Good)

Phép thử Hồi quy (Regression test) cốt lõi chứng minh Nền tảng "Dựng Khung Trước Xuất Bản Sau" (Build-before-publish):

```java
@Test
void invalidRefreshDoesNotDestroyCurrentSnapshot() {
    java.util.concurrent.atomic.AtomicInteger calls =
            new java.util.concurrent.atomic.AtomicInteger();

    RoutingConfigClient client = () -> {
        if (calls.getAndIncrement() == 0) return snapshot(41);
        return new RoutingSnapshot(42, Map.of()); // Tung Rác Dữ Liệu
    };
    PaymentRoutingRegistry registry = new PaymentRoutingRegistry(client);

    registry.refresh();
    assertCompleteGeneration(registry.snapshot(), 41);

    // Kích hoạt Cơ chế Lọc Rác - Hệ thống Văng Lỗi Validation
    assertThrows(IllegalArgumentException.class, registry::refresh);

    // Bằng chứng Thép: Cấu trúc Gốc Nguyên vẹn
    assertCompleteGeneration(registry.snapshot(), 41); 
}
```

Chứng minh một Đợt làm mới Đổ nát không kéo theo sự Diệt vong của Thế hệ Đang Phục Vụ.

## 5. Thí Nghiệm 4: Khảo Sát Bức Tranh Lỗ Hổng Công Bố Dữ Liệu (Unsafe Publication Stress)

Tuyệt đối cấm khai báo JUnit bắt buộc "Reader must observe stale value", vì Bộ Lập Lịch không hỗ trợ Ký Hợp Đồng Hệ Quả (Happens-before). Ứng dụng OpenJDK JCStress để vạch trần thảm họa cho phép theo định nghĩa JMM:

```java
@JCStressTest
@Outcome(id = "0", expect = Expect.ACCEPTABLE_INTERESTING,
        desc = "Reader đã va vào Bản Chụp Lỗi Thời")
@Outcome(id = "1", expect = Expect.ACCEPTABLE,
        desc = "Reader quan sát Bản Chụp Mới")
@State
public class UnsafeSnapshotPublicationStress {

    private Map<String, PaymentRoute> routes = Map.of();

    @Actor
    public void writer() {
        routes = Map.of(
                "merchant-a",
                new PaymentRoute("provider-a", true, 1)
        );
    }

    @Actor
    public void reader(I_Result result) {
        result.r1 = routes.size(); // Bóc trần Data Race
    }
}
```

Kết quả `0` phơi bày việc Thiếu Hợp Đồng Khả Kiến. Dẫu cho đã bọc `volatile` hay `AtomicReference`, Thiết kế Chuẩn Mực Outcome buộc phải phản ánh Trật Tự (Ordering) Nghiệp Vụ, Khước từ quy chụp Đặc tính Lập Lịch (Scheduling) làm Trật Tự Bộ Nhớ (Memory).

## 6. Tiêu Chuẩn Ngữ Nghĩa ConcurrentHashMap

Nếu đánh đổi kiến trúc Bản Chụp Bất Biến lấy tốc độ của `ConcurrentHashMap`, bài Kiểm Định Bắt Buộc chứng minh:
- Không xuất hiện Đứt gãy Cấu trúc và Vắng bóng `ConcurrentModificationException`.
- Định vị Dữ liệu Bất khả Xâm phạm Nguyên Tử (Atomic) cho Từng Đơn vị Khóa.
- Báo cáo Toàn Bảng chỉ được phép định danh là "Ước Tán" (Approximate).
*Cảnh báo: Phế truất ngay Lựa chọn này nếu Nghiệp vụ ép buộc Chuẩn Toàn Bảng Chung Thế Hệ.*

## 7. Khung Giám Sát Khai Thác (Production Blueprint)

Đặc tả Vận hành:
- Tuổi thọ Bản Chụp và Định danh Thế Hệ (Generation ID).
- Số khối lượng (Entry Count) và Chứng Chỉ Tính Toàn Vẹn (Checksum).
- Thống kê tỷ lệ Từ Chối Bản Chụp Trễ Hạn (Stale publish rejected).
- Đo lường Yêu cầu Hụt Tuyến (Fallback).
- Bản đồ Phân Phối Thế Hệ giữa Liên Nút (Node).

## 8. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [ ] Bài thử Broken Test giam luồng Đọc trong Khe hở Cập nhật bằng Latch.
- [ ] Xóa bỏ Phép Đo Lường Vận Mệnh bằng `Thread.sleep`.
- [ ] 100% Khối Chặn Latch/Future bọc vỏ Timeout.
- [ ] Kiểm định Hồi Quy bao phủ Tổng Số Lượng và Tính Đồng Nhất Thế Hệ.
- [ ] Bài Test Phơi bày Áp Lực Đa Tuyến Ghi và Khước từ Thế Hệ Lỗi Thời.
- [ ] Xác nhận Sập Cập Nhật luôn được Neo Giữ bởi Bản Chụp Tốt Nhất Cuối Cùng.
- [ ] Ngữ Nghĩa Vòng Lặp phản chiếu chính xác Lựa chọn Hạch Tâm Collection.
- [ ] Hậu Kiểm Thử: Đóng Executor và Khôi Phục Cờ Tín Hiệu Ngắt (Interrupt).
