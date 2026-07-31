# Bệnh DB-004 — Bóng Ma "Phantom Row" Bóp Nát Vạch Giới Hạn Sức Chứa (DB-004 — Phantom rows làm vỡ capacity check)

## 1. Tóm tắt Câu Chuyện (Tóm tắt)

Một cái Bể Xử Lý (Processing pool) tên là `P-42` có giới hạn sức chứa (capacity) là `10` chỗ và hiện tại đã có `9` người đang bám vào (trạng thái `ACTIVE`). Đột nhiên, Thằng Khách A và Thằng Khách B đồng thời nhào vô để xin chỗ. Hai cu cậu chạy trên 2 dòng Giao Dịch (Spring transactions) hoàn toàn độc lập, chả ai nhìn thấy ai. Cả hai đều đếm số người đang ngồi:

```sql
select count(*)
from slot_allocation
where pool_id = :poolId
  and status = 'ACTIVE';
```

Thằng A thấy `9 < 10` (Còn trống 1 ghế!), thằng B cũng y chang thấy `9 < 10`. Thế là cả 2 thằng đều hả hê giành lấy, nhét vào DB 2 Dòng Phân Bổ (allocation rows) mới tinh khác nhau. Bùm! Hai thằng Chốt Sổ (commit) đều thành công; Cuối cùng, tổng số người "ACTIVE" đè vào bể là `11`!

Lời Dặn Khắc Cốt Ghi Tâm (Invariant):

```text
Với bất cứ cái Bể nào:
  số người đang ngồi <= giới hạn thiết kế

Với thằng P-42 cụ thể:
  giới hạn = 10
  số người hiện tại = 9
  thì CHỈ ĐƯỢC 1 THẰNG vào cửa, thằng còn lại phải cút (rejected)
```

> **Sếp chốt lại:** Trò đi Xí Chỗ 1 cái Dòng mới (row lock lúc INSERT) KHÔNG THỂ BẢO VỆ ĐƯỢC sự thật "còn đúng một ghế trống" kia! Vì cái Giới Hạn Sức Chứa (capacity) bản chất là 1 Luật Sinh Tử áp lên cả 1 TẬP HỢP CÁC DÒNG (predicate invariant), chứ không phải áp lên 1 dòng riêng lẻ!

## 2. Dàn Diễn Viên & Sân Khấu Sát Phạt (Actors và shared state)

| Gương Mặt Vàng | Vai Diễn Sinh Tử |
| --- | --- |
| Thằng Cấp Phát A | Nhìn thấy chỗ trống rồi lén lút nhét `A-101` vô. |
| Thằng Cấp Phát B | Cũng liếc thấy chỗ trống, nhét luôn `B-202` vô không nể nang. |
| Bảng `processing_pool` | Cái vỏ giữ cái Mốc Giới Hạn Sức Chứa (capacity). |
| Bảng `slot_allocation` | Cái đống chứa mấy tờ vé chỗ ngồi của khách, kèm chữ status. |
| Anh Cảnh Sát PostgreSQL MVCC | Người chụp ảnh tĩnh (snapshot) phát cho mấy thằng query đếm số nhìn. |
| Thằng Lười Hibernate | Lôi từng cục entity nhả thành các lệnh INSERT rời rạc chả liên quan mẹ gì nhau. |

Hiện trạng mồi lúc chưa cháy (Initial committed state):

```text
Bể chứa processing_pool(P-42): sức chứa tối đa=10
Đang có: 9 tờ vé ACTIVE
```

Hiện trạng vỡ mồm sau thảm họa (Broken final state):

```text
A-101 = Báo Danh ACTIVE
B-202 = Báo Danh ACTIVE
Tổng số vé ACTIVE hiện đang cầm: 11
CẢ HAI thằng lách qua cửa và đinh ninh mình làm đúng (ACCEPTED)
```

## 3. Khung Giao Dịch & Khúc Cua Tử Thần (Transaction boundary và contention point)

Mỗi lần gọi `allocate()` là chui qua cái máy quét của Spring Proxy:

```text
MỞ CỬA BƯỚC VÀO (BEGIN)
  NGÓ XEM sức chứa tối đa là bao nhiêu?
  ĐẾM XEM có bao nhiêu vé (COUNT(*)) ở bể P-42 đang ACTIVE?
  Đập bàn: Ờ, còn dư 1 ghế kìa tụi bây!
  MÓC BÚT KÝ 1 vé ACTIVE mới nhét vô! (INSERT)
ĐÓNG CỬA ĐI RA (COMMIT)
```

Quá trình Dính Liền Nát Bét (Non-atomic sequence):

```text
ngó đếm 1 đống -> bốc đếm số đó so sánh giới hạn -> nhét thêm 1 thằng mới vô cái đống đó
```

Khúc Cua Tử Thần (Contention point) Ở Đây ĐÉO PHẢI LÀ CÁI MÃ SỐ VÉ. Nó là cái Lưới Điều Kiện Lọc (predicate):

```text
tất cả những dòng nào thuộc pool_id=P-42 và có chữ status=ACTIVE
```

A chèn thẻ `A-101`, B chèn thẻ `B-202`; Bọn Rào Chắn Unique Key không hề chửi nhau xíu nào. Kể cả hàm ĐẾM (Plain COUNT) bình thường cũng đéo có chức năng Dành Trước chỗ ngồi!

## 4. Thế Trận của Các Lớp Đọc (`READ COMMITTED` và `REPEATABLE READ`)

| Lớp Cách Ly (Isolation) | Giao Dịch Đọc Lại Liền Thì Ra Sao? | Đua Tranh Dành Chỗ Cắn Nhau Ra Sao (Capacity race) |
| --- | --- | --- |
| Dễ Dãi `READ COMMITTED` | Đứa khác chốt sổ rồi là Mày THẤY liền thêm dòng ma (visible phantom) | Hai thằng cùng mù quáng đếm được `9`, cùng Nhét và cùng Chốt Sổ! |
| Cứng Hơn Chút `REPEATABLE READ` | Cảnh sát đưa cho Mày đúng cái ảnh cũ, mày vẫn ĐẾM ra `9` suốt! | Vẫn không sao! Thằng nào nhét dòng của thằng nấy, cuối trận vẫn ra `11`! |
| Trùm Cuối `SERIALIZABLE` | Mắt thần SSI soi luôn cả 2 thằng dẫm đạp lên điều kiện tìm kiếm. | Một thằng sẽ bị Đạp Vào Mặt (abort) với lỗi `40001`; Ứng dụng phải Tự Vác Mặt Qua Xin Chạy Lại (retry). |

Anh PostgreSQL Cầm Bảng `REPEATABLE READ` Giúp Mày Không Bị Bóng Đè (không thấy dòng mới khi đếm lại liên tục), nhưng Cái Tờ Ảnh Cũ Đấy ĐÉO Tự Động Biến Thành Người Giữ Cửa bảo vệ Cái Giới Hạn Sức Chứa Gom Tụ Nhé (aggregate capacity invariant).

## 5. Hiện Thực Đắng Cay Lệch Hẳn Kỳ Vọng (Expected và actual)

| Bước Đi | Khách A | Khách B | Kết Cục Máu Mủ (Final) |
| --- | --- | --- | --- |
| CÙNG ĐẾM SỐ | 9 | 9 | |
| PHÁN QUYẾT TỰ SƯỚNG | LÊN XE | LÊN XE | |
| NHÉT DÒNG XUỐNG DB | `A-101` | `B-202` | |
| KẾT CỤC TOANG HOÁC | Chốt Sổ (commit) | Chốt Sổ (commit) | Dư Xe Tới 11 Thằng! |
| KẾT CỤC MƠ ƯỚC | 1 Thằng Lên Xe | 1 Thằng Bị Đạp Xuống Trạm | Vừa Y 10 Thằng. |

## 6. Sổ Tay Nhập Môn Để Tránh Chết Oan (Thuật ngữ cần biết)

| Chữ Nghĩa | Giải Nghĩa Tiếng Người |
| --- | --- |
| Dòng Bóng Ma (phantom row) | Con số đếm bất thình lình lòi thêm 1 dòng khi thằng khác chốt sổ mà ban đầu mình không thấy. |
| Luật Sinh Tử Tập Hợp (predicate invariant) | Cái luật cấm áp lên 1 đám đông dựa theo điều kiện lọc cụ thể, chứ không phải một mình 1 dòng. |
| Nhìn Xong Mới Sờ (check-then-insert) | Đếm trước 1 cục dữ liệu rồi rớ tay vào sửa thêm bằng cái Trí Nhớ có khi đã Lỗi Thời Chết Tiệt rồi (stale). |
| Bức Ảnh Tĩnh (stable snapshot) | Trạng thái thế giới "Đông Đá" đứng yên mà Giao Dịch nhìn thấy khi chạy `REPEATABLE READ`. |
| Gắn Bó Mật Thiết Từng Vùng (predicate dependency) | Dính lứu Ghi/Đọc Dẫm Đạp Nhau trên hẳn 1 Khu Vực, không chỉ 1 Dòng cụ thể. |
| Kính Lúp SSI | Hàng Xịn Serializable Snapshot Isolation bảo vệ Giao Dịch Trùm của nhà PostgreSQL. |
| Thảm Họa Đạp Nhau Ở Cấp Trùm (serialization failure) | Trạng thái 1 Lệnh bị Phản Đòn Bể Mặt Giao Dịch, Ọi Ra Lỗi `40001`. |
| Bộ Đếm Độc Tôn (authoritative counter) | Một Ô Kẻ Cột Nhỏ Xíu Đếm Cộng Trừ ngay trên DB cho Sạch Sẽ (atomic) chứ đéo phải đếm Từng Đứa. |

## 7. Hành Trình Tham Khảo Bản Đồ Sống Còn (Điều hướng)

- [Code Vỡ Nát Lúc Đếm Xong Nhét (Broken count-then-insert)](broken-code.md)
- [Bóc Phốt Bóng Ma Dưới Lăng Kính Phóng Phóng Sự DB (MVCC and predicate analysis)](analysis.md)
- [Đơn Thuốc Chữa Bằng Atomic Counter Lẫn Khóa Mẹ-Con Phủ Đầu (Atomic counter, parent lock and serializable solutions)](solutions.md)
- [Buộc Tội Tại Hiện Trường Có Bày Bin Cảnh Báo (Deterministic PostgreSQL experiments)](experiments.md)
- [Gia Phả MVCC Của PostgreSQL](../../concepts/postgresql-mvcc.md)
- [Hàng Rào Lớp Cách Ly Giao Dịch](../../concepts/isolation-levels.md)
- [Biết Code Concurrent Thì Test Nó Đi!](../../concepts/concurrency-testing.md)

## 8. Sập Hầm Chết Cười Ngoài Thực Tế (Hậu quả production)

- Cái Bể Nhận Gánh Đồ Hút Tải Gấp Mấy Lần Thực Lực Thiết Kế, Nổ Memory, Cháy CPU, Rụi Tài Nguyên.
- Cả Hai Cu Gọi Lệnh Đều Nghĩ Mình Ngon Trúng Giải Độc Đắc, Chả Có Thằng Nào Báo Bể Kèo, Thế Mới Khốn.
- Lên Bảng Dashboard (Dashboard count) Số Hiện Quá Ngưỡng (vượt quá 10) Mặc Dù Từng Tờ Vé Trong Đó Hoàn Toàn Xanh Đẹp Đều Hợp Lệ Cả!!!
- Cái Rào Unique ID Chả Có Ích Mẹ Gì Hết Đâu Để Đỡ Được Quả Trục Trặc Tập Hợp Này Nhé.
- Xoạc Ngang Tăng Máy Chạy App Lên (Tăng application instances) Chỉ Làm Cái Cửa Tử Thần Dễ Cho Bọn Nó Lọt Xuống Nhanh Thêm Thôi.
- Kêu Mấy Chú Vào Rửa Tay Hốt Rác (Manual cleanup) Bóp Chết Lầm Thẻ Người Chơi Hoặc Trừ Kép Thêm Nát (double-decrement counter).
- Thấy Lỗi Mà Gọi Cứu Nháp (Retry) Bù Trù Làm Đẻ Thêm Nhiều Dòng Mới Rác Nếu Bạn Code Mù Idempotent Chưa Chuẩn.

## 9. Cẩm Nang Cứu Mạng Xử Lý Ổ Rác Khuyên Dùng (Hướng sửa khuyến nghị)

Chọn Cách Ép Buộc Cho Cái Sức Chứa Này Có Xác Có Hồn (capacity hữu hình) Ngay Đi:

1. **Ông Nội Phán Quyết Kép Lệnh:** Có Một Dòng Giữ Cửa Sinh Sinh Sự Tồn Tại, Dùng Tính Bơm Kép Cọng Trừ Bộ Đếm (atomic conditional counter update) Xong Nhét Ticket (insert allocation) Trực Tiếp Vô Trong 1 Lệnh Giao Dịch.
2. Nếu Ám Ảnh Ghét Đếm Tạm Counter, Tóm Cổ Luôn Cái Ông Quản Lý Bể (lock parent) Bằng Cái Lệnh Móc Họng `FOR UPDATE`, Ngồi Đó Từ Từ Đếm RỒI Mới Chèn Vô; Mọi Đứa Nào Viết (writer) Bắt Buộc Đánh Xe Đi Theo Cái Cửa Đó.
3. Biết Chắc Chắn Số Ghe Ngồi Có Hạn (finite slots), Rải Sẵn Tất Cả Vé Trống Lên (pre-create slot rows) Rồi Chỉ Việc Mò Lấy Cục `FOR UPDATE SKIP LOCKED` Giựt Từng Dòng, Nhẹ Não!
4. Nếu Lưới Bộ Lọc Quá Loằng Ngoằng Hóc Búa Éo Gôm 1 Dòng Kịp, Xài Trùm `SERIALIZABLE` Và Viết Cái Luật Cầm Kim Bơm Uống Lại Chạy Lại Chặt Cứng (bounded full-transaction retry).

Rào Trấn Unique Mệnh Lệnh (Unique request constraint) Bắt Buộc Giữ Dùng Kè Để Ép Cái Thằng Call Gọi Trùng Điển Mệnh, Nhưng Nhớ Nó Là 1 Cái Vị Trí Khác Gắn Khác Cái Nghĩa Sức Chứa Này Nhé!

## 10. Khoanh Vùng Vấn Đề Này Sâu Tới Đâu (Phạm vi)

Case Nghiên Cứu Lần Này Sài Vỏ Giả Lập Một Cái Trạm Bể Xử Lý Chung Kéo Với Một Cục Lưới Điều Kiện Sức Chứa (predicate capacity). Chuyện Đặt Phòng Khách Sạn Gây Hấn Semantics (Hotel/room) Quăng Qua Kho `BOOK-001` Giải Quyết. Đâm Lệnh Trái Chiều Cong Veo Quán Tính (Write skew) Dính Vào Bộ Gọi Ca Trực Chạy Qua Trạm `DB-005` Chặt Nhé.

