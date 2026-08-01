# Phân Tích Chuyên Sâu: Nạn Đói Tài Nguyên Và Ràng Buộc Tiến Trình

## 1. Dòng Thời Gian Xung Đột (Interleaving Timeline)

| Trình tự | Luồng Công Nhân 1 | Luồng Công Nhân 2 | Trạng Thái Hàng Đợi |
| --- | --- | --- | --- |
| 1 | Khởi chạy Tác vụ Cha P1 | Khởi chạy Tác vụ Cha P2 | Hoàn toàn rỗng |
| 2 | P1 đẩy Tác vụ Con C1 vào Pool | P2 đẩy Tác vụ Con C2 vào Pool | Giam giữ C1, C2 |
| 3 | P1 đóng băng chờ C1 | P2 đóng băng chờ C2 | C1, C2 kẹt cứng, không thể kích hoạt |

Hoàn toàn vắng bóng cấu trúc Giám sát vòng lặp chờ (Monitor wait-for cycle); Nguồn cơn bế tắc nằm ở đặc tính Sức Chứa (Capacity): Tác vụ Cha giam cầm Luồng để ngóng Tác vụ Con, Tác vụ Con lại mòn mỏi ngóng Luồng hiện đang bị chính Tác vụ Cha giam cầm.

## 2. Đối Chiếu Kết Quả (Expected vs Actual)

**Kỳ Vọng (Expected):** Các yêu cầu đã được tiếp nhận phải đi đến điểm hoàn tất nội trong Hạn mức thời gian (Deadline), và biến động Hàng đợi phản chiếu chính xác khối lượng công việc CÓ KHẢ NĂNG được phục vụ. 

**Thực Tế (Actual):** Thông số Luồng Vận Hành (Active count) luôn chạm trần Pool Size, chỉ số Hoàn Thành (Completed count) đóng băng vĩnh viễn, Hàng đợi chật cứng Tác vụ con, và biểu đồ tiêu thụ CPU có thể tụt xuống mức cực thấp.

## 3. Bản Chất Nguồn Gốc Lỗi (Root Cause)

Quy mô Executor (Executor sizing) bị định đoạt hoàn toàn cô lập, tước bỏ mối liên hệ với Đồ thị Phụ thuộc (Dependency graph). Hệ số Đóng băng (Blocking coefficient) hay bất kỳ công thức toán học tính toán Pool Size nào cũng trở nên vô nghĩa trước một Hệ hình thái (Topology) nơi mọi Luồng công nhân đồng loạt tự giam mình chờ đợi các tác vụ nằm chung Pool. Hành vi Chờ lồng nhau (Nested blocking) đã chém đứt Quy tắc Tiến Trình (Progress invariant).

Lệnh `Future.get()` kiến tạo mối liên hệ Hệ quả (Happens-before) ngay khi Tác vụ con hoàn thành, ngặt nỗi Tác vụ con không bao giờ có cơ hội hoàn thành. Tính năng Quá hạn (Timeout) chỉ có năng lực phá vỡ Khối chờ sau khi Hạn mức kết thúc; Tuy nhiên, Tín hiệu Hủy (Cancellation) bắt buộc phải truyền dẫn xuyên suốt (Propagate) và mọi Tác vụ phải tôn trọng Cờ Ngắt (Interrupt).

## 4. Kiểm Soát Quá Tải Và Giảm Áp (Overload & Backpressure)

Cấu trúc Hàng đợi giới hạn (Bounded queue) bắt buộc phải đồng bộ Chính sách Từ chối (Rejection policy) với Giao diện API: Trả mã ngắt nhanh (Fail-fast) 429/503, Hàng đợi Caller giới hạn hoặc Trạm kiểm soát Đầu vào (Admission semaphore). Tuyệt đối cấm hành vi Thử lại (Retry) dội ngược vào chính cá thể (Instance) đang nghẽn mạng.
Hạn mức thời gian (Deadline) là tổng hòa của: Độ trễ Hàng đợi (Queue delay) + Thời gian Thi hành (Execution) + Độ trễ Phụ thuộc (Dependency latency).

## 5. Xử Lý Phân Loại Lỗi, Tắt Hệ Thống Và Môi Trường Phân Tán

- Khi Tác vụ Cha vỡ Hạn mức thời gian (Timeout), lập tức phát tín hiệu Hủy Tác vụ Con và dọn sạch Ngữ cảnh bộ nhớ (Cleanup context).
- Lệnh `shutdown()` đóng cửa từ chối Yêu cầu mới; Lệnh `shutdownNow()` phóng tín hiệu Ngắt (Interrupt) nhưng không có thẩm quyền buộc các luồng Nhập/Xuất viễn trình (Remote I/O) dừng tay.
- Trong mạng phân tán, mỗi Nút (Node) sở hữu Pool riêng; Bộ Cân Bằng Tải (Load balancer) hoàn toàn có nguy cơ tiếp tục nã đạn (gửi yêu cầu) vào các Nút đang hấp hối (Saturated).
- Tốc độ Phóng to quy mô (Autoscaling) luôn chậm chạp hơn một cơn bão Thử lại (Retry storm); Bắt buộc thiết lập Trạm kiểm soát Đầu vào / Phân tán Tải (Admission/Load-shedding) cục bộ trên từng Nút.
- Vấn nạn Cạn kiệt kết nối JDBC (JDBC connection starvation) là một hình thái bế tắc tài nguyên độc lập, được mổ xẻ tại module `SPR-007`.

## 6. Giám Sát Môi Trường Khai Thác (Production Blueprint)

Đo lường sát sao: Số luồng Đang chạy/Tối đa, Kích thước/Sức chứa Hàng đợi, Tuổi đời tác vụ già nhất trong Hàng đợi, Tỷ lệ Tác vụ Đệ trình/Bắt đầu/Hoàn tất/Từ chối, Thời gian thi hành, Độ trễ xếp hàng, Tỷ lệ Hủy bỏ và Tuân thủ Hạn mức.
Trích xuất Bảng phân luồng (Thread dump) sẽ phơi bày sự thật: Các luồng công nhân đang kẹt cứng ở lệnh `Future.get` trong khi các Tác vụ Con vẫn nằm bất động chưa từng được khởi chạy.

> **Nguyên tắc kỹ thuật:** Chỉ số Hiệu suất sử dụng (Utilization) buộc phải đánh giá song hành cùng Chỉ số Tiến Trình (Progress); "100% công nhân bận rộn" hoàn toàn có thể là bức bình phong cho sự thật "100% công nhân đang đóng băng" chứ không hề tạo ra bất kỳ giá trị xử lý nào.
