# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Cội Nguồn Vấn Đề

## 1. Trạng Thái Khởi Điểm (Initial state)

Trước khi xảy ra lỗi, hệ thống của mình có tình trạng như sau:

```text
resources = {}
resourceKey = "tenant-a"
Bộ đếm kích hoạt factory (factory open count) = 0

Luồng T1 triệu gọi register("tenant-a")
Luồng T2 triệu gọi register("tenant-a")
```

Mình có vài quy tắc bất di bất dịch (Business Invariant) cần phải giữ:

```text
Số lượng tài nguyên đang hoạt động cho "tenant-a" <= 1
Mọi Caller phải thu thập chính xác cùng một phiên bản tài nguyên đang được Registry quản trị
Hệ số kích hoạt Factory cho một phiên đăng ký = 1
```

## 2. Kịch Bản Lỗi: Thứ Tự Thực Thi Đan Xen (Interleaved Execution)

Đây là lúc mọi chuyện bắt đầu rối tung lên do 2 luồng chạy đan xen:

```text
Luồng T1                                       Luồng T2
---------------------------------------------  ----------------------------------
resources.get("tenant-a") -> null
                                                resources.get("tenant-a") -> null
factory.open("tenant-a") -> Kích hoạt Tài nguyên A
                                                factory.open("tenant-a") -> Kích hoạt Tài nguyên B
resources.put("tenant-a", Tài nguyên A)
Hoàn trả Tài nguyên A
                                                resources.put("tenant-a", Tài nguyên B) (Ghi đè A)
                                                Hoàn trả Tài nguyên B
```

Lúc này, cái Map của bạn sẽ trông như thế này:

```text
resources = { "tenant-a" -> Tài nguyên B }
```

Vấn đề ở đây là cả 2 tài nguyên A và B đều đã được sinh ra. Tài nguyên A được trả về cho luồng T1 xài, nhưng nó lại bị "đá" ra khỏi Registry (do bị B đè lên). Hậu quả là lúc bạn dọn dẹp hệ thống, bạn chỉ tìm thấy B trong Map và tắt nó đi, còn A thì nằm bơ vơ, chả ai quản, gây ra lỗi rò rỉ tài nguyên rành rành.

## 3. Đối Chiếu Kết Quả (Expected vs Actual)

```text
Mục Tiêu Đề Ra (Expected):
  Tần suất gọi factory = 1
  Tài nguyên hoạt động = 1
  Phản hồi cho T1      = Tài nguyên A
  Phản hồi cho T2      = Tài nguyên A
  Dữ liệu lưu Registry = Tài nguyên A

Thực Tế Vận Hành (Actual):
  Tần suất gọi factory = 2 (Gây áp lực hệ thống)
  Tài nguyên hoạt động = 2 (Rò rỉ)
  Phản hồi cho T1      = Tài nguyên A
  Phản hồi cho T2      = Tài nguyên B
  Dữ liệu lưu Registry = Tài nguyên B
  Tài nguyên mồ côi    = Tài nguyên A (Không thể dọn dẹp)
```

> **Nguyên tắc kỹ thuật:** 2 đối tượng thực sự đã được cấp phát trên RAM do bạn đưa ra quyết định dựa trên cái trạng thái "chưa có gì" ban đầu.

## 4. Phân Tích Lớp Trách Nhiệm Gây Lỗi (Root Cause Mapping)

### Hệ Sinh Thái Vòng Đời Spring Bean
Class `ManagedResourceRegistry` được khai báo là Singleton trong Spring. Điều đó nghĩa là mọi request gọi tới nó đều xài chung 1 object Map duy nhất. Nhưng Spring không tự động khóa (sync) các hàm này lại cho bạn đâu.

### Đặc Điểm Của ConcurrentHashMap
Từng hàm như `get` hay `put` của nó thì thread-safe, chả ai cãi. Nhưng logic của bạn thì lại là một chuỗi liên hoàn:

```text
Kiểm tra biến Key -> Nếu vắng mặt -> Cấp phát giá trị -> Công bố
```

Đây là ví dụ kinh điển của lỗi **Kiểm tra rồi hành động (Check-then-act)**. Bạn không có một điểm chốt chặn chung nào cả. Luồng T1 check thấy vắng, T2 cũng check thấy vắng, và cả 2 đều đinh ninh mình được phép tạo mới.

### Giới Hạn Của Bộ Nhớ Java (Java Memory Model)
`ConcurrentHashMap` chỉ giúp bạn ở phần là khi 1 object đã vào map, luồng khác đọc ra sẽ thấy nó ở trạng thái hoàn chỉnh (Safe publication). Lỗi ở đây không phải do đọc sai dữ liệu, mà là bạn đã để cho **cả 2 luồng cùng tạo ra 2 object riêng biệt**!

- **Safe Publication**: Giúp luồng khác thấy đối tượng đã khởi tạo xong.
- **Tính Nguyên Tử (Atomicity)**: Đảm bảo chỉ 1 thằng duy nhất được quyền tạo mới. Bạn thiếu cái này.

### Lằn Ranh Giới Hạn Giao Dịch Database
Nhớ kỹ, `@Transactional` sinh ra để xử lý db, không phải để lock Java Map trong bộ nhớ. Nó cũng chẳng biết cách huỷ Thread hay Socket nếu luồng chạy bị lỗi đâu.

## 5. Điểm Hiệu Lực Duy Nhất (Linearization Point)

Để fix dứt điểm, bạn cần một **Điểm Hiệu Lực Duy Nhất (Linearization point)**. Hiểu nôm na là cái khoảnh khắc mà hệ thống chốt hạ "Ai là người thắng cuộc" và ép những thằng còn lại dùng chung kết quả.

- Khai thác `computeIfAbsent`: Hàm này của map nó tự làm nguyên tử luôn (atomic), mình khỏi lo.
- Khai thác `putIfAbsent`: Chốt hạ ở lúc `put` vào map. Luồng nào `put` thành công thì giữ, luồng thua thì phải lo dọn rác do mình vừa sinh ra.
- Khai thác `FutureTask` Placeholder: Chốt hạ ở lúc cắm cái mỏ neo vào Map. Anh nào cắm được thì anh đó mới đi gọi Factory chạy thật.

## 6. Xử Lý Phân Loại Lỗi, Timeout Và Dọn Dẹp Tài Nguyên

- Lỡ hàm Factory quăng lỗi trước khi return thì Map vẫn trống. Factory phải tự biết dọn rác của mình.
- Dùng `computeIfAbsent` mà Factory tung lỗi thì Map cũng không lưu, luồng sau có thể thử lại.
- Dùng `FutureTask` thì nếu mỏ neo bị lỗi (Exception), nhớ phải xóa mỏ neo đi để mấy luồng sau còn Retry được.
- Nếu app sập (Crash), mất sạch bộ nhớ thì Registry cục bộ này cũng bay màu luôn.
- Nếu quy trình mở/đóng tài nguyên phức tạp, bạn nên tách nó ra 1 module quản lý vòng đời riêng để tránh đóng nhầm tài nguyên của luồng khác.

## 7. Ứng Dụng Trên Kiến Trúc Phân Tán (Multi-instance Fallacy)

```text
Nút Mạng A: registry A -> Kích hoạt Tài nguyên A
Nút Mạng B: registry B -> Kích hoạt Tài nguyên B
```

Bạn có xài `computeIfAbsent`, `synchronized`, hay `FutureTask` thì cũng chỉ là giải quyết bài toán trên 1 server (JVM). Qua nhiều server thì mạnh thằng nào thằng nấy tạo, không đồng bộ được đâu.

> **Nguyên tắc kỹ thuật:** Fix lỗi cục bộ ngon nghẻ không có nghĩa là kiến trúc cụm (cluster) của bạn đang đúng.

## 8. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Tạo tè le tài nguyên vô ích, rò rỉ bộ nhớ.
- Các request vào cùng 1 key lại nhận về 2 object khác nhau.
- Map thì lưu 1 kiểu, ngoài đời lại chạy 1 kiểu.
- Thread, Socket bị bung bét, server ì ạch.
- Đoạn code `map.size()` vẫn báo chuẩn nên chả ai phát hiện ra.

### Hệ Quả Nghiệp Vụ
- Khách hàng kết nối bị lỗi hoặc bị đúp.
- Bắn event dư thừa đi lung tung.
- Tiền tài nguyên mạng, băng thông tăng chóng mặt.
- Server mau cạn kiệt resource và dễ sập hơn.

## 9. Giới Hạn Áp Dụng (Out of Scope)

Bài này mình chỉ giải quyết bài toán Registry trên 1 server thôi nhé. Đừng bê nó đi giải:
- Dữ liệu sống sót qua màn khởi động lại server.
- Khoá dữ liệu phân tán nhiều server.
- Cơ chế thuê (lease) hoặc fencing trên mạng.
- Bài toán Check-then-insert của Database.
- Xử lý chi tiết các bước Đăng ký - Ngắt - Xóa phức tạp.
