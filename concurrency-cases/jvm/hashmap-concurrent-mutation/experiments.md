# Môi Trường Thực Nghiệm: Cấu Trúc Kiểm Định Rủi Ro & Đánh Giá Khai Thác

## 1. Chiến Lược Kiểm Thử Cốt Lõi (Core Strategy)

Kế hoạch test của chúng ta sẽ xoáy sâu vào 3 luồng rủi ro chính:
1. Chứng minh đoạn code lỗi (Broken code) sẽ để lộ ra dữ liệu gãy vụn lúc đang cập nhật.
2. Chứng minh Cấu trúc Bản Chụp Bất Biến (Immutable snapshot) bảo vệ an toàn 100% dữ liệu và bắt buộc các Thế Hệ phải tăng dần đều.
3. Rà soát xem cách duyệt danh sách (Iteration) có đáp ứng đúng mong mỏi của người gọi hàm hay không.

Để làm được, chúng ta xài `Latch` (chốt chặn) để điều khiển các luồng lao vào đúng ngay cái "Tọa độ tử thần" (lúc tranh chấp). Tuyệt đối KHÔNG dùng `Thread.sleep` để đoán thời gian vì cái Bộ lập lịch (Scheduler) của máy ảo nó "quay xe" bất ngờ lắm. Và nhớ, phải đặt Thời gian chờ tối đa (Timeout) cho mọi lệnh chốt chặn để test không bị treo vĩnh viễn.

## 2. Thí Nghiệm 1: Trình Diễn Bóc Tách Bản Chụp Phân Mảnh (Partial Snapshot)

Ở bài test này, chúng ta sẽ "tóm cổ" Luồng ghi ngay lúc nó vừa mới nhét được ĐÚNG 1 phần tử của Thế hệ mới vào Map. Ngay khoảnh khắc Luồng ghi bị đứng hình, Luồng đọc sẽ lao vào. Lỗi này là chắc chắn 100% xảy ra, không phụ thuộc vào may rủi.

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

Mục đích ở đây không phải là ép nó văng lỗi `ConcurrentModificationException`. Chúng ta muốn chứng minh sự vi phạm luật chơi (Invariant): Luồng đọc nhìn thấy cái Map chỉ có ĐÚNG 1 mẩu dữ liệu, trong khi cả Thế hệ cũ và mới đáng lẽ phải luôn chứa 2 mẩu!

## 3. Thí Nghiệm 2: Tôn Tôn Trọng Bản Chụp Nguyên Tử & Chặn Đứng Tuyến Ghi Chậm Trễ (Stale Writer)

Đây là cuộc chiến sinh tử giữa 2 Luồng Ghi. Kịch bản: Luồng cầm Thế hệ 42 bị kẹt đèn đỏ; Luồng cầm Thế hệ 43 phóng lên trước xuất bản thành công. Khi Luồng 42 được nhả ra, cơ chế So-Sánh-Và-Gán (CAS) sẽ thẳng tay từ chối cái dữ liệu quá đát của nó.

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

Kiểm định luật chơi: Bản chụp luôn giữ nguyên vẹn 2 mẩu khóa và 100% dữ liệu phải mang chung cái mộc Thế Hệ. Đừng bao giờ lôi số đếm (size) hay rình rập Exception ra làm thước đo chuẩn mực!

## 4. Thí Nghiệm 3: Thẩm Định Năng Lực Trụ Vững Khi Cập Nhật Vỡ Lở (Failure Keeps Last-Known-Good)

Bài test cực kỳ quan trọng để bảo vệ triết lý "Xây nhà xong hết mới tung biển quảng cáo" (Build-before-publish):

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

Bài test chứng minh rõ ràng: Một đợt cập nhật bị thối (tung rác) sẽ bị chặn đứng, và cấu trúc đang chạy ngon lành ở Thế hệ cũ (41) vẫn vĩnh viễn an toàn.

## 5. Thí Nghiệm 4: Khảo Sát Bức Tranh Lỗ Hổng Công Bố Dữ Liệu (Unsafe Publication Stress)

Đừng dại mà viết JUnit test kiểu "bắt Luồng Đọc phải thấy dữ liệu cũ", bởi vì Bộ lập lịch không bao giờ hứa hẹn luật lệ đó (Happens-before). Hãy dùng tool `OpenJDK JCStress` để tra tấn và lột trần cái mớ lý thuyết thảm họa của Java Memory Model:

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

Nếu kết quả ra `0`, chứng tỏ bạn đang quên ký "Hợp Đồng Khả Kiến" giữa các luồng. Dù bạn đã bọc `volatile` hay `AtomicReference`, bạn vẫn phải thiết kế luật lệ sao cho nó chuẩn Nghiệp Vụ, chứ đừng nương tựa vào sự hên xui của Trật tự Bộ Lập Lịch.

## 6. Tiêu Chuẩn Ngữ Nghĩa ConcurrentHashMap

Nếu bạn chấp nhận xài `ConcurrentHashMap` để đổi lấy tốc độ và đánh rơi Bản chụp Bất biến, thì bài test phải chứng minh được:
- Ứng dụng không bao giờ bị nát cấu trúc và văng lỗi `ConcurrentModificationException`.
- Dữ liệu ở từng chiếc Khóa (Key) luôn chuẩn xác 100% (Atomic).
- Việc in ra toàn bộ Map phải gọi rõ ràng là "Ước Tán" (Approximate).
*Cảnh báo: Nếu sếp bạn bắt buộc bảng dữ liệu phải đồng nhất Thế Hệ cùng một lúc, dẹp ngay cái giải pháp này đi nhé!*

## 7. Khung Giám Sát Khai Thác (Production Blueprint)

Đồ chơi mang lên Production cần có:
- Đo xem tuổi thọ của Bản Chụp sống được bao lâu, Thế Hệ (ID) đang là mấy.
- Lượng Khóa (Entry Count) và Chứng Chỉ Tính Toàn Vẹn (Checksum).
- Thống kê xem có bao nhiêu đợt cập nhật trễ bị tát văng (Stale publish rejected).
- Đếm số ca đi tìm đối tác không ra phải xài dự phòng (Fallback).
- Biểu đồ xem Thế hệ đang rải rác thế nào giữa các máy chủ (Node).

## 8. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

Trước khi đóng code, rà soát lại:
- [ ] Bài test Broken Test đã kẹp cổ Luồng Đọc vào giữa khe hở cập nhật bằng Latch chưa?
- [ ] Xóa bỏ hết thói hư tật xấu dùng `Thread.sleep` để đoán mò chưa?
- [ ] 100% các Khối Chặn Latch/Future đã có cái bọc Timeout chưa?
- [ ] Chạy Hồi Quy (Regression) kiểm chứng Tổng Số Lượng và tính Đồng Nhất Thế Hệ chưa?
- [ ] Chạy tra tấn xem nhiều Luồng Ghi có đấm đá được nhau để chối bỏ Thế hệ quá đát không?
- [ ] Chứng nhận rằng cập nhật tịt ngòi thì hệ thống vẫn giữ nguyên Bản Chụp cuối cùng.
- [ ] Luật duyệt danh sách đúng chuẩn với Loại Collection bạn chọn.
- [ ] Dọn dẹp chiến trường: Đóng Executor và trả lại Cờ Tín Hiệu (Interrupt) sạch sẽ.
