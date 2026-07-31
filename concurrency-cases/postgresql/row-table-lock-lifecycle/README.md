# Bệnh DB-007 — Vòng Đời Trớ Trêu Của Row Lock, Table Lock Khóa Rắn Khóa Buông (DB-007 — Row lock, table lock và lock lifetime)

## 1. Tóm Tắt Chuyện Gì Đã Xảy Ra (Tóm tắt)

Một Khách Hàng (Tenant) mã `T-42` đang có mức hạn ngạch (quota) được chốt cứng là `10`. Một Ông Admin A mở cửa Giao Dịch ra, chạy câu lệnh ngó `SELECT` chay bình thường, rồi đứng lỳ ở đó mải mê chạy vòng xác thực ở máy của mình (local validation). Cả Team Dev đều ngây thơ đinh ninh rằng: "Cú SELECT đó chắc đã lấy tay KHÓA CHẶT DÒNG ĐÓ LẠI (row lock) rồi, Ông Admin B tới sau ắt sẽ phải đứng xếp hàng chờ!". Đời đâu như mơ, ông B tỉnh queo đâm lút cán lệnh `UPDATE` sửa thẳng quota thành `8` rồi Chốt Sổ (commit) chạy thoát ngay lập tức!

Lúc phát giác ra, mấy anh tài trong team liền hốt hoảng đắp thêm chữ `FOR UPDATE` vào. Rồi lại tưởng bở rằng: "Kỳ này thì bố đứa nào xài lệnh đọc cũng bị chặn đứng cổ!". Nhưng sự thật phũ phàng lại hiện ra khi A đang bấm chặt cái khóa row lock và tay còn đang cầm khư khư mức quota nháp là `12` (chưa commit):

- Bất cứ thằng B nào muốn Bơm Dữ Liệu Chống Lên (competing writer) hay Bốc Bùa Khóa Y Chang B (locking reader) ĐỀU phải ngoan ngoãn ngậm mỏ chờ đợi đến khi gãy cổ đứt (timeout);
- Mấy Cái Bảng Quản Trị C (Dashboard C) chỉ dùng mắt soi chay `SELECT` vẫn thản nhiên lao qua ĐÉO chờ một giây, và Mắt Nó Thấy con số nháp kia Không? KHÔNG! Nó thấy đúng Tờ Ảnh Cũ đã Xong Xuôi Mượt Mà (old committed quota) là `8`;
- Bộ Gông Khóa CHỈ Bị Vứt Chìa Đi (release) ngay lúc A dập gõ ván bài Commit/Rollback xuống, CHỨ KHÔNG PHẢI lúc kết thúc thoát vòng chạy cái hàm (repository method return) như mấy bé lầm tưởng!

Luật Bất Thành Văn Ép Tuân Thủ (Invariant/contract):

```text
Cấm Vọng Động: Đã muốn sửa quota theo kiểu "Dựa Trên Tình Trạng Hiện Tại (current state)", thì bắt buộc phải Nắm Được Khóa Thật Sự Độc Quyền Xung Đột (authoritative conflict) rồi hẵng phán quyết! Và phải KẸP NÁCH CÁI KHÓA ĐÓ tới tận lúc tung cờ trắng hay phất cờ chiến (commit/rollback) cuối cùng!

Mấy ông quan sát ngoài đường (Plain observers) thì chỉ được nhìn Vạn Vật Dưới Góc Ảnh Chốt Chết MVCC mà thôi (committed MVCC state).
TUYỆT ĐỐI không có cái khái niệm Ngáo Đá Nào Cho Rằng "Hễ Tóm Được Khóa (row lock) là có Quyền Ngó Số Chưa Chốt Nháp (uncommitted data)" Của Người Ta Đâu Nhé!
```

> **Sếp chốt lại:** Trấn Khóa Từng Dòng (Row lock) Sinh Ra là để Buộc Mấy Kẻ Có Tâm Địa Sửa Tranh Nhau (conflicting mutations) PHẢI Nhượng Bộ Xếp Hàng! Nó KHÔNG BAO GIỜ có mục đích biến nguyên cái PostgreSQL thành Bãi Mìn Rằng "Một Hằng Ghi Đang Khóa Thì TẤT CẢ Lũ Ngó Trông Bất Kể Sống Chết Đều Phải Hóa Đá!".

## 2. Dàn Diễn Viên & Tình Cảnh Xài Chung (Actors và shared state)

| Gương Mặt Vàng | Vai Diễn Sinh Tử |
| --- | --- |
| Ông Kẹ Admin A | Mò Đọc Xong Nặn Vặn Số Quota giữa chừng vòng Giao Dịch |
| Cu Trẻ Admin B | Kẻ Chen Đâm Hông Sửa Số (competing writer) Lẫn Kẻ Rình Khóa (locking reader) |
| Cậu Bảng Dashboard C | Khán Giả Chỉ Đứng Coi Dạo Hiện Thực Chốt Sổ (plain committed-state observer) |
| `tenant_quota` | Dòng Cầm Phán Quyết Độc Tôn Hạng Mức |
| Anh Cảnh Sát PostgreSQL MVCC | Người Lấy Bức Ảnh Chốt Đã Lưu Cho Khán Giả C Xài Đỡ Ghiền |
| Áo Choàng Spring Transaction | Người Quyết Vòng Đời Tắt Thở Cho Cái Ổ Khóa (lock lifetime) |

Hiện trạng mồi lúc chưa cháy (Initial committed row):

```text
tenant_id = T-42
quota     = 10
revision  = 5
```

## 3. Ranh Giới Giao Dịch Bắt Ngợp Lên Mũi (Transaction boundaries)

Cái Trí Tưởng Tượng Chết Người Đổ Bể (Broken assumption):

```text
Ông A: BẮT ĐẦU (BEGIN) -> Ngó Chay (SELECT quota=10) -> Thẩm định -> LÊN LỆNH UPDATE
Ông B: BẮT ĐẦU (BEGIN) -> CHÉM LỆNH UPDATE Cùng Đúng 1 Chỗ Dòng Đó -> (Cả Team Mơ Rằng B Đang Khóc Chờ Đợi)
```

Hiện Thực Ăn Tát Kêu Vang (Actual): Lệnh dòm của ông A CÓ GIỮ CÁI KHÓA MẸ NÀO ĐÂU; Thằng B rảnh rang thong dong phóng Cú Update Và Ép Sổ Chốt Nhanh Rụng Nụ Cười!

Ép Cọc Khóa Kẹp Bạo Hành (Explicit locking):

```text
Ông A: BẮT ĐẦU (BEGIN) -> NGÓ XONG XÍ CHỖ KHÓA `SELECT ... FOR UPDATE` -> Dọng UPDATE quota=12 -> Mở Nhạc Chờ -> COMMIT
Ông B: Vác Lệnh UPDATE/FOR UPDATE Ngắm Trúng Dòng Của A -> Phải Đứng Úp Mặt Vô Tường Chờ/Nổ Hạn Chết Mòn (waits/fails)
Thím C: Ngó Chay `plain SELECT` Vào Lỗ Dòng Đó -> Vẫn Cứ Là Lấy Về Tờ Ảnh Chốt Số Cũ (old committed version)
```

Nút Nghẹn Cổ Chai Ác Chiến Nhất (Contention point) Ở Lại Cái Khe `tenant_quota(T-42)`. Dây Khóa Chỉ Khởi Bắt Được Đầu Tròng (Lock acquire) Vào Đúng Thời Khắc Cú Lệnh Ràng Buộc Khóa/Lệnh Sửa (locking statement/update) Được Phát Kêu Lên! Và NÓ Mở Van Tụt Dây (release) Ở Trạm Bến Cuối Về Đích Đóng Đứt Vòng Transaction Vật Lý (physical transaction end).

## 4. Bản Đồ Rút Gọn Sinh Mệnh Dây Khóa (Lock matrix rút gọn cho case)

| Lúc A Đang Cầm Trịch Gì Kìa? | Cu B Rởn Mỡ Kéo Trò Gì? | Kết Cục Máu Mủ (Expected behavior) |
| --- | --- | --- |
| Éo Có Chút Khóa Row Nào Sất, Cứ Ngó Chay (plain SELECT) Thôi | Phóng UPDATE | B Luồn Mịn Màng Lách Xuyên Lên (B tiếp tục) |
| Kẹp Nách Vòng Kim Cô `FOR UPDATE` (row lock) | Phóng Lệnh UPDATE Cùng Chỗ Dòng A Kẹp | B Phải Hóc Chờ Hoặc Gãy Cánh Đứt Xích Tiêu (waits/timeout) |
| Kẹp Nách Vòng Kim Cô `FOR UPDATE` | Cũng Ra Bài Khóa `FOR UPDATE` Dọc Mặt | B Lại Hóc Chờ Kéo Đứt Thở Nghẽn Cổ (waits/timeout) |
| Đang Kẹp Rét Chén Khóa `FOR UPDATE` Độc Cầm | Chơi Màn Ngó Chay `plain SELECT` Thử | Cu C Luồn Lách Đi Đọc Cái Ảnh Kỷ Niệm Cũ Chưa Sửa! (committed version) |
| Dập Luôn Thẻ Xích Trọn Đỉnh Table Bằng `ACCESS EXCLUSIVE` | Cũng Cố Chen Ngó Chay `plain SELECT` | Chết Trân Liền! Thím C KHÔNG LỌT ĐƯỢC CHÚT ẢNH NÀO MÀ CŨNG PHẢI XẾP HÀNG Ọc (waits/timeout) |

Chú Lũ Đọc Xong Nhớ Kỹ Nhé! Bảng Này Rải Chỉ Tiết Các Đoạn Trường Cho Chuyên Mục Án Này Thôi Nhé! DB Bác PostgreSQL Nhà Ta Đầy Mưu Chước Rằng Trăm Kiểu Row/Table Lock Khác Bọc Nhau Trùng Điệp (nhiều row/table lock modes khác).

## 5. Sổ Tay Nhập Môn Để Tránh Chết Oan (Thuật ngữ cần biết)

| Chữ Nghĩa | Giải Nghĩa Tiếng Người |
| --- | --- |
| Khóa Nút Họng Một Dòng (row-level lock) | Còng Chân Cột Chặt Nhau Buộc Xếp Hàng Thay Đổi (mutate) Dính Lên Đúng 1 Hàng Vị Trí Nhất Định. |
| Mức Đàn Áp Đè Bảng (table-level lock mode) | Áo Khoác Chắn Quan Hệ Bọc Ngoại Diện (Relation lock mode) Tự Quấn Tròng Lên Rộng Khắp Nhờ Cú Lệnh Sút (statement). |
| Trùm Nắm Cán (lock holder) | Đứa Nào Đang Nhét Trong Túi Quả Chốt Khóa (transaction) Mắc Ngoe Nghịch Dấu Mà Kẻ Khác Mắc Xung Đột Phải Nghẹn Đứng (incompatible lock). |
| Kẻ Ngửi Khói Chờ (waiter) | Trâu Chậm Đành Đứng Cổ Nín Chờ Đứa Nắm Trùm Đóng Nắp Rút Thẻ Trả (lock holder kết thúc). |
| Lệnh Dòm Ngó Trơn Đuột (plain SELECT) | Tò Tò Trích Đọc Kiểu Bốc Tấm Ảnh MVCC MÀ KHÔNG Quàng Cái Vòng Câu Chữ Ép Khóa Chốt Góc (locking clause) Nào Thấy. |
| Phép Ấn Chú Giữ Nhà Khống `FOR UPDATE` | Tròng Sợi Xích Kẹp Mỏ Độc Ác Đi Đọc Lấy Áo Xích Độc Tôn (exclusive) Hợp Mệnh Khớp Dọc Hàng Nút Bịt (row-level intent). |
| Vòng Hơi Đời Dây Xích (lock lifetime) | Khúc Độ Dài Bắt Đầu Quấn Lấy Nhặt Nắm Vào Tới Đoạn Kéo Phạch Còi Kết Cục Buông Thả Bỏ (acquisition tới commit/rollback). |
| Phút Nghẹt Tràn Hết Mức `lock_timeout` | Nước Bờ Trống Quá Chịu Sút Đứt Không Có Lỳ Đứng Ôm Cổ Chờ Đứt Khóa (giới hạn thời gian chờ database lock). |
| Án Chặn Hóa Đá Nát Table Bằng `ACCESS EXCLUSIVE` | Cuộc Cờ Buộc Khóa Rộng Hoàn Toàn Bàn Đập Cả Cảnh Ngó Thường Trơn (plain SELECT) Rũ Xương Chết! |

## 6. Hành Trình Tham Khảo Bản Đồ Sống Còn (Điều hướng)

- [Trí Tưởng Tượng Chết Chìm Về SELECT-lock (Broken SELECT-lock assumptions)](broken-code.md)
- [Bóc Băng Tua Nhanh Lúc Bốc, Kẹp Lẫn Đứt Dây Khóa Và Ảo Thị Hiện Hình (Lock acquisition, visibility and release analysis)](analysis.md)
- [Đơn Thuốc Chữa Theo Pessimistic, Atomic Cùng Optimistic Áp Cho Kịp (Pessimistic, atomic and optimistic solutions)](solutions.md)
- [Vào Đấu Trường Thử Lửa Ép Phun Lỗi Giao Tuyến Chốt PostgreSQL Khóa Lồng (Deterministic PostgreSQL lock experiments)](experiments.md)
- [Võ Lâm Xài Pessimistic Locking](../../concepts/pessimistic-locking.md)
- [Đồ Tể Nhát Khóa PostgreSQL](../../concepts/postgresql-locks.md)
- [Gia Phả MVCC Của PostgreSQL](../../concepts/postgresql-mvcc.md)
- [Biết Code Concurrent Thì Test Nó Đi!](../../concepts/concurrency-testing.md)

## 7. Sập Hầm Chết Cười Ngoài Thực Tế (Hậu quả production)

- Mấy Tay Viết Tạc Đè Sài (Writer) Rớ Trúng Phải Bóng Data Mốc Thiu (stale state) Vì Dòng Mò Mắt Đọc Thường Cháy `plain SELECT` Rõ Kém, Ép Khách Nhét Sập Bầu Quẳng Cứt (reserve row).
- Team Xúm Kéo Hộ Xây Bức Khóa Trắng Phanh Ảo Lỗi JVM Lên Nhưng Vác Ra Nhiều Quán Trọ Trạm Rẽ (multiple nodes) Bức Vỡ Đè Chửi Nhau Như Thường.
- Có Ôm Cục Kéo Dây Khóa Cháy `FOR UPDATE` Rút Chặt RỒI DÁM LƯỢN RẼ Dạo Remote I/O (Lên Cõi Gọi Mạng Táp API Ngoài), Là Gãy Ốp Cháy Nghẹn Chùm Đuôi (wait queue/pool exhaustion).
- Bọn Cày Bảng Soi Dashboard Mất Não Tắc Cổ Buộc Đợi Hàng Trống Vắng Lầm Bóp Khóa Nhá (serialize không cần thiết) Do Rập Phản Xạ Đánh Rắm Kẹp Đi Cọc Khóa Á (locking read).
- Nghẹn Chờ Giết Trục Sập Khóa `timeout` Mà Kẻ Trực Ống Ỉm Áo Gọi Dội "Gia Thế Kẻ Này Báo Row Không Khắc Mọc Đâu" Oan Uổng!
- Đoạn Áo Khép Sổ Mở Dấu Ràng Nhầm Ranh Giới Hạn Cứu (Commit/rollback boundary) Tệ Tạo Lệnh Treo Khóa Lỳ Ngâm Quá Date Ám Code Lính Điểm Chết Dở Trò Tụt Code Reviewer Lầm Hiểu Nổi Chút Cho Coi.
- Gắn Khóa Chốt Vướng Vít Bàn Lớn Cọc Cao Table Làm Cả Lũ Khách Khác Ở Ngôi Làng Kế Cạnh Tịch Áp Ăn Oan Vạ Lây Quá Đáng Hét (unrelated tenants bị block).

## 8. Cẩm Nang Cứu Mạng Xử Lý Ổ Rác Khuyên Dùng (Hướng sửa khuyến nghị)

- Nghe Nè Writer: Nhớ Chốt Đi Xào Cờ Giăng Bọc Vỏ Lỗi Đọc-Sửa-Ghi (read-modify-write serialization): Phóng Áo `PESSIMISTIC_WRITE` (Hibernate) Kẹp Sát Thùng `FOR UPDATE` Cuộc Cháy Liền Tròn Tròn Liếc Sống Ngắn (short transaction), HOẶC Cọ Đạp Gọn Bằng Môn Atomic Giáp Thập Quyền Chéo Ghi Lệnh SQL (atomic conditional SQL) Khi Mà Khéo Chống Đỡ Kịp Đi Nói Được Rạch Ý Nghĩa Nè (diễn đạt được).
- Khán Giả Đi Chơi Đọc Kéo: Đâm Vét Cái Trơn Tuột `plain SELECT` Đều Tay Và Cam Kết Ăn Dữ Liệu Bụng Đái Cũ (committed-staleness contract).
- Rà Phá Tranh Nhau Rối Rắm KHÔNG NHẤT THIẾT PHẢI Cản Chặn Khóa Bóp Sập Nhau Đâu (blocking): Bơm Cái Bùa Lạc Ấn `@Version` Áp Chặn Khi Tình Trạng Hủy Lùi Chọc Retry/Reject Làm Tròn Vị Lắm Đó Nha.
- Chỉ Khi Bứng Cây To Đập Nền Bàn Cả Xóm Rụng Lớn Rộng Vết (Whole-table/schema operation) Mới Bấu Vụng Khóa Đỉnh Table (explicit table lock mode).
- Ép Châm Luôn Có Trói Ngự Chặn Bứt Quật Sút Lủng Trán `bounded timeout`, Rải Sập Hàng Nhịp Kế Cứ Đi Bộ Tươi Mượt Không Lì Deterministic Tách Lệnh Chữa Cháy Multi-Row Lẫn Đồ Đo Mức Hại Nhé!

## 9. Khoanh Vùng Vấn Đề Này Sâu Tới Đâu (Phạm vi)

Vực Bẫy Trọng Án Này Soi Rọi Đúng Đít Bọc Cái Mỏ Cơ Chế Và Nhịp Thở Sống Cuộc Vòng Đời Trói Cọc Gắn Lưới Khóa Nhen. Bốc Mút Lót Đít Ôm Gắn Gọn Rập Việc Nào Đó Dùng Tay Cắp Tán Xí Khung Nhảy `SKIP LOCKED` Sang DB-010 Mà Hóng Nhé; Rừng Nhặt Hình Khóa Xếp Chọn Pattern Lách Bước Vô Sâu `LOCK-003`; Còn Án Giao Dịch Chỏi Mũi Kẹp Nghẽn Dao Deadlock Lên Mâm Dọc Chéo Đối Phóng Găm Nát Cổ Thì Lộn Trại Trút `DB-008` Ghi Sạch Nhé Hén!
