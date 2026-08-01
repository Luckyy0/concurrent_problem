# Giải Pháp Kiến Trúc Tối Ưu và Chiến Lược Hoạch Định Sức Chứa

## 1. Phương Án 1: Bỏ Ngay Việc Ném Task Lồng Nhau (Drop Nested Submission)

Cách tốt nhất là: Đã có luồng rồi thì gọi trực tiếp hàm xử lý luôn, đừng đẻ thêm hàng đợi làm gì cho rườm rà.

```java
public EnrichedOrder enrich(Order order) {
    Price price = pricingClient.loadPrice(order.productId()); // Gọi trực tiếp luôn cho khỏe
    return new EnrichedOrder(order.orderId(), price);
}
```

Cách này nhổ tận gốc rễ vấn đề "Nạn đói nội bộ". Lưu ý nhỏ: Khi gọi API ra ngoài thì tự cài timeout cho nó nhé. Và số luồng của Pool không được lớn hơn tổng số connection cấp phép ở bên dưới.

## 2. Phương Án 2: Dùng Người Điều Phối Trực Tiếp (Orchestrator Submits Leaf Tasks)

Thay vì để Cha tự gọi Con, hãy tạo một người quản lý chung (Orchestrator). Ông này đứng ngoài Pool, giao việc nhỏ (Leaf tasks) thẳng vào Pool luôn. Cấm tiệt chuyện đưa ông Cha vào chung mâm với các việc nhỏ. Nếu quá hạn (Timeout), ông quản lý sẽ hú còi hủy hết các tác vụ đang kẹt. Chi tiết hơn xem bài `JVM-009`.

## 3. Phương Án 3: Tách Riêng Pool Ra (Separate Pools)

Nếu buộc phải đẻ thêm tác vụ, thì Pool của Cha và Pool của Con phải riêng biệt. Cơ mà cách này cực kỳ mệt óc lúc setup giới hạn (Sizing). Pool Cha sinh việc nhanh quá mà Pool Con gánh không nổi thì Pool Con cũng banh xác. Tức là chỉ chuyển bệnh từ phòng này sang phòng khác thôi.

## 4. Cấu Hình ThreadPoolExecutor Chuẩn Thực Tế (Production Baseline)

```java
new ThreadPoolExecutor(
        coreSize,
        maxSize,
        30, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(queueCapacity),
        threadFactory,
        new ThreadPoolExecutor.AbortPolicy() // Quá tải thì văng lỗi, từ chối thẳng thừng!
);
```

Bắt được lỗi `RejectedExecutionException` thì báo luôn cho user là "Hệ thống quá tải", tuyệt đối không được nuốt lỗi (Swallow). Dùng `CallerRunsPolicy` chỉ khi bạn muốn hệ thống tự làm chậm lại, nhưng cẩn thận vì luồng gọi sẽ bị kẹt lại để làm thay tác vụ đó.

## 5. Chặt Chẽ Với Deadline Và Hủy Bỏ (Deadline & Cancellation)

Luôn dùng `future.get(thời gian còn lại)`. Nếu lố giờ thì gọi `cancel(true)` ngay để giải phóng. Gọi API hay DB thì phải có timeout riêng của nó, đừng quá tin tưởng vào mỗi cờ Hủy (Interrupt).

## 6. Lời Bàn Về Luồng Ảo (Virtual Threads)

Luồng ảo cực ngon để xử lý mấy vụ kẹt I/O, code nhìn rất gọn. NHƯNG, số luồng có thể ảo, còn Database và băng thông mạng thì không ảo đâu nha em. Vẫn phải có chốt chặn (Semaphore) đàng hoàng. Đừng nghĩ luồng ảo là thuốc tiên trị được lỗi kiến trúc hệ thống.

## 7. Đánh Đổi Giữa Các Giải Pháp (Trade-offs)

| Giải Pháp | Khả Năng Xử Lý Chơn Chu | Rủi Ro Quá Tải | Độ Phức Tạp |
| --- | --- | --- | --- |
| Gọi Trực Tiếp (Direct Call) | Hết kẹt cứng 100% | Bị giới hạn bởi lượng luồng tối đa của Pool | Dễ òm |
| Người Điều Phối (Leaf Tasks) | Không sợ Cha ôm luồng | Cần canh Timeout và Hủy cẩn thận | Vừa phải |
| Tách Pool (Separate Pools) | Không kẹt vòng lặp nữa | Đau đầu tính toán sức chứa cho cả 2 Pool | Khá Cao |
| Luồng Ảo + Cấp Phép | Chấp hết các loại ngủ đông chờ | Phải có bộ cấp phép (Permits) cản ở ngoài | Vừa phải |
| Buff Pool/Queue to lên | Lỗi vẫn hoàn lỗi | Chỉ kéo dài sự sống thêm vài giây | Dễ, nhưng dở |

## 8. Khung Giám Sát Khi Đưa Lên Production (Production Checklist)

- [ ] Phải dùng hàng đợi có giới hạn.
- [ ] Từ chối dứt khoát khi quá tải.
- [ ] Set Deadline cho toàn bộ quy trình.
- [ ] Set Timeout cho từng cuộc gọi xuống tầng dưới.
- [ ] Lỗi là phải truyền tín hiệu Hủy xuống tận cùng.
- [ ] Cấm lặp vòng chung Pool.
- [ ] Gắn đồ thị đo lường (Metrics) đầy đủ.
- [ ] Dập Load test thẳng vào xem nó chết ở đâu trước khi release.
