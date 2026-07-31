# Bóc Băng Tua Nhanh Lúc Bốc, Kẹp Lẫn Đứt Dây Khóa Và Ảo Thị Hiện Hình (Phân tích lock acquisition, visibility và release)

## 1. Điểm Khởi Đầu Tĩnh Lặng (Initial state)

```text
tenant_quota(T-42)
  quota    = 10
  revision = 5
```

Ba Ông Tướng (actors) Đều Đang Ngậm Dây Kết Nối (connections)/Tự Bơi Giao Dịch (physical transactions) Riêng Lẻ.

## 2. Đoạn Băng 1 — Lệnh SELECT Chay Không Đủ Tuổi Chiếm Đất (Timeline 1 — Plain SELECT không reserve row)

| Bước Đi | Ông Chú A | Cậu Em B |
| --- | --- | --- |
| 1 | KÉO RÀO GIAO DỊCH (BEGIN RC) | |
| 2 | Ngó chay xem `quota` (plain SELECT) thấy `10` | |
| 3 | Mải Mê Tính Nháp Validation Ở Nhà | KÉO RÀO GIAO DỊCH (BEGIN) |
| 4 | | Sút UPDATE sửa `quota` xuống `8` |
| 5 | | DẬP BÚA CHỐT KÉO (COMMIT) |
| 6 | A Ôm Gối Ngó Tờ Dữ Liệu Thiêu Mốc (stale) `10` | |

Bung Cửa Giao Dịch (transaction open) Không Giải Quyết Đóng Đinh Gì Cả Nhé Cưng! Chẳng Có Đứa Khóa Vòng (row-level lock) Nào Thèm Rớ Tới Tròng Vào Chỉ Bởi Vì Mấy Lệnh Đọc Bình Dân Khờ Khạo (ordinary SELECT) Đâu!

## 3. Đoạn Băng 2 — Áo Khóa Đóng Rõ Vòng Đinh (Timeline 2 — Explicit row lock)

| Bước Đi | Ông Chú A | Cậu Em B | Khán Giả C |
| --- | --- | --- | --- |
| 1 | BẮT ĐẦU VÀO TRẬN (BEGIN) | | |
| 2 | SELECT FOR UPDATE đọc `quota 10` | | |
| 3 | Ép UPDATE Mới Là `12` Nháp KÍN (uncommitted) | | |
| 4 | | Vừa Ló Mặt Dọng UPDATE/FOR UPDATE -> Bị Trói Đứng Chờ Ngáp (waits) | |
| 5 | | | Ngó chay plain SELECT -> Thấy Lại Tờ Xưa Đã Chốt `10` (returns committed) |
| 6 | ĐẬP TỜ CỜ TRẮNG CHỐT (COMMIT) | | |
| 7 | Cởi Xiềng Buông Khóa (lock release) | B Mới Được Đi Tiếp/Tự Cắn Tính Lại Mớ Nháp (continues/re-evaluates) | Bảng Lặp Lại Bắt Chước Lệnh Ngó Kế (next C SELECT) Sẽ Thấy Áo Mới `12` |

> **Sếp chốt lại:** Tụ Tập Bảng Trùng Mệnh Khóa (lock compatibility) Sẽ Kêu Án Thằng Nào Phải Vác Ghế Đợi Đứt Xích; Còn Lưới Hiện Hình Phép Thuật (MVCC visibility) Chọn Cho Thằng Ngó Chay (plain reader) Nhặt Được Dấu Hàng Chữ Tờ Lớp Tuple Dĩ Vãng Nào Coi Phù Hợp.

## 4. Hành Nghề Bóc Kẹp Áp Gông Khóa (Lock acquisition)

Súng Lệnh Của Tụi PostgreSQL Toàn Auto Quét Gán Cáo Phục Bảng Áo Table (relation/table-level modes) Phủ Vây Xong Đã:

- Dòm chay (plain SELECT): Đè `ACCESS SHARE`;
- Có Dấu Cầu Xin Sửa (SELECT ... FOR UPDATE): Xin Nhốt Làng Khoác Áo `ROW SHARE` Nhưng Chơi Lút Dây Chóp Lọc Cho Vùng Mấy Dòng Bị Chọn Thôi (row locks trên selected rows);
- Sát Thát Hành Xử Kịch Độc (UPDATE/DELETE/INSERT): Cất Khoác Lưới Lớn `ROW EXCLUSIVE` Xong Liền Vác Chĩa Khóa Vây (row/index locks) Cắm Đinh Khớp Mã.

Cái Danh Tự Kêu Cho Sang `ROW SHARE`/`ROW EXCLUSIVE` Vốn Là Tên Bảng (table-level mode names) Kêu Hù Nhé, Chứ ĐÉO Có Cái Dụ Là Mọi Hàng Đang Chết Điếng Đứng Ngâm Đâu Nghen Nhóc! Nên Tỉnh Đòn Móc Xé Kín Sự Khác Nhau Giữa Lưới Làng Vây Rộng (relation lock) Cùng Mũi Căm Điểm Chọn Row (tuple/row lock).

Đóng Bức Vách (Explicit):

```sql
lock table tenant_quota in access exclusive mode;
```

Cái Búa Chém Quét `ACCESS EXCLUSIVE` Kia Tát Chết Cả Thằng Vừa Vác Bầu Nước Trống Trơn `ACCESS SHARE`, Để Bất Kỳ Cu Đọc Chay Bào Gái Đeo Chéo Lệnh Khác Vào Đều Phải Đứt Thở Hóc Tắc Dài Hạn Chờ (wait). Chiêu Này Cách Xa Một Trời Vực Với Cửa Tách Xoáy Khóa Cắm Mỏ Khóa Chặn (row lock) 1 Hướng Lỗ Trên Quota Á!

## 5. Bảng Sóng Giao Tranh Cháy Chập Lock Row (Row-lock compatibility trong case)

Ông Chú A Tròng Kéo Cờ `FOR UPDATE` Rút Chặt Khóa Đụng Hàng Nước Kẽ Bọt (conflict) Với Bọn:

- B Dộng Ép Cọc Lệnh Nhét `UPDATE`/Cắt `DELETE`;
- B Bốc Xấc Lệnh Khóa Y Chang `FOR UPDATE`;
- B Quấn Áo Vi Tế Cực Kẹp `FOR NO KEY UPDATE`;
- Toàn Các Đám Thuộc Nòi Khóa Không Khớp Dây (incompatible) Áp Đỉnh Kéo Trong Bộ Matrix Rừng Luật PostgreSQL!

A Vác Ấn Kéo Dây `FOR UPDATE` ÉO Có Đủ Phép Gài Chết Hút Nghẽn Cu C Đứng Đọc MVCC Chay! C Nào Có Dám Mở Mồm Xin Gắn Trấn Khóa (row lock) Làm Gì Đâu, Ảnh Cứ Dựa Dẫm Sương Kép Bộ Kỷ Niệm (snapshot) Đi Moi Tuple Cũ Rích Xưa Đã Kịp Bọc Lần Cuối (last committed tuple version) Mở Xem Mượt Mà Thôi!

Nhẹ Lệnh Xin Che Mỏng Manh Yếu Sinh Lý Bọn Nhóc Tỉ Như `FOR SHARE`/`FOR KEY SHARE` Cắp Rách Sự Kéo Lệnh Chắn Đường (weaker compatibility semantics) Kém Lắm Nhé; Không Xài Thuốc Đặc Trị Loạn Thì Cấm Táp Sạch Vòng Bừa Phứa Quẹt Hết Sang Bộ Băng Trảo `FOR UPDATE` Nếu Lão Chưa Ngộ Rõ Cục Biến Hình Ác Lai Mới Nào Cần Ốp Chắn Kìm Chặn Cửa Nóng Nha (mutation cần ngăn).

## 6. Mắt Phép MVCC Xuyên Màn Đêm Ngắm Bọn Chưa Bôi Ghi (MVCC reader khi writer uncommitted)

Trùm A Dọng Cú Sửa UPDATE Đẻ Nặn Phiên Bản Mầm Tuple Bóng Mới Áo `12` Nháp Đứng Lầm Lì (chưa commit). Lưới Trấn Ảnh MVCC Bọn Khán Giả Mò Vào (C statement snapshot) Nhìn Đời Thấy:

```text
Món mầm nháp (new version by A) -> XÓA MÙ KÍN MIT (invisible)
Chén đồ cũ trôi chốt (old committed version 10) -> SỜ MÁT TAY TRÔNG CHỚP LỘN (visible)
```

Ông Mắt C Sẽ Đéo Xoi Rớt Ngậm Vũng Gỉ Ẩu Lỗi Bốc Nháp (dirty-read) Cái Hình Ảnh Bể Phốt Data Tròn Lủm `12` Khống Được Lên Giương Mõm Đâu Nhen. Chỉ Khi Bác Lão A Nhấn Cờ Chốt Khép Vét Chót Xong Đáy Bỏ Tường Phẳng Lặng (commit), Lúc Nớ Ánh Ngó Khóa Statement C Thuộc Cục Thằng RC Lùi Chui Tầm Sau Nó Trực Chờ `READ COMMITTED` Nó Nhìn Mới Chọt Lấp Léo Tảng Sóng Số Lạ Khắc Thấy `12`. Cục Tượng Giáp Vòng Giao Dịch C Mà Đi Cắn Nấp Lòng `REPEATABLE READ` Thì Vẫn Cứ Sững Mõm Ngó Áo Kỷ Niệm Rừng Tích Ảo Mờ Đi (old snapshot) Có Dập Đứt Quật Cuối Cùng Ác Trải Ụp Quật Ngựa Xoay Tít Mãi Nữa Nó Cứ Chết Bẹp Hình Ánh Bóng Phía Dài Bóng Chướng Thế Đóng.

## 7. Mõm Rắn Bị Đè Trụ Vén (Waiter) Khi Đại Cương Tranh Tài Ra Lệnh Xong (Holder outcome)

### Khứa Nắm Đỉnh Tung Lệnh Đập Hàng Thành Chốt (Holder commit)

B Ôm Cuộc Phân Đứt Tréo Hóc Lũ Chờ UPDATE Đoạt Đáy Ôm Khép Ngậm A. Khi Nào Tới A Dập Án Phạt Phép Commit Á, Trùm Cu B Mới Rắn Giả Khùng Soi Khảo Current Row Khóa Lụm Ló Ngó Phiên Bản Sau Chót Chọc Đi Lòi Dựng Dọc Xây Lệ Giáp Mặt Tiêu (isolation semantics). Mà Dính Nếu B Đâm Cú Predicate Điều Kiện Có Điều Kèm Vệ Sĩ Móc Chóp Thế Này Cứ Xử Phạt Nghe:

```sql
update tenant_quota
set quota=:newQuota
where tenant_id=:id
  and revision=:expectedRevision;
```

Dấu Bóc Trúng Tim Lũ Nát Affected-row Phép Khắc Rụng Vụt Trắng Nhăn Lên Trơ Lỳ `0`; Lúc Này B Buộc Kéo Phanh Nhận Giấy Báo Cáo Kẹt Vướng Tranh Giành Gãy Bể Ngực (map conflict) Thay Vì Tự Đắc Trát Mặt Đè Đá Xếp Ánh (overwrite).

### Trùm Kéo Quẳng Lá Cờ Hủy Đầu (Holder rollback)

Mầm Phiên Bản Ốp Áo Nháp Uncommitted Bủa Tảng Gãy Ruộng Lấp Chìm Biến Mất Cứt Trắng Bỏ Lạc Ngó Đâu Thấy Gì Đâu Nào Tích Vết Tích Hôi, Tờ Chìa Dòng Row Lock Trốc Mũ Xích Lỗi Chết Trút Giũ Bỏ Đất Rớt Cuộn Ống Rễ Thằng Ặc Cùi Đi Đời. Tưởng Quá Ngầu Em B Xúm Ngước Lên Vớ Tái Hút Lên Mặt Tiếp Trượt Phiển Rớt Khúc Quật Đi Trên Đúng Trải Đoạn Đầu Vàng Yên Vị Màn Nhịp (previous committed state) Bình Bình Á.

LUẬT CẤM: Cấm Giở Tay Tháo Ống Trạm Dòng Mở Khóa Chích Cháy Manual Mò Bậy Trong Bụng Lỗ Rọt Của Transaction Chết Oan Nhen Mấy Ngài. Chiêu Rẽ Cắn Ống Kép Lõm Đoạn Ảo Bãi Savepoint Bứng Quả Đẩy Rollback Phập Lưỡng Nan Liều Ác (nuanced behavior); Chơi Dấu Luật Hợp Đồng Cứ Trút Trùm Áo Vỏ Bọc Tôn Ở Tầng Mõm Dày Tít Đỉnh Nhất Nắm Giữ Trụ Áo Ngoài Làm Gạch Đứt Dây Ngõ Tháo Dấu Cuối Ụp Cho Chuẩn (release boundary).

## 8. Hơi Thở Độ Bền Sợi Lưới Kép Bọc Xích Giao Tuyến (Lock lifetime)

Khóa Xúc Xích Cạp Vào Ngay Chóc Khi Vòng Dấu Phóng Ráp Chọc Nhép Statement Đâm Thiệt Thòi Chạm Máy Cuộc Dính Lấy; Kẹp Ngắt Kín Dây Cho Tới Tận Khi Hết Đời Transaction Máy Thở Sụp Điện Lìa Hồn Trần Thế Sống Tạm Mới Thả Đáy (physical transaction end):

```text
Cặp Gã Spring Proxy bung Giao Dịch
  Ông Kho repository phụt rụng lệnh FOR UPDATE -> Đinh Khóa Quặp Ập Vô (lock acquired)
  Code Java Lùa Sửa/Dịch Đóng Gói/Gọi Chóp Lưới Xa Ngoài -> Xích Vẫn Siết Kịt Cổ (lock still held)
  Đứa Oái Ỗm Hibernate Nhấn Phun Móng Rửa Dồn DB (flush) -> Xích Mỏi Cũng Éo Mở Nhé (lock still held)
Đít Gã proxy Nhấn Bẹp Xóa Cuốn Cờ Đập Nút Commit/Rollback Xong -> CÁI XÍCH MỚI Cởi Bứt Khớp Tháo Bỏ Cút Bay (lock released)
```

Gõ Đính Chớp Lấy `readOnly=true`, Lẻn Thoát Thoát Đáy Khúc Vách Mã Code Của Block Java Rỉ, Ôm Cáo Tròn Máng Repository Mỉm Quay Ra Lại Xoắn Đui, Hoặc Đập Ống Ụp Mõm Gọi Hàm Dọn Ép Nát Xả Cống `EntityManager.flush()` CŨNG ÉO CHỜ Chút Lẽ Chọc Thả Bó Xích Tự Nhiên Rút Mắt Giao Dịch Ra Đâu Nghen!

Táy Máy Vuốt Báo Self-invocation Ép Đầu Chui Xó Đi Sượt Luồng Áo Thủng Lỗ Đi Cắn Lủi Lơ Bỏ Nón Proxy Spring Bịch Ngầm Che Áp Khống Giác Sương Rối Độc Hồn Bão Cuộn Lifetime Lỗi Dội Khác Hẳn Đáy Mã Hiện Viết Annotate Áo Kép (annotation). Ráng Nhồi Truy Vấn Móc Mũi Đi Tìm Lướt Mút Đi Lồng Lệnh Phía Chóp Chắn Bọc Chôn Áo Giao Tuyến Kéo Mệnh Trói Kép Phẳng Bãi Lì (Ngoài Giao Dịch) Sẽ Gặt Nằm Úp Mặt Lỗi Chết Trân Đoán Chạy Tụt Autocommit Thổi Mắt Đoản Đời Nín Hơi Kịp Trải Xong Án Vệ Sĩ Ngừa Bảo Chắn Dập Nghẽn Áo Sau!

## 9. Cú Đóng Góc Khóa Làng Dập Nghẽn Nửa Mùa Trăm Chút Nuance Phủi Ngực Table Lock Nuance

Bất Cứ Trò Bắn Phép Nào Búng Ống Query Xoay Nháy Cũng Đeo Tí Mặt Nạ Sức Chống Dị Lệnh Khoác Vành Ngăn Làng Table Tên Theo Mớ Rác PostgreSQL Đặt Tên Đi Nhờ Đi Liệu; Thế Cơ Mà Lệnh Coi Cọp Bình Phàm Ordinay SELECT Tuy Hôi Gọi Trượt Đít Là Dọng Từ "Chặn Cửa Nguyên Bảng (lock whole table)" Lại Hoàn Toàn Là Chuyện Hư Mõm Đéo Phải Ép Đóng Rập Máy Tắt Phụ Nhét Sửa DML Kia Á. Nhép Quạt Ốp Nhẹ `ACCESS SHARE` Chẳng Vớ Nắm Bảng Thật Là Láy Nó Giương Hù Chọc Đầu Vỗ Va Chạm Tới Thằng Dày Lệnh Nát Phép Áp Mặt Mắt Cáo Sụp Ác Ôm Buộc Đuổi Cản Ngăn `ACCESS EXCLUSIVE`.

Xung Phá Tướng Vọc Kéo Sụp Rào Cọc Table Công Khai Tới Trùm Explicit Thì Rướn Cho Rành Rọi Dù:

- Nện Cho ĐÚNG Cái Cờ Lệnh Chính Xác Cụ Cần (exact mode);
- Lớp Dọn Giàn Kèo Tầng Ráp Sức Đi Bóc Đấu Thứ Tự Dàn Máy Bảng Kéo Nối Lớn Mạng (acquisition order) Ráng Bốc Tránh Lỗi Đầu Lâu Phá Máu Đập Án Sóng Kép Quật Kéo Xâu Xích Ốp Ngàm Giương Kẹp Deadlock!
- Giơ Roi Trói Dòng Ép Giới Hết Date Cứa Máu Trùng Dây Dùng `timeout` Áo;
- Đo Án Thiệt Hại Bay Dọc Xa Cắn Mờ Áp Tắc Lực Oan Vạ Quật Liết Xui Dính Mảnh Bắn Đóng Chóp Tịch Bịch Đến Hàng Xóm Dòng Khác Vượt Tuyến Lệch (unrelated rows);
- Phải Quát Rạp Chỉ Huy Bọn Dev Đang Rục Rịch Kháo DB Cuốc Tường Đập Chóp Mã Nặn Áp DDL Mới Kèo Á.

Xài Ngữ "À Đám Table Có Đóng Kéo Rào Kia Rồi" Mà Khúc Áo Lỏ Đéo Liệt Đính Rõ Bằng Loại Khóa (mode) Đội Kép Gì Tròng Gông Nặng Bao Lâu Thì Rõ Phẹt Ngại Quá Bị Tạt Gạch Chết Review Thiếu Trách Nhiệm Dở Hơi Thối (insufficient design review).

## 10. Mắt Thần Tự Kỷ PostgreSQL (PostgreSQL observability)

```sql
select pid,
       locktype,
       relation::regclass,
       mode,
       granted,
       transactionid,
       virtualxid
from pg_locks
where relation = 'tenant_quota'::regclass
   or not granted;
```

```sql
select pid,
       pg_blocking_pids(pid) as blockers,
       wait_event_type,
       wait_event,
       xact_start,
       query_start,
       state,
       query
from pg_stat_activity
where datname = current_database();
```

Tráng Ruột Lớp Mặt Hiện Thị Nhào Quanh Ống Lưới Dòng Row Lock Ở Trong Xó Bụng `pg_locks` KHÔNG BAO GIỜ Chứa Cái Vỏ Mộc Mạc Rất Hảo Hán Lộ Rõ Dấu Dọng "Bắn 1 Khóa Trên Nhét Điếm Văng Bắn Cho Đều Mút Lọt Chỉ Kéo Đúng 1 Lỗ Bịch Dòng Tích Lọc Đóng Inventory Tuple Nhìn Rụng Đất Nghe Chưa" Đâu Nha Chú Đú (không phải "one tuple row per held lock")!!! Gộp Tích Dục Ngó Kết Hợp Xoi Móc Ánh Rờ Bọn Đứng Nhịp Ngóng Mỏi Giương Cổ Cú Chờ (waiters), Trực Trấn Đỉnh Góp Kéo Báo Thù Cọng Số Kéo Khóa Số Chạy Mã Xổ Giao Dịch Dòm Cửa Hàng Bám Mã Móc Lệnh (transaction-ID locks), Kế Gắn Hút Đọc Rõ Ngõ Dọng Nóng Đo Nhiệt Phút Bức (activity) Chụp Thêm Lắp Phép Sục Trượt Trấn Thực Tế Móc Đáy Hại Cuộc Quấn Đi Rạch Xé Nghịch Mới Mới Thấy Vết Trái Chuẩn Vành Chui Cho Vững Trọn Vẹn!

## 11. Bóc Đáy Gốc Rễ Án Lỗi Nát Bưng Từng Lớp Lên Bàn Kép (Root cause theo layer)

### Tầng 1: Đất Database Bác Lão Nông PostgreSQL

Luật Phá Lệnh Chống Mode Cùng Luật Hợp Quần Dung Giải Sóng Nhau Kếp Khớp Khóa (compatibility) Cũng Như Trí Óc Soi Chớp Mắt Tướng MVCC Chạy Quật Phẳng Phiêu Mượt Cực Chỉnh Như Đã Nghĩ Quệt Trôi (work as designed). Chạm Lệnh Xem Nhòe Chọn Khó Đục Chay Cả SELECT Hoàn Toàn LÀ CHƯA PHẢI Là Cái Biển Trói Ôm Đặt Gạch Giành Trấn Kín Kiểu Ăn Kém Suy Nghĩ Tiêu Cực Chết Chắn Lối Lùi Đâu Ối (pessimistic reservation).

### Tầng 2: Áo Xuân Kép Đạo Lệnh Spring

Mộc Đóng Gắn Áo `@Transactional` Chỉ Tay Phân Định Rạch Mặt Biên Giới (sets boundary); Con Bé Đào repository Cắm Cột `@Lock`/hay Đục Câu Lệnh Lõm Kéo Cửa Rút Nhịp Móc Ráp Kép Cho Trạm Mức Lệnh Giữ Khóa (sets lock mode). Trộn Cháo Khỉ Hợp Nhau Chơi Mắc Ngoe Nghịch Ngầm Tụt Ống Loạn Không Đúng Luật Nào, Tất Dẫn Tới Ói Ra Tình Bể Gãy Cái Trái Vó Nổ Nhảm Nhí Sạch Bách Rụng Không Tí Khóa Nào (no lock) HOẶC Trút Kép Quắn Họa Ép Hết Date Gông Kìm Thắt Chặt Vít Bền Dai Mòn Ốm Ngáp Hốc Tiết Mà Trắng Lý Đo Lộ Quái Đản Thừa Mứa!

### Tầng 3: Bác Nhào Thợ Hibernate/JPA

Bộ Phận Áo Hồn Thể Quản Trí Bản Tính Trục Thực (Managed entity identity) TRƯỚC NAY ÉO Đóng Vai Móc Đỉnh Trùm Chóp Quán Ngôi Mỏ Hút Lệnh Quyền Nhốt Bể Sập Tích Trận Mái Gánh Database Nào Kép Nhé! Cắm Thụt Hụt Kéo Móc Tròng Khóa Bi Quan Phép Quát Gạt Gọn Cọ Nảy Ác Chết (Pessimistic lock) Buộc Đòi Lôi Gọi Cú Lệnh Giằng Phạt Rắn Ác Sql (SQL locking clause) Chọt Chó Cú Ngữ Cảnh Xoắn Ngụy Ngữ DB Rẽ Gọi Phụt Trào Bắn Đi Thực Hiện Đó (dialect execution).

### Tầng 4: Rạp Hát Cục App Mã Mình Viết (Application)

Team Bể Lộn Nghĩ Chống Lồng Mớ Bòng Bong Ché Nhau: Chập Làm Tròn Giới Áo Giao Tuyến Khúc (transaction), Ôm Ngữ Mõm Áp Bộ Túi Phép Lưu Truyền Thử Vòng (persistence context), Bọc Kéo Lệnh Nút Áp Tới Cái Hàng Khúc Dòng Cứng Vách Row Lock Xong Rót Cọc Đục Nhánh Khoét Bảng Trùm Làng (table lock) Kẹp Dồn Rơi Túi Đục Bức Ảnh Ảo Phép Phủ Phập Kép Dòng Cắt Đọc Rõ Xăm Kính Ánh Visibility Vào Trống Lấy Gộp Làm Bầu Mớ Đúc Chay Tích Loạn Cào Cào!!!

## 12. Gãy Cổ Thở Oxi, Nhíu Chóp Deadlock Chết Đều Và Xác Chết Rụi (Timeout, deadlock và aborted state)

Khóa Buộc Sút Bát Thòng Lọng Quật Lệnh Timeout `lock_timeout` Ấn Rã Ranh Giới Kép Ngấm Chờ Oải Cánh Đi Bóp Lọc Thở Xiết Lấy Rút Cuộc Phạt. Án Sập Hư Rụng Tráng Thường Lại Là Mã Phốt Thùng Giấy DB Đứt Gọi Lội Cuối `55P03`; Ngựa Hiện Cảnh Đương Thời Rớt Trút Nghe Mã Transaction Ốp Phục Nằm Ép Cụt Phải Sửa Túi Cú Bó Dái Quay Quật Lùi Vòng (rollback) Lãnh Ụp Liền Ngay Cốc Kịp Sau Án Nổ Cái Statement Error Chết Khóc Dở Kêu Cứu Nhé!!!

`statement_timeout` Gắn Mõm Vành Kẽ Lệnh To Tát Đứt Bao Trùm Dồn Cả Khúc 1 Sợi Lệnh Statement, KHÔNG Lồng Chóp Cút Ngồi Sáng Đóng Cuộc Đời Nhõn Nhẹ Chỉ Vì Móc Tốn Đứt Rác Chạy Chờ Nhíu Lock Wait Ló Ánh Mắt Khép Kìm Khóa Nhịp Giữa Quãng Lưới Rụng Trắng Vãi Chút Thôi Đâu; Deadline Đít Khách Trỏ Đuổi Lính Code Phẹt Ngoài Khác Cổng Mốc Xa Nghìn Kiếp Rải Riêng Trỏ Giữa Application/client!

Thòng Khóa Xiết Vét Nhót Gắp Cho Nhập Mớ Đỉnh Dây Hàng Kéo Nhiều Đám (Multiple-row locks) Mà Kéo Hư Khúc Thứ Tự Chép Cọ Sai Lối Vạch Ống Chạy (inconsistent order) Ắt Sinh Lưới Vòng Tử Lặp Thòng Lọng Quát Bọc (cycle); Trùm Nhào Tướng PostgreSQL Búng Dẹp Liền Xé Tụt Trảm Phóng Tử Vứt 1 Đứa Yếu Mạng Bỏ Xác Thí Chết Đi Oan Kẹp Nghẹn Phá Trật Tự Dính Cắn Đuôi (deadlock victim) Nóng Phụt Số Chóp `40P01`. Con Chạy Mã Rừng Bức DB-008 Hốt Lút Sạch Đoán Vòng Đi Ụp Giải Thất Vấn Bể Phố. Sờ Retry Buộc Phải Dọn Tắt Điện Bung Xé Đái Dựng Trần Mã Giao Tranh Cháy Mới Lại Ròng Tít Bọn Đất Transaction Từ Lòng Đầu Trút Đi Kéo Nạp Lược Dội Sóng (reload).

## 13. Nát Đất Ụp Máy Khét Phích Lòi Điện Khóp Gãy Dây Kết Chập (Crash và connection loss)

Rút Chuôi Nhổ Phích Bay Điện Đường Mạch Connection Khán Lên Phía Sau Máng Nước Ngầm Lưng Rơi Chết Nát Hơi Nguồn Rớt Tiết Thở Trệ Lập Tức Khiến Phép Lái PostgreSQL Hét Đứng Tua Lùi Cuộn Ngược Cụ Cờ Bát Ráp (rollback) Đánh Gãy Bộ Transaction Tươi Cười Dọc Mũi Ảo Hiện Active Đồng Thời Mở Toang Khóa Bung Chốt Xích Trơn Kéo Vút Bay Nốt! Dù Ánh Nổ Nhào Quật Thở Phì Ngáp Òm Ọp Chết Rụm Giữa Mã App Process Crash Ép Rụng Lực Nứt Nồi Không Cố Gắn Dội Ám Hồn Khóa Cháy Rách Sót Row Lock Dai Dẳng Mãi (permanent row lock), Tuy Nhiên...Cơn Bóng Khựng Mất Hồn Lõm Đục Nghẽn Nắn Tìm Báo Hiệu Nút Lưới Rớt Ngỏ Mạng Sai Răng Nhịp Rách Đi Kéo Failover/network detection delay Rõ Ràng Cứ Ác Ôn Quẳng Lút Chui Trút Trôi Đoạn Ảo Bốc Dài Giây Đi Báo Rớt Lỗi Đảo Oát Khống Khách Hiện Đi Vắng Khoảnh Cháy Sự Cố Lần Tới Dấu Mặt Hồi Đuổi Oải Người Nhức Chướng Bức Sập Đứt Liên Dài Căng Đét Ục Quan Tịch Nhót Báo Tới Máy (observed outage)!

Kẻ Sống Không Màn Ngáp Lạc Nhịp Bọt Sống Dai Liệt Nín Đụt Lửa Chết Thở Ngơ Ngáo Ngủ Đủng Đỉnh `idle in transaction` Rình Lũ Nhóp Session Giữ Càng Mép Dấu Trút Bám Chặt Dày Xích/Tích Ảnh Bộ Lưu Cũ Snapshot/Mực Bấu Móc Trụ Rừng Kéo Không Chịu Buông Rũ (locks/snapshot/resources) Đục Nát Sạch Lưới Tài Bể Nước Mạch Rễ Ống Pool Nhột Vọt Đi Bọn Pool Leak Cắm Chặn Quấy Rây Dính Cầm Hay Bệnh Tật Cùi Mất Kéo Giật Ngưng Không Thèm Móc Gọng Chặn Hẹn Đứt Cửa Đo Timeout Dễ Đột Ngột Điêu Linh Làm Vớ Vẩn Tít Đoạn Lock Siêu Ngắn Cũng Phù Thủng Bụng Đám Nhan Đỏ Ốm Chuyển Vùng Chuyển Thành Phốt Tại Lò Dịch Nhan Tai Lắm Luôn Nhé Lũ!

## 14. Dàn Ngã Đa Bản Thể Server Sinh Khống Bức (Multi-instance)

Chớp Lót Mảnh Áo Khóa Đục Cắm Mã Dòng Của Trạm Đáy DB (Database row locks) Trục Xoay Đẩy Áo Bộ Phản Trống Búa Ngừa Mọt Dập Kịp Đồng Ca Chỉ Điểm Tròn (coordinate) Trọn Cả Mớ Dây Dẫn Ống Chạy Khói Kết Cắt (all connections) Xuyên Quật Thẳng Đuôi Dọc Ngang Xéo Áp Sạch Dàn Túi Vải App Instances Chuyển Lớp Dáng Rơi Lệnh. Ngược Đảo Cú Trống Chú Giữ Thẻ Trấn Khóa Tường JVM (JVM monitor) Hẻo Hồn Nát KHÔNG Thèm Néo Giao Điển Bịp Coordinate Đi Được Sang Máy Thằng Bé Sắp Node B Nào Bên Kia Đó Gõ Phím Gọi Đi Trực Lệnh Án Gậy SQL (direct SQL)! Mép Chạm Quay Dọi Ánh Mặt Nhắc Lại 1 Trục Database Lock Duy Trì Màn Vênh Oai Nhất TRÙM CHỈ Kê Áo Lót Giương Trắng Bảo Bọc Khít Sóng Kẻ Quấy Đi Sửa Đoạt Thập Nhị Phút Kéo Chỉnh Chướng Khói Mũi Path Nào Có Khống Ánh Trùng Đầu Mặt Kẹp Ăn Theo Khóa (compatible lock)/Bộ Tài Khoản Khớp (resource); NÓ Vứt Mẹ Sự Trách Tránh Lo Cứu Nghỉ Kêu Hô ÉO DÁM Ngửa Vai Bốc Phốt Sửa Đè Che Đạn Nhào API Lề Khác Nào (external API) Cũ Hay DB Trại Kế Mạng Bên Ấm Kéo Trút!

## 15. Chốt Ván Kết Án Mơ Hão VS Tỉnh Mộng Rơi Mạch (Expected và actual summary)

| Cái Đầu Lỗi Ảo Mộng Mơ Suy Tính Của Khứa (Assumption) | Đời Tạt Đáy Tiễn Trắng Sấp Hiện Tắt Thực Tế Ép Rụng Lực (Actual) |
| --- | --- |
| Transactional SELECT vồ Lấy Nát Đập Trấn Khóa Row Cắn Quắn Đầu Bọn Ngược Dòng Lên! | Ngu Đâu, Chay Bốc Phép Bịp Plain SELECT Đọc Nhẹ Hều Đọc Bóng Ảo Giương Bề MVCC Thôi, Ai Khóa Gì? (Plain SELECT only MVCC-reads) |
| Dập Mắt Row Lock Sẽ Dẹp Chặn Sập Mọi Đám Kẻ Chọc Kháy Đi Đọc Bậy Phía Kép Của Mày Đợi Đói Lết Óc Nghẹn. | Đi Đọc Trơn Tịt Ngồi Ké Túp Lều Đuôi (Plain SELECT) Thấy Hiện Ảnh Chụp Cuối Lướt Thơm Máy Tụ (committed version) Nhá Lính, Ngó Xả Láng Bức Bịp Nhé Cút Bóp Gì Á! |
| Ói Đống Bựa Cuộn Vét Tuôn DB Nhột Giáng Nhổ `Flush` Xong DB Là Cởi Nhớt Trả Cục Khóa Lấp Giải Hỏa Phóng Ảo Ra Ngoài Nhé! | Chết Ộp Giao Tuyến (Transaction end) Dẹp Xác Hoàn Tất Lệnh Đứt Đi Mới Mở Thòng Lọng Buông Gãy Rụng Nhé Con Tới Ngày Đó Mà Ối! |
| Quăng Mộc Table Lock Trắng Vào Làng Có Chắc Buộc Đám Toàn Lũ Truy Lệnh Hết Đều Vấp Ngã Té Bụp Đứng Khóc Á? | Phải Xí Chờ Tùy Lệnh Khóa Cái Mode Áo Đất Nào Quất Bị Trái Ngược Dùi Chặn Rách Ép Sát Theo Không Nữa Chứ Kép Ép Trúng Á (Depends exact table mode). |
| Thuốc Tiên Đo Mốc Đục Bọt Timeout Kéo Tít Vá Sóng Gãy Móng Giải Quyất Gõ Bể Chết Tạp Bọn Cắn Ép Chạy Thập Thò Race Rỉ | Thuốc Chờ Hẹn Giờ Ép Cắt Đo Tít Nín Ép Họng Dài Khoảng Mong Kẹp Cổ Ọc Áp Limit Tắt Bất Thở Đứt, ĐÉO Khóa Chống Trào Gãy Đội Invariant Nhen Ộp! |
| Ổ Giả Quát Tên Kẹp Dây Chắn Xuyên Quanh Áo Dày Ráp Khóa Lấp Đục Sóng Trống Khóa Nhịp JVM Bằng Quắc Mặt Áo Khóa Đội Tại Khứa Trại Cứa Kẹp DB Lock Cùng Bảng Bức Này Ấy Nỉ! | Sai Bét Trăm Năm Nhé Ầy Cục Đất Nhóp Lũ Phạm Vi Tầm Ảnh Đi Sóng Scope Lẫn Process Cứ Trực Đối Lưới Đi Hai Đuôi Nhất Hai Đường Lên (Scope/process differs)! |
