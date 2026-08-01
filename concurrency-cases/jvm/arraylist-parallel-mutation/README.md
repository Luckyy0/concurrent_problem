# Bài toán JVM-004 — Thay đổi ArrayList dùng chung trong tác vụ song song

## 1. Tóm tắt vấn đề (Overview)

Xét ngữ cảnh một dịch vụ (Spring service) tiếp nhận các yêu cầu báo giá theo lô (batch) và phân phối thành các tác vụ (task) chạy song song. Các tác vụ này đồng loạt gọi phương thức `results.add(...)` để nạp dữ liệu vào cùng một danh sách `ArrayList`; đồng thời, API giám sát tiến độ (progress endpoint) có thể truy xuất danh sách này để hiển thị trạng thái hiện thời.

`ArrayList` hoàn toàn không hỗ trợ cơ chế thay đổi dữ liệu song song (concurrent mutation). Khi hai tác vụ cùng thực thi thao tác `add`, chúng có thể ghi đè lên cùng một chỉ mục nội bộ hoặc xung đột khi cập nhật thuộc tính `size`. Hậu quả là kết quả báo giá bị thất thoát mặc dù toàn bộ tiến trình báo cáo thành công. Quá trình tra duyệt (iterator) cũng đối mặt với nguy cơ kích hoạt ngoại lệ `ConcurrentModificationException`. Ngoài ra, nếu luồng điều phối (coordinator) đọc danh sách trước rào cản hoàn tất (completion barrier), dữ liệu thu được sẽ mang tính chất phân mảnh (partial) hoặc chứa trạng thái cũ.

Trọng tâm của bài toán này là việc duy trì các **Quy tắc Bất biến (Business Invariants)** trong giới hạn một máy ảo (JVM):

```text
- Mỗi yêu cầu hợp lệ kiến tạo chính xác một cấu trúc QuoteResult tương ứng.
- Tập hợp kết quả cuối cùng phải toàn vẹn: Không khuyết thiếu, không dư thừa, và không chứa phần tử null.
- Tuyệt đối không công bố kết quả chung cuộc (final result) trước khi toàn bộ tác vụ thành công hoặc lô xử lý được đánh dấu thất bại.
- Nếu đặc tả API cam kết bảo lưu thứ tự đầu vào (input order), cấu trúc kết quả đầu ra phải tuân thủ nghiêm ngặt, bất kể thứ tự hoàn thành của từng tác vụ lẻ.
```

> **Nguyên tắc kỹ thuật:** Khởi chạy đa luồng không đồng nghĩa với việc cấp quyền cho đa tác vụ trực tiếp biến đổi một cấu trúc dữ liệu dùng chung. Chiến lược an toàn nhất quán là triển khai cơ chế ủy quyền: mỗi tác vụ trả giá trị độc lập, luồng điều phối duy nhất chịu trách nhiệm tổng hợp dữ liệu.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Cấu trúc không an toàn đa luồng (`non-thread-safe list`) | Tập hợp không hỗ trợ các quy ước đảm bảo tính toàn vẹn khi có đa luồng can thiệp cấu trúc. |
| Thay đổi cấu trúc (`structural modification`) | Các hành vi bổ sung, loại bỏ hoặc tái phân bổ bộ nhớ (resize) làm thay đổi quy mô tập hợp. |
| Cô lập quyền sở hữu (`thread confinement`) | Đối tượng có khả năng biến đổi (Mutable object) bị giới hạn vòng đời truy cập vào một luồng duy nhất. |
| Rào cản hoàn tất (`completion barrier`) | Điểm kiểm soát đồng bộ xác thực toàn bộ tác vụ đã kết thúc trước khi cho phép tra xuất kết quả. |
| Cơ chế thu thập song song (`concurrent collector`) | Thuật toán gom cụm dữ liệu có đặc tả thiết kế đáp ứng tiến trình xử lý đa luồng. |
| Thứ tự tiếp nhận (`encounter order`) | Trình tự xử lý dữ liệu được thiết lập bởi nguồn cung (source stream) hoặc danh sách đầu vào. |
| Trình lặp ngắt nhanh (`fail-fast iterator`) | Trình duyệt dữ liệu có năng lực phát hiện sự can thiệp cấu trúc và kích hoạt ngoại lệ, nhưng không đóng vai trò đồng bộ hóa. |
| Dữ liệu phân mảnh (`partial result`) | Tập hợp kết quả chưa hoàn thiện, không đáp ứng điều kiện để trở thành dữ liệu chung cuộc. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Kiến trúc hệ thống thương mại điện tử/tài chính yêu cầu truy xuất báo giá cho một lô giao dịch:

- Dữ liệu đầu vào chứa chuỗi `QuoteRequest` theo trình tự tiếp nhận.
- Mỗi tác vụ triệu gọi `QuoteClient` biệt lập, có độ trễ (latency) không đồng nhất.
- Luồng điều phối (coordinator) chờ vòng lặp lô hoàn tất và hoàn trả `List<QuoteResult>`.
- Khối vận hành có thể giám sát tiến độ thực thi qua API độc lập.
- Hệ thống gọi (caller) yêu cầu kết quả cuối cùng duy trì thứ tự gốc nhằm ghép nối chuẩn xác.

Hoàn tất sớm một tác vụ không cấp quyền cho tác vụ đó tự ấn định vị trí dữ liệu bằng cách nối chèn (append) vào mảng dùng chung. Thứ tự hoàn thành (Completion order) và Thứ tự đầu vào (Input order) là hai hệ quy chiếu hoàn toàn khác biệt.

## 4. Các Thực thể và Trạng thái chia sẻ (Shared state & Contention)

| Thành phần | Vai trò và Trạng thái |
| --- | --- |
| Tài nguyên chia sẻ | Cấu trúc `ArrayList<QuoteResult>` trực thuộc phạm vi lô xử lý. |
| Chủ thể ghi (Writer) | `N` tác vụ cạnh tranh thực thi hàm `add`. |
| Chủ thể đọc (Reader) | Luồng điều phối hoặc API giám sát sao chép/duyệt danh sách. |
| Thao tác lỗi (Faulty Op) | Tiến trình gọi `ArrayList.add`, mở rộng mảng cấp thấp (resize array), tăng biến `size++` và quá trình duyệt. |
| Điểm xung đột | Đa tác vụ can thiệp đồng thời vào biến `size` và mảng vật lý bên dưới. |
| Rào cản hoàn tất | Chỉ thiết lập khi toàn thể danh sách `Future` phản hồi thành công. |
| Ranh giới Giao dịch | Hoạt động ngoài phạm vi kiểm soát của Database Transaction. |
| Phạm vi Bất biến | Cục bộ trong không gian cấp phát của một Application Instance. |

Cấu trúc `Future.get()` kiến tạo thành công chuỗi quan hệ hệ quả (`happens-before`) giữa tác vụ và luồng điều phối. Tính chất này đảm bảo khả năng hiển thị (visibility) sau hoàn tất, nhưng không có năng lực khôi phục các mảnh dữ liệu đã bị ghi đè ngầm định do xung đột đa luồng trên `ArrayList`.

## 5. Phạm vi tác động (Scope Boundary)

Tài liệu này tập trung vào kỹ thuật tổng hợp dữ liệu trên bộ nhớ (Memory result aggregation):

- Phân định quyền sở hữu đối với tập hợp dữ liệu Mutable.
- Tách bạch ranh giới giữa an toàn đa luồng (Thread-safe) và ngữ nghĩa ảnh chụp trạng thái (Snapshot/Ordering semantics).
- Kỹ thuật kiểm soát luồng kết thúc, quá thời gian (timeout), hủy bỏ (cancellation) và lỗi cục bộ (partial failure).
- Triển khai kiểm định chống thất thoát, dư thừa và thay đổi thứ tự.

Tài liệu không thảo luận cấu trúc đồ thị đa tầng của `CompletableFuture` (thuộc chuyên đề `JVM-009`) hay đồng bộ dữ liệu giao dịch cơ sở dữ liệu.

## 6. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình JPA Và `FOR UPDATE` (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Kích thước tập kết quả (`results.size()`) sai lệch so với số lượng tác vụ hoàn thành.
- Dữ liệu bị ghi đè hoặc mảng tham chiếu chứa trạng thái không nhất quán.
- Các yêu cầu trích xuất Snapshot giám sát phát sinh `ConcurrentModificationException`.
- Chuỗi cấu trúc trả về bị nhiễu loạn theo thứ tự hoàn thành thay vì thứ tự hệ thống.
- Hết thời gian chờ (Timeout) tại mức độ Lô nhưng các tác vụ vẫn ngầm thay đổi cấu trúc danh sách.

### Hệ Quả Nghiệp Vụ
- Thất thoát báo giá mà không phát sinh thông điệp Lỗi báo cáo.
- Hệ thống gọi (Caller) ghép nối báo giá sai lệch thông tin gốc.
- Sai số trong tổng giá trị tài chính do mất mát dữ liệu đầu vào.
- Quá trình thử lại (Retry) tạo hệ lụy gọi API dư thừa hoặc nhân bản tác động (Duplicate side effect).
- Lỗi khó xác minh và phân tích log do tính chất bất định của luồng tương tranh.

## 8. Khuyến Nghị Áp Dụng (Best Practices)

Chiến lược tiêu chuẩn áp dụng là **Cô lập quyền sở hữu (Thread confinement)**: mỗi tác vụ trả về định dạng độc lập `QuoteResult`; duy nhất luồng điều phối khởi tạo và thao tác `ArrayList` sau khi thu thập chuỗi phản hồi từ hệ thống `Future`. Khai báo tập `Future` theo thứ tự tiếp nhận bảo chứng kết quả đầu ra đồng nhất mà không cần kỹ thuật nối chèn chia sẻ (shared append).

Đối với các luồng chuyển đổi phụ thuộc khối lượng CPU (CPU-bound stream), áp dụng chuỗi `parallelStream().map(...).toList()` để chuyển giao quyền quản lý phân mảng và hợp nhất cho Framework; tuyệt đối loại bỏ các cấu trúc tích lũy bên ngoài (external mutable accumulator).

Cấu trúc mảng đồng thời (Concurrent collection) chỉ tối ưu khi kiến trúc hệ thống buộc nhiều bên cung cấp (Producer) công bố kết quả ngay trong quá trình xử lý lô, và đặc tả API chấp thuận dữ liệu sắp xếp theo tốc độ hoàn thành hoặc khả năng hiển thị chậm.

## 9. Phân Tích Phương Án Chọn Lựa (Architectural Alternatives)

- **Tác vụ trả trị + Điều phối hợp nhất:** Lựa chọn ưu tiên. Phục vụ hệ thống yêu cầu cấu trúc đầy đủ, quản trị lỗi tập trung và cố định trình tự.
- **Phân bổ Khe tĩnh (Per-task indexed slot):** Quy mô đầu vào xác định. Mỗi tác vụ sở hữu chỉ mục riêng biệt. Điều phối chỉ quét sau khi qua rào cản hoàn tất.
- **Tiến trình xử lý song song (Parallel stream collector):** Tác vụ trọng tâm CPU, miễn nhiễm I/O chặn (blocking I/O) và phù hợp nguyên lý thực thi luồng tuần tự.
- **Cấu trúc `ConcurrentLinkedQueue`:** Kiến trúc đa tác vụ cung cấp kết quả theo thời gian thực (completion-order); bản chụp cuối cùng (Final snapshot) chỉ công bố sau khi toàn bộ ngưng hoạt động.
- **Cấu trúc `Collections.synchronizedList`:** Đặc thù khi toàn bộ tiến trình truy xuất (kể cả duyệt mảng) phục tùng chung một khung kiểm soát (monitor); yêu cầu chi phí bảo trì ranh giới sở hữu cao hơn.
