# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Ranh Giới Điểm Hiệu Lực

## 1. Dòng Thời Gian Đan Xen (Interleaving Timeline)

Giả định Trạng Thái Gốc bảo lưu Thế hệ 41:
```text
merchant-a → Provider-X, Thế hệ=41
merchant-b → Provider-Y, Thế hệ=41
```

Hệ thống Dịch vụ cấu hình phát lệnh Thế hệ 42:
```text
merchant-a → Provider-Z, Thế hệ=42
merchant-b → Provider-Y, Thế hệ=42
```

Tuyến Ghi (Writer) tháo dỡ bằng `routes.clear()` rồi đổ dữ liệu bằng `routes.putAll(loaded)`. Mấu chốt kỹ thuật: `putAll` KHÔNG phải một thao tác nguyên khối; dưới góc độ ngữ nghĩa (semantics), nó bị phân rã thành một chuỗi liên hoàn các đột biến (mutations) cày xới trên bề mặt Map.

### Cửa Sổ Rủi Ro Đứt Gãy

| Trình tự | Luồng Làm mới (T1) | Luồng Yêu cầu (T2) | Luồng Giám sát (T3) | Trạng Thái Hệ Thống Thực Tế |
| --- | --- | --- | --- | --- |
| 1 | Nạp xong Thế hệ 42 | | | Chứa toàn vẹn Thế hệ 41 |
| 2 | Kích hoạt `routes.clear()` | | | Bản đồ bộ nhớ bị san phẳng (Rỗng) |
| 3 | (Đóng băng) | Triệu gọi `get("merchant-b")` | | Nhận `null`, buộc dạt vào Fallback |
| 4 | Nạp Khóa `merchant-a` | | | Map biến dị mang một mảnh của Thế hệ 42 |
| 5 | (Đóng băng) | | Lặp qua `entrySet()` | Giám sát báo cáo hệ thống chỉ có `merchant-a` |
| 6 | Nạp Khóa `merchant-b` | | | Tái cấu trúc thành công Thế hệ 42 |

Không cần tới 2 Tuyến Ghi để xé nát Quy tắc Bất biến (Invariant). Chỉ 1 Tuyến Ghi và 1 Tuyến Đọc là quá đủ làm sập hệ thống. Nếu vòng lặp (Iteration) đâm sầm vào Cấu trúc đang biến đổi, `HashMap` sẽ thẳng tay ném ngoại lệ `ConcurrentModificationException`. Đặc tính Ngắt nhanh (Fail-fast) chỉ là nỗ lực vớt vát (Best-effort), nghiêm cấm coi đây là Tấm khiên bảo vệ.

> **Nguyên tắc kỹ thuật:** Kết quả cuối cùng hoàn toàn có thể đúng, nhưng sự thật đằng sau là Luồng Yêu Cầu đã tự phán xét và xử lý Sinh Mệnh Giao Dịch ngay tại thời khắc Dữ liệu đang vỡ nát.

## 2. Đối Chiếu Kết Quả (Expected vs Actual)

| Tiêu Chí Đánh Giá | Kỳ Vọng (Expected) | Hệ Quả Thực Tế (Broken code) |
| --- | --- | --- |
| Tính Toàn Vẹn | Tuyến Đọc thụ hưởng 100% Thế hệ 41 hoặc 42 | Tuyến Đọc hứng chịu Map Rỗng hoặc Mảnh vỡ Thế hệ 42 |
| Tính Nhất Quán | Dữ liệu quét qua thuộc chuẩn 1 Thế hệ | Truy xuất đứt gãy lai tạp giữa 2 Thế hệ |
| Khả Kiến (Visibility) | Ghi thành công báo hiệu tức thời cho Mạng lưới | Vắng bóng Hợp đồng Khả kiến do gán qua Thuộc tính thường |
| Vòng Lặp (Iteration) | Giám sát bóc tách Bản chụp tĩnh | Vòng lặp mất Khóa, Đội dữ liệu, hoặc Văng Ngoại Lệ |
| Sai Số (Failure) | Cập nhật lỗi thì lùi về Bản chụp Tốt Gần Nhất | Đột biến Dữ liệu Phá hủy trạng thái phục vụ vĩnh viễn |

## 3. Bản Chất Cội Nguồn Lỗi Theo Các Lớp Phân Tách

### Khuyết Tật Của Cấu Trúc HashMap
`HashMap` tuyên bố cự tuyệt khả năng Chống biến đổi Cấu trúc Đồng thời (Concurrent structural modification). Vắng mặt Cơ chế Đồng bộ Ngoại vi (External synchronization), giao lộ Đọc/Ghi biến thành một Đấu trường Đua dữ liệu (Data race). Tuyệt đối cấm xây dựng độ tin cậy của hệ thống dựa trên ảo mộng "phiên bản JDK hiện tại dường như chạy ổn".

### Hành Vi Phức Hợp (Compound Action)
Góc độ Nghiệp vụ coi Đợt làm mới là Một Hành Vi Đơn Thể. Góc độ Bộ nhớ coi Đợt làm mới là Một Chuỗi Hành Vi Phân Rã (`clear` + `put` + `put`). Không thiết lập **Điểm Hiệu Lực Duy Nhất (Linearization point)**, Tuyến Đọc ngang nhiên lách qua khe hở của sự biến thiên.

### Góc Khuất Mô Hình Bộ Nhớ (Java Memory Model)
Cho dù tái cấu trúc mã thành Sao-Chép-Và-Tráo-Đổi (Copy-and-swap), nhưng nếu Biến tham chiếu (Reference field) là một cấu trúc Biến thường, JMM sẽ không bao giờ vẽ ra đường gạch nối `happens-before` giữa Tuyến Đọc và Tuyến Ghi. Việc xây Map cục bộ trước khi Gán là đúng với Quy tắc Trình tự (Program order) của Luồng, nhưng Vô nghĩa trong việc Khai báo sự Khả kiến.

`volatile`, Monitor Lock, hay Atomic Variable là các chìa khóa thiết lập Ranh giới `happens-before`.

### Ảo Giác Container Spring
Định nghĩa Singleton chỉ trói buộc Quy mô Vòng Đời, không trói buộc Chính Sách Đồng Bộ. Container xuất bản cá thể một cách an toàn, tuy nhiên Phương thức Làm Mới Định Kỳ (Scheduled) và Tuyến Yêu Cầu Controller có quyền tự do tung hoành trên nhiều Luồng và cùng chà đạp lên một Biến Khả Biến (Mutable field).

### Giới Hạn Cơ Chế Giao Dịch
Không Tồn tại Database Row, không có MVCC, Commit hay Rollback. `@Transactional` không có tư cách bảo vệ Trạng Thái Bộ Nhớ Cục Bộ (Memory state). Nếu Luồng Cập Nhật đục phá Map rồi văng Ngoại lệ, Transaction DB sẽ tự hoàn tác, nhưng Mảnh rác trong Java Map vĩnh viễn không thể lùi về trạng thái cũ.

## 4. Ranh Giới Điểm Hiệu Lực (Linearization Point) Chuẩn Mực

Áp dụng Bản chụp Bất biến (Immutable snapshot), Tuyến Ghi phân rã thành 3 tiến trình:
1. Tải và Validate dữ liệu bên trong Không gian Biến cục bộ.
2. Đóng gói Bản chụp hoàn chỉnh qua `Map.copyOf(...)`, phong ấn biến đổi.
3. Xuất bản qua lệnh `AtomicReference.set` hoặc write `volatile`.

Giai đoạn 3 là Điểm Hiệu Lực Độc Tôn. Tuyến Đọc nắm giữ Tham chiếu 1 lần duy nhất, đảm bảo tính Toàn vẹn Thế hệ cho 100% logic xử lý. Nếu đa Tuyến Ghi cùng hội tụ, Hoán đổi Nguyên tử chỉ bảo vệ Tính Toàn Vẹn, chưa bảo vệ Tính Tươi Mới (Độ Trễ). Bắt buộc phải áp dụng Cờ So Sánh (`compare-and-set`) và phế truất Bản chụp Lỗi thời (Stale).

## 5. Xử Lý Phân Loại Lỗi Hệ Thống Và Vòng Đời Cấu Trúc

- Tuyệt đối cấm `clear()` Trạng thái Đang Phục Vụ để dọn chỗ cho dữ liệu rác chưa qua Khâu Validate.
- Nếu Bản chụp Hoán đổi bằng phương pháp Nguyên tử duy nhất, Hệ thống không cần lo lắng về Lỗi Rollback Phân mảnh.
- Application Crash sau khi Swap, Yêu cầu cục bộ đã thấy Thế hệ Mới, nhưng không có nghĩa các Nút (Node) khác cũng tự động đồng bộ Thế hệ đó.

## 6. Giới Hạn Phạm Vi Và Môi Trường Phân Tán

Bản chất `AtomicReference` chỉ cai quản ranh giới JVM nội bộ. Không một khóa (JVM Lock) nào đủ thẩm quyền ép buộc Nút B phải nghe theo Nút A. Đối với Lệnh Yêu cầu Tức thời (Vô hiệu hóa Provider), cần xây dựng Cấu trúc Giao thức từ gốc Config: Mã Thế hệ (Generation order), Kênh Phân phối Sự kiện (Event delivery), Chứng nhận Tiếp nhận (Acknowledgement) hoặc một ranh giới Cấu trúc Phân Tán (Distributed Database).
