# Phân Tích Chuyên Sâu: Nạn Đói Tài Nguyên Và Ràng Buộc Tiến Trình

## 1. Dòng Thời Gian Xung Đột (Interleaving Timeline)

Hãy xem quá trình bế tắc xảy ra thế nào:

| Bước | Anh Công Nhân 1 | Anh Công Nhân 2 | Hàng Đợi |
| --- | --- | --- | --- |
| 1 | Nhận Tác vụ Cha P1 | Nhận Tác vụ Cha P2 | Trống không |
| 2 | P1 đẩy Tác vụ Con C1 xuống | P2 đẩy Tác vụ Con C2 xuống | C1, C2 vào nằm chờ |
| 3 | P1 ngủ đông chờ C1 | P2 ngủ đông chờ C2 | C1, C2 kẹt cứng ngắc vì không có ai chạy |

Lỗi ở đây là Cha chiếm luồng để ngóng Con, còn Con thì mòn mỏi chờ luồng (đang bị Cha chiếm). Vòng luẩn quẩn này khiến hệ thống đứng im.

## 2. Đối Chiếu Kết Quả (Expected vs Actual)

**Anh em kỳ vọng (Expected):** Các request nhận vào sẽ chạy mượt mà, đúng deadline. Hàng đợi chỉ chứa lượng công việc mà hệ thống có khả năng nhai được.

**Thực tế đắng lòng (Actual):** Luồng nào cũng báo đang bận (Active max), số việc xong (Completed) bằng 0, hàng đợi đầy nhóc, nhưng CPU thì nhàn rỗi vì mấy luồng toàn đang "ngủ" chờ nhau.

## 3. Bản Chất Nguồn Gốc Lỗi (Root Cause)

Chúng ta thường hay set size cho Pool một cách hời hợt mà quên mất mối quan hệ cha-con giữa các tác vụ. Mọi công thức tính toán Pool size đều vứt đi nếu tất cả luồng đều tự trói mình để chờ nhau. Đây gọi là Chờ lồng nhau (Nested blocking) - nó phá vỡ quy luật vận hành của hệ thống.

Lệnh `Future.get()` bắt anh Cha đợi anh Con làm xong. Khổ nỗi anh Con có được chạy đâu mà xong. Nếu dùng Timeout thì sau một lúc anh Cha sẽ thoát, nhưng nó không giải quyết được gốc rễ vấn đề, và cờ hủy (Interrupt) nhiều khi không được truyền chuẩn xuống dưới.

## 4. Kiểm Soát Quá Tải Và Giảm Áp (Overload & Backpressure)

Khi dùng hàng đợi giới hạn (Bounded queue), phải đi kèm với chính sách từ chối (Rejection) dứt khoát: Trả luôn lỗi 429/503 để báo khách hàng biết là đang bận. Đừng để hệ thống liên tục retry vào một chỗ đang tắc.
Nhớ rằng: Timeout = Thời gian xếp hàng + Thời gian chạy + Thời gian gọi API ngoài.

## 5. Xử Lý Phân Loại Lỗi, Tắt Hệ Thống Và Môi Trường Phân Tán

- Khi Cha bị Timeout, phải lập tức hủy Con để dọn rác bộ nhớ.
- `shutdown()` ngăn không nhận thêm việc; `shutdownNow()` gửi cờ ngắt để dừng luồng, nhưng chưa chắc dừng được mấy cuộc gọi mạng (Remote I/O) đâu nhé.
- Trong hệ thống lớn, Load Balancer không biết ông Node nào đang kẹt nên cứ đẩy thêm request vào, làm mấy ông Node càng nhanh chết.
- Đợi Auto-scale thì không kịp, cơn bão Retry sẽ quật ngã hệ thống trước. Cần có chốt chặn quá tải ngay tại mỗi server.
- Còn nếu bị hết connection database thì đó là một câu chuyện khác, xem ở phần `SPR-007`.

## 6. Giám Sát Môi Trường Khai Thác (Production Blueprint)

Cần theo dõi sát: Số luồng bận, độ đầy hàng đợi, thời gian kẹt trong hàng đợi, tỷ lệ lỗi và thời gian thực thi.
Nếu kéo Thread dump ra xem, em sẽ thấy sự thật phũ phàng: Cả dàn công nhân đang đờ đẫn tại `Future.get()`, trong khi đám việc Con thì chưa từng được nhúc nhích.

> **Mẹo nhỏ:** Đừng chỉ nhìn vào "100% tài nguyên đang dùng". Đầy lúc 100% công nhân đang bận... bận đứng im chờ nhau, chứ chả làm được tích sự gì!
