# Phản Mẫu Thiết Kế (Anti-Patterns): Cấu Trúc Khuyết Tật

## 1. Phân Tích Mã Nguồn Suy Thoái (Broken Implementation)

```java
List<Enrichment> results = new ArrayList<>();
List<CompletableFuture<Void>> tasks = inputs.stream()
    .map(input -> CompletableFuture
        .supplyAsync(() -> client.load(input), executor)
        .thenAccept(results::add)) // LỖI: Điểm giao tranh đa luồng trên một List thiếu bảo vệ
    .toList();
// LỖI: Phá vỡ chuỗi kiểm soát; Không Hủy được nếu một child lỗi
CompletableFuture.allOf(tasks.toArray(CompletableFuture[]::new)).join();
return results; // Trả về cấu trúc có rủi ro bị Mutate ngầm định
```

Hệ thống Callbacks hoàn toàn có năng lực đua tài cùng lúc (chạy đồng thời); tuy nhiên, cấu trúc nguyên thủy `ArrayList.add` thiếu vắng chứng chỉ an toàn luồng (Thread-safe).
Rào cản `allOf` không mang chức năng Tuần tự hóa (Serialize) các Callbacks và càng không tự động trang bị lớp giáp Nguyên tử (Atomic) cho khối Tích lũy (Accumulator).
Kịch bản đứt gãy: Nếu một tác vụ con (Child) sụp đổ, lệnh `join` ném ra ngoại lệ `CompletionException`, nhưng các tác vụ con khác hoàn toàn có thể tiếp tục nhởn nhơ vận hành và cấy rác (Mutate) vào cấu trúc `results`.
Việc hoàn trả hoặc lưu trữ bộ đệm (Cached) một tham chiếu Khả Biến (Mutable) vô hình trung mở toang cánh cửa cho phép Trạng thái dữ liệu bị lén lút sửa đổi dẫu cho khối phản hồi (Response) đã được trả về cho hệ thống.

## 2. Tiền Đề Kích Hoạt Lỗ Hổng (Conditions for Replication)

1. Tồn tại bối cảnh tối thiểu hai Callback xảy ra tình trạng Đan chéo (Overlap).
2. Khối Tích lũy chia sẻ (Shared Accumulator) bị giam hãm vào trong định dạng Closure (Captured).
3. Hiện tượng Lỗi/Hết thời hạn không phát động luồng Hủy bỏ (Cancel) toàn bộ các tác vụ anh em (Sibling).
4. Hệ thống gọi (Caller) tiến hành quan sát và trích xuất cấu trúc List trước khi vòng đời hệ thống (Lifecycle) kịp đóng sập.

## 3. Các Phương Án Sửa Lỗi Tạm Bợ Cần Tránh (Insufficient Fixes)

- **Cấu trúc `Collections.synchronizedList`**: Chỉ bít lỗ hổng tại lệnh `add`, nhưng hoàn toàn bó tay trong việc định hình Quy chuẩn Trình tự (Order) hay Giải quyết Ngoại lệ (Failure).
- **Cấu trúc `CopyOnWriteArrayList`**: Đốt cháy tài nguyên vô độ cho hệ thống tập trung Ghi (Write-heavy).
- **Cấu trúc `allOf`**: Thiếu vắng cơ chế Ngắt Nhanh (Fail-fast) hay Hủy Bỏ anh em (Cancel sibling).
- **Sử dụng `exceptionally` trả về `null`**: Hành vi cố tình che đậy Ngoại lệ, dẫn đến hệ quả khó chẩn đoán.
- **Khai thác Common Pool cho Blocking I/O**: Mở ra con đường dẫn thẳng tới nạn Đói Tài Nguyên (Starvation) — Phân tích chi tiết tại JVM-008.

> **Nguyên tắc kỹ thuật:** Bọc một tập hợp dữ liệu bằng lớp màng An Toàn Luồng (Thread-safe collection) hoàn toàn KHÔNG liên quan tới việc thiết lập quy chuẩn thiết kế Lô: Giao dịch Tuyệt Đối (All-or-Nothing) hay Giao dịch Phân Rã (Per-item outcome).
