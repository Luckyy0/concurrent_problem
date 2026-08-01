# Môi Trường Thực Nghiệm: Xác Minh Tương Tranh Dòng Đời Thực

## 1. Thí Nghiệm Đan Xen Lỗi Luồng (Deterministic Broken Race)

Sử dụng kết cấu mỏ neo kép (hai Latch) để giam hãm chuỗi Callbacks, tiếp đó cấp quyền cho cả hai luồng đồng loạt nã pháo (gọi) vào móc cấu trúc (Hooked accumulator) tại chính xác cùng một vị trí Chỉ Mục Trọng Thâm (Logical index); tiến hành xác thực (Assert) trạng thái kích thước Size là 1 thay vì 2. 
Cảnh báo: Đối với cấu trúc `ArrayList` vật lý (Thật), bắt buộc vận hành qua luồng Stress Suite độc lập — Không được phép lạm dụng một lần vượt ải (Pass) ngẫu nhiên làm chứng chỉ bảo đảm an toàn.

## 2. Kiểm Định Cấu Trúc Khắc Phục (Fixed Regression)

```java
@Test
void composesValuesInInputOrder() {
    CompletableFuture<String> first = new CompletableFuture<>();
    CompletableFuture<String> second = new CompletableFuture<>();
    List<CompletableFuture<String>> futures = List.of(first, second);
    
    // Cấu trúc hợp nhất Giá trị (Value Composition) thay vì Mutation
    CompletableFuture<List<String>> aggregate = CompletableFuture
        .allOf(first, second)
        .thenApply(x -> futures.stream().map(CompletableFuture::join).toList());
        
    // Mô phỏng đảo ngược tiến trình hoàn tất
    second.complete("B");
    first.complete("A");
    
    // Khẳng định: Kết quả BẮT BUỘC tôn trọng trình tự Input Order (A -> B)
    assertEquals(List.of("A", "B"),
        aggregate.orTimeout(5, TimeUnit.SECONDS).join());
}
```

## 3. Kiểm Định Hệ Thống Đứt Gãy / Hủy Bỏ (Failure & Cancellation)

Khởi tạo kịch bản một Tác vụ con (Child) đổ vỡ (Fail), trong khi một Child khác bị bóp nghẹt (Block) trên Latch; Thực hiện xác thực trạng thái khối Aggregate đã rơi vào Ngoại lệ (Exceptional), tín hiệu Hủy (Cancel) đã được phát đi truy quét Sibling, và tuyệt đối KHÔNG có bất kỳ Final List nào được rò rỉ (Publish) ra bên ngoài.
Lưu ý: Mọi tín hiệu Latch/Future bắt buộc phải trang bị rào cản Thời gian (Timeout); Nghiêm cấm hoàn toàn sự xuất hiện của thủ thuật `Thread.sleep`.

## 4. Giám Sát Môi Trường Khai Thác (Production Verification)

Thiết lập hệ đo lường (Metric) chặt chẽ cho các chỉ số: Lệnh Input, Thành công (Success), Thất bại (Failure), Hủy lệnh (Cancel), Trễ trạm (Late-completion), Hạn mức khối Aggregate (Deadline), Hàng đợi Executor (Queue) và Tín hiệu Từ chối (Rejection).
Áp dụng xác thực tính Toàn vẹn quy mô (Cardinality), Định danh độc bản (Unique input ID), Trật tự (Order), và đặc tính Bất Biến (Immutable) của đối tượng Output.

## 5. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [ ] Hiện tượng đan chéo Callback (Overlap) đã được điều phối khắc nghiệt bằng hệ thống Latch.
- [ ] Output chỉnh sửa bắt buộc duy trì Trật tự đầu vào (Input order) bất chấp trình tự hoàn tất luồng bị đảo lộn.
- [ ] Trạng thái Thất bại tuyệt đối không bộc lộ (Publish) các mảnh dữ liệu dư thừa (Partial result).
- [ ] Phản ứng Hủy bỏ (Cancellation) và Hạn mức Timeout đã vượt qua hệ thống Test gắt gao.
- [ ] Tài nguyên Executor/Tín hiệu Rejection liên tục được đưa vào tầm quan sát.
