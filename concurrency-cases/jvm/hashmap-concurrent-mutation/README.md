# Bài toán JVM-003 — Đột biến HashMap đồng thời và Công bố trạng thái bất an toàn (Unsafe Publication)

## 1. Tóm tắt vấn đề (Overview)

Hãy tưởng tượng bạn có một Spring bean (singleton) lưu trữ các quy tắc định tuyến thanh toán ngay trong bộ nhớ (RAM) bằng một `HashMap`.
- **Luồng đọc (Request thread):** Đọc các quy tắc này để quyết định chọn đối tác thanh toán nào.
- **Luồng ghi (Scheduled refresh thread):** Cứ cách một khoảng thời gian lại kéo cấu hình mới về và ghi đè trực tiếp lên cái `HashMap` đó.

Lỗi chết người ở đây là luồng ghi gọi `clear()` rồi mới `putAll()` dữ liệu mới. Trong khoảnh khắc ngắn ngủi giữa hai lệnh này, luồng đọc có thể lao vào và thấy một danh sách rỗng, hoặc một danh sách đang được cập nhật dở dang (lẫn lộn cả cũ và mới), hoặc thậm chí văng luôn lỗi ứng dụng (exception) khi đang duyệt danh sách.

Ngay cả khi bạn "lách luật" bằng cách tạo hẳn một cái map mới rồi mới gán lại biến (thay vì sửa trực tiếp map cũ), nhưng nếu bạn không xử lý để chia sẻ biến đó an toàn giữa các luồng (gọi là "safe publication"), thì luồng đọc vẫn dính lỗi "Data race" (cạnh tranh dữ liệu). Máy ảo Java (JVM) không hề hứa hẹn rằng luồng đọc sẽ ngay lập tức nhìn thấy cái map mới đó đâu!

**Quy tắc bắt buộc phải tuân thủ trong hệ thống:**

```text
- Từng yêu cầu đơn lẻ phải được quyền truy xuất một Thế hệ (Generation) trọn vẹn của bảng định tuyến.
- Tuyệt đối cấm để lọt trạng thái rỗng hoặc trạng thái lai tạp giữa hai Thế hệ vào tầm ngắm của bất kỳ yêu cầu nào.
- Một tuyến đường thanh toán (PaymentRoute) đã được công bố thì bất khả xâm phạm; mọi biến đổi sau lưng đều bị cấm.
```

> **Nguyên tắc kỹ thuật:** Luồng cập nhật phải thay toàn bộ dữ liệu chỉ bằng MỘT thao tác duy nhất (như kiểu tráo đổi cái rụp). Tuyệt đối không được vừa tháo dỡ vừa đắp dữ liệu mới ngay trước mắt các luồng đang đọc.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Tập hợp bất an toàn luồng (`non-thread-safe collection`) | Các cấu trúc dữ liệu không tự bảo vệ được mình khi bị nhiều luồng vừa đọc vừa ghi cùng lúc (ví dụ: HashMap thông thường). |
| Công bố an toàn (`safe publication`) | Cách bạn bàn giao một đối tượng cho luồng khác một cách an toàn, đảm bảo luồng kia thấy đầy đủ dữ liệu nguyên vẹn của nó. |
| Quan hệ hệ quả (`happens-before`) | Luật của Java đảm bảo rằng thao tác của luồng A chắc chắn sẽ hoàn thành và hiển thị kết quả cho luồng B thấy. |
| Bản chụp bất biến (`immutable snapshot`) | Một bản copy trạng thái tại một thời điểm, làm xong là "đóng băng" luôn, không ai sửa được nữa. |
| Hoán đổi tham chiếu nguyên tử (`atomic reference swap`) | Việc tráo đổi từ dữ liệu cũ sang dữ liệu mới diễn ra chỉ trong một tích tắc, không bị chia cắt hay xen ngang. |
| Vòng lặp nhất quán yếu (`weakly consistent iteration`) | Kiểu vòng lặp không sợ bị lỗi khi duyệt, nhưng có thể đọc trúng dữ liệu nửa cũ nửa mới. |
| Biến đổi cấu trúc (`structural modification`) | Các thao tác làm thay đổi khung sườn của cấu trúc (như thêm, xóa phần tử hoặc gọi `clear()`). |
| Thế hệ (`generation`) | Phiên bản dữ liệu đánh dấu mỗi đợt làm mới thành công. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Hệ thống Fintech của chúng ta xử lý thanh toán dựa theo mã đối tác (`merchantId`):

- **Luồng Yêu cầu:** Gọi hàm `selectRoute(merchantId)` để tìm đối tác thanh toán.
- **Luồng Làm mới:** Cứ 30 giây lại chạy ngầm để lấy dữ liệu mới từ Cấu hình.
- Thỉnh thoảng có người dùng API để ép hệ thống tải lại dữ liệu ngay lập tức (Manual refresh).
- Dữ liệu mỗi lần tải về được xem là một "Thế hệ" (ví dụ: Thế hệ 41, 42).

Yêu cầu kinh doanh là việc cập nhật phải theo kiểu "Tất cả hoặc không gì cả" (All-or-nothing). Các giao dịch thanh toán cứ việc dùng dữ liệu cũ hoặc mới đều được, nhưng cấm tuyệt đối việc dùng dữ liệu đang chắp vá. Cái `PaymentRoute` cũng phải dùng `record` trong Java để không ai sửa được các giá trị bên trong.

## 4. Các Thực thể và Trạng thái chia sẻ (Shared state & Contention)

| Thành phần | Vai trò và Trạng thái |
| --- | --- |
| Đối tượng dùng chung | Singleton `PaymentRoutingRegistry` |
| Trạng thái dùng chung | Bảng ánh xạ `merchantId → PaymentRoute` |
| Tuyến Đọc (Reader) | Luồng Yêu cầu (Request) và Luồng Giám sát (Diagnostic) |
| Tuyến Ghi (Writer) | Luồng Làm mới (Scheduled) hoặc Luồng Cập nhật thủ công |
| Chuỗi thao tác khuyết tật | `clear() → put(...) → put(...)` trên cùng một cá thể `HashMap` |
| Ranh giới đứt gãy | Quãng thời gian kiến trúc Map chắp vá dữ liệu Thế hệ mới |
| Phạm vi Ràng buộc | Không có Transaction Database; Thuần túy là Bộ nhớ JVM |

Nhớ nhé, Spring khởi tạo các Bean an toàn không có nghĩa là mọi thao tác đọc/ghi vào các biến bên trong cái Bean đó đều an toàn với đa luồng!

## 5. Giới hạn Áp dụng (Out of Scope)

Trong phạm vi bài toán này, chúng ta chỉ bàn về:
- Việc giữ cho cái Map trong RAM không bị vỡ.
- Cách chia sẻ bản dữ liệu (Snapshot) cho các luồng thấy nhau (Visibility).
- Đảm bảo đọc danh sách định tuyến an toàn.
- Chọn cấu trúc dữ liệu phù hợp.

Chúng ta KHÔNG bàn đến:
- Các giao dịch cập nhật nhiều khóa (Multi-key transaction) ở cấp độ Database.
- Việc đồng bộ hóa dữ liệu giữa nhiều máy chủ (Multi-instance). (Nếu chạy nhiều máy thì mỗi máy tự xài phiên bản riêng cũng không sao, trừ khi bạn có cơ chế đồng bộ tập trung).

## 6. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Luồng tìm đối tác thanh toán bị trả về `null` dù thực tế dữ liệu đó có ở cả bản cũ lẫn bản mới.
- Ứng dụng tự nhiên văng lỗi `ConcurrentModificationException` khi đang duyệt (iterator), hoặc đếm sai số lượng khóa.
- Cập nhật rồi mà luồng đọc vẫn dùng dữ liệu cũ mềm, vì ứng dụng không báo cho luồng kia biết là đã cập nhật.
- Các hệ thống giám sát (Health check, Metric) trả về số liệu loạn xạ.

### Hệ Quả Nghiệp Vụ
- Khách hàng bị từ chối thanh toán oan uổng, hoặc hệ thống phải đẩy qua đối tác dự phòng kém ngon hơn.
- Đối tác bị áp dụng sai luật định tuyến.
- Có sự cố khẩn cấp (cần khóa đối tác lại ngay) nhưng lệnh lúc ăn lúc không.
- Bug kiểu này rất khó tái hiện (chỉ xảy ra khi "đúng người, đúng thời điểm"), test bằng tay rất khó ra.

## 8. Khuyến Nghị Phân Lớp Áp Dụng (Best Practices)

1. **Nếu đọc rất nhiều + làm mới toàn bộ:** Hãy dùng **Bản chụp Bất biến (Immutable snapshot)**. Tức là tạo hẳn một cái map mới hoàn toàn bên ngoài, chuẩn bị xong xuôi hết rồi mới dùng `AtomicReference` hoặc biến `volatile` để tráo đổi cái rụp vào.
2. **Nếu cần cập nhật lẻ tẻ từng mục:** Có thể xài `ConcurrentHashMap` nếu bạn chấp nhận việc đôi khi lặp qua danh sách sẽ thấy dữ liệu hơi chắp vá một tí. Đừng dùng nó nếu bạn cần gộp nhiều lần cập nhật lại thành một cục bắt buộc đi liền nhau.
3. **Nếu bắt buộc phải đóng băng mọi thứ để đọc:** Xài `ReentrantReadWriteLock`. Cách này bắt các luồng đọc phải đợi nếu đang có luồng cập nhật, rất an toàn nhưng dễ làm hệ thống bị nghẽn (bottleneck).

### Phác Đồ Lựa Chọn Cấu Trúc
- `AtomicReference<Map<...>>`: Dành cho lúc thay mới toàn bộ, đọc nhiều, cần trạng thái cập nhật đồng nhất tuyệt đối, có lưu thêm thông tin thế hệ.
- `volatile Map<...>`: Giống cái trên, nhưng đơn giản hơn, dùng khi chỉ cần 1 biến đơn lẻ mà không cần chức năng So-sánh-và-đổi (compare-and-set).
- `ConcurrentHashMap`: Khi các khóa được cập nhật lẻ tẻ độc lập, không cần cập nhật theo một cụm lớn (không cần Snapshot toàn cục).
- `ReentrantReadWriteLock`: Khi quy trình đọc/ghi phức tạp, bạn sẵn sàng hy sinh tốc độ (độ trễ) để đổi lấy sự chắc chắn tuyệt đối.
- Giao Thức Phân Tán (Distributed Protocol): Khi bạn cần 100% tất cả các máy chủ đều cập nhật dữ liệu cùng một lúc.
