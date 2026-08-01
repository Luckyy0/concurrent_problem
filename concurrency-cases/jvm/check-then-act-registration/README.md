# Bài toán JVM-002 — Đăng Ký Tài Nguyên Theo Mô Hình "Kiểm Tra Rồi Hành Động" (Check-Then-Act)

## 1. Tóm tắt vấn đề (Overview)

Hệ thống duy trì một bộ đăng ký trung tâm (registry) trên bộ nhớ, cấp phát một đối tượng `ManagedResource` tương ứng với mỗi định danh `resourceKey`. Quá trình triển khai lạm dụng `ConcurrentHashMap` và nhầm tưởng rằng kiến trúc đã tự động đạt chuẩn an toàn đa luồng (Thread-safe). Tuy nhiên, lỗ hổng cốt lõi nằm ở việc phân rã quy trình thành hai bước rời rạc:

```text
Kiểm tra key chưa tồn tại → Khởi tạo và đưa tài nguyên mới vào Map
```

Trong thực tế vận hành, hai luồng độc lập hoàn toàn có thể đồng thời vượt qua chốt kiểm tra ban đầu và cùng nhau khởi tạo tài nguyên. Dù cấu trúc Map cuối cùng chỉ lưu trữ duy nhất một đối tượng, đối tượng bị loại trừ vẫn ngầm định được mở (open) và tiếp tục chiếm dụng tài nguyên hệ thống (Thread, Socket, File handle) một cách vô thừa nhận.

Trọng tâm của bài toán này là việc duy trì các **Quy tắc Bất biến (Business Invariants)** trong không gian của một máy ảo (JVM):

```text
- Mỗi resourceKey chỉ được phép liên kết tối đa với một đối tượng ManagedResource đang hoạt động.
- Mọi luồng yêu cầu cấp phát (caller) cho cùng một key phải nhận về chính xác cùng một phiên bản tài nguyên do registry quản trị.
- Hàm kiến tạo (Factory) chỉ được cấp phép thực thi duy nhất một lần cho quá trình đăng ký một key.
```

> **Nguyên tắc kỹ thuật:** Tính an toàn của các thao tác đơn lẻ trên Map không đồng nghĩa với tính an toàn của một quy trình phức hợp. Một chuỗi lệnh "Kiểm tra rồi mới khởi tạo" (check-then-act) luôn tiềm ẩn khe hở để các luồng khác chen ngang, gây ra hiệu ứng chồng lấp trạng thái.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Mô hình `check-then-act` | Đánh giá điều kiện trước khi hành động, tạo ra cửa sổ thời gian (window of vulnerability) để luồng khác thay đổi trạng thái nền. |
| Thao tác phức hợp (`compound action`) | Một quy trình logic lớn được lắp ghép từ nhiều chỉ lệnh nguyên thủy rời rạc. |
| Thao tác Map nguyên tử (`atomic map operation`) | Quá trình can thiệp cấu trúc Map tại một Điểm hiệu lực duy nhất, miễn nhiễm với sự can thiệp từ luồng ngoại vi. |
| Phương thức `computeIfAbsent` | Đặc tả API đảm bảo nguyên lý: Khởi tạo giá trị chỉ khi Key vắng mặt, bảo chứng tính Nguyên tử nội tại của Map. |
| Điểm hiệu lực duy nhất (`linearization point`) | Thời điểm duy nhất mà toàn bộ hệ thống công nhận một thao tác đã chính thức thiết lập trạng thái. |
| Tài nguyên vô thừa nhận (`orphan resource`) | Đối tượng đã được cấp phát nhưng nằm ngoài vùng kiểm soát của registry, không thể được thu hồi chuẩn xác. |
| Công bố an toàn (`safe publication`) | Hành vi đưa đối tượng vào môi trường chia sẻ sao cho các luồng khác chỉ nhìn thấy trạng thái đã khởi tạo hoàn thiện. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Ứng dụng quản trị một Local Client cho mỗi hệ thống khách hàng (Tenant) hoặc Điểm cuối (Endpoint):

- Luồng T1 yêu cầu cấp phát tài nguyên cho định danh `tenant-a`;
- Luồng T2 đồng thời phát sinh yêu cầu tương tự;
- Phương thức `ManagedResourceFactory.open(...)` kích hoạt các tài nguyên nặng như Worker thread, Socket hoặc File watcher;
- Registry có trách nhiệm thu hồi và tái sử dụng tài nguyên đã tồn tại thay vì kích hoạt bừa bãi tài nguyên mới.

Hệ quy chiếu ở đây giới hạn tại một registry cục bộ của một Application Instance. Nếu bài toán yêu cầu đảm bảo tính duy nhất xuyên suốt Cụm máy chủ (Cluster), cấu trúc Local Map không đủ thẩm quyền; vấn đề đó thuộc chuyên đề `DB-006` và `DIST-001`.

## 4. Các Thực thể và Trạng thái chia sẻ (Shared state & Contention)

| Thành phần | Vai trò và Trạng thái |
| --- | --- |
| Tài nguyên chia sẻ | Singleton trung tâm `ManagedResourceRegistry` |
| Bộ nhớ trạng thái | Cấu trúc `ConcurrentMap<String, ManagedResource>` |
| Chủ thể cạnh tranh | Hai hoặc nhiều luồng yêu cầu (Request/Worker thread) |
| Chuỗi thao tác lỗi | `get(key) → open(key) → put(key, resource)` |
| Vị trí đứt gãy | Khoảng trống thời gian hậu phương thức `get` và tiền phương thức `put` |
| Ranh giới Giao dịch | Hoạt động ngoài phạm vi kiểm soát của Database Transaction |
| Phạm vi Bất biến | Cục bộ trong không gian cấp phát của một Application Instance |

## 5. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 6. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Hàm kiến tạo (Factory) bị lạm dụng khởi chạy nhiều lần trên cùng một định danh.
- Các tài nguyên chậm chân không được bảo chứng quyền quản lý vào registry.
- Gây rò rỉ nghiêm trọng các tài nguyên gốc như Thread, Socket, File handle, Subscription.
- Phân tách cấu trúc trả về: Các luồng cùng gọi đăng ký nhưng thu thập về các đối tượng vật lý khác nhau.
- Quy trình dọn dẹp (Cleanup/Shutdown) mất phương hướng vì chỉ dò tìm được tài nguyên hợp lệ trên Map.

### Hệ Quả Nghiệp Vụ
- Hệ thống vô ý thiết lập đa luồng xử lý hoặc đa kết nối cho cùng một khách hàng.
- Sinh ra luồng sự kiện (Event) lặp hoặc đẩy hệ thống đối tác vượt ngưỡng giới hạn cấp phép (Rate limit).
- Máy chủ kiệt quệ định danh kết nối (Connection/File descriptor) khi hứng chịu tải trọng cao.
- Sự cố hiển thị ngẫu nhiên theo xác suất phân luồng (Timing), gây trở ngại cực lớn cho quy trình tái hiện và gỡ lỗi.

## 7. Khuyến Nghị Áp Dụng (Best Practices)

- Ưu tiên phương thức `computeIfAbsent` khi thao tác kiến tạo nhẹ, không gọi lại chính khối Map nội tại và không đòi hỏi xử lý Remote I/O kéo dài.
- Áp dụng kỹ thuật Điền Trống (Placeholder) qua `FutureTask` nếu quá trình cấp phát tốn kém hoặc mang tính chất biến đổi ngoại vi (Side effect); chỉ duy nhất luồng "thắng" được quyền thi hành Factory, các luồng thụ động chuyển sang trạng thái chờ đồng bộ kết quả.
- Trong trường hợp chấp thuận việc hàm kiến tạo lặp, áp dụng `putIfAbsent` nhưng bắt buộc phải triển khai quy trình Thu hồi khẩn cấp (Clean up) đối với các tài nguyên dư thừa.
- Tránh ảo tưởng dùng cơ chế Registry Cục Bộ để phân xử tính độc quyền (Uniqueness) trên quy mô Kiến trúc Đa máy chủ (Multi-instance).

## 8. Phân Tích Phương Án Chọn Lựa (Architectural Alternatives)

- **Sử dụng `computeIfAbsent`**: Tương thích Cache/Registry cục bộ, tiến trình khởi tạo tức thời và hiếm khi phát sinh lỗi.
- **Mô hình Placeholder (`FutureTask`)**: Đáp ứng khởi tạo khối lượng lớn, hệ thống Caller yêu cầu chia sẻ đồng bộ đối tượng, cần cấu trúc thu dọn khi hàm kiến tạo thất bại để hỗ trợ chu trình Thử lại (Retry).
- **Hợp nhất `putIfAbsent` & Cleanup**: Xảy ra dư thừa tài nguyên là có thể chấp nhận, miễn là có năng lực tự định vị để đóng tài nguyên rác.
- **Từ khóa `synchronized`**: Quy mô bộ đăng ký siêu nhỏ, tỷ lệ cạnh tranh thấp và chấp thuận đặc tả mọi luồng xếp hàng chung chờ cấp khóa (Global lock).
- **Ràng buộc Database hoặc Điều phối Phân tán**: Các Quy tắc Bất biến chia sẻ hệ sinh thái nhiều Nút mạng hoặc phải bảo lưu sau tiến trình Tái khởi động (Restart).
