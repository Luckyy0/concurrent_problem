# Soi Kính Lúp: Vòng Lặp Kẹt Xe, Kẻ Chết Thay và Quá Trình Dọn Rác (Phân tích chu trình chờ, victim và rollback)

## 1. Bối Cảnh Ban Đầu (Trạng thái ban đầu)

Tưởng tượng có hai cái ví đã chốt sổ êm xuôi:

```text
Ví 101 (A): Số dư = 1_000
Ví 202 (B): Số dư = 1_000
```

Thằng T1 bắt đầu hô biến `transfer(101, 202, 100)`. Cùng lúc đó, thằng T2 cũng gào lên `transfer(202, 101, 70)`.
Mỗi đứa chui qua một cái Cổng uỷ quyền (Spring proxy), tự mở một Giao dịch riêng (transaction) ở chế độ `READ COMMITTED` trên hai Đường truyền (physical connection) hoàn toàn độc lập.

## 2. Diễn Biến Án Mạng Kẹt Xe Dưới Đáy PostgreSQL (Timeline tạo deadlock trong PostgreSQL)

| Bước | Lính T1 — A chuyển B | Lính T2 — B chuyển A | Lưới Chờ Kẹt Xe (Wait-for graph) |
| ---: | --- | --- | --- |
| 1 | Hô `BEGIN` | Hô `BEGIN` | Trống trơn, chưa ai đợi ai |
| 2 | Chộp khóa ví A bằng `FOR UPDATE` | | T1 cầm đầu A |
| 3 | | Chộp khóa ví B bằng `FOR UPDATE` | T2 cầm đầu B |
| 4 | Thò tay xin khóa B và ĐỨNG CHỜ | | T1 phải nịnh nọt T2 |
| 5 | | Thò tay xin khóa A và ĐỨNG CHỜ | T1 nịnh T2, T2 nịnh lại T1 (Vòng Luẩn Quẩn) |
| 6 | Còi báo động rú lên (detector thấy cycle) | Còi báo động rú lên (detector thấy cycle) | Giám thị PostgreSQL bốc thăm chọn 1 đứa Bắn Bỏ |
| 7 | T1 sống sót | Ví dụ T2 lãnh đạn nhận mã lỗi `40P01` | Lệnh chết, Giao dịch tiêu tùng (aborted) |
| 8 | T1 vớ được khóa B và xông lên | T2 Dọn rác (rollback), Nhả khóa B ra | Vòng Luẩn Quẩn Vỡ Tan |
| 9 | T1 trừ A, cộng B, Chốt sổ (commit) | | Ví A còn `900`, Ví B lên `1_100` |

Thằng Dính Đạn (Victim) có thể là T1 thay vì T2. Lập trình đúng đắn và viết Test chỉ được đo ni "Bảo đảm có ĐÚNG 1 KẺ CHẾT THAY để giải quyết kẹt xe", ĐÉO ĐƯỢC chỉ định cứng (hard-code) thằng connection nào bắt buộc phải thua.

> **Nói ngắn gọn:** Thằng Database PostgreSQL nó dẹp đường bằng cách Giết Thí Mạng một Giao Dịch để mấy thằng khác chạy tiếp; Chứ DB đéo rảnh mà quyết định giùm App là cái Giao dịch chết yểu kia Nên Làm Lại (retry) hay Quăng Lỗi Chửi Khách (business error).

## 3. Mong Ước Nhỏ Nhoi và Thực Tế Phũ Phàng (Kết quả mong đợi và thực tế)

| Khía cạnh | Đội Dev Nằm Mơ Cười Khúc Khích | Thực Tế Bị Đấm Sưng Mắt |
| --- | --- | --- |
| Cơ Chế Khóa (Locking) | Nghĩ rằng 2 Lệnh Chuyển Tự Rồng Rắn Xếp Hàng | Mỗi Lệnh ôm khư khư 1 cái Ví và Đứng Nhìn Nhau Say Đắm |
| Tiến độ (Progress) | Tưởng cả 2 đều Ghi Sổ êm đẹp ngay lần đầu | Một Kẻ Chết Thay bị văng Lỗi Báo Tử `40P01` |
| Tính Trọn Vẹn (Atomicity) | Tiền Trừ và Tiền Cộng đi đôi với nhau | PostgreSQL quăng rác của Kẻ Chết Thay đi, Nhưng Thằng Viết Code (caller) phải tự Lên Kịch Bản Xử Lý Cái Chết Đó! |
| Làm Lại (Retry) | Viết cái Vòng Lặp Retry nhỏ xíu sau chữ `catch` | Giao dịch cũ ĐÃ THÚI HOẮC; Phải Dọn Dẹp (Rollback) rồi Bắt Đầu 1 Lần Chạy Mới Tinh (fresh attempt) |
| Bơm Scale (Scale-out) | Tưởng dán bùa `synchronized` ở Class là Yên Tâm | Code App chỉ khóa được trong máy đó, Còn 10 máy cùng phi vào thì vẫn Kẹt Xe ở trạm thu phí PostgreSQL! |

Bọc Giao Dịch (`@Transactional`) chỉ giúp em Dọn Rác Sạch Sẽ (atomic rollback) khi có biến, CHỨ NÓ KHÔNG CÓ PHÉP MÀU ép mọi thứ phải chạy Trót Lọt, Càng Không Tự Động Viết Ra Cái Trật Tự Khóa Chuẩn Mực (canonical lock order) giùm em đâu.

## 4. Giải Phẫu Lưới Chờ Kẹt Xe (`wait-for graph` trong PostgreSQL)

Lúc lính T1 phang câu lệnh:

```sql
select * from account where id = 202 for update;
```

Cái ví B đã bị T2 cất vào túi (khóa) mất rồi. Lính T1 phải Ngậm Ngùi Đứng Chờ thằng T2 hoàn lương (kết thúc giao dịch), vì T2 phải chốt xong thì T1 mới biết xài cái Phiên Bản Dữ Liệu/Khóa nào. Y Chang vậy, T2 lại đang đứng chực chờ T1 nhả cái ví A ra. Hai Sợi Dây Nối Móc Vào Nhau thành 1 Vòng Tròn:

```text
T1 ôm giữ Ví A ──────── Đứng há mỏ chờ Ví B (mà T2 đang ôm)
  ▲                                             │
  └──────── Ôm Giữ Ví B, Lại Há Mỏ Chờ Ví A ────┘
```

PostgreSQL đéo rảnh mà đi check kẹt xe từng mili-giây. Phải đợi tụi nó cắn nhau đủ thời gian quy định (cái gọi là mốc `deadlock_timeout`), ông Giám Thị (detector) mới xách đèn pin ra rọi đồ thị. Thấy cái Vòng Luẩn Quẩn, Ổng Bắn Bỏ Một Thằng ngay để Đám Đông đằng sau còn đi tiếp.

Nhớ kỹ: `deadlock_timeout` chỉ là cái Cú Chỉnh Đồng Hồ Báo Thức của Giám Thị, KHÔNG PHẢI MỐC THỜI GIAN NGHIỆP VỤ (business deadline). Em có vặn cái giờ này ngắn lại thì Bọn Nó Vẫn Kẹt Xe, lại còn làm Giám Thị chạy rông tốn cơm (tốn CPU kiểm tra) ở những nơi Người Ta Đứng Chờ Rất Bình Thường.

## 5. Bóc Trần Các Ổ Khóa Tham Chiến (Các loại lock thực sự tham gia)

Mỗi lần thốt lên `SELECT ... FOR UPDATE`:

- Xin luôn Cục Khóa Bảng (relation-level `ROW SHARE` lock) trên nguyên cái Bảng `account`;
- Chọt xin Cục Khóa Dòng (row-level lock) trên đúng cái dòng tìm được;
- Trúng cái dòng đang bị thằng khác giấu, lính của em sẽ ngồi xếp bằng Đợi Mã Giao Dịch (transaction ID) của cái thằng giấu kia Chạy Xong.

Hai cục khóa Bảng `ROW SHARE` của T1 và T2 hoàn toàn Bắt Tay Sống Chung Hoà Bình với nhau. Điểm Bùng Nổ của Kẹt Xe Nằm Trực Tiếp Ở Khóa Dòng/Chờ Giao Dịch, ĐỪNG CÓ ĐỔ THỪA LÀ DO "KHÓA NGUYÊN CẢ BẢNG NÊN KẸT".

Lục trong bảng `pg_locks`, một thằng đang há mỏ chờ (waiter) sẽ hiện trạng thái đang đợi ở cột `transactionid`; Còn mấy cái Khóa Dòng đéo phải lúc nào cũng mọc ra 2 dòng chữ `tuple` rõ ràng dễ hiểu đâu. Em phải xâu chuỗi một nùi bảng: `pg_stat_activity`, `pg_locks`, `pg_blocking_pids(pid)` rồi móc với Mã Vạch (correlation ID) của App em mới tìm ra Kẻ Thủ Ác, chứ dòm vô một dòng khóa là Bó Tay!

## 6. Món Nghề Phiên Bản MVCC Và Ảnh Chụp Tức Thời (MVCC và statement snapshot)

Mặc định ở chế độ `READ COMMITTED`, mỗi câu lệnh đều Cầm Máy Chụp một Tấm Ảnh Mới (snapshot mới). Nhưng xài Bùa `SELECT ... FOR UPDATE` thì đéo phải chỉ Nhìn (đọc visible version); NÓ PHẢI GIÀNH GIẬT ĐƯỢC Ổ KHÓA CỦA CÁI DÒNG ĐÓ rồi mới chịu nhả số liệu về cho App.

Chiếu theo Lịch Sử Tranh Giành:

- Tấm ảnh lúc T1 khóa ví A vẫn dòm thấy Số Dư Chốt (committed balance) là `1_000`;
- Tấm ảnh lúc T2 khóa ví B cũng tự dòm thấy Số Dư Chốt là `1_000`;
- Cả hai thằng, Tới phát Lệnh Khóa Thứ 2 ĐỀU BIẾT TÒNG TỌC LÀ CÁI DÒNG ĐÓ CÓ TỒN TẠI, Nhưng Đều Phải Há Mỏ Chờ Mở Khóa;
- Thằng Dashboard Phèn phèn quăng lệnh `SELECT` thường vô, Nó Vẫn Đọc Được Cục Dữ Liệu Đã Chốt như thường, và NÓ ĐÉO BAO GIỜ BỊ DÍNH VÀO VÒNG KẸT XE;
- Khi Thằng Ôm Khóa Chết Hay Chốt Xong, Thằng Đang Há Mỏ Sẽ Lấy Được Khóa Theo Luật Của Tấm Ảnh/Lệnh PostgreSQL, LÚC ĐÓ MỚI PHẢI BÊ VỀ APP MÀ THẨM ĐỊNH LẠI DỮ LIỆU VỪA CHỚP ĐƯỢC (revalidate state đã khóa).

Tăng Cấp Độ Cô Lập Lên Cao Cũng ĐÉO SỬA ĐƯỢC CÁI BỆNH KHÓA NGƯỢC CHIỀU (opposite ordering). Xài `REPEATABLE READ`/`SERIALIZABLE` lại còn rước thêm Cả Đống Mìn Hậu Quả Tranh Chấp khác; (Xem case `DB-009` để biết Cấp Cứu Lỗi Vi Phạm Phân Tách - Serialization failure).

## 7. Giải Mã Cái Chết Thay: "Bị Abort" Là Thế Nào? (Transaction victim bị abort nghĩa là gì?)

Câu lệnh lãnh viên kẹo đồng `40P01` sẽ sụp đổ, kéo theo Cả Cái Bọc Giao Dịch (transaction) Bật Đèn Đỏ Văng Fail. TOÀN BỘ mọi Múa Phụ Vụ của Giao Dịch đó — KỂ CẢ CÁI MỚ DỮ LIỆU NÓ VỪA SỬA TRƯỚC KHI BỊ KẸT CHẾT — ĐỀU TIÊU TÁN THÀNH MÂY KHÓI (Không bao giờ được commit).

Lúc này, nếu App của Em "Vô Học" Mà Cố Đấm Ăn Xôi Gõ Lệnh:

```sql
select 1;
```

vào thẳng cái xác Giao dịch chưa kịp đem chôn (chưa rollback), PostgreSQL sẽ Quát Thẳng Vào Mặt:

```text
25P02: current transaction is aborted, commands ignored until end of transaction block
(Mày Mù À? Giao dịch này chết ngoắc rồi, Mọi Lệnh Của Mày Đều Bị Quăng Thùng Rác Cho Tới Khi Mày Chịu Dọn Xác Kéo Băng Rôn Rollback)
```

Thằng Vệ Sĩ của Spring (transaction interceptor) sẽ chụp cái lỗi văng ra khỏi Hàm và tự đi Bấm Nút Dọn Dẹp (Rollback). Dọn Dẹp Xong Thì:

- Mọi Ổ Khóa (dòng, bảng, giao dịch) của Kẻ Chết Thay được vứt bỏ (release);
- Bộ đệm chứa rác (persistence context) của lần thử đó Tuyệt Đối Bị Tiêu Hủy, Không Xài Lại;
- Thằng Hồi Nãy Đang Đứng Há Mỏ Chờ (waiter) Hốt Được Ổ Khóa Và Chạy Tiếp;
- Còn Muốn Làm Lại (retry), Nếu Luật Cho Phép, PHẢI TẠO 1 GIAO DỊCH MỚI TINH, Kéo (reload) Dữ Liệu Hai Cái Ví Lại Từ Đầu!

Lấy Nước Xịt `EntityManager.clear()` ĐÉO THAY ĐƯỢC BƯỚC DỌN DẸP `ROLLBACK`. Viết Hàm Bắt Lỗi `catch` xong rồi Cắm Đầu Xài Tiếp Trong Cùng 1 cái Hàm `@Transactional` đó LÀ SAI LỆCH RANH GIỚI TRẦM TRỌNG!

## 8. Hồi Kết Của 2 Kẻ Chọi Nhau (Các kết quả commit và rollback)

### Kẻ Thắng Cuộc (Transaction thắng commit)

Thằng chiến thắng Vớ được Khóa thứ 2 (sau khi thằng bại trận đi Dọn Xác). Nó Lôi Dữ Liệu Vừa Khóa Ra Thẩm Định Số Dư Lại, Phun 2 Bãi `UPDATE` Đẩy Vào DB (flush), rồi Vênh Mặt Hô `COMMIT`. Cả Cục Tiền Trừ Và Tiền Cộng Ùa Ra Hiện Hình Cùng Lúc; Chìa Khóa Vứt Đi.

### Kẻ Chết Thay Chấp Nhận Số Phận (Transaction victim rollback, không retry)

Kẻ Viết Code (Caller) Phải Chấp Nhận Một Cái Kết Đắng Lòng: Lỗi Xung Đột Kỹ Thuật (temporarily unavailable). Đéo có Mẩu Dữ Liệu Nào Được Sống Sót. BÚT SA GÀ CHẾT: Nếu Cái API Đã Lỡ Hứa Với Khách "Dạ Em Sẽ Xử Lý Dần (Bất đồng bộ)", THÌ PHẢI TÌM CHỖ KHÁC NGOÀI GIAO DỊCH NÀY MÀ LƯU LẠI CÁI LỆNH ĐÓ (durably); CẤM CÓ GIẢ ĐÒ CHÉM GIÓ LÀ CHUYỂN THÀNH CÔNG RỒI.

### Kẻ Chết Thay Đứng Lên Làm Lại Cuộc Đời (Transaction victim rollback rồi retry)

Sống Lại với một Tấm Ảnh Tươi Mới Mẻ, một Đường Chuyền Mới, một Bộ Đệm Trống Không và Chìa Khóa Mới Tinh. Nhưng Khoan: Giờ Đây Nó Có Thể Dòm Thấy Cái Cục Dữ Liệu Mới Toanh Mà Thằng Chiến Thắng Vừa Nhét Vào, NÊN NÓ BẮT BUỘC PHẢI THẨM ĐỊNH LẠI TẤT CẢ LUẬT LỆ NGHIỆP VỤ TỪ ĐẦU (business validation). Đéo phải Chỉ Bật Chạy Lại Câu Lệnh Chờ Thứ 2 Nhé!

Giả dụ Hai Lệnh Chuyển tiền (đá nhau) ĐỀU HỢP LỆ VÀ ĐỦ TIỀN, thì Kẻ Thua Cược Làm Lại Ngay Sau Đó Vẫn Cứ Là Về Đích êm xuôi:

```text
Thằng T1 Xong Trước: A tụt còn 900, B lên 1100
Thằng T2 Bò Lên Dứt Điểm Sau: A lên lại 970, B tụt lại 1030
(Tổng vẫn là 2000, Quá Đẹp!)
```

Nhớ nhé, Mấy Cái Số Lẻ Tẻ Này Để Hù Em Thôi; Cái Bắt Buộc Nhớ Là Tính Trọn Vẹn (atomicity) Và Không Mất 1 Xu Nào (conservation) Tròng Vòng Trái Đất Này.

## 9. Đừng Trộn Lẫn 2 Từ Này Kẻo DB Chửi: Kẹt Xe vs Trễ Giờ (Phân biệt deadlock và timeout)

| Tai Nạn | Mã Vạch Lỗi Đi Kèm | Nguồn Cơn Tội Ác | Đơn Thuốc Cứu Sinh |
| --- | --- | --- | --- |
| Phát Hiện Kẹt Xe (deadlock detected) | `40P01` | Giám thị tóm gọn Vòng Luẩn Quẩn, Bắn Chết 1 Đứa | Lôi xác Dọn Dẹp (Rollback); Chạy Lại Từ Đầu Một Vòng Đời Mới (nếu Luật Cho Phép Cứu) |
| Đợi Khóa Bạc Đầu (lock timeout) | `55P03` | Cắm Trại chờ lâu quá (`lock_timeout`) rụng răng, Không Thấy Kẹt Xe Nào | Lôi xác Dọn Dẹp Giao Dịch Này; Xách Đèn Đi Điều Tra Thằng Nào Ngậm Khóa Quá Lâu/Mạng Yếu. |
| Câu Lệnh Văng Hố (statement canceled) | `57014` | Hết Giờ Chạy Lệnh (`statement_timeout`) Hoặc Bị Ép Cắt Lệnh (cancel) | Dọn Xác (Rollback) vì Giao Dịch Rách Nát; Học Cách Tuân Thủ Mốc Giờ Quy Định (Deadline). |
| Va Đập Cô Lập (serialization failure) | `40001` | Va Chạm Ảnh Chụp Ở Mức Độ Cô Lập Khắt Khe SSI | Lôi xác Dọn Dẹp (Rollback) Lên Đồ Chạy Lại Toàn Tập (xem case DB-009) |

Đừng có Dở Hơi Đi Biến Đám Lỗi Của Đáy Xã Hội Này Thành Thông Báo "Khách Hàng Không Đủ Tiền" Hoặc Bắn Trả HTTP 200 Success Láo Toét! Mẹ Ơi, Đánh Nhau Bể Trán Trong Máy (Technical contention) Khác Xa Lỗi Vi Phạm Luật Của Kế Toán (Business rejection) Nhé!

## 10. Tại Sao Xếp Hàng Theo Chuẩn Lại Phá Được Vòng Chờ Luẩn Quẩn? (Vì sao canonical order phá cycle?)

Lập Bàn Thờ Khắc Câu Luật Xếp Hàng Dựa Trên Số Mã Vạch ID Cứng (stable unique account ID):

```text
Thằng Bé Nhất  = Bắt buộc phải là Số min(fromId, toId)
Thằng Bự Hơn   = Bắt buộc phải là Số max(fromId, toId)
```

Kiểu Gì Thì Kiểu (A sang B Hay B sang A) Tụi Nó Đều PHẢI VÁC MẶT ĐI XIN KHÓA CÁI VÍ A TRƯỚC (Vì A mang Số ID 101 nhỏ hơn B mang ID 202). Nếu Lính T1 ẵm Cái A Rồi, Thằng T2 Phải Hóc Chờ A Trước Khi Được Quyền Đụng Vào Cái B. Vì T2 Chưa Nắm B, Lính T1 Xung Phong Chộp Nốt Được B Rất Mượt; SẼ KHÔNG CÒN CÁI CẠNH NÍU ĐUÔI T1 → T2 RỒI T2 → T1 Nữa Đâu!

Sách Võ Công Này Mất Thiêng Ngay Lập Tức NẾU Và CHỈ NẾU Mọi Ngóc Ngách Lệnh Code Của Em Dùng Chung 1 Cây Thước (comparator) Và Chung 1 Chủng ID (resource identity). Mấy Kiểu Bóp Dái Này Làm Vỡ Mặt Lập Tức:

- Cái Nút API Chuyển Tiền Thì So Sánh Mã Số Hệ Thống (numeric ID), Còn Thằng Nhân Viên Chạy Đêm (batch) Lại Đi So Sánh Mã Thẻ Ngân Hàng Số Khắc Ngoài (external account number);
- Cây Thước (Comparator) Của Em Lại Tính Sai Gộp 2 Thằng Trạc Trạc Nhau Thành 1 Đứa;
- 1 Con Đường Code Lại Vòng Đi Khóa Customer Xong Mới Khóa Ví (account), Đứa Khác Lại Hô Khóa Ví Rồi Mới Đi Khóa Customer;
- Còn Bị Vạ Lây Do Mấy Cái Trigger, Khóa Ngoại Rác, Cọc Lệnh Bí Mật Đứng Ngoài Luồng Nó Lao Vào Ăn Cắp Khóa Không Tuân Luật.

Chốt Hạ: Cầm Cuốn Bí Kíp Xếp Hàng Theo Chuẩn Này Giúp Đuổi Bọn Kẹt Xe Biến Mất Càng Nhiều Càng Tốt (Và Lỗi Nổi Lên Càng Dễ Bắt), NHƯNG KHÔNG CÓ NGHĨA LÀ APP CỦA EM ĐƯỢC QUYỀN LƯỜI BIẾNG VỨT LUÔN CÁI BỘ VÁ LỖI VÀ QUY ĐỊNH LÀM LẠI (retry policy).

## 11. Đừng Để Hai Thằng Lính Spring Và Hibernate Lừa Tình (Ranh giới Spring và Hibernate)

Gắn Bùa `@Lock(PESSIMISTIC_WRITE)` Sẽ Bắt Xung Đột Xì Khói Ngay Lập Tức Khi Chạy Câu Lệnh Của Repository. Nhưng Cái Tính Nết Hay Trì Hoãn Kiểm Tra Rác (deferred dirty checking) Của Hibernate, Mãi Tới Lúc Chốt Xả Dữ Liệu (flush/commit) Nó Mới Phụt Ra Dòng Chừ Tiền `UPDATE`. Trong Khi Đó Án Mạng Kẹt Xe Lại Xuất Hiện Rất Sớm Ở Câu Lệnh Đang Đòi Ổ Khóa `SELECT` THỨ 2!

Nếu Em Viết Code Thọc Tay Trừ Tiền Thằng Chuyển TRƯỚC Xong Mới Thò Đi Khóa Đứa Nhận, Thằng Đệ Hibernate Rất Có Thể Nó Tự Mở Vòi Xả Rác (auto-flush) TRƯỚC Khi Kịp Xin Ổ Khóa Thứ 2 (Tùy Vào Cách Chỉnh flush mode/query space). Nghĩa Là Điểm Nổ Hoặc Kẹt Xe Lại Bay Trật Khỏi Chỗ Em Dự Báo Trên Tờ Giấy! BỌN MÀY VIẾT CODE ĐÚNG KHÔNG THỂ NGỒI CẦU NGUYỆN KIỂU "Trời Sinh Nó Lỗi Chắc Chắn 100% Nằm Ở Câu Chọc DB Của Repository Đâu!"

Thằng Bảo Mẫu Spring Rất Hay Dịch Đống Mã Lỗi Hiểm Hóc Của Hibernate/JDBC Ra Thành `CannotAcquireLockException` Hoặc 1 Lớp `DataAccessException` Nào Đó Cho Dễ Nuốt. Thằng Trưởng Phòng Xét Duyệt Làm Lại (Retry classifier) PHẢI LỤC TÌM TỚI TẬN CÁI RỄ `SQLException#getSQLState()` BẰNG ĐƯỢC CHỮ `40P01`, Chứ Không Phải Quét Cái Tên Vỏ, Và Nhớ Dán Cả Chuỗi Nguyên Nhân Để Trong Nhật Ký Ghi Chép (log/trace) Của App.

Gắn Đè 2 Lá Bùa `@Retryable` VÀ `@Transactional` Lên Cùng 1 Hàm Của Đứa Nhân Viên (Method) Sẽ Biến Câu Chuyện Chạy Đúng Hay Sai Trở Thành May Rủi (Phụ Thuộc Đứa Nào Đứng Trước). Cách Xếp Trận Bất Bại Dễ Đọc Là Tách Rời Hẳn Tên Đội Trưởng Retry KHÔNG GIAO DỊCH Lên Ra Khỏi Đứa Culi Giao Dịch Làm Viêc Nặng Ở Class Khác.

## 12. Gây Án Đời Thực Và Rủi Ro Làm Lại 2 Lần Tiền (External side effect và nguy cơ trùng lặp)

Khi PostgreSQL Tuyên Bố Bỏ Chạy (rollback), NÓ ĐÉO THỂ THU HỒI LẠI ĐƯỢC Cái Lá Thư Điện Tử (email), Tiếng Sét API (HTTP call) Hay Tín Hiệu Em Đã Bắn Bậy Ra Thế Giới Bên Ngoài. Nếu Giao Dịch Đang Chạy Lại Thích Ra Oai Gọi Cái Thế Giới Đời Thực Đó TRƯỚC Khi Chốt Sổ (commit) Và Rồi Vô Phúc Trúng Cử Ăn Đạn Kẻ Chết Thay:

- PostgreSQL Khóc Thét Xóa Sạch Dữ Liệu Rác Ở Database;
- Nhưng Cục Khách Hàng Ngoài Đời THÌ LẠI HIỆN RA CHỮ "GIAO DỊCH THÀNH CÔNG VKL";
- Thằng Đội Trưởng Retry Ở App Làm Lại Phát Nữa -> Lại Gửi Thư Báo Nhận Thêm Lần Nữa. KHÁCH MÚA TUNG TRỜI!

Thuốc Giải Duy Nhất: CẤM TUYỆT ĐỐI Gọi Lệnh Đi Mạng (remote I/O) Trong Lúc Còn Đang Ở Trong Căn Phòng Đang Ôm Khóa Bầu Cử DB. Đem Xài Mấy Cục Cứu Tinh (outbox/idempotency/reconciliation) Phù Hợp Đi! Cái Trò Bấm Nút Làm Lại (Deadlock retry) KHÔNG PHẢI LÀ BÙA THẦN Ép Chạy Đúng 1 Lần Đâu!

## 13. Sập Tiệm Và Đứt Cáp Đột Ngột (Process crash và mất connection)

Đang Chạy Mà Điện Cúp, Server Máy App Văng Hồn, Cáp Mạng Nổ Cái Đùng! Thằng PostgreSQL Tự Ngắt Ngay Giao Dịch Đang Treo Chân, Bấm Nút Quăng Rác (rollback) Rồi Nhả Sạch Ổ Khóa. Thằng Xếp Hàng Chờ Bên Cạnh Tự Động Được Tiến Lên Lấy Cục Vàng.
VẤN ĐỀ LÀ: Đứa Kêu Lệnh Ở Phía Gọi Sẽ Đéo Thể Biết Được Lệnh Đã Thực Sự Qua Vòng Chốt Sổ Hay Chưa, Nhất Là Nếu Rơi Mạng Trúng Ngay Vào Đúng Lúc Đang Hô Câu Chốt `commit`.
Mọi Mệnh Lệnh Định Đoạt Tiền Bạc (Command quan trọng) BẮT BUỘC Phải Kèm Chìa Khóa Kiểm Lỗi (idempotency key/status lookup) Đề Phòng Nạn Kết Quả Mờ Mịt (ambiguous outcome); Mù Quáng Nhấn Nút Làm Lại Bừa Bãi Chỉ Đem Lại Thiệt Hại Cộng Kép (duplicate business operation) Cho App Em Thôi.

## 14. Đẩy Lên Nhiều Máy (Multi-instance)

Nhét Bùa Chặn Cửa Cùi Bắp Khóa Ở Máy Chạy App Của Java Như `synchronized`, `ReentrantLock` Hoặc Tự Chế Bộ Nhớ Kẹt Xe Trong Bộ Não RAM Của Đứa App 1... THÌ ĐÉO XI NHÊ GÌ VỚI Đứa App 2 Nhé.
Cái Bảng Quyền Lực Ở Dưới DB (Authoritative rows) Và Dàn Trống Trận Ôm Khóa PostgreSQL (PostgreSQL locks) MỚI CHÍNH LÀ VŨ KHÍ CẦM TRỊCH CUỘC CHƠI CHO CẢ THẾ GIỚI MÁY CỦA APP.

Kéo thêm Cả Làng (Scale-out) Máy Ra Chỉ Làm Tăng Số Lượng Phiên Giao Dịch Cùng Cướp Chung 1 Cái Bát Vàng Ở Cửa Hàng Đông Khách. VÌ VẬY, LUẬT XẾP HÀNG THEO CHUẨN CỨNG (Canonical row order) PHẢI TRỞ THÀNH 1 HIẾN PHÁP (Protocol toàn hệ thống) Mà API, Đội Chạy Ngầm (batch, scheduler) Tới Thằng Dò Lỗi Trừ Tiền Cuối Ngày CŨNG PHẢI TUÂN THEO RAM RÁP. Và Nhớ Rằng Mỗi 1 Cỗ Máy Con Kéo App Lên Lúc Đứng Há Mỏ Đợi (lock wait) LÀ ĐANG GIỮ CHẶT 1 CUỐNG KẾT NỐI (connection), Nên Cái Hồ Bơi Và Khóa Giờ Thở Của Toàn Bộ App Phải Trông Cùng Với Ổ Khóa DB.

## 15. Dò Bệnh Cấp Nào Chết Cấp Đó (Nguyên nhân gốc theo từng layer)

| Tầng Hệ Thống | Kịch Bản Thủ Vai |
| --- | --- |
| Lớp Của Code Tụi Mày (Application) | Chọn Cách Bốc Lấy Ổ Khóa Bằng Đích Hay Nguồn Lung Tung, Tự Cắn Đuôi Mình Làm Vòng Tròn (circular wait) |
| Bảo Mẫu (Spring) | Đặt Giao Dịch Đúng Chỗ Rồi, Mà Sáng Kiến Xài Nút Retry Trong Cùng Vòng Trái Đất (boundary) Thì Bị Lú Rồi Đó Con |
| Lớp Tích Hợp (Hibernate/JPA) | Sủa Câu Lệnh Bắt Ổ Khóa `SELECT` Xong, Phun Ra Báo Cáo Xung Đột Nhùng Nhằng Dịch Tào Lao Bí Đao (propagate conflict) |
| Bà Hoàng Dưới Đáy (PostgreSQL) | Lắm Giữ Mọi Ổ Khóa Dòng, Rọi Đèn Thấy Vòng Tròn Đóng Kín, Sút Bay Đầu 1 Đứa Thành Kẻ Chết Thay Gửi Nhãn Hiệu `40P01` |
| Bùa Lởm (JVM-local lock) | Không Có Phép Lực Nào Bảo Vệ Giao Dịch Đang Chạy Ở Tòa Nhà Kế Bên Đâu Con (application instance khác) |

TÓM LẠI: NGUYÊN NHÂN CỐT LÕI KHÔNG PHẢI LÀ POSTGRESQL CHẠY CHẬM, CÀNG ĐÉO PHẢI LÀ 2 ĐỨA TỚI CÙNG LÚC, MÀ LÀ TỘI YẾU KÉM KHI GOM NHẶT NHIỀU NGUỒN TÀI NGUYÊN (multi-resource acquisition) MÀ LẠI ĐÉO CÓ MỘT TRÌNH TỰ BÀN THỜ NÀO CHO TỤI NÓ THEO DÕI CẢ (total order chung).

## 16. Kính Chiếu Yêu Quản Trị Hệ Thống (Khả năng quan sát - `observability`)

Các Cần Ăn Gắp Phải Quăng Lên Tường Báo:

- Cột Điểm Đếm Tổng Kẹt Xe (`deadlock_detected_total`) Chia Theo Ngạch Nhiệm Vụ, KHÔNG BAO GIỜ Nhồi Mớ ID Rác Rưởi (high-cardinality) Tài Khoản Của Khách Vào Nha!
- Ghi Chú Đủ Các Dòng Trạng Thái Thử Lại: Số Lần Làm (attempt), Thắng Thua Thế Nào (outcome/exhaustion) Bị Ngắt Cáp Giữa Chừng Phút Số (elapsed deadline) Bao Lâu.
- Mã Lỗi Tội Phạm Đi Kèm SQLSTATE, Dây Xích Liên Quan Tới DB (transaction/correlation ID), Và Luật Xếp Thứ Tự Chuẩn Của Đám Đồ (canonical resource order).
- Soi `pg_stat_database.deadlocks` Từ Ngoài Dashboard!
- Mở Rộng Băng Video Ghi Kẹt Xe (deadlock log) Ở PostgreSQL Vô Nhờ Cấu Hình Bật `log_lock_waits` Cho Sạch Mắt Đỡ Ngứa Lưng;
- Khi Có Biến Mà Máy Còn Đang Chết Ngắt: Đè Khẩn Cấp `pg_stat_activity`, `pg_locks` Và `pg_blocking_pids` Ra Bắt Tội Phạm Cắn Khóa Còn Tươi (live waits)!
- Trạm Bơm Nước Chờ Xin Giao (Connection pool): Xem Bao Nhiêu Lệnh Đang Chạy, Đợi Ở Trạm Hay Bị Nhốt Qua Đêm (timeout).

Tuyệt Đối CHÉM ĐẦU Nếu Mày Ghi Sổ Số Tiền, Cục Password Token Hoặc Bí Kíp (bind value nhạy cảm)! Dải Lệnh Kẹt Xe Ghi Đè (Deadlock log) LÀ BẢN KẾT ÁN VÒNG LẶP ĐÃ XẢY RA; Còn Máy Đo Vòng Đợi Khóa Bạc Đầu (lock wait metric) LÀ QUẢ CẦU TIÊN TRI GIÚP MÀY ĐI DỌN ĐƯỜNG TRƯỚC KHI TRẦN GIAN GẶP THẢM HỌA.
