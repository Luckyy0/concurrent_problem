# Môi Trường Thực Nghiệm: Cấu Trúc Kiểm Định Rủi Ro & Đánh Giá Khai Thác

## 1. Thí Nghiệm 1: Tái Hiện Bế Tắc (Self-Starvation)

Chúng ta dùng Latch để cố tình cho 2 Tác vụ Cha chạy và chiếm trọn 2 luồng công nhân, rồi mới cho chúng gọi Tác vụ Con ra chạy:

```java
@Test
void parentsStarveChildrenOnTheSamePool() throws Exception {
    ThreadPoolExecutor pool = new ThreadPoolExecutor(
            2, 2, 0, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(10)
    );
    CountDownLatch bothParentsRunning = new CountDownLatch(2);

    Callable<String> parent = () -> {
        bothParentsRunning.countDown();
        if (!bothParentsRunning.await(5, TimeUnit.SECONDS)) {
            throw new IllegalStateException("Giả lập lỗi rồi");
        }
        // Cha đẩy Con vào chung Hàng đợi rồi đứng đực ra đợi
        Future<String> child = pool.submit(() -> "child-done");
        return child.get(30, TimeUnit.SECONDS); 
    };

    Future<String> first = pool.submit(parent);
    Future<String> second = pool.submit(parent);
    try {
        // Test sẽ pass nếu bị Timeout - chứng tỏ hệ thống đã đứng hình
        assertThrows(TimeoutException.class,
                () -> first.get(300, TimeUnit.MILLISECONDS));
        assertEquals(2, pool.getActiveCount()); // 100% Công nhân đang kẹt chờ
        assertEquals(2, pool.getQueue().size()); // 100% Việc con mắc kẹt dưới hàng đợi
        assertEquals(0, pool.getCompletedTaskCount()); // Chả làm được tích sự gì
    } finally {
        first.cancel(true);
        second.cancel(true);
        pool.shutdownNow();
        assertTrue(pool.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Dùng Latch để chắc chắn hệ thống luôn kẹt đúng kịch bản mong muốn, thay vì dùng `Thread.sleep` hên xui.

> **Mẹo nhỏ:** Việc lôi ra bằng chứng "Luồng đầy, hàng đợi đầy, việc xong bằng 0" đáng giá hơn rất nhiều so với việc chỉ nhìn thấy một cái báo lỗi Timeout.

## 2. Thí Nghiệm 2: Giải Quyết Bằng Cách Gọi Trực Tiếp (Direct Call)

```java
@Test
void directDependencyCallCompletesAtPoolCapacity() throws Exception {
    ExecutorService pool = Executors.newFixedThreadPool(2);
    CountDownLatch start = new CountDownLatch(1);
    Callable<String> task = () -> {
        if (!start.await(5, TimeUnit.SECONDS)) throw new IllegalStateException();
        return loadPriceDirectly(); // Gọi hàm xử lý luôn, khỏi qua hàng đợi
    };
    Future<String> first = pool.submit(task);
    Future<String> second = pool.submit(task);
    start.countDown();
    try {
        assertEquals("price", first.get(5, TimeUnit.SECONDS));
        assertEquals("price", second.get(5, TimeUnit.SECONDS));
    } finally {
        pool.shutdownNow();
        assertTrue(pool.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Chạy thẳng hàm `loadPriceDirectly()` luôn, không thèm quăng vào queue nữa, thế là ngon lành!

## 3. Thí Nghiệm 3: Test Lỗi Từ Chối Và Hủy Bỏ (Rejection & Cancellation)

Test này mình tạo Pool 1 luồng + Queue 1 chỗ. Khóa luồng bằng Latch, nhét 1 việc vào Queue, xong nhét thêm việc thứ 3. Hệ thống phải quăng lỗi `RejectedExecutionException` ngay lập tức.
Sau đó mở Latch, xác nhận 2 việc kia chạy bình thường.
Nếu có tác vụ nào bị báo Timeout, thì mình cũng phải test xem nó có nhận được lệnh `cancel(true)` để dọn dẹp sạch sẽ hay không.

## 4. Test Sức Chịu Tải (Load/Stress Checks)

Bơm request dồn dập vào hệ thống xem nó chịu nhiệt ra sao. Nhớ kiểm tra:
- Hàng đợi có bị bung nóc không?
- Lỗi báo từ chối (Rejection) phải văng ra trước khi RAM bị đầy nghẹt.
- Số lượng task làm xong vẫn phải tăng đều đặn.
- Thời gian chờ của mấy thằng xếp hàng không được quá lâu.
- Bọn bị từ chối không được lặp lại nhồi đạn (Retry) vào đúng cái server đang sấp mặt đó.

*(Lưu ý: Test chịu tải này không dùng để thay thế cho cái bài Test logic ở bước 1 đâu nha.)*

## 5. Cần Giám Sát Những Gì Trên Production (Production Blueprint)

Đưa lên môi trường thật thì nhớ check:
- Pool đang chạy bao nhiêu luồng, max bao nhiêu, hàng đợi chứa bao nhiêu.
- Task cũ nhất dưới hàng đợi đã nằm đó bao lâu rồi.
- Tốc độ nhận việc, chạy việc, hoàn thành và từ chối.
- Nếu kẹt, lấy Thread Dump ra xem có phải mấy ông kẹ đang đứng ngáp ở hàm `Future.get()` không.
- Coi chừng xài hết băng thông hay connection xuống DB.
- Lúc tắt server (Graceful shutdown) nó tốn bao lâu và có bao nhiêu task tèo giữa chừng.

## 6. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [ ] Đã dùng Latch để test chắc chắn 100% Cha chiếm luồng.
- [ ] Tất cả hàm `await` hay `get` đều phải nhét Timeout vào.
- [ ] Bỏ hẳn thói quen dùng `Thread.sleep` khi viết Test.
- [ ] Lúc văng lỗi đói tài nguyên, phải log đủ cả Active, Queued, Completed.
- [ ] Test xong nhớ Cleanup sạch sẽ.
- [ ] Phải có bài Test chứng minh cách fix trên một Pool duy nhất.
- [ ] Test đủ cơ chế Rejection khi dồn tải qua cửa.
- [ ] Trọn bộ Monitor đo được rõ thời gian kẹt trong hàng đợi và thời gian chạy thực tế.
