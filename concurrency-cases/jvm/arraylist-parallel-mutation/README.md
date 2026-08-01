# Bài toán JVM-004 — Thay đổi ArrayList dùng chung trong tác vụ song song

## 1. Tóm tắt vấn đề (Overview)

Xét ngữ cảnh một dịch vụ (Spring service) tiếp nhận các yêu cầu báo giá theo lô (batch) và phân phối thành các tác vụ (task) để chạy song song. Các tác vụ này đồng loạt gọi phương thức `results.add(...)` để thêm dữ liệu vào cùng một danh sách `ArrayList`. Đồng thời, một API giám sát tiến độ (progress endpoint) có thể đọc danh sách này để hiển thị trạng thái hiện tại.

Vấn đề là `ArrayList` hoàn toàn không hỗ trợ việc thay đổi dữ liệu từ nhiều luồng cùng lúc (concurrent mutation). Khi hai tác vụ cùng thực thi thao tác `add`, chúng có thể ghi đè lên cùng một vị trí trong mảng nội bộ hoặc xảy ra xung đột khi tăng biến đếm `size`. 

**Hậu quả:**
- Kết quả báo giá bị mất (mặc dù các tác vụ đều báo cáo là thành công).
- Quá trình đọc danh sách (iterator) có nguy cơ văng lỗi `ConcurrentModificationException`.
- Nếu luồng điều phối (coordinator) đọc danh sách trước khi tất cả các tác vụ hoàn tất (completion barrier), dữ liệu thu được sẽ bị thiếu hụt hoặc chứa trạng thái cũ.

Trọng tâm của bài toán này là duy trì các **Quy tắc Bất biến (Business Invariants)** trong bộ nhớ của ứng dụng (JVM):

```text
- Mỗi yêu cầu hợp lệ sẽ tạo ra chính xác một kết quả QuoteResult tương ứng.
- Tập hợp kết quả cuối cùng phải toàn vẹn: Không bị thiếu, không bị thừa, và không chứa phần tử null.
- Tuyệt đối không trả về kết quả cuối cùng (final result) trước khi toàn bộ các tác vụ thành công (hoặc lô xử lý bị đánh dấu thất bại).
- Nếu API yêu cầu phải giữ nguyên thứ tự như lúc gửi vào (input order), thì danh sách kết quả đầu ra cũng phải đúng thứ tự đó, bất kể tác vụ nào chạy xong trước.
```

> **Nguyên tắc kỹ thuật:** Khởi chạy đa luồng không có nghĩa là cho phép nhiều luồng trực tiếp thay đổi một cấu trúc dữ liệu dùng chung. Chiến lược an toàn nhất là: mỗi tác vụ tự trả về giá trị độc lập của nó, sau đó luồng chính (coordinator) sẽ chịu trách nhiệm gom các kết quả này lại.

## 2. Các Thuật ngữ (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Cấu trúc không an toàn đa luồng (`non-thread-safe list`) | Cấu trúc dữ liệu không hỗ trợ việc nhiều luồng cùng đọc/ghi một lúc (ví dụ: `ArrayList`). |
| Thay đổi cấu trúc (`structural modification`) | Các thao tác thêm, xóa hoặc tự động cấp phát lại bộ nhớ (resize) làm thay đổi kích thước của tập hợp. |
| Giới hạn quyền sở hữu trong luồng (`thread confinement`) | Đối tượng có thể thay đổi (mutable object) chỉ được phép truy cập và chỉnh sửa bởi một luồng duy nhất. |
| Rào cản hoàn tất (`completion barrier`) | Điểm chờ (đồng bộ) đảm bảo toàn bộ các tác vụ đã kết thúc trước khi cho phép đọc kết quả. |
| Bộ thu thập song song (`concurrent collector`) | Thuật toán hoặc cấu trúc dữ liệu được thiết kế riêng để gom dữ liệu an toàn trong môi trường đa luồng. |
| Thứ tự tiếp nhận (`encounter order`) | Trình tự xử lý dữ liệu được thiết lập bởi danh sách đầu vào ban đầu. |
| Trình lặp ngắt nhanh (`fail-fast iterator`) | Quá trình duyệt danh sách có khả năng phát hiện xem danh sách có bị luồng khác sửa đổi giữa chừng hay không. Nếu có, nó sẽ ném ngoại lệ ngay lập tức để báo lỗi. |
| Dữ liệu phân mảnh (`partial result`) | Tập hợp kết quả chưa hoàn thiện, chưa gom đủ dữ liệu cuối cùng. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Trong các hệ thống thương mại điện tử hoặc tài chính, thường có yêu cầu lấy báo giá cho một lô giao dịch:

- Dữ liệu đầu vào là một danh sách `QuoteRequest` theo một trình tự nhất định.
- Mỗi tác vụ sẽ gọi đến `QuoteClient` độc lập. Thời gian phản hồi (latency) của từng tác vụ là khác nhau (có cái nhanh, cái chậm).
- Luồng điều phối (coordinator) sẽ chờ cho đến khi toàn bộ lô hoàn tất và trả về danh sách `List<QuoteResult>`.
- Quản trị viên có thể giám sát tiến độ thực thi thông qua một API khác.
- Hệ thống gọi (caller) yêu cầu kết quả cuối cùng phải giữ đúng thứ tự ban đầu để dễ ghép nối dữ liệu.

Lưu ý: Việc một tác vụ hoàn thành sớm không có nghĩa là nó được phép tự ý nhét dữ liệu (append) vào mảng dùng chung. **Thứ tự hoàn thành (Completion order)** và **Thứ tự đầu vào (Input order)** là hai khái niệm hoàn toàn khác biệt.

## 4. Các Thực thể và Trạng thái chia sẻ (Shared state & Contention)

| Thành phần | Vai trò và Trạng thái |
| --- | --- |
| Tài nguyên chia sẻ | Biến `ArrayList<QuoteResult>` dùng chung cho cả lô xử lý. |
| Chủ thể ghi (Writer) | `N` tác vụ cạnh tranh nhau gọi hàm `add`. |
| Chủ thể đọc (Reader) | Luồng điều phối hoặc API giám sát đọc danh sách. |
| Thao tác lỗi (Faulty Op) | Quá trình gọi `ArrayList.add` gây ra việc tự động nới rộng mảng (resize array), tăng biến đếm `size++` không đồng bộ. |
| Điểm xung đột | Nhiều tác vụ đồng thời sửa biến `size` và mảng vật lý bên dưới. |
| Rào cản hoàn tất | Chỉ được thiết lập khi toàn bộ danh sách `Future` báo cáo thành công. |
| Ranh giới Giao dịch | Xảy ra trong bộ nhớ ứng dụng, nằm ngoài phạm vi kiểm soát của Database Transaction. |
| Phạm vi Bất biến | Giới hạn trong bộ nhớ (heap) của một instance ứng dụng. |

Việc gọi `Future.get()` giúp tạo ra mối quan hệ "xảy ra trước" (`happens-before`) giữa tác vụ và luồng điều phối. Tính chất này đảm bảo luồng điều phối sẽ thấy được kết quả sau khi tác vụ hoàn thành. Tuy nhiên, nó **không thể** khôi phục lại các phần tử đã bị ghi đè lên nhau do xung đột đa luồng trong `ArrayList`.

## 5. Phạm vi tác động (Scope Boundary)

Tài liệu này tập trung vào kỹ thuật tổng hợp dữ liệu trên bộ nhớ (Memory result aggregation):

- Phân định rõ quyền sở hữu đối với các tập hợp dữ liệu có thể thay đổi (Mutable collection).
- Phân biệt giữa an toàn đa luồng (Thread-safe) và việc đảm bảo thứ tự dữ liệu (Ordering semantics).
- Kỹ thuật kiểm soát khi kết thúc, xử lý quá thời gian (timeout), hủy bỏ (cancellation) và lỗi cục bộ (partial failure).
- Triển khai kiểm tra để tránh mất mát, trùng lặp hoặc sai lệch thứ tự.

Tài liệu này **không** đi sâu vào cấu trúc đồ thị đa tầng của `CompletableFuture` (thuộc chuyên đề `JVM-009`) hay đồng bộ dữ liệu giao dịch dưới cơ sở dữ liệu.

## 6. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Các Giải Pháp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)

## 7. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Kích thước tập kết quả (`results.size()`) không khớp với số lượng tác vụ đã hoàn thành.
- Dữ liệu bị ghi đè, hoặc mảng chứa giá trị `null` không nhất quán.
- Khi gọi API giám sát để lấy Snapshot, hệ thống có thể quăng lỗi `ConcurrentModificationException`.
- Danh sách trả về bị đảo lộn theo thứ tự hoàn thành thay vì thứ tự yêu cầu.
- Mặc dù bị quá giờ (Timeout) ở cấp độ Lô, nhưng các tác vụ đang chạy ngầm vẫn tiếp tục sửa đổi danh sách.

### Hệ Quả Nghiệp Vụ
- Mất báo giá nhưng hệ thống không hề báo lỗi.
- Hệ thống gọi (Caller) ghép nối báo giá sai lệch thông tin gốc.
- Sai số trong tổng giá trị tài chính do mất dữ liệu đầu vào.
- Nếu thử lại (Retry) toàn bộ lô, có thể dẫn đến việc gọi API thừa hoặc nhân đôi thao tác (Duplicate side effect).
- Bug rất khó tái hiện và phân tích log do tính chất ngẫu nhiên của luồng chạy song song.

## 8. Khuyến Nghị Áp Dụng (Best Practices)

Chiến lược tốt nhất là **Giới hạn quyền sở hữu (Thread confinement)**: mỗi tác vụ chỉ trả về kết quả `QuoteResult` độc lập của nó. Duy nhất luồng điều phối (coordinator) mới là người tạo và điền dữ liệu vào `ArrayList` sau khi đã gom đủ kết quả từ các `Future`. Việc tạo danh sách `Future` theo đúng thứ tự yêu cầu ban đầu sẽ đảm bảo kết quả đầu ra luôn đúng thứ tự mà không cần phải gọi chung một hàm `add`.

Đối với các bài toán nặng về tính toán (CPU-bound stream), nên dùng `parallelStream().map(...).toList()` để giao phó việc chia nhỏ và gộp dữ liệu cho Framework Java xử lý. Tuyệt đối tránh việc dùng một danh sách bên ngoài để gộp kết quả.

Các danh sách hỗ trợ đa luồng (Concurrent collection) chỉ nên dùng khi hệ thống yêu cầu các tác vụ (Producer) phải đưa kết quả ra ngay lập tức, và hệ thống chấp nhận việc thứ tự dữ liệu phụ thuộc vào việc ai chạy xong trước.

## 9. Tổng Hợp Phương Án (Architectural Alternatives)

- **Tác vụ trả về giá trị + Luồng chính gộp lại:** Đây là lựa chọn ưu tiên nhất. Phù hợp cho hệ thống cần trả về kết quả đầy đủ, quản lý lỗi tập trung và giữ nguyên trình tự.
- **Phân bổ vị trí cố định (Per-task indexed slot):** Dùng khi biết trước số lượng đầu vào. Mỗi tác vụ ghi vào một vị trí (index) cố định trong mảng. Luồng điều phối chỉ đọc mảng sau khi tất cả đã hoàn thành.
- **Tiến trình xử lý song song (Parallel stream collector):** Phù hợp cho các tác vụ nặng tính toán, không bị chờ I/O chặn (blocking I/O).
- **Cấu trúc `ConcurrentLinkedQueue`:** Phù hợp khi cần cung cấp kết quả theo thời gian thực (completion-order). Danh sách cuối cùng (Final snapshot) chỉ được công bố sau khi toàn bộ đã xong.
- **Cấu trúc `Collections.synchronizedList`:** Đặc thù dùng khi toàn bộ quá trình đọc/ghi đều tuân theo chung một cơ chế khóa (monitor lock); cách này dễ gây chậm hệ thống do tranh chấp khóa.
