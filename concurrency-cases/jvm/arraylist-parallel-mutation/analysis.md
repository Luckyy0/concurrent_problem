# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Cội Nguồn Vấn Đề

## 1. Trạng Thái Khởi Điểm (Initial state)

Cấu trúc Lô xử lý (Batch) bao gồm hai yêu cầu đầu vào được lập mục lục tuần tự:

```text
Chỉ mục 0 → request-a
Chỉ mục 1 → request-b
```

Tập hợp `results` là cấu trúc `ArrayList` rỗng được phân bổ bộ nhớ khởi tạo ở mức 2 (capacity 2):

```text
Kích thước logic (size) = 0
Mảng vật lý (backing array) = [null, null]
```

Tác vụ T1 xử lý kết quả A, tác vụ T2 xử lý kết quả B. Hai luồng thực thi đồng thời triệu gọi `results.add(...)` trong môi trường thiếu vắng cơ chế đồng bộ (synchronization).

## 2. Kịch Bản Lỗi: Thất Thoát Phần Tử Do Mất Đồng Bộ (Lost Update)

Cấu trúc bên dưới trình bày logic vận hành nội tại của phương thức `add`: trích xuất biến `size` làm chỉ mục chèn (insertion index), ghi dữ liệu vào khe (slot), và cuối cùng tái cập nhật giá trị `size`. Chuỗi hành vi này không tuân thủ tính Nguyên Tử (Atomic).

| Trình tự | Tác vụ T1 | Tác vụ T2 | Trạng thái Bộ nhớ Cục bộ |
| --- | --- | --- | --- |
| 1 | Truy xuất `size = 0`, xác định chỉ mục 0 | | Toàn bộ mảng vật lý trống (`null`) |
| 2 | Chuyển đổi ngữ cảnh (Context switch) | Truy xuất `size = 0`, xác định chỉ mục 0 | Xung đột hệ thống: Hai luồng nhận định chung chỉ mục 0 |
| 3 | Cập nhật kết quả A vào khe 0 | | Mảng: `[A, null]` |
| 4 | | Cập nhật kết quả B vào khe 0 | Mảng: `[B, null]` (A bị ghi đè) |
| 5 | Tái cập nhật `size = 1` | Tái cập nhật `size = 1` | Kích thước mảng sai lệch so với logic (1 thay vì 2) |
| 6 | `Future.get()` hoàn tất vòng đời | `Future.get()` hoàn tất vòng đời | Luồng điều phối tiếp nhận mảng thiếu khuyết (chỉ chứa 1 đối tượng) |

Triển khai cấu trúc nội tại có thể biến thiên tùy phân bản JDK, tuy nhiên `ArrayList` chưa từng bảo chứng mức độ an toàn cho các thao tác thay đổi cấu trúc đa luồng. Việc lạm dụng tính năng dựa trên xác suất hoặc đặc tính hẹp của một phân bản là hành vi phản kỹ thuật.

Phương thức `Future.get()` đảm bảo thiết lập rào cản tầm nhìn (visibility barrier) lên các luồng độc lập, nhưng hành vi này diễn ra SAU khi các lỗi logic (ghi đè) đã khắc sâu vào bộ nhớ. Rào cản bộ nhớ không có năng lực truy hồi trạng thái đã bị hủy hoại.

> **Nguyên tắc kỹ thuật:** Quy trình chờ kết thúc đa luồng chỉ đảm bảo tính toàn vẹn khả kiến của kết quả cuối cùng tại thời điểm chốt, nhưng không định hình tính Nguyên Tử (Atomicity) cho chuỗi hành động biến đổi dữ liệu đứt gãy bên trong.

## 3. Hiện Tượng Bất Ổn Snapshot Dữ Liệu

| Trình tự | Luồng Ghi (T1) | Luồng Đọc Tiến Độ (T3) |
| --- | --- | --- |
| 1 | Khởi chạy `add(resultA)` | |
| 2 | Can thiệp cấu trúc cấp thấp (Resize) | Yêu cầu kết xuất `List.copyOf(results)` |
| 3 | Tái định hình Mảng vật lý hoặc biến kích thước | Kích hoạt chu trình duyệt qua tập tham chiếu |
| 4 | Chốt trạng thái `add` | Phản hồi mảng đứt đoạn (Partial) hoặc Kích hoạt Exception |

Hành vi `ConcurrentModificationException` đóng vai trò phản hồi đứt gãy nhanh (Fail-fast signal). Trình duyệt mảng (Iterator) không chịu trách nhiệm giăng bắt mọi xung đột dữ liệu. Ngay cả khi không phát sinh Ngoại lệ, cấu trúc mảng Snapshot cũng không đủ điều kiện đảm bảo Tính Nhất Quán (Consistency).

## 4. Đối Chiếu Kết Quả (Expected vs Actual)

| Tiêu Chí | Kế Hoạch Đề Ra | Biểu Hiện Thiết Kế Lỗi |
| --- | --- | --- |
| Quy Mô (Cardinality) | Khớp số lượng tác vụ hoàn tất | Nguy cơ thiếu hụt (Lost Update) |
| Định Danh (Identity) | Mỗi `requestId` tồn tại độc lập duy nhất | Rủi ro biến mất định danh; Retry gây nhân bản |
| Toàn Vẹn (Null safety) | Kết quả cuối không chứa `null` | Mảng vật lý có nguy cơ chưa chốt tham chiếu |
| Thứ Tự (Ordering) | Tuân thủ thứ tự truy xuất đầu vào | Bị thay đổi ngẫu nhiên theo thời điểm tác vụ hoàn tất |
| Tiến Độ (Progress) | Snapshot có cơ chế an toàn | Tiến trình đọc gây nhiễu luồng cập nhật hiện thời |
| Lỗi Cục Bộ (Failure) | Cấm phát tán tập kết quả chưa hoàn thiện | Dữ liệu ngầm định bị sửa đổi bất chấp Lô xử lý đã hết hạn (Timeout) |

## 5. Phân Tích Lớp Trách Nhiệm Gây Lỗi (Root Cause Mapping)

### Lớp Cấu Trúc ArrayList
Cấu trúc `ArrayList` tối ưu kiến trúc bộ nhớ cho chuỗi truy xuất tuần tự, loại trừ kiểm soát tương tranh. Mối liên kết giữa sức chứa (Capacity), Mảng Vật Lý (Backing array), Chỉ số đếm (`size`) và Bộ theo dõi (Modification count) là một khối liền mạch. Thiết lập Khóa cục bộ (Lock) lên một phần tử không tạo nên tính toàn vẹn hệ thống.

### Lớp Cấu Trúc Bộ Nhớ Java (Java Memory Model)
Các luồng thao tác Read/Write vô định hình sinh ra hiện tượng Rượt đuổi dữ liệu (Data race). Biến tham chiếu tĩnh (`final`) chỉ giữ cố định đối tượng. Khóa cập nhật (`volatile`) chỉ bảo vệ liên kết vùng nhớ, không áp đặt quy chế độc quyền (Mutual exclusion) trên các phương thức nội tại của `ArrayList`. Đọc thêm về **[Kiến Trúc Java Memory Model](../../concepts/java-memory-model-and-atomicity.md)**.

### Lớp Điều Phối Spring & Executor
Mặc dù Singleton `BrokenBatchQuoteService` được thiết lập làm Cổng tiếp nhận (Gateway), khối danh sách `ArrayList` nội tại bị phát tán ranh giới kiểm soát vào các vùng Thread-pool Executor. Quy mô Bean và Vòng đời Executor không thiết lập chủ quyền quản trị lên các đối tượng truy xuất qua Lambda.

### Sự Đánh Tráo Giữa Cấu Trúc An Toàn Và Đặc Tả Nghiệp Vụ
Kể cả khi xử lý chặn toàn bộ luồng `add`, thứ tự kết quả vẫn tuân thủ cơ chế Hoàn tất (Completion order). Nếu hệ thống quy định đặc tả Duy Trì Thứ Tự Đầu Vào, bắt buộc phải bảo lưu cơ chế Đánh Chỉ Số (Index) hoặc gom cụm theo Trình Tự Khởi Tạo. Tính năng An Toàn Luồng (Thread safety) và Tính Tuần Tự (Ordering) là hai khía cạnh độc lập.

### Tính Cách Ly Giao Dịch
Quá trình phân giải không có sự hiện diện của Cơ sở dữ liệu (Transaction, MVCC). Cơ chế `@Transactional` không có thẩm quyền khôi phục khối tài nguyên List biến thiên trên bộ nhớ RAM. Đối với các tương tác qua mạng (Remote Quote Call), yêu cầu phải có chính sách chống lặp (Idempotency) để khử độc tính nhân bản từ hệ thống gốc.

## 6. Phạm Vi Áp Dụng Điểm Tối Ưu

Mô hình kiến trúc "Tác Vụ Phản Hồi - Điều Phối Tổng Hợp":

1. Bộ điều phối (Coordinator) phân bổ danh sách `Future` theo trật tự gốc.
2. Tác vụ (Worker) độc lập thiết lập `QuoteResult`, loại bỏ sự can thiệp tập hợp kết quả.
3. Bộ điều phối đánh giá hạn mức thời gian, giám sát từng thành phần `Future`.
4. Luồng điều phối duy nhất tiếp nhận trách nhiệm chèn (append) dữ liệu.
5. Snapshot `List.copyOf(results)` công bố danh mục báo giá hợp pháp cuối cùng.

Cấu trúc bộ nhớ chia sẻ được đưa vào tình trạng **Cô Lập Sở Hữu (Thread confinement)**. Triệt tiêu hoàn toàn sự cần thiết của Cấu trúc Đa Luồng (Concurrent list).

## 7. Xử Lý Phân Loại Ngoại Lệ, Bỏ Lỡ Và Dữ Liệu Phân Mảnh

Đặc tả Lô xử lý (Batch API) yêu cầu quyết định cấu trúc nghiêm ngặt:

- **Nguyên lý Tuyệt đối (All-or-nothing):** Đình chỉ toàn bộ lô khi một tác vụ thất bại/quá hạn. Chặn đứng hành vi trả về tệp dữ liệu rác (Partial result).
- **Nguyên lý Định Tuyến Cục Bộ (Per-item outcome):** Trả về đặc tả đa hình (Success/Failure) trên từng kết quả đơn lẻ. Giữ nguyên Quy mô đầu ra.
- **Tiến trình Dòng Chảy (Streaming progress):** Phát tán tín hiệu tức thời qua kênh giao tiếp (Concurrent channel). Bác bỏ tư cách của một Tập hợp chung cuộc.

Khởi chạy `Future.cancel(true)` chỉ kiến tạo tín hiệu ngắt (Interrupt). Tiến trình xử lý (Remote call) phải tự thiết lập cơ chế giới hạn băng thông mạng (Read/Connect timeout). Không hoàn trả kết quả ảo trong khi các luồng ngầm định vẫn tiêu thụ bộ nhớ. Hạn mức thời gian (Deadline) phải quy định cho Tổng Thể Lô.

## 8. Ứng Dụng Trên Kiến Trúc Phân Tán (Multi-instance Fallacy)

Một tiến trình xử lý thông thường tập trung trên một Nút mạng (Node), điều này biện minh cho quyền sở hữu cục bộ. Tuy nhiên, nếu khách hàng yêu cầu dò xét Tiến độ từ hệ thống Cân bằng tải (Load Balancer) sang các Nút khác, cấu trúc `ConcurrentHashMap` hoàn toàn mất tác dụng. Bắt buộc triển khai Định tuyến Cố định (Sticky routing) hoặc Cơ sở lưu trữ ngoài (External Key-Value Store).

## 9. Phân Tích Lớp Trách Nhiệm Gây Lỗi (Root Cause Mapping)

### Hệ Quả Kỹ Thuật
- Kích thước tập kết quả mất cân xứng, trật tự hệ thống sụp đổ.
- Lỗi kích hoạt rải rác trên Snapshot giám sát.
- Tài nguyên hệ thống (Executor) chịu áp lực nặng nề nếu loại bỏ ranh giới thời gian.
- Sai lệch phân tích Log hệ thống vì trạng thái nội tại che đậy xung đột.

### Hệ Quả Nghiệp Vụ
- Tiến trình kiểm toán xác định Lô hoàn thành nhưng Khách hàng từ chối do thiếu hụt thông tin báo giá.
- Tổ chức phân phát dữ liệu cấu trúc chéo gây thiệt hại về định tuyến nghiệp vụ.
- Đối chiếu số liệu đối tác (Reconciliation) bất khả thi do hệ thống thiếu bằng chứng ngoại lệ rõ ràng.

## 10. Giới Hạn Áp Dụng (Out of Scope)

Tài liệu loại trừ cấu trúc đồ thị luồng xử lý (Graph composition) qua nền tảng `CompletableFuture` (Phân định riêng tại module `JVM-009`). Đồng thời, không bao quát các nghiệp vụ truy vấn cơ sở dữ liệu hay bộ trừ trùng (Deduplication) liên Server.
