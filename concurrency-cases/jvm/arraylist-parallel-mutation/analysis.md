# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Cội Nguồn Vấn Đề

## 1. Trạng Thái Khởi Điểm (Initial state)

Giả sử hệ thống nhận được một lô xử lý (Batch) chứa hai yêu cầu đầu vào được đánh số thứ tự như sau:

```text
Chỉ mục 0 → request-a
Chỉ mục 1 → request-b
```

Tập hợp `results` là một mảng `ArrayList` rỗng, được cấp phát sẵn bộ nhớ cho 2 phần tử (capacity = 2):

```text
Kích thước logic (size) = 0
Mảng vật lý (backing array) = [null, null]
```

Luồng T1 chạy tác vụ xử lý cho yêu cầu A, luồng T2 xử lý yêu cầu B. Cả hai luồng cùng lúc gọi lệnh `results.add(...)` mà không có cơ chế đồng bộ (synchronization) nào.

## 2. Kịch Bản Lỗi: Thất Thoát Dữ Liệu (Lost Update)

Bên dưới lớp vỏ của hàm `ArrayList.add` là ba bước thực hiện tách rời: lấy biến `size` làm vị trí chèn (insertion index), ghi giá trị vào vị trí đó trong mảng, và tăng giá trị `size` lên 1. Vì 3 bước này không được gói gọn thành một khối duy nhất không thể chia cắt (không có tính Nguyên Tử - Atomic), chúng dễ dàng bị các luồng cắt ngang.

| Trình tự | Luồng T1 | Luồng T2 | Trạng thái Mảng Nội Bộ |
| --- | --- | --- | --- |
| 1 | Đọc `size = 0`, chọn vị trí 0 | | Mảng đang rỗng: `[null, null]` |
| 2 | Đổi luồng (Context switch) | Đọc `size = 0`, chọn vị trí 0 | Cả 2 luồng đều nghĩ vị trí tiếp theo là 0 |
| 3 | Ghi kết quả A vào vị trí 0 | | Mảng thành: `[A, null]` |
| 4 | | Ghi kết quả B vào vị trí 0 | Mảng thành: `[B, null]` (kết quả A đã bị ghi đè mất) |
| 5 | Cập nhật `size = 1` | Cập nhật `size = 1` | Biến `size` là 1 thay vì 2 |
| 6 | Kết thúc (`Future.get()` hoàn tất) | Kết thúc (`Future.get()` hoàn tất) | Luồng chính nhận mảng bị thiếu dữ liệu (chỉ có 1 kết quả) |

Tùy vào phiên bản Java (JDK) mà code bên dưới `ArrayList` có thể viết khác nhau đôi chút, nhưng tựu chung lại, `ArrayList` chưa bao giờ được thiết kế để chạy an toàn với đa luồng. Việc cầu may hay lợi dụng một phiên bản Java cụ thể nào đó để chạy mã này là phản kỹ thuật.

Mặc dù việc gọi `Future.get()` giúp đảm bảo luồng chính chờ các luồng phụ làm xong mới đi tiếp, nhưng thao tác ghi đè (ở bước 4) đã lỡ xảy ra rồi. Việc đồng bộ chờ kết thúc không thể khôi phục lại dữ liệu đã bị ghi đè.

> **Nguyên tắc kỹ thuật:** Việc chờ luồng hoàn tất (`Future.get()`) chỉ đảm bảo luồng chính thấy được kết quả cuối cùng, chứ không giúp quá trình thêm dữ liệu `add()` bên trong trở nên an toàn (Atomic).

## 3. Hiện Tượng Lỗi Khi Đọc Tiến Độ (Snapshot)

| Trình tự | Luồng Ghi (T1) | Luồng Đọc Tiến Độ (T3) |
| --- | --- | --- |
| 1 | Bắt đầu chạy `add(resultA)` | |
| 2 | Mảng đang tự nới rộng (Resize) | Gọi lệnh copy mảng `List.copyOf(results)` |
| 3 | Mảng cấp thấp hoặc biến size đang đổi | Bắt đầu vòng lặp duyệt qua mảng |
| 4 | Hoàn thành lệnh `add` | Quăng lỗi `ConcurrentModificationException` hoặc trả về rác |

Lỗi `ConcurrentModificationException` đóng vai trò là một cái bẫy phát hiện sớm (Fail-fast). Iterator của Java cố gắng ném lỗi này khi phát hiện mảng bị thay đổi giữa chừng. Tuy nhiên, nó chỉ làm theo kiểu "cố gắng hết sức" (best-effort) chứ không đảm bảo 100% bắt được lỗi. Dù nó không quăng lỗi đi nữa, mảng được copy ra cũng có nguy cơ chứa toàn dữ liệu rác (null hoặc thiếu phần tử).

## 4. Đối Chiếu Kết Quả (Expected vs Actual)

| Tiêu Chí | Kế Hoạch Đề Ra | Biểu Hiện Thực Tế Lỗi |
| --- | --- | --- |
| Quy Mô (Số lượng) | Bằng đúng số lượng tác vụ hoàn tất | Ít hơn do luồng ghi đè lên nhau (Lost Update) |
| Định Danh (Tính duy nhất)| Mỗi `requestId` chỉ có 1 kết quả duy nhất | Kết quả bị đè mất; nếu gọi chạy lại (Retry) sẽ sinh ra tác dụng phụ |
| Toàn Vẹn (Null safety) | Kết quả không chứa giá trị `null` | Mảng cấp thấp có thể để hở các khoảng `null` chưa kịp ghi |
| Thứ Tự (Ordering) | Giữ đúng thứ tự như lúc đưa vào | Trả về lộn xộn theo thứ tự hoàn thành của các luồng |
| Tiến Độ (Progress) | Có thể copy mảng an toàn để xem | Khiến ứng dụng văng lỗi khi đang duyệt mảng |
| Lỗi Cục Bộ (Failure) | Không được trả về kết quả nửa vời | Trả về mảng bị thiếu hụt ngay cả khi Lô xử lý đã bị quá hạn (Timeout) |

## 5. Phân Tích Cội Nguồn Vấn Đề (Root Cause Mapping)

### Tại Cấu Trúc ArrayList
`ArrayList` được thiết kế để tối ưu tốc độ đọc/ghi cho 1 luồng duy nhất, hoàn toàn không có cơ chế kiểm soát đa luồng. Biến lưu kích thước mảng (`size`), mảng chứa dữ liệu (Backing array) và biến đếm số lần sửa mảng (`modCount`) là những thứ đi liền với nhau. Nếu chỉ dùng một cái Khóa (Lock) sơ sài bọc bên ngoài một vài chỗ cũng không đảm bảo an toàn cho cả hệ thống.

### Tại Mô Hình Bộ Nhớ Java (Java Memory Model)
Khi nhiều luồng cùng đọc và ghi vào một vùng nhớ mà không xếp hàng, chúng tạo ra hiện tượng chạy đua (Data race). 
Khai báo biến bằng `final` chỉ giúp biến không trỏ sang một danh sách khác. Dùng `volatile` chỉ giúp dữ liệu đồng bộ xuống bộ nhớ chính. Cả hai không hề cung cấp cái Khóa (Mutual exclusion) để bảo vệ hàm `add` của `ArrayList`. Đọc thêm về **[Kiến Trúc Java Memory Model](../../concepts/java-memory-model-and-atomicity.md)**.

### Tại Cách Dùng Luồng (Spring & Executor)
Mặc dù service `BrokenBatchQuoteService` là một biến duy nhất (Singleton), nhưng nó lại vứt các tác vụ vào một Thread-pool bên ngoài xử lý. Framework Spring không tự động bao bọc và bảo vệ các đoạn mã Lambda được ném vào Executor.

### Lẫn Lộn Giữa "An Toàn Luồng" và "Đúng Thứ Tự"
Cho dù bạn có dùng Lock để các luồng phải xếp hàng khi gọi hàm `add()`, thì thứ tự kết quả trong mảng vẫn bị lộn xộn (ai chạy xong trước thì được ghi vào mảng trước). Nếu nghiệp vụ bắt buộc "Thứ tự ra phải giống thứ tự vào", bạn bắt buộc phải dùng mảng định sẵn vị trí (Index) hoặc chờ xong xuôi hết rồi mới gom lại. Tính an toàn đa luồng (Thread safety) và việc giữ nguyên thứ tự (Ordering) là hai chuyện hoàn toàn khác nhau.

## 6. Chiến Lược Tối Ưu

Mô hình "Tác vụ tự trả kết quả - Luồng chính đi thu gom":

1. Luồng điều phối (Coordinator) ném các tác vụ vào Executor và nhận lại một danh sách các `Future`, danh sách này giữ đúng thứ tự ban đầu.
2. Tác vụ (Worker) tự nó tính toán và trả về một cái `QuoteResult` độc lập, tuyệt đối không đụng chạm vào một mảng chung nào cả.
3. Luồng điều phối sẽ chờ tất cả các `Future` chạy xong (có quản lý timeout).
4. Sau khi các `Future` đã xong, chính tay luồng điều phối sẽ tuần tự đọc kết quả và thêm (append) vào mảng `ArrayList`. Lúc này mảng chỉ bị một luồng tác động nên hoàn toàn an toàn.
5. Danh sách `List.copyOf(results)` cuối cùng mới là kết quả chuẩn xác.

Phương pháp này gọi là **Giới hạn quyền sở hữu trong luồng (Thread confinement)**. Nhờ đó, ta chẳng cần phải xài đến mảng đa luồng (Concurrent collection) phức tạp nào cả.

## 7. Xử Lý Các Trường Hợp Lỗi, Bỏ Lỡ Và Dữ Liệu Bị Cắt Khúc

Khi làm API xử lý theo lô (Batch API), cần thống nhất quy tắc thật rõ ràng:

- **Luật Tất-cả-hoặc-không-có-gì (All-or-nothing):** Dừng toàn bộ lô ngay nếu có bất kỳ tác vụ nào bị lỗi hoặc quá hạn. Tuyệt đối không trả về một mảng chứa kết quả lở dở (Partial result).
- **Luật Trả lẻ từng cái (Per-item outcome):** Vẫn trả về đúng số lượng kết quả như lúc đầu vào, tác vụ nào lỗi thì đánh dấu trạng thái riêng của nó là lỗi (Success/Failure). 
- **Báo cáo luồng (Streaming progress):** Nếu cần báo tiến độ liên tục, hãy dùng các kênh Stream/Websocket thay vì cố gắng copy lại một danh sách cuối cùng.

Lưu ý: Gọi `Future.cancel(true)` chỉ là hình thức báo hiệu ngừng (Interrupt). Bên trong tác vụ (khi gọi API ngoài), bạn phải tự thiết lập thời gian hết hạn mạng (Read/Connect timeout) rõ ràng. Tránh tình trạng ứng dụng trả về lỗi nhưng tác vụ vẫn ngầm chạy và ăn tốn bộ nhớ.

## 8. Khái Niệm Xử Lý Đa Máy Chủ (Multi-instance Fallacy)

Phân tích trên tập trung vào một máy chủ (Node). Nhưng nếu khách hàng muốn xem tiến độ qua Load Balancer và request rớt trúng một máy chủ khác, thì biến `ConcurrentHashMap` lưu tiến độ ở máy chủ hiện tại sẽ trở nên vô dụng. Lúc đó, bạn buộc phải dùng Redis hoặc Database để lưu trạng thái lô xử lý.

## 9. Tổng Kết Hậu Quả

### Hậu Quả Kỹ Thuật
- Kích thước tập kết quả không khớp, thứ tự lộn xộn.
- Thỉnh thoảng văng lỗi khó hiểu khi gọi API xem tiến độ.
- Ăn mòn bộ nhớ và quá tải luồng (Executor) nếu các tác vụ treo mà không có Timeout rõ ràng.
- Log hệ thống không có lỗi nhưng dữ liệu trả về bị thiếu hụt.

### Hậu Quả Nghiệp Vụ
- Khách hàng không chấp nhận lô dữ liệu bị hụt thông tin báo giá.
- Hệ thống gửi thiếu dữ liệu cho các hệ thống đối tác.
- Việc đối soát (Reconciliation) trở nên bất khả thi vì ứng dụng không hề ghi nhận là đã xảy ra lỗi.
