# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Cội Nguồn Vấn Đề

## 1. Trạng Thái Khởi Điểm (Initial state)

Cấu hình tài nguyên hệ thống trước thời điểm xung đột:

```text
resources = {}
resourceKey = "tenant-a"
Bộ đếm kích hoạt factory (factory open count) = 0

Luồng T1 triệu gọi register("tenant-a")
Luồng T2 triệu gọi register("tenant-a")
```

Hệ quy tắc Bất biến (Business Invariant) cần được bảo chứng tuyệt đối:

```text
Số lượng tài nguyên đang hoạt động cho "tenant-a" <= 1
Mọi Caller phải thu thập chính xác cùng một phiên bản tài nguyên đang được Registry quản trị
Hệ số kích hoạt Factory cho một phiên đăng ký = 1
```

## 2. Kịch Bản Lỗi: Thứ Tự Thực Thi Đan Xen (Interleaved Execution)

```text
Luồng T1                                       Luồng T2
---------------------------------------------  ----------------------------------
resources.get("tenant-a") -> null
                                                resources.get("tenant-a") -> null
factory.open("tenant-a") -> Kích hoạt Tài nguyên A
                                                factory.open("tenant-a") -> Kích hoạt Tài nguyên B
resources.put("tenant-a", Tài nguyên A)
Hoàn trả Tài nguyên A
                                                resources.put("tenant-a", Tài nguyên B) (Ghi đè A)
                                                Hoàn trả Tài nguyên B
```

Trạng thái Map cuối cùng:

```text
resources = { "tenant-a" -> Tài nguyên B }
```

Tuy nhiên, tiến trình (Process) của hệ thống lúc này đã cấp phát cả Tài nguyên A và Tài nguyên B. Tài nguyên A đã được phân phối cho T1 nhưng lại hoàn toàn nằm ngoài tầm kiểm soát của Registry. Do đó, các chu trình Dọn dẹp/Shutdown dò qua Registry hoàn toàn "mù" trước sự tồn tại của Tài nguyên A, dẫn đến rò rỉ.

## 3. Đối Chiếu Kết Quả (Expected vs Actual)

```text
Mục Tiêu Đề Ra (Expected):
  Tần suất gọi factory = 1
  Tài nguyên hoạt động = 1
  Phản hồi cho T1      = Tài nguyên A
  Phản hồi cho T2      = Tài nguyên A
  Dữ liệu lưu Registry = Tài nguyên A

Thực Tế Vận Hành (Actual):
  Tần suất gọi factory = 2 (Gây áp lực hệ thống)
  Tài nguyên hoạt động = 2 (Rò rỉ)
  Phản hồi cho T1      = Tài nguyên A
  Phản hồi cho T2      = Tài nguyên B
  Dữ liệu lưu Registry = Tài nguyên B
  Tài nguyên mồ côi    = Tài nguyên A (Không thể dọn dẹp)
```

> **Nguyên tắc kỹ thuật:** Cấu trúc vật lý của Map không sụp đổ, nhưng Quy tắc Nghiệp vụ đã hoàn toàn vỡ vụn do hai tập tài nguyên vật lý đã bị phân bổ trước khi Map đưa ra phán quyết chọn lựa cuối cùng.

## 4. Phân Tích Lớp Trách Nhiệm Gây Lỗi (Root Cause Mapping)

### Hệ Sinh Thái Vòng Đời Spring Bean
`ManagedResourceRegistry` được cấu hình dưới dạng Singleton, đồng nghĩa mọi yêu cầu trong cùng `ApplicationContext` dùng chung một tham chiếu Map. Mặc định, Spring không cung cấp bộ chặn đồng bộ để ép buộc các lệnh gọi `register(...)` diễn ra tuần tự.

### Đặc Điểm Của ConcurrentHashMap
Bản thân các lệnh `get` và `put` đơn lẻ là bất khả xâm phạm. Tuy nhiên, Registry yêu cầu một chu trình logic phức hợp:

```text
Kiểm tra biến Key -> Nếu vắng mặt -> Cấp phát giá trị -> Công bố
```

Đây là nguyên mẫu của lỗ hổng **Kiểm tra rồi hành động (Check-then-act)**. Hệ thống thiếu vắng một *Điểm Hiệu Lực Duy Nhất* trói buộc toàn chuỗi. Cả T1 và T2 đều có khả năng đánh giá trạng thái "Vắng mặt Key", và căn cứ vào Snapshot cục bộ để thực thi các hành vi hợp pháp hóa (hóa ra lại gây thảm họa hệ thống).

### Giới Hạn Của Bộ Nhớ Java (Java Memory Model)
`ConcurrentHashMap` cam kết hành vi Công bố An Toàn (Safe publication) giá trị sau khi đã nạp vào Map. Lỗi không nằm ở việc T2 đọc nhầm một đối tượng dở dang; Lỗi ở chỗ hai đối tượng thực thụ (Hoàn chỉnh vật lý) đã cùng được sinh ra độc lập.

Sự tách biệt giữa **Công Bố An Toàn (Safe Publication)** và **Tính Nguyên Tử (Atomicity)**:
- Safe Publication đảm bảo các luồng nhìn thấy phiên bản khởi tạo hoàn chỉnh.
- Atomicity đòi hỏi duy trì quyền Độc tôn (chỉ một chủ thể duy nhất được cấp quyền tạo và đăng ký).

### Lằn Ranh Giới Hạn Giao Dịch Database
Cấu trúc lỗi này phi cơ sở dữ liệu. `@Transactional` không có thẩm quyền trên Java Map nội bộ, không có chức năng Rollback khi Factory lỡ mở Socket/Thread và cũng không áp đặt cơ chế Lock trên các thuộc tính của Singleton.

## 5. Điểm Hiệu Lực Duy Nhất (Linearization Point)

Bất kỳ một cấu trúc khắc phục chuẩn mực nào cũng bắt buộc định hình một **Điểm Hiệu Lực Duy Nhất (Linearization point)**: Thời khắc thao tác được hệ thống công nhận quyền thống trị (Winning actor) và ép các bên còn lại tuân phục kết quả.

- Khai thác `computeIfAbsent`: Điểm hiệu lực nằm ngầm trong khối mã Nguyên Tử của Map.
- Khai thác `putIfAbsent`: Điểm hiệu lực đo bằng thời khắc lệnh chèn (insert) trả về kết quả thành công; Tuy nhiên, phải trả giá bằng việc truy quét và dọn dẹp các tài nguyên sinh thừa.
- Khai thác `FutureTask` Placeholder: Điểm hiệu lực là lúc mỏ neo (Placeholder) chèn thành công; Kể từ đó chỉ Luồng thắng được kích hoạt Factory.

## 6. Xử Lý Phân Loại Lỗi, Timeout Và Dọn Dẹp Tài Nguyên

- Nếu Factory bộc phát ngoại lệ (Exception) trước khi hoàn trả, cấu trúc Map chưa tiếp nhận Entry. Factory buộc phải Tự thân thu dọn (Clean up) các thành phần tài nguyên dở dang.
- Triển khai `computeIfAbsent` nếu hàm ánh xạ bị hủy (throw Exception), Map từ chối cập nhật giá trị; Luồng Caller tiếp theo được quyền Retry.
- Áp dụng `FutureTask`, các Entry lỗi phải tuân thủ điều kiện gỡ bỏ (Remove) để giải phóng môi trường cho chu trình Retry.
- Khi Application Process Crash, toàn bộ Registry cục bộ bốc hơi (Đây không phải là Durable State).
- Nếu chu trình hoạt động đi kèm các quy trình Unregister/Close phức tạp, bắt buộc thiết lập một mô hình Quản Trị Vòng Đời độc lập để không "đóng nhầm" tài nguyên Caller khác đang khai thác.

## 7. Ứng Dụng Trên Kiến Trúc Phân Tán (Multi-instance Fallacy)

```text
Nút Mạng A: registry A -> Kích hoạt Tài nguyên A
Nút Mạng B: registry B -> Kích hoạt Tài nguyên B
```

Toàn bộ các chiến lược `computeIfAbsent`, `putIfAbsent`, `synchronized`, hoặc `FutureTask` chỉ đóng vai trò phân xử các Thread trong phạm vi 1 máy ảo JVM. Chúng tuyệt đối không đảm bảo tính độc nhất của cấu trúc tài nguyên nếu xét trên tổng thể Cluster.

> **Nguyên tắc kỹ thuật:** Đảm bảo tính toàn vẹn cục bộ cho một Nút hoàn toàn có thể vô tình sinh ra hàng loạt tài nguyên độc lập rải rác trên từng Nút của hệ thống.

## 8. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Kích hoạt ồ ạt tài nguyên dư thừa, rò rỉ không hồi kết.
- Phân tách cấu trúc, hai Caller lưu trữ hai định dạng object rẽ nhánh trên cùng một Key.
- Trình thu dọn tài nguyên vô tác dụng trước các đối tượng bị ghi đè ngầm (Overwrite).
- Số lượng Thread/Socket/File descriptor phình to tỷ lệ thuận với xung đột luồng.
- Lỗ hổng ẩn mình hoàn toàn trước lệnh đo lường `map.size()`.

### Hệ Quả Nghiệp Vụ
- Khách hàng (Tenant) đối mặt với sự phân tán Consumer hoặc lỗi kết nối.
- Logic tích lũy thông báo Event/Callback bị kích hoạt lặp.
- Tổn hao chi phí kết nối ngoại vi và vượt trần băng thông cấp phép (Quota).
- Triển khai Release dưới mức Tải cao làm rút ngắn tuổi thọ máy chủ trước khi Crash.

## 9. Giới Hạn Áp Dụng (Out of Scope)

Chuyên đề này đóng khung trong phạm vi cấu hình Đăng ký Tài nguyên Cục bộ (Local Registration). Cấu trúc KHÔNG giải quyết các bài toán sau:

- Bảo lưu đặc tính Độc nhất (Uniqueness) sau quá trình Tái Khởi Động.
- Ràng buộc phân tán toàn cục giữa nhiều Application Instance.
- Triển khai hệ thống Thuê phân tán (Distributed lease / Fencing).
- Mô hình "Kiểm tra rồi Chèn" (Check-then-insert) trên Cơ Sở Dữ Liệu.
- Vòng đời phức hợp Đăng ký - Ngắt kết nối - Phân rã (Register/Unregister/Close).
