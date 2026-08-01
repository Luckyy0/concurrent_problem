# Môi Trường Thực Nghiệm: Phân Tích Và Đo Lường Cấu Trúc Khóa Đồng Thời

## 1. Chiến Lược Kiểm Thử Cốt Lõi (Core Strategy)

Để chứng minh khóa chạy xịn hay chạy cùi, bài test của chúng ta phải trả lời được 4 câu:
1. Xem thử 2 cái khóa (Lock objects) khác nhau có cản được 2 luồng (Actors) cùng lao vào hay không. (Chắc chắn là không).
2. Xem thử 2 chuỗi Key nội dung giống y chang nhưng khác địa chỉ ô nhớ thì có ép dùng chung 1 khóa được không. (Cũng không luôn).
3. Đánh giá sức mạnh của Khóa phân dải (Striped Lock) trong việc bảo vệ dữ liệu trên một máy (Service Instance).
4. Phơi bày sự bất lực của khóa nội bộ trên một máy khi phải chống chọi trong môi trường đa máy chủ (Multi-node).

Mình sẽ xài cái hàng rào giả (Fake Store Latch) chắn ngang ngay sau hàm `exists` để dụ tất cả các luồng vào thấy "File chưa có". Tuyệt đối không dùng `Thread.sleep` cùi bắp; dùng Timeout đếm ngược cho chuẩn!

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

Hàng rào (Barrier) này chỉ sụp khi cả 2 luồng cùng đến được điểm `exists`. NẾU khóa xịn, thằng đến sau phải bị Timeout cản lại. Còn đằng này, vì mỗi thằng đẻ 1 cái khóa mới nên cả 2 luồng vượt rào êm ru, tạo file 2 lần và ghi 2 lần. Toang!

> **Nguyên tắc kỹ thuật:** Đừng có soi mã nguồn xem lock kiểu gì. Chạy bài test này, nếu nó cho phép 2 luồng cùng xử lý 1 file duy nhất thì code đó sai bét.

## 3. Thí Nghiệm 2: Trị Số Chuỗi Khớp (Equal) KHÔNG Bằng Đồng Nhất Tham Chiếu Khóa (Monitor)

Mô phỏng lại cái Ảo giác Khóa (tưởng chuỗi giống nhau là khóa dính nhau):

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
Nhìn kìa, nội dung chuỗi giống hệt nhau, nhưng chúng là 2 object khác nhau. Kết cục? Hai luồng ôm 2 cái khóa riêng rẽ, mạnh ai nấy chạy qua cửa!

## 4. Thí Nghiệm 3: Năng Lực Trấn Thủ Của Khóa Phân Dải (Striped Lock)

Ở đây, request tới sau phải bị nghẽn lại ở dải khóa. Chờ luồng đầu tiên xuất bản file xong xuôi, thằng đến sau tỉnh mộng nhận thông báo "Có file rôi má ơi" rồi quay xe, không tính toán lại dư thừa nữa.

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

Kết quả quá mỹ mãn! Chỉ có 1 lần tính toán (render) và 1 lần lưu (put). Hệ thống đứng vững.

## 5. Thí Nghiệm 4: Hai Bản Thể Máy Chủ Phơi Bày Giới Hạn Khóa Nội Bộ

Thử làm kiến trúc 2 Server. Mỗi máy có 1 bộ khóa phân dải `StripedKeyLocks` riêng biệt, nhưng cùng đâm vào cái Kho dữ liệu chung `BarrierStore`.

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

Cú lừa ngoạn mục! Máy nào khóa máy nấy, chả liên quan gì nhau. Cả 2 cùng qua lọt, render 2 lần và ghi 2 lần lên cái Store chung. Đây là bài test bắt buộc phải có để xóa tan ảo mộng về việc dùng Khóa cục bộ bảo vệ cụm máy chủ!

## 6. Thí Nghiệm 5: Lệnh Cập Nhật Kho Khởi Tạo Có Điều Kiện Định Hình Vương Quyền Độc Tôn (Conditional Create)

Bức tường Kho Lưu Trữ tự thân vận động phán xử bằng hàm Atomic `putIfAbsent`:

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

Dù 2 máy cùng cắm đầu chạy render tốn CPU, nhưng khi lao tới Store thì Store gạt phăng thằng chậm chân, trả về lỗi "Đã có người lưu rồi ba". Hệ thống an toàn tuyệt đối. Thực chiến là phải xài cái này nha anh em!

## 7. Khung Kiểm Định Quản Trị Khóa Mở Rộng: Timeout, Interruption, Tình Trạng Lỗi

Đừng quên test thêm các "thế kẹt" hiểm hóc sau:
- Luồng 2 văng lỗi `ArtifactBusyException` do đứng chờ khóa quá lâu (Timeout) trong khi Luồng 1 vẫn giữ chặt.
- Thử gửi tín hiệu ngắt (Interrupt) cho Luồng 2 xem nó có bung ra và giữ nguyên cờ Ngắt không.
- Làm cho đoạn code Render văng Exception, xem khóa có nhả ra đàng hoàng cho đứa sau không (hay ôm chết luôn).
- Giả lập Kho Store bị timeout; lúc này nên cảnh báo anh em làm tính năng "Thử lại và đối soát".
- Thử băm 2 file khác nhau vào chung một khóa phân dải xem nó có bị lỗi data không (không hề, nó chỉ chạy tuần tự lại xíu thôi).
- Thử quăng cái khóa qua Luồng 2 bắt nó xả khóa dùm Luồng 1. KHÔNG BAO GIỜ CHO PHÉP NHÉ!

## 8. Khung Giám Sát Khai Thác Môi Trường (Production Blueprint)

Để ý mấy chỉ số vàng này nha:
- Thời gian phải chờ khóa, tỉ lệ bị quá hạn chờ (Timeout), số luồng đang chạy.
- So sánh số lần render với số lần ghi file thành công.
- Theo dõi bao nhiêu lần đâm nhau lên Kho mà bị từ chối (báo `ALREADY_EXISTS`).
- Quan sát độ dài hàng đợi và xem có dải khóa nào bị "nhồi" nhiều quá không (Hot spot).

Lưu ý nhỏ: Đừng bao giờ dựa vào cái hàm soi số luồng đang chờ của `ReentrantLock` để làm cơ sở logic, sai bét đấy.

## 9. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [ ] Phơi bày được cấu trúc khóa nhờ mấy hàng rào Latch/Barrier.
- [ ] Vạch trần lỗi khóa khi 2 chuỗi khác địa chỉ ô nhớ.
- [ ] Khảo sát việc ghi đè vô duyên do khóa tồi.
- [ ] Đo thử độ tệ hại của khóa nội bộ trên 2 Server giả lập.
- [ ] Test ranh giới của việc khởi tạo dữ liệu xài lệnh Có Điều Kiện trên Store.
- [ ] Dẹp luôn `Thread.sleep` khỏi hệ thống test.
- [ ] Buộc phải có giới hạn thời gian (Timeout) cho mọi hàng rào Latch.
- [ ] Thử thách khối `try-finally` trước các sự cố văng Exception và Gián đoạn (Interrupt).
- [ ] Đảm bảo dọn dẹp sạch sẽ mớ tiểu trình (Thread Pool) sau khi chạy test.
