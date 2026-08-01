# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Cội Nguồn Bế Tắc

## 1. Trạng Thái Khởi Điểm (Initial state)

Tệp Đối Soát `settlement/day-1.csv` hoàn toàn vắng mặt. Hai Yêu cầu T1 và T2 đổ bộ trên cùng một Máy Chủ, đồng loạt triệu gọi `generate`. Mấu chốt sai lầm: Mỗi lần gọi tự động sinh ra một Bản Thể `ReentrantLock` hoàn toàn mới.

## 2. Kịch Bản Đan Xen: Khóa Mới Mỗi Lần Triệu Gọi (New Lock Per Call)

| Tiến Trình | Luồng T1 | Luồng T2 | Kho Lưu Trữ (Shared Store) |
| --- | --- | --- | --- |
| 1 | Khởi tạo Lock-A, Khóa Thành Công | | Rỗng |
| 2 | | Khởi tạo Lock-B, Khóa Thành Công | Rỗng |
| 3 | `exists(key)` → false | | Rỗng |
| 4 | | `exists(key)` → false | Rỗng |
| 5 | Vận Hành Render Nội Dung A | Vận Hành Render Nội Dung B | Rỗng |
| 6 | Thực thi `put(key, A)` | | Lưu Trữ A |
| 7 | | Thực thi `put(key, B)` | Lỗi Ghi Đè / Dữ Liệu Nhân Bản |
| 8 | Giải Phóng Lock-A | Giải Phóng Lock-B | Đều Báo Cáo "Đã Tạo Mới" |

Thực chất, Hai Khóa đều vận hành hoàn hảo với phận sự của chúng; Bi kịch ở chỗ Không một Chủ thể (Actor) nào lao vào cạnh tranh chung một Ổ Khóa.

> **Nguyên tắc kỹ thuật:** Bản thân công cụ Khóa không hề hỏng hóc. Cấu trúc ánh xạ (Mapping) từ Tài nguyên Logic sang Cấu trúc Khóa mới là kẻ thủ ác. Một Chìa Khóa Nghiệp Vụ đã bị gán cho hai Ổ Khóa Vật Lý hoàn toàn cách ly.

## 3. Kịch Bản Đan Xen: Vùng Tới Hạn Mỏng Manh (Narrow Critical Section)

Dẫu cho T1 và T2 cùng chia sẻ chung một Khóa (Monitor):

| Tiến Trình | Luồng T1 | Luồng T2 |
| --- | --- | --- |
| 1 | Khóa, Báo Cáo False, Nhả Khóa | Bị Chặn (Chờ Đợi) |
| 2 | Kích hoạt Render Mở Toang (Ngoài Lock) | Khóa, Báo Cáo False, Nhả Khóa |
| 3 | Thi Hành Render | Thi Hành Render |
| 4 | Thực Thi Phép Ghi (Put) | Thực Thi Phép Ghi (Put) |

Lớp Khóa chỉ có tác dụng Serialize khâu "Kiểm Tra", hoàn toàn đánh rơi Đặc Quyền Serialize Quyết định "Nếu Vắng Thì Tạo Mới". Vùng Tới Hạn bắt buộc phải bọc lót trọn vẹn Cụm Hành Vi (Compound action) hoặc Phép Ghi phải tự trang bị Cơ chế Dò Cạnh Tranh Nguyên Tử (Atomic conflict detection) tại Kho Lưu Trữ Uy Quyền (Authoritative Store).

## 4. Kịch Bản Đan Xen: Thu Hồi Khóa Quá Sớm (Premature Lock Eviction)

1. T1 ôm gọn Lock-X dành cho Khóa K.
2. T2 bốc Tham chiếu Lock-X từ Sổ Đăng Ký và rơi vào Chờ Đợi.
3. T1 Nhả Khóa rồi Vô tình Xóa Khóa K khỏi Sổ.
4. T2 Hân hoan chiếm đoạt Lock-X.
5. T3 gọi lệnh `computeIfAbsent(K)`, lập tức sinh ra Lock-Y và Chiếm Khóa.
6. Hậu Quả: T2 và T3 Đồng Đạo Diễn chung Vùng Tới Hạn cho cùng một Khóa K.

Cấu trúc `ConcurrentHashMap` chỉ bảo chứng Tác Vụ Map an toàn; Nó hoàn toàn Từ Chối trách nhiệm Quản trị Vòng đời của các Khóa đang nằm kẹt và những kẻ đang ngóng đợi (Waiter).

## 5. Đối Chiếu Kết Quả (Expected vs Actual)

| Hạng Mục | Kỳ Vọng (Expected) | Hệ Quả Thực Tế (Broken code) |
| --- | --- | --- |
| Định Danh Khóa | Cùng Mã Khóa trên Nút chia sẻ chung Ổ Khóa | Khóa Mới/Khóa Cục Bộ tự rẽ nhánh Bản Thể |
| Vùng Tới Hạn | Bao trọn Check-Render-Publish | Chỉ khóa kín khâu Check |
| Đồng Thời Khóa | Khóa Độc Lập cho phép chạy Song Song | Lệnh `synchronized(this)` trói chung mọi Khóa |
| Vòng Đời Khóa | Bất khả thi có 2 Khóa Chạy cho 1 Mã Khóa | Xóa Sớm đẻ ra 1 Khóa Cũ - 1 Khóa Mới Cùng Tồn Tại |
| Đa Nút Phân Tán | Xung đột bị Đè bẹp bởi Kho/DB | Từng Nút mộng tưởng tự phong Khóa Cục Bộ |
| Đổ Vỡ Hệ Thống | 100% Khóa Giải Phóng ở mọi Tình Huống | Thiếu vắng `finally` chôn sống Khóa vô thời hạn |

## 6. Phân Tích Lớp Trách Nhiệm Gây Lỗi (Root Cause Mapping)

### Định Danh Monitor (Monitor Identity)
Lệnh `synchronized (object)` sử dụng Định danh Tham chiếu (Reference identity). Nó Tuyệt Đối Cự Tuyệt cơ chế gộp Monitor thông qua hành vi `equals()`. Phép màu `happens-before` nối từ Lệnh Unlock tới Lock kế tiếp chỉ hiển linh khi và chỉ khi 2 Actor chạm vào cùng 1 Monitor.

### Cấu Trúc ReentrantLock
`ReentrantLock` sở hữu cơ chế Định Danh y hệt Monitor. Đóng gói nó vào Biến Cục Bộ của hàm làm mất hoàn toàn tính Chia Sẻ. Khóa phải được Nhả bởi chính Luồng đã Chiếm nó; Cấm tuyệt đối chiếm Khóa trước Cửa Ngõ Bất Đồng Bộ (Async boundary) rồi Vất vưởng Nhả Khóa ở một Luồng Hồi Đáp (Callback thread) khác.

### Ranh Giới Phạm Vi Khóa Và Trạng Thái
Vành đai Đồng Bộ hóa phải Tương Đương hoặc Lớn Hơn Vành Đai Bất Biến. Nếu Trạng thái chia sẻ là Mã Artifact, mọi con đường Check/Publish phải chịu chung 1 Khung Phán Xét. Nếu Trạng thái nằm rải rác trên Kho Đối Tượng Liên Nút, Khóa Heap Cục Bộ hoàn toàn mất Tự Cách Trở Thành Tuyến Phòng Thủ Tuyệt Đối.

### Mạng Lưới Phân Tán
Máy A và Máy B Không Thể dùng chung Vùng Nhớ Heap. Biến Tĩnh (Static field) chỉ mang tính Tĩnh nội bộ Classloader/JVM. Bất Biến Liên Nút buộc phải tìm đến Lệnh Độc Quyền CSDL (Unique record), hoặc Giao thức Phân Tán (Distributed Protocol with Lease/Fencing).

## 7. Ranh Giới Điểm Hiệu Lực (Linearization Point)

- **Local Striped Lock:** Sát na chiếm Dải Khóa (Stripe) là Điểm Quyết Định; Lệnh `put` hoàn thiện là mốc Artifact Dính Chặt vào Mạch Luồng.
- **Conditional Create:** Kho Lưu Trữ phán quyết `CREATED` hoặc `EXISTS` tại một Sát na Ghi Nguyên Tử (Atomic write); Đây là Điểm Phán Quyết Tuyệt Đối cho Toàn Mạng.
- **DB Unique Constraint:** Vành Đai Độc Nhất phân xử Kẻ Thắng Bại bằng Luật Transaction Database.
- **Distributed Lease:** Nắm Thẻ Thuê (Lease) là chưa đủ. Thẻ Chắn (Fencing Token) buộc phải bị Thẩm Định Bắt Buộc tại Kho Lưu Trữ trước mỗi Đợt Ghi.

## 8. Xử Lý Phân Loại Lỗi, Hết Thời Hạn (Timeout), Gián Đoạn (Interruption)

- Thiết lập Bộ đo Thời Gian cho Khóa Dương và Triển khai `tryLock` / `lockInterruptibly` để khước từ Nạn Chờ Đợi Vĩnh Hằng.
- Bắt được Tín Hiệu Ngắt, Buộc phải dựng lại Cờ Interrupt và báo Lỗi Cụ Thể.
- Mở Khóa Độc Quyền tại Khối `finally`, CHỈ KHI chiếm Khóa Thành Công.
- Thất Bại Render CẤM công bố Artifact; Nhưng Khóa vẫn phải được Nhả.
- Lưu trữ Tắt Nghẽn Mạng (Store timeout) sinh ra Hành Vi Vô Định (Có thể Write đã lọt). Vòng Thử Lại phải đọc lại Dữ liệu hoặc sử dụng Giao Thức Độc Đẳng (Idempotent).
- Máy Ảo Sập Tự Động Phóng Thích Khóa Nội Bộ nhưng KHÔNG Tự Xóa Dấu Vết External Write dở dang.

## 9. Bài Toán Tương Tranh Và Độ Mịn Của Khóa (Granularity)

- Khóa Diện Rộng (Coarse lock) An Toàn nhưng Đổi Lại Nghẽn Đầu Cầu (Head-of-line blocking).
- Khóa Đơn Tuyến (Per-key) Tối Đa Hóa Song Song nhưng Nan Giải Vòng Đời.
- Khóa Phân Dải (Striped lock) Số Lượng Tĩnh Khống Chế Race Condition, nhưng Chấp Nhận Va Chạm Băm (Hash Collision) Gây Chờ Đợi Oan Uổng.

Chế độ Công Bằng (Fair `ReentrantLock`) không can thiệp Tính Đúng Đắn mà chỉ Tàn Phá Thông Lượng (Throughput). Chỉ Kích Hoạt khi Nạn Đói (Starvation) trở thành Khủng Hoảng Hiện Hữu.

## 10. Ứng Dụng Trên Kiến Trúc Đa Máy Chủ (Multi-instance Fallacy)

Hai Cá Thể Nút cấu hình Hai Bộ Stripe Array Độc Lập. Khóa Cục Bộ Vẫn Rực Sáng Giúp Xóa Đóng Khối Lượng Trùng Lặp Nội Bộ. Nhưng Ranh Giới Tính Đúng Đắn Buộc Phải Giao Phó Cho Cửa Ngõ Dữ Liệu `putIfAbsent` hoặc Giao thức Phân Tán. Mọi Hành Vi Dựa Dẫm Local Lock Xuyên Cluster Là Một Ảo Mộng Chết Người.
