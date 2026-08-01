# Môi Trường Thực Nghiệm: Xác Minh Tương Tranh Dòng Đời Thực

## 1. Thí Nghiệm Đan Xen Lỗi Luồng (Deterministic Broken Race)

Mình sẽ dùng 2 cái Latch để "neo" các callback lại, rồi thả cho 2 luồng cùng lúc lao vào ghi dữ liệu ở chung một vị trí trong mảng. Lúc này, kiểm tra (Assert) xem kích thước mảng có phải là 1 (bị đè mất data) thay vì là 2 hay không.
**Lưu ý:** Nếu bạn test trên `ArrayList` thật thì phải dùng các bài test chịu tải (Stress suite) đàng hoàng. Chứ đừng có thấy chạy qua (pass) 1 lần ăn hên rồi vội tin là nó ngon lành nhé!

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

Hãy thử tạo một kịch bản: Cho 1 task con tạch (Fail) và 1 task khác bị treo (Block) luôn ở cái Latch. Sau đó mình kiểm tra xem tiến trình chính có ném ra lỗi không, lệnh Hủy (Cancel) có được gọi xuống cho các task anh em không, và tuyệt đối KHÔNG có tí data dở dang nào lọt ra ngoài.
Nhớ kỹ nè: Bất kỳ cái Latch hay Future nào cũng phải gắn Timeout đàng hoàng. Cấm tuyệt đối cái trò dùng `Thread.sleep` để đợi nhe.

## 4. Giám Sát Môi Trường Khai Thác (Production Verification)

Phải gắn Metric kỹ càng vô để đo đếm: Lượng Input, Số Thành công (Success), Số Thất bại (Failure), Số Hủy lệnh (Cancel), Số luồng xong trễ (Late-completion), Deadline chung, tình trạng Hàng đợi (Queue) và Số lần đẩy ra (Rejection).
Kiểm tra xem đầu ra có đủ số lượng không, ID có độc nhất không, có đúng thứ tự không, và Output đã bị khóa cứng (Immutable) chưa.

## 5. Khung Tiêu Chuẩn Thực Nghiệm (Quality Checklist)

- [ ] Lấy Latch ép cho mấy cái Callback chạy đè lên nhau thử xem sao.
- [ ] Output trả về phải đúng thứ tự Input, kể cả thằng nào chạy xong trước xong sau cũng kệ.
- [ ] Hễ fail là không có chuyện trả về mảng dữ liệu sứt sẹo đâu nha.
- [ ] Mấy vụ Timeout với Cancel phải pass qua được đợt test khắc nghiệt này.
- [ ] Phải luôn theo dõi tài nguyên của Executor và các lệnh bị Rejection.
