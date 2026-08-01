# Bài toán JVM-003 — Đột biến HashMap đồng thời và Công bố trạng thái bất an toàn (Unsafe Publication)

## 1. Tóm tắt vấn đề (Overview)

Khảo sát một cấu trúc Spring singleton đóng vai trò lưu trữ bảng định tuyến thanh toán (payment routing) trực tiếp trên bộ nhớ (memory). Tác vụ xử lý yêu cầu (Request thread) đọc các quy tắc để lựa chọn đối tác cung cấp (provider), trong khi đó một luồng làm mới định kỳ (Scheduled refresh thread) có nhiệm vụ tải cấu hình mới và tiến hành ghi đè trực tiếp lên chính cấu trúc `HashMap` đó.

Lỗ hổng chết người phát sinh khi luồng làm mới thực thi chuỗi lệnh `clear()` nối tiếp bằng `putAll()`. Luồng yêu cầu (Request) hoàn toàn có khả năng vướng vào khoảng thời gian tranh chấp và thu về một bản đồ rỗng, một mảnh dữ liệu chắp vá, hoặc thậm chí kích hoạt ngoại lệ hệ thống ngay giữa vòng lặp (iterate). 
Nếu nhà phát triển lảng tránh việc sửa đổi trực tiếp (mutate) bằng cách khởi tạo một bản đồ mới rồi hoán đổi tham chiếu (reference), nhưng lại bỏ qua cơ chế công bố an toàn (safe publication) cho thuộc tính (field), thì Luồng đọc (Reader) vẫn rơi vào trạng thái đua dữ liệu (Data race). Trong kịch bản này, Mô hình Bộ nhớ Java (Java Memory Model - JMM) hoàn toàn không có nghĩa vụ bảo chứng cho việc Luồng đọc có nhìn thấy bản chụp (snapshot) mới hay không.

Bài toán thiết lập ranh giới bảo vệ các **Quy tắc Bất biến (Business Invariant)** nội trong một không gian Máy ảo (JVM):

```text
- Từng yêu cầu đơn lẻ phải được quyền truy xuất một Thế hệ (Generation) trọn vẹn của bảng định tuyến.
- Tuyệt đối cấm để lọt trạng thái rỗng hoặc trạng thái lai tạp giữa hai Thế hệ vào tầm ngắm của bất kỳ yêu cầu nào.
- Một tuyến đường thanh toán (PaymentRoute) đã được công bố thì bất khả xâm phạm; mọi biến đổi sau lưng đều bị cấm.
```

> **Nguyên tắc kỹ thuật:** Luồng làm mới bắt buộc phải thay thế toàn bộ khối dữ liệu thông qua một thao tác duy nhất với Điểm hiệu lực (Linearization point) minh bạch. Cấm tuyệt đối hành vi tháo dỡ bảng dữ liệu cũ rồi chắp vá dữ liệu mới ngay trước mặt các luồng yêu cầu.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Tập hợp bất an toàn luồng (`non-thread-safe collection`) | Cấu trúc dữ liệu không tự bảo chứng tính toàn vẹn khi phải hứng chịu hỏa lực đọc/ghi đan xen từ đa luồng. |
| Công bố an toàn (`safe publication`) | Giao thức bàn giao đối tượng cho một luồng khác, bảo chứng đối tượng và trạng thái nội tại đã được khởi tạo toàn vẹn và khả kiến. |
| Quan hệ hệ quả (`happens-before`) | Đặc tả của Java Memory Model cam kết kết quả của luồng A sẽ hiển hiện rành mạch trước mắt luồng B. |
| Bản chụp bất biến (`immutable snapshot`) | Trạng thái toàn vẹn của một khoảnh khắc, vĩnh viễn không thể biến dạng sau khi xuất bản. |
| Hoán đổi tham chiếu nguyên tử (`atomic reference swap`) | Quá trình chuyển giao từ bản chụp cũ sang mới gói gọn trong một chỉ lệnh vật lý duy nhất, cấm chia cắt. |
| Vòng lặp nhất quán yếu (`weakly consistent iteration`) | Cơ chế duyệt an toàn trước biến động đồng thời, nhưng mang rủi ro quan sát thấy một số thay đổi dở dang (không triệt để). |
| Biến đổi cấu trúc (`structural modification`) | Các thao tác tái định hình kiến trúc vật lý của cấu trúc bộ nhớ (thêm, xóa, hoặc `clear()`). |
| Thế hệ (`generation`) | Phiên bản logic đánh dấu một chu kỳ vòng đời toàn vẹn của tập hợp dữ liệu sau mỗi đợt làm mới. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Hệ thống Công nghệ tài chính (Fintech) điều phối giao dịch thanh toán dựa trên định danh Đối tác (`merchantId`):

- Luồng Yêu cầu triệu gọi `selectRoute(merchantId)` để xác định đối tác xử lý.
- Luồng Làm mới (Scheduled) truy xuất dữ liệu từ Dịch vụ Cấu hình (Config service) mỗi chu kỳ 30 giây.
- Hệ thống hỗ trợ cổng API ép buộc làm mới thủ công (Manual refresh).
- Mỗi đợt tải dữ liệu hoàn trả một Thế hệ (Generation) trọn vẹn, được đánh số định danh (Ví dụ: Thế hệ 41 hoặc 42).

Tiêu chuẩn Làm mới (Refresh) bắt buộc tuân thủ nguyên tắc Giao dịch tuyệt đối (All-or-nothing): Yêu cầu có quyền sử dụng Thế hệ Cũ hoặc Mới, nhưng nghiêm cấm việc định tuyến qua một bảng dữ liệu đang trong quá trình xây cất. Cấu trúc `PaymentRoute` phải được thiết lập dạng `record` (Bất biến). Bản thân một cấu trúc Map bất biến chỉ khóa lớp vỏ (cấu trúc Map), nó hoàn toàn vô hại đối với sự biến thiên của các thuộc tính (field) bên trong nếu bản thể dữ liệu vẫn là Khả biến (Mutable).

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

Sự kiện Spring cấp chứng chỉ an toàn (Safe publication) cho một cá thể Singleton tại thời điểm khởi chạy hoàn toàn không đồng nghĩa với việc mọi thao tác ghi đè lên thuộc tính của cá thể đó sau này cũng tự nhiên trở thành An toàn Luồng (Thread-safe).

## 5. Giới hạn Áp dụng (Out of Scope)

Chuyên đề tập trung xử lý: 
- Tính toàn vẹn cấu trúc của bộ nhớ Map.
- Tính khả kiến (Visibility) khi chuyển giao Bản chụp.
- Tính nhất quán của tiến trình đọc bảng quy tắc.
- Tiêu chí lựa chọn Collection theo chuẩn ngữ nghĩa nghiệp vụ.

Cấu trúc này KHÔNG sinh ra để gánh vác các Giao dịch nhiều Khóa (Multi-key transaction) cấp độ Database hay Đồng bộ hóa phiên bản xuyên suốt Đa Máy Chủ (Multi-instance). Hệ thống các Nút phân tán vẫn có thể lưu giữ các Thế hệ khác biệt nếu hạ tầng Cấu hình không cung cấp cơ chế Phiên bản (Versioning) hay Bộ điều phối giao thức (Coordination protocol) thích hợp.

## 6. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Yêu cầu thu về kết quả `null` ngay cả khi Định tuyến hợp lệ tồn tại ở cả Thế hệ cũ lẫn mới.
- Vòng lặp (Iterator) văng ngoại lệ `ConcurrentModificationException` hoặc trích xuất sai lệch số lượng Khóa.
- Tuyến đọc ngoan cố bám trụ Bản chụp cũ vì quy trình cập nhật không kích hoạt Cờ Khả kiến (Visibility).
- Số liệu đo lường (Metric), Giám sát (Health) và Yêu cầu Định tuyến (Routing) ghi nhận những Trạng thái phân mảnh khác nhau.

### Hệ Quả Nghiệp Vụ
- Thanh toán bị từ chối oan uổng hoặc ép chuyển sang Đối tác dự phòng (Fallback).
- Đối tác bị áp đặt sai lệch Quy tắc khác Thế hệ.
- Lệnh Vận hành khẩn cấp (Khóa đối tác) có hiệu lực chập chờn, phân tán.
- Sự cố mang rủi ro Trùng khớp chu kỳ (Timing), thách thức mọi giới hạn của Kiểm thử tuần tự.

## 8. Khuyến Nghị Phân Lớp Áp Dụng (Best Practices)

1. Cường độ Đọc lớn (Read-heavy) + Làm mới Toàn bảng: Áp dụng **Bản chụp Bất biến (Immutable snapshot)** và xuất bản qua cấu trúc `AtomicReference` hoặc biến `volatile`. Yêu cầu Tuyến Ghi xây dựng bản chụp tĩnh hoàn toàn bên ngoài Tuyến Đọc và chỉ hoán đổi Tham chiếu một lần duy nhất.
2. Nhu cầu Cập nhật riêng lẻ từng Khóa (Key-level): Sử dụng `ConcurrentHashMap` nếu hệ thống chấp nhận sự lỏng lẻo của Vòng lặp nhất quán yếu. Không lạm dụng nó để nhóm nhiều lệnh cập nhật Khóa thành một Cụm Nguyên Tử.
3. Ràng buộc bảo toàn cấu trúc Khả biến: Áp dụng `ReentrantReadWriteLock` khi Tuyến đọc bắt buộc cần Khóa để chiêm ngưỡng một Khối Trạng Thái đồng nhất. Phương án này đi kèm rủi ro Nghẽn cổ chai (Contention) và bắt buộc 100% điểm chạm phải tuân thủ kỷ luật Khóa.

### Phác Đồ Lựa Chọn Cấu Trúc
- `AtomicReference<Map<...>>`: Thay toàn bộ, nặng về Đọc, Snapshot Tuyệt Đối (All-or-nothing), cần đính kèm Metadata (Thế hệ).
- `volatile Map<...>`: Kế thừa mô hình Snapshot nhưng Trạng thái thuần túy là 1 biến đơn lập, không đòi hỏi `compare-and-set`.
- `ConcurrentHashMap`: Khóa biến thiên độc lập, không mưu cầu Snapshot Toàn Cục.
- `ReentrantReadWriteLock`: Chuỗi Đọc/Ghi phức hợp, đánh đổi Độ trễ (Latency) lấy Trạng thái Tuyệt Đối.
- Giao Thức Phân Tán (Distributed Protocol): 100% Nút Mạng phải đồng bộ nhất quán tại một sát na Nghiệp vụ.
