# Môi Trường Thực Nghiệm: Cấu Trúc Kiểm Định Rủi Ro & Đánh Giá Khai Thác

## 1. Thí Nghiệm 1: Trình Diễn Bế Tắc Khép Kín (Self-Starvation)

Sử dụng chốt điều hướng (Latch) để đảm bảo hai Tác vụ Cha cùng đoạt quyền kiểm soát Luồng Công Nhân trước khi phóng Tác vụ Con ra tiền tuyến:

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
            throw new IllegalStateException("Quy trình giả lập đan chéo đổ vỡ");
        }
        // Cha đẩy Con vào Hàng đợi và chôn chân chờ đợi
        Future<String> child = pool.submit(() -> "child-done");
        return child.get(30, TimeUnit.SECONDS); 
    };

    Future<String> first = pool.submit(parent);
    Future<String> second = pool.submit(parent);
    try {
        // Tái xác nhận hệ thống đã đóng băng và văng lỗi Quá hạn
        assertThrows(TimeoutException.class,
                () -> first.get(300, TimeUnit.MILLISECONDS));
        assertEquals(2, pool.getActiveCount()); // 100% Công Nhân đang kẹt
        assertEquals(2, pool.getQueue().size()); // 100% Con bị nhốt
        assertEquals(0, pool.getCompletedTaskCount()); // Không một ai hoàn thành
    } finally {
        first.cancel(true);
        second.cancel(true);
        pool.shutdownNow();
        assertTrue(pool.awaitTermination(5, TimeUnit.SECONDS));
    }
}
```

Hệ thống Timeout chỉ mang ý nghĩa làm Người gác cổng (Watchdog/Assertion), trong khi cấu trúc Latch mới là kiến trúc sư định hình Hình thái Cố định (Deterministic topology). Quy trình Hủy thử nghiệm (Test cleanup) tác động lên Cha; Cơ chế Ngắt (Interruption) sẽ ép hàm `child.get()` buông tay.

> **Nguyên tắc kỹ thuật:** Bằng chứng pháp lý "Công nhân chạy toàn bộ, Trẻ em kẹt trong hàng đợi, Hoàn thành bằng Không" khắc họa chân dung Nạn đói Tài nguyên một cách đanh thép hơn hàng vạn lần so với một tín hiệu Timeout đơn lẻ.

## 2. Thí Nghiệm 2: Truy Vấn Trực Tiếp Giải Tỏa Tiến Trình (Direct Call)

```java
@Test
void directDependencyCallCompletesAtPoolCapacity() throws Exception {
    ExecutorService pool = Executors.newFixedThreadPool(2);
    CountDownLatch start = new CountDownLatch(1);
    Callable<String> task = () -> {
        if (!start.await(5, TimeUnit.SECONDS)) throw new IllegalStateException();
        return loadPriceDirectly(); // Truy xuất trực tiếp, không qua Hàng đợi
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

Kiến tạo Giả lập (Fake dependency) bắt buộc tuân thủ tính Định Lượng (Bounded/Deterministic); Khước từ vĩnh viễn cấu trúc Ngủ (Sleep).

## 3. Thí Nghiệm 3: Cơ Chế Khước Từ Và Hủy Bỏ (Rejection & Cancellation)

Kiến trúc thử nghiệm: Pool 1 + Queue 1. Phong tỏa Luồng Công Nhân qua Latch, tống khứ một Tác vụ vào Hàng đợi, tiếp tục nhồi Tác vụ thứ 3. Kích hoạt xác thực `RejectedExecutionException` ngay tại Cổng kiểm soát (Admission boundary).
Kế tiếp, phá khóa Latch và nghiệm thu hai Tác vụ lọt lưới đã hoàn tất quy trình. 
Luồng Kiểm định Timeout (Timeout path) bắt buộc xác nhận các Tác vụ Con phải hứng chịu tín hiệu `cancel(true)` và cờ Trạng Thái Ngắt (Interrupt status) của Luồng Công Nhân đã được khôi phục.

## 4. Xác Minh Sức Chịu Tải (Load/Stress Checks)

Kích thích Cường độ Yêu cầu (Request rate) vượt ngưỡng Cổng kiểm soát và chứng thực:
- Dung lượng Hàng đợi không bùng nổ vượt giới hạn.
- Sự kiện Từ chối (Rejection) xuất hiện trước khi Dung lượng Bộ nhớ (Memory growth) phình to.
- Chỉ số Hoàn Tất (Completed count) vận hành bền bỉ.
- Độ tuổi P99 của Hàng đợi (P99 queue age) nằm gọn trong Hạn mức cho phép.
- Cơ chế Thử lại (Retry) không tự lao đầu vào lại chính Nút (Node) đang tử nạn.
Lưu ý: Bộ Stress Test tuyệt đối không thay thế đặc tính chứng minh logic của bài Test Chờ Lồng Nhau.

## 5. Giám Sát Môi Trường Khai Thác (Production Blueprint)

Đặc tả vận hành:
- Giám sát Luồng Active/Max, Thông lượng Hàng đợi/Sức chứa, Tuổi đời Tác vụ tồn đọng (Oldest task age);
- Biên độ biến động Tác vụ Đệ trình/Bắt đầu/Hoàn Tất/Từ Chối/Hủy (Delta);
- Khảo sát Thời gian Chờ (Queue wait), Thời lượng Thi Hành (Execution), Độ trễ Cấu trúc phụ thuộc (Dependency latency) và Hạn mức Cuối (End-to-end deadline);
- Trích xuất Thread Dump vạch trần Ngăn Xếp Luồng Công Nhân (Worker stack) mắc kẹt tại `Future.get` / Nhập Xuất Viễn Trình (Remote I/O);
- Đánh giá khả năng Tiêu thụ Tài nguyên Ngoại vi / Hạn Mức Tải (Downstream connection/Quota utilization);
- Theo dõi Độ trễ Tắt Hệ Thống (Graceful shutdown elapsed) và Khối lượng Tác vụ Chết yếu (Unfinished task count).

## 6. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [ ] Latch chứng minh hai Tác vụ Cha đã độc chiếm Luồng Công Nhân trước thềm Tác vụ Con ra đời.
- [ ] 100% Mỏ neo `await`/`get` được gắn Timeout.
- [ ] Xóa bỏ hoàn toàn định dạng `Thread.sleep`.
- [ ] Chứng thực Nạn đói Tài nguyên bao hàm cả chuỗi số liệu: Active, Queued, Completed.
- [ ] Tái lập quy trình Cleanup: Hủy Tác vụ và Neo hệ thống chờ Executor Shutdown.
- [ ] Fixed test chứng minh Mạch Tiến Trình thông suốt trên cùng cấu trúc Pool.
- [ ] Chính sách Khước từ (Rejection policy) đụng độ Bài kiểm định Cửa ngõ Quá tải (Overload boundary).
- [ ] Metric Production phân định rạch ròi Độ Trễ Hàng Đợi (Queue wait) và Thời lượng Thi Hành.
