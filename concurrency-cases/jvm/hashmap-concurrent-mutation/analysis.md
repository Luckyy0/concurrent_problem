# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Ranh Giới Điểm Hiệu Lực

## 1. Dòng Thời Gian Đan Xen (Interleaving Timeline)

Giả sử hiện tại hệ thống đang chạy với "Thế hệ 41":
```text
merchant-a → Provider-X, Thế hệ=41
merchant-b → Provider-Y, Thế hệ=41
```

Bây giờ Cấu hình báo có bản cập nhật mới, gọi là "Thế hệ 42":
```text
merchant-a → Provider-Z, Thế hệ=42
merchant-b → Provider-Y, Thế hệ=42
```

Lúc này, Luồng ghi (Writer) bắt đầu dọn dẹp bằng `routes.clear()` rồi đổ dữ liệu vào bằng `routes.putAll(loaded)`. Vấn đề lớn nhất ở đây là: Lệnh `putAll` KHÔNG làm xong mọi thứ trong chớp mắt (nguyên khối). Thực tế, nó chia nhỏ thành từng bước lắt nhắt để cập nhật Map.

### Cửa Sổ Rủi Ro Đứt Gãy

Hãy xem bảng thời gian này để thấy rủi ro:

| Trình tự | Luồng Làm mới (T1) | Luồng Yêu cầu (T2) | Luồng Giám sát (T3) | Trạng Thái Hệ Thống Thực Tế |
| --- | --- | --- | --- | --- |
| 1 | Nạp xong Thế hệ 42 | | | Vẫn đang phục vụ Thế hệ 41 |
| 2 | Kích hoạt `routes.clear()` | | | Bản đồ bộ nhớ bị xóa sạch bách (Rỗng) |
| 3 | (Tạm nghỉ) | Gọi hàm `get("merchant-b")` | | Nhận về `null`, phải đẩy sang đối tác dự phòng (Fallback) |
| 4 | Nạp Khóa `merchant-a` | | | Map bây giờ chỉ có 1 phần của Thế hệ 42 |
| 5 | (Tạm nghỉ) | | Đang duyệt `entrySet()` | Báo cáo giám sát nói hệ thống chỉ có mỗi `merchant-a` |
| 6 | Nạp Khóa `merchant-b` | | | Hoàn thành Thế hệ 42 trọn vẹn |

Bạn thấy đấy, không cần tới nhiều luồng ghi, chỉ cần 1 Luồng ghi và 1 Luồng đọc là đủ để hệ thống "ăn hành". Khi luồng lặp dữ liệu đâm sầm vào lúc Map đang đổi, `HashMap` sẽ ném cái lỗi `ConcurrentModificationException`. Lỗi này không phải là "tấm khiên" bảo vệ đâu, nó chỉ là một cố gắng nhỏ nhoi để "vớt vát" mà thôi.

> **Nguyên tắc kỹ thuật:** Kết quả có vẻ đúng, nhưng sự thật là Luồng Yêu Cầu đã tự xử lý giao dịch dựa trên một dữ liệu nát bét lúc đó rồi.

## 2. Đối Chiếu Kết Quả (Expected vs Actual)

| Tiêu Chí Đánh Giá | Kỳ Vọng (Mình muốn gì) | Hệ Quả Thực Tế (Code vỡ nó làm gì) |
| --- | --- | --- |
| Tính Toàn Vẹn | Luồng đọc nhận đủ 100% Thế hệ 41 hoặc 42 | Luồng đọc nhận cái Map Rỗng, hoặc dữ liệu nửa mùa |
| Tính Nhất Quán | Dữ liệu đồng nhất cùng 1 Thế hệ | Truy xuất bị gãy, dính cả cũ lẫn mới |
| Khả Kiến (Visibility) | Ghi xong là các luồng khác thấy ngay | Gán biến thường nên luồng khác có thể vẫn thấy bản cũ |
| Vòng Lặp (Iteration) | Lấy được 1 bản chụp dữ liệu tĩnh để giám sát | Vòng lặp mất Khóa, dữ liệu lộn xộn, hoặc Văng Lỗi |
| Sai Số (Failure) | Lỗi cập nhật thì giữ nguyên dữ liệu Tốt gần nhất | Chỉnh sửa trực tiếp làm vỡ dữ liệu, hệ thống không phục hồi được |

## 3. Bản Chất Cội Nguồn Lỗi Theo Các Lớp Phân Tách

### Khuyết Tật Của Cấu Trúc HashMap
`HashMap` thông thường không hề có cam kết bảo vệ dữ liệu khi nhiều luồng cùng đục khoét (Concurrent structural modification). Không có "khóa", việc vừa đọc vừa ghi trở thành một cuộc chiến dữ liệu (Data race). Đừng bao giờ ảo tưởng rằng "bản JDK này chạy thử thấy ổn nên chắc không sao".

### Hành Vi Phức Hợp (Compound Action)
Với bạn, làm mới cấu hình là MỘT việc. Nhưng với máy ảo, làm mới là NHIỀU lệnh nhỏ (`clear` rồi `put`, rồi lại `put`). Bạn không có một **Điểm Hiệu Lực Duy Nhất (Linearization point)** để chốt sổ, nên luồng đọc cứ thế lách qua kẽ hở để đọc sai.

### Góc Khuất Mô Hình Bộ Nhớ (Java Memory Model - JMM)
Dù bạn thông minh xài trò "Tạo bản sao mới rồi tráo" (Copy-and-swap), nhưng nếu biến bạn lưu bản sao chỉ là biến bình thường, JMM chả hứa hẹn gì về việc các luồng khác sẽ thấy cái biến đó ngay (Luật `happens-before`). 
Bạn cần `volatile`, Monitor Lock, hoặc biến Atomic để tạo cầu nối cho các luồng thấy nhau.

### Ảo Giác Container Spring
Bean Singleton chỉ có nghĩa là nó sống "duy nhất 1 bản" từ đầu đến cuối, chứ không hứa hẹn là nó chịu được đa luồng! Cái `@Scheduled` của bạn và cái Controller hứng yêu cầu là chạy ở các luồng khác nhau, vô tư đâm chém vào một biến đổi được (Mutable field).

### Giới Hạn Cơ Chế Giao Dịch
Đừng nghĩ xài `@Transactional` là thoát. Cái đó chỉ cho Database thôi. Nếu Luồng cập nhật xóa sạch Map rồi văng lỗi giữa chừng, Transaction DB sẽ Rollback lại, nhưng cái Map trên RAM thì đã bị xóa sạch sẽ, vĩnh viễn không cứu lại được.

## 4. Ranh Giới Điểm Hiệu Lực (Linearization Point) Chuẩn Mực

Để an toàn, hãy dùng chiến thuật **Bản chụp Bất biến (Immutable snapshot)**. Luồng ghi sẽ làm đúng 3 bước:
1. Tải và kiểm tra dữ liệu ở một vùng kín (Biến cục bộ).
2. "Đóng băng" nó lại thành một khối không thể sửa bằng `Map.copyOf(...)`.
3. Xuất bản nó lên bằng `AtomicReference.set` hoặc biến `volatile`.

Bước 3 chính là Điểm Hiệu Lực Duy Nhất! Luồng đọc chỉ cần lấy tham chiếu 1 lần là lấy nguyên cục Thế hệ 100% chuẩn, không sợ ai thay đổi. Nếu có nhiều Luồng ghi cùng lúc, phải dùng cơ chế **So sánh - và - Đặt (`compare-and-set`)** để luồng đến sau biết mà lùi bước nếu bản chụp của nó đã lỗi thời.

## 5. Xử Lý Phân Loại Lỗi Hệ Thống Và Vòng Đời Cấu Trúc

- Tuyệt đối cấm dùng `clear()` lúc dữ liệu đang phục vụ khách, chỉ để dọn chỗ cho cái mới chưa kịp kiểm tra.
- Đã tráo đổi nguyên tử (Atomic Swap) thì không bao giờ sợ bị vỡ vụn lúc hỏng hóc (Rollback phân mảnh).
- Ứng dụng lỡ sập sau khi tráo đổi, dữ liệu tại máy đó đã là Mới, nhưng các máy chủ (node) khác không có nghĩa là cũng tự động theo kịp.

## 6. Giới Hạn Phạm Vi Và Môi Trường Phân Tán

Nhớ nhé, `AtomicReference` chỉ cai quản bộ nhớ bên trong MỘT máy ảo JVM. Không có lệnh khóa (Lock) nào ép được Máy chủ B phải cập nhật giống Máy chủ A. Nếu muốn cấm một Đối tác ngay tức khắc trên toàn mạng lưới, bạn phải giải quyết ở bài toán Hệ thống Phân tán (như Event, Versioning ở Config hay DB dùng chung).
