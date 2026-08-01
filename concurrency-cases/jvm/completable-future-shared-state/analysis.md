# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Cội Nguồn Vấn Đề

## 1. Dòng Thời Gian Đan Xen Luồng (Interleaving Timeline)

Cứ tưởng tượng luồng T1 và T2 cùng về đích một lúc. Cả hai đều tranh nhau đọc kích thước (`size`) của mảng rồi ghi đè dữ liệu lên đó. Thế là toang, data của thằng này có thể bị thằng kia ghi đè mất hút luôn!
Nếu luồng T3 có bị lỗi (Exception), nó sẽ làm cho kết quả gom chung bị lỗi theo. Ấy vậy mà luồng T2 vẫn tỉnh bơ chạy xong và tiếp tục ghi bậy vào cái mảng kết quả đó.

## 2. Đối Chiếu Kết Quả (Expected vs Actual)

**Anh em mình kỳ vọng (Expected):**
- Trả về danh sách dữ liệu không thể thay đổi (Immutable).
- Chạy đủ 100%, giữ nguyên thứ tự ngon lành.
- Nếu có lỗi thì vứt ra cái lỗi (Exception) đàng hoàng.

**Thực tế nhận được (Actual):**
- Dữ liệu trong mảng bị rớt mất.
- Thứ tự lộn xộn, ai chạy xong trước thì nằm trước.
- Dù có báo lỗi rồi mà data bên trong vẫn ngấm ngầm bị thay đổi.
- Cần hủy (Cancel) mà nó chả chịu truyền cái lệnh hủy đó xuống cho mấy task con.

## 3. Bản Chất Nguồn Gốc Lỗi (Root Cause)

Bản thân các luồng (Completion stages) nó chạy hoàn toàn độc lập với nhau. Khi bạn dùng `join` ở cuối, nó chỉ chắc chắn là nhận được kết quả cuối cùng thôi, chứ nó mù tịt không hề biết trước đó các luồng đã giành giật nhau ghi đè dữ liệu như thế nào.
Thằng `allOf` chỉ như cái vạch đích báo hiệu "chạy xong hết rồi nè", chứ không phải là công cụ đóng gói quy trình (Transaction) hay hỗ trợ hủy đồng loạt.

## 4. Xử Lý Phân Loại Lỗi, Timeout Và Hủy Bỏ (Failure, Timeout, and Cancellation)

- Bạn nên gom chung 1 cái deadline (Timeout) cho nguyên cục. Khi hết hạn hoặc có lỗi, phải cố gắng đi hủy (Cancel) hết các luồng con đang chạy.
- Nếu bạn gọi ra các dịch vụ ngoài (Remote client), tụi nó cũng phải tự có Timeout riêng nhe.
- Tuyệt đối không đưa cái List data cho người ta xài khi chưa chạy xong xuôi hoàn toàn.
- Hàm `join` sẽ bọc cái lỗi lại thành `CompletionException`, nên khi code bạn phải chủ động bóc lớp vỏ đó ra (Unwrap) để biết chính xác là lỗi gì.

## 5. Ứng Dụng Trên Kiến Trúc Phân Tán (Multi-instance Fallacy)

- Toàn bộ trạng thái lúc này chỉ giới hạn trong một con Java (JVM) thôi. Khi chạy đa luồng (Async), nó sẽ mất luôn cái context của Spring Transaction.
- Mất Transaction Context ra sao, bạn có thể xem kỹ hơn ở bài SPR-002 nhé.

> **Nhớ nè:** Lúc bạn ghép các Future lại, nhớ xử lý cả phần Dữ liệu (Value) và phần Lỗi (Failure). Những thao tác nào chọc ngoáy dữ liệu bên ngoài mà không nằm trong quản lý của Future thì coi như "mồ côi", Future sẽ không kiểm soát được đâu.
