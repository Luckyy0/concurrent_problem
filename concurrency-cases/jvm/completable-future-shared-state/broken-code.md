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

Ở đây á, các Callbacks có thể chạy đồng thời (chạy đua với nhau), nhưng `ArrayList.add` thì lại không hề an toàn khi chạy đa luồng (thread-safe). 
Bạn xài `allOf` chỉ để đợi chúng nó chạy xong, chứ không phải để bắt chúng chạy tuần tự, và nó cũng chả giúp bảo vệ cái List `results` kia đâu.
Nếu có một task con bị vỡ, thằng `join` sẽ quăng lỗi `CompletionException`. Nhưng mấy task con khác vẫn có thể sống nhăn răng và tiếp tục đút dữ liệu (mutate) rác vào mảng `results`.
Tệ hơn, việc bạn trả về một cái List có thể bị sửa đổi (mutable list) làm cho dữ liệu có thể bị ai đó lén thay đổi, kể cả khi bạn đã trả kết quả về cho hệ thống rồi.

## 2. Tiền Đề Kích Hoạt Lỗ Hổng (Conditions for Replication)

Lỗi này sẽ bung bét khi:
1. Có ít nhất hai Callback chạy chồng chéo (overlap) lên nhau.
2. Bạn để cho cái List dùng chung bị "kẹt" vào trong hàm closure.
3. Khi có lỗi hay timeout, hệ thống không dọn dẹp và hủy (cancel) hết các task anh em đang chạy.
4. Bên gọi hàm (Caller) bốc cái List ra xài trước khi các luồng kịp đóng hoàn toàn.

## 3. Các Phương Án Sửa Lỗi Tạm Bợ Cần Tránh (Insufficient Fixes)

Nhiều người nghĩ thế này là xong nhưng không phải nha:
- **Dùng `Collections.synchronizedList`**: Khóa được hàm `add`, nhưng không lo được cái vụ thứ tự (order) với xử lý lỗi (failure).
- **Dùng `CopyOnWriteArrayList`**: Cứ ghi là nó copy cả mảng, tốn tài nguyên khủng khiếp nếu bạn ghi liên tục (write-heavy).
- **Phụ thuộc vào `allOf`**: Thằng này không có trò ngắt ngay khi có lỗi (fail-fast) hay hủy các luồng khác đâu.
- **Xài `exceptionally` để trả về `null`**: Làm thế này là bạn đang cố giấu cái lỗi đi, sau này debug có mà khóc.
- **Dùng Common Pool cho I/O (chặn luồng)**: Rất dễ dẫn đến tình trạng cạn kiệt tài nguyên (Starvation) — cái này có bài phân tích riêng ở JVM-008 rồi.

> **Nhớ nè:** Bọc data vô một mảng an toàn đa luồng (thread-safe) hổng có nghĩa là bạn giải quyết được bài toán logic của cả lô: Hoặc là làm được hết, hoặc là tạch cả lô.
