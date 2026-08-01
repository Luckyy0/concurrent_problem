# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Cội Nguồn Vấn Đề

## 1. Dòng Thời Gian Đan Xen Luồng (Interleaving Timeline)

Giả định Tác vụ T1 và T2 đồng thời chạm đích (hoàn tất), cả hai luồng cùng truy xuất vào thông số `size` nội tại của cấu trúc mảng và tiến hành hành vi ghi đè (write). Hậu quả: một kết quả vật lý có nguy cơ cao bị xóa sổ hoàn toàn khỏi bộ nhớ.
Kịch bản Tác vụ T3 sụp đổ sẽ khiến khối tổng hợp (Aggregate) rơi vào trạng thái ngoại lệ (Exceptional); tuy nhiên, luồng T2 vẫn thản nhiên hoàn thành công việc sau đó và tiếp tục gây biến dạng cấu trúc dữ liệu lỗi (Mutate partial state).

## 2. Đối Chiếu Kết Quả (Expected vs Actual)

**Kỳ Vọng (Expected):**
- Trả về tập dữ liệu Bất Biến (Immutable).
- Quy mô đầy đủ, giữ chuẩn trình tự (Ordered success).
- Hoặc cung cấp một ngoại lệ (Explicit failure) rõ ràng nếu hệ thống đứt gãy.

**Thực Tế (Actual):**
- Mảng dữ liệu bị khuyết thiếu (Lost update).
- Trật tự sắp xếp bị bẻ cong theo tốc độ hoàn thành luồng (Completion-order).
- Trạng thái ngầm thay đổi dẫu cho Cờ Ngoại Lệ đã được giương lên.
- Tín hiệu Hủy (Cancellation) hoàn toàn tắc nghẽn, không được kế thừa truyền dẫn (Propagate) cho các tác vụ con.

## 3. Bản Chất Nguồn Gốc Lỗi (Root Cause)

Các chặng xử lý (Completion stages) duy trì đặc tính độc lập tuyệt đối về lịch trình phân bổ (Scheduling). Rào cản hệ quả `Happens-before` liên kết từ điểm hoàn tất tới lệnh `join` chỉ mang tính chất bảo chứng quyền Khả kiến (Visibility) của trạng thái được ghi chốt cuối cùng; nó hoàn toàn "Mù" trước các hành vi Đột biến đan xen (Concurrent mutation) đã xảy ra trước đó.
Lệnh `allOf` đóng vai trò là một Rào cản Khép Vòng (Completion barrier), nó không được sinh ra để định danh cho một Chuỗi Giao Dịch (Transaction) hay một Cấu trúc Hủy Bỏ Đồng Bộ (Structured cancellation).

## 4. Xử Lý Phân Loại Lỗi, Timeout Và Hủy Bỏ (Failure, Timeout, and Cancellation)

- Thiết lập một Ngân sách Thời gian (Deadline) chung cho toàn khối; khi khối tổng hợp (Aggregate) gặp lỗi hoặc quá hạn, tiến hành kích hoạt tín hiệu Hủy (Cancel) các tiến trình con theo năng lực tối đa (Best-effort).
- Các Client viễn trình (Remote client) bắt buộc vẫn phải tự cấu hình cơ chế Timeout độc lập.
- Nghiêm cấm công bố cấu trúc Tích lũy (Accumulator) ra ngoài môi trường trước khi chốt phán quyết chung cuộc (Terminal outcome).
- Phương thức `join` tự động bao bọc các mã lỗi vào trong khuôn khổ `CompletionException`; hệ thống cần thiết kế chủ đích mở gói (Unwrap cause) để nhận diện lỗi gốc.

## 5. Ứng Dụng Trên Kiến Trúc Phân Tán (Multi-instance Fallacy)

- Toàn bộ khối Trạng thái (State) giới hạn cô lập trong một máy ảo JVM; chặng xử lý bất đồng bộ (Async stage) mặc định không kế thừa Khung Ngữ cảnh Giao dịch (Spring Transaction Context) gốc.
- Bài toán đánh mất Ngữ cảnh Giao dịch (Transaction context loss) được phân tích chuyên sâu tại module SPR-002.

> **Nguyên tắc kỹ thuật:** Đồ thị cấu trúc hợp nhất (Composition graph) bắt buộc phải nắm giữ cả dòng chảy Giá trị (Value flow) lẫn dòng chảy Ngoại lệ (Failure flow); Mọi tác động ngoại vi (Shared side effect) đứng chênh vênh bên ngoài đồ thị sẽ vĩnh viễn thoát ly khỏi Hợp đồng Hoàn tất (Completion contract).
