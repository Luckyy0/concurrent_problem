# Phân Tích Chuyên Sâu: Dòng Thời Gian Tranh Chấp Và Cội Nguồn Bế Tắc

## 1. Trạng Thái Khởi Điểm (Initial state)

Cùng xem kịch bản này: File `settlement/day-1.csv` chưa hề tồn tại. Hai anh chàng Luồng 1 (T1) và Luồng 2 (T2) trên cùng một máy chủ lao vào gọi hàm `generate` cùng một lúc. Lỗi chết người ở đây là: Mỗi lần gọi, hàm lại đẻ ra một ổ khóa `ReentrantLock` mới toanh.

## 2. Kịch Bản Đan Xen: Khóa Mới Mỗi Lần Triệu Gọi (New Lock Per Call)

| Tiến Trình | Luồng T1 | Luồng T2 | Kho Lưu Trữ (Shared Store) |
| --- | --- | --- | --- |
| 1 | Tạo Lock-A, Khóa thành công | | Rỗng |
| 2 | | Tạo Lock-B, Khóa thành công | Rỗng |
| 3 | `exists(key)` → false (chưa có) | | Rỗng |
| 4 | | `exists(key)` → false (chưa có) | Rỗng |
| 5 | Hì hục tạo file A | Hì hục tạo file B | Rỗng |
| 6 | Thực thi `put(key, A)` | | Đã lưu A |
| 7 | | Thực thi `put(key, B)` | Lỗi Ghi Đè / Dữ liệu bị nhân bản |
| 8 | Giải Phóng Lock-A | Giải Phóng Lock-B | Cả hai đều báo "Đã Tạo Mới" |

Đấy, bạn thấy không? Hai cái khóa hoạt động rất hoàn hảo, mỗi tội chả ai tranh giành với ai vì đứa nào cũng có cái khóa riêng!

> **Nguyên tắc kỹ thuật:** Vấn đề không phải do công cụ Khóa bị hỏng. Lỗi là do cách chúng ta ánh xạ khóa logic sang khóa vật lý bị sai. Một mã nghiệp vụ lẽ ra chỉ có 1 ổ khóa, nhưng bạn lại cấp cho nó tận 2 ổ riêng biệt.

## 3. Kịch Bản Đan Xen: Vùng Tới Hạn Mỏng Manh (Narrow Critical Section)

Ngay cả khi T1 và T2 xài chung một ổ khóa, nếu bạn chỉ khóa vùng nhỏ xíu thì toang vẫn hoàn toang:

| Tiến Trình | Luồng T1 | Luồng T2 |
| --- | --- | --- |
| 1 | Khóa, Thấy file chưa có, Nhả khóa luôn | Bị chặn, đứng chờ |
| 2 | Bắt đầu chạy render mệt mỏi | Khóa, Thấy file chưa có, Nhả khóa |
| 3 | Tiếp tục tạo file | Hì hục tạo file theo T1 |
| 4 | Đẩy file lên Store | Lại đẩy file đè lên Store |

Việc khóa kiểu này chỉ giải quyết khâu "Kiểm tra" chứ không khóa nguyên quá trình "Nếu chưa có thì tạo và lưu". Bạn phải bắt buộc bọc khóa cho TOÀN BỘ cụm hành động này. Hoặc nếu không, kho lưu trữ (Database/Store) phải có cơ chế tự bắt lỗi nếu bị ghi đè.

## 4. Kịch Bản Đan Xen: Thu Hồi Khóa Quá Sớm (Premature Lock Eviction)

1. T1 ôm trọn Lock-X dành cho khóa K.
2. T2 lấy Lock-X từ trong cái Map ra và đứng chờ xếp hàng.
3. T1 xong việc, nhả khóa ra, rồi "tiện tay" xóa luôn Khóa K khỏi Map.
4. T2 hớn hở chiếm được Lock-X và chạy.
5. Ngay lúc đó, T3 lao tới, gọi Map để lấy khóa nhưng thấy mất rồi nên nó đẻ ra Lock-Y mới tinh và ôm lấy.
6. Hậu quả: T2 ôm Lock-X cũ, T3 ôm Lock-Y mới. Cả hai anh cùng dắt tay nhau vào vùng cấm. Xong phim!

Dùng `ConcurrentHashMap` thì cái Map an toàn, nhưng nó KHÔNG HỀ quản lý hộ bạn vòng đời của những cái khóa bên trong. 

## 5. Đối Chiếu Kết Quả (Expected vs Actual)

| Hạng Mục | Đáng lẽ phải thế này | Thực tế thê thảm (Broken code) |
| --- | --- | --- |
| Định Danh Khóa | Cùng Mã Khóa thì xài chung 1 ổ khóa. | Mỗi lần chạy đẻ 1 khóa mới tinh. |
| Vùng Tới Hạn | Khóa trọn gói Check-Render-Publish. | Chỉ khóa đúng lúc Check. |
| Đồng Thời Khóa | Khóa ai nấy lo, chạy song song ầm ầm. | Xài `synchronized(this)` bắt tất cả xếp hàng dài. |
| Vòng Đời Khóa | Một mã chỉ có 1 khóa đang chạy. | Xóa Map sớm đẻ ra 2 khóa song song. |
| Đa Nút Phân Tán | Xung đột bị tóm dính ở Database/Store. | Mỗi máy tự thủ dâm cái khóa cục bộ. |
| Đổ Vỡ Hệ Thống | 100% được mở khóa ở mọi tình huống. | Quên khối `finally`, luồng bị lỗi là chết cứng cái khóa. |

## 6. Phân Tích Lớp Trách Nhiệm Gây Lỗi (Root Cause Mapping)

### Định Danh Monitor (Monitor Identity)
Lệnh `synchronized (object)` dựa vào **địa chỉ ô nhớ** chứ không quan tâm cái ruột bên trong (không xài `equals()`). Khóa chỉ phát huy tác dụng chốt chặn khi nhiều luồng cùng bám vào duy nhất 1 địa chỉ ô nhớ đó.

### Cấu Trúc ReentrantLock
Tương tự như Monitor, đóng gói `ReentrantLock` vào một cái biến nội bộ (local variable) trong hàm là tự sát. Và nhớ là: Luồng nào cầm khóa thì chính luồng đó phải nhả khóa. Đừng có cầm khóa ở luồng A rồi quăng sang luồng B bắt nó nhả.

### Ranh Giới Phạm Vi Khóa Và Trạng Thái
Vòng khóa phải ôm trọn dữ liệu nhạy cảm. Nếu bạn phải check và publish chung một mã file, thì khóa phải trùm lên cả Check và Publish. Nếu lưu trữ nằm trên máy chủ khác (Store), khóa cục bộ chả có ý nghĩa phòng thủ tuyệt đối nào đâu.

### Mạng Lưới Phân Tán
Máy A và Máy B xài Ram khác nhau. Từ khóa `static` cũng chỉ xài trên 1 máy. Muốn khóa đa máy chủ, phải nhờ đến Database (như Unique constraint) hoặc mấy trò Phân Tán xịn sò.

## 7. Ranh Giới Điểm Hiệu Lực (Linearization Point)

- **Local Striped Lock:** Khoảnh khắc ôm được Khóa là điểm chốt chặn cục bộ. Lúc đẩy `put` lên kho thành công là lúc file đã nằm trên Mạng.
- **Conditional Create:** Quyết định TẠO MỚI hay ĐÃ CÓ từ Kho Lưu Trữ là phán quyết Tối Cao cho toàn hệ thống.
- **DB Unique Constraint:** Ai ghi trước vào DB thì thắng, dựa theo luật Transaction của Database.
- **Distributed Lease:** Có "Vé Thuê" (Lease) chưa đủ. Bạn cần "Thẻ Chắn" (Fencing Token) kiểm tra chặt chẽ trước khi ghi lên Kho.

## 8. Xử Lý Phân Loại Lỗi, Hết Thời Hạn (Timeout), Gián Đoạn (Interruption)

- Nhớ cài Timeout cho ổ khóa (xài `tryLock` hoặc `lockInterruptibly`) để không bắt người ta chờ tới chết.
- Nếu bị cản (Interrupt), phải ném lỗi rõ ràng và set lại cờ Interrupt.
- Mở khóa thì bỏ vào cục `finally`, và CHỈ MỞ khi đã chiếm được khóa thành công!
- Đang tạo file bị lỗi thì cấm được publish lên store, nhưng nhớ là VẪN PHẢI MỞ KHÓA.
- Nếu Store bị nghẽn mạng thì không biết lúc đó file đã lên chưa. Đoạn này phải ráng mà check lại dữ liệu.
- Máy sập nguồn thì khóa cục bộ tự hủy, nhưng dữ liệu ghi dang dở trên Store thì vẫn rác ở đó.

## 9. Bài Toán Tương Tranh Và Độ Mịn Của Khóa (Granularity)

- Khóa 1 cục to (Coarse lock): An toàn đấy nhưng gây kẹt xe nghẽn mạch.
- Khóa theo từng ID (Per-key): Chạy song song bá cháy nhưng lại khó dọn dẹp khóa cũ (vòng đời).
- Khóa Phân Dải (Striped lock): Vừa có giới hạn cố định để không kẹt, lại vừa cho phép chạy song song. Điểm trừ nhỏ là thỉnh thoảng hai ID khác nhau bị gộp chung 1 dải làm chờ oan uổng xíu thôi.

Cái `ReentrantLock` công bằng (Fair) chỉ tổ làm chậm hệ thống (Throughput) chứ chả giúp đúng đắn gì hơn đâu. Chỉ xài khi thực sự có luồng bị "bỏ đói" quá lâu.

## 10. Ứng Dụng Trên Kiến Trúc Đa Máy Chủ (Multi-instance Fallacy)

Hai máy chủ sẽ có 2 khoá phân dải độc lập. Khóa cục bộ lúc này vẫn tốt nha, nó giúp máy chủ ĐỠ TÍNH TOÁN DƯ THỪA nội bộ. NHƯNG, quyền quyết định ai sống ai chết phải nhường lại cho hàm `putIfAbsent` của Kho dữ liệu chung. Ảo tưởng khóa nội bộ chống đè data trên cụm máy chủ là cực kỳ sai lầm!
