# Mổ Xẻ Tận Cùng: Ảo Ảnh Của Từng Câu Lệnh (Statement snapshot)

## 1. Bối cảnh ban đầu (Initial state)

Trước khi 2 diễn viên chính (Ông A và Ông B) ra sân, dưới DB PostgreSQL đang có dòng Luật như sau đã chốt (committed):

```text
merchant_refund_policy(M-42)
  auto_refund_limit = 100.00
  active            = true
  revision          = 7
```

Khách yêu cầu hoàn lại `80.00`. Ở Phiên bản số `7`, lệnh duyệt tự động dư sức chạy qua; nhưng nếu gặp Phiên bản `8` với mức trần `50.00` thì rớt đài cái chắc.

## 2. Ảo tưởng sức mạnh của Lập trình viên (Expected theo broken design)

Dev nhà mình hay lẩm nhẩm cái suy luận ngây ngô này:

```text
Ông A ĐÃ BẮT ĐẦU (BEGIN) Giao dịch (transaction) rồi...
  -> Nên Luật lệ sẽ đứng yên không bao giờ đổi trong suốt mọi lần Đọc của A.
  -> Do đó, cái Phiên bản lúc dùng để Phán Quyết và cái Phiên bản lúc đem đi Lưu Sổ sẽ luôn là MỘT.
```

Nghe hợp lý ha? Nó chỉ đúng nếu bạn được phát một Bức Ảnh Giao Dịch Xuyên Suốt (stable transaction snapshot) kìa! Còn với PostgreSQL ở mức mặc định `READ COMMITTED`, nơi mà nó chỉ phát Bức Ảnh Chụp Vội cho Từng Lệnh SELECT (plain SELECT), thì suy luận này sai bét!

## 3. Thực tế phũ phàng diễn ra (Actual interleaving)

| Bước | Lính Lác A (Xét duyệt - Refund evaluator) | Sếp B (Đổi Luật - Risk administrator) |
| --- | --- | --- |
| 1 | `BẮT ĐẦU VỚI MỨC CÔ LẬP READ COMMITTED` | |
| 2 | Lệnh SELECT #1 lấy được Bức ảnh S1 | |
| 3 | Đọc Hạn mức = `100`, Phiên bản = `7` | |
| 4 | Ký lệnh `APPROVED` cho số tiền `80` (Duyệt trong RAM) | `BẮT ĐẦU` |
| 5 | | Lệnh UPDATE Hạn mức xuống `50`, Phiên bản lên `8` |
| 6 | | `CHỐT SỔ (COMMIT)` |
| 7 | Lệnh SELECT #2 lấy được Bức ảnh MỚI TOANH S2 | |
| 8 | Đọc Hạn mức = `50`, Phiên bản = `8` | |
| 9 | Lệnh INSERT báo cáo: Đã Duyệt 80, dựa trên Limit 100, LUẬT PHIÊN BẢN 8 | |
| 10 | `CHỐT SỔ (COMMIT)` | |

Kết cục tàn tạ (Final state):

```text
Luật hiện tại trên DB: Hạn mức=50, Phiên bản=8
Quyết định lưu lại:    Số tiền=80, APPROVED, Limit dùng xét=100, Gắn nhãn Phiên bản=8
```

Cả A và B đều cười toe toét báo Thành công. Tại sao không có lỗi? Vì hai giao dịch không đánh nhau trên cùng 1 dòng dữ liệu (A thì UPDATE/INSERT dòng khác, B thì UPDATE dòng Luật). Cho nên Lỗi Ghi Đè (lost-update) hoàn toàn không thèm hiện ra.

> **Nói ngắn gọn:** PostgreSQL rất uy tín ở chỗ: Đảm bảo không cho bạn thấy dữ liệu dở dang chưa chốt. NHƯNG nó KHÔNG HỀ Tuyên thệ rằng: Nhiều câu lệnh ở chế độ `READ COMMITTED` ghép lại sẽ vẽ ra Cùng Một Bức Tranh Tổng Thể. Nó chỉ là 2 tấm ảnh chụp ở 2 thời điểm khác nhau thôi!

## 4. Máy Chụp Ảnh Hoạt Động Ra Sao? (Snapshot chính xác của từng statement)

Ở chế độ `READ COMMITTED`, quản gia PostgreSQL chỉ bấm máy chụp Ảnh (snapshot) đúng vào khoảnh khắc Câu Lệnh chạy:

```text
Ảnh S1 chụp TRƯỚC KHI sếp B chốt
  -> Thấy Luật Phiên bản 7

Sếp B chốt cái Luật Phiên bản 8 (commits)

Ảnh S2 chụp SAU KHI sếp B chốt
  -> Hiển nhiên thấy Luật Phiên bản 8 Mới Nhất
```

Cú SELECT #1 hoàn toàn KHÔNG PHẢI là Đọc Rác (dirty read) hay bị "Sai" khi sếp B chốt sổ. Bức ảnh S1 hoàn toàn vinh quang và hợp lệ ở thời điểm đó.
Lỗi là do Mấy Trò Xào Nấu (application) ở trên App, vác Quyết Định từ Tấm S1 đem trộn chung với Số Liệu của Tấm S2 mà không hề có động tác Khám lại xem Luật lệ đã đổi chưa (revalidate).

## 5. Các Mảnh Đời Song Song (MVCC tuple versions)

Khi B chạy lệnh UPDATE, DB không ghi đè mất xác dòng cũ, mà tạo ra một Phiên bản Dòng (tuple version) mới tinh:

```text
Bản cũ (old tuple): Limit=100, Phiên bản=7
Bản mới (new tuple): Limit=50,  Phiên bản=8
```

Sau khi sếp B chốt sổ:

- Tấm ảnh S1 của lính A vẫn trỏ về Bản Cũ 7 (dữ liệu vật lý vẫn còn đó).
- Lệnh Đọc mới của lính A (ảnh S2) tự động chọn Bản Mới 8.
- Bản Cũ 7 vẫn sống sót vật lý (chờ đội dọn rác Vacuum dẹp đi sau) để phục vụ cho mấy bức ảnh cũ chưa coi xong.
- Còn App của bạn thì tưởng 2 lần đọc đó là cùng 1 sự vật hiện tượng (cùng logical row). Đời không như mơ!

Công nghệ MVCC sinh ra là để ĐỌC KHÔNG ĐỢI GHI, GHI KHÔNG ĐỢI ĐỌC (giảm blocking). Nó KHÔNG sinh ra để Đóng Băng thời gian sự vật cho bạn xài từ đầu chí cuối ở mọi isolation level.

## 6. Trò Chơi Ổ Khóa (Lock behavior)

Khi sếp B UPDATE, sếp bợ luôn cái Ổ Khóa Của Dòng Luật (row-level lock) và giữ khư khư tới khi chốt sổ (commit).
Còn ông lính A gọi Lệnh SELECT chay:

- Hoàn toàn KHÔNG xin Khóa Đọc (`FOR SHARE` hay `FOR UPDATE`).
- Vẫn đọc ào ào trôi tuột vì MVCC phát cho Bức Ảnh Cũ (không bị block bởi B).
- Không thèm đặt gạch xí chỗ cho lần đọc sau.
- Không có quyền gì cấm sếp B chen ngang vào giữa hai statements.

Khi ông A chèn giấy tờ (INSERT `refund_decision`), ông A chỉ xin khóa trên cái bảng Nhật Ký, nên chả đụng chạm gì sếp B đang giữ khóa Bảng Luật. Đó là lý do CSDL mù màu không thể tuýt còi báo Xung Đột.

## 7. Khối Lệnh Rời Rạc (Non-atomic sequence)

Quy trình vỡ nát:

```text
Đọc Luật (Tấm Ảnh V1)
  -> Trong RAM: Ra phán quyết dựa trên V1
  -> Đùng! DB Chốt thành V2
  -> Đọc Luật (Lấy Audit từ V2)
  -> GHI: Trộn phán quyết V1 với Audit V2 lại thành một mớ hổ lốn.
```

Cái Bọc Giao Dịch (`@Transactional`) chỉ giúp các Lệnh GHI dính chùm vào nhau (atomic writes). Nó KHÔNG giúp đóng gói các lệnh ĐỌC RỜI RẠC thành 1 Sự Kiện Duy Nhất (atomic observation) dưới chế độ `READ COMMITTED`.

## 8. Đổ Lỗi Cho Ai Bây Giờ? (Root cause theo layer)

### Lớp PostgreSQL

Chế độ `READ COMMITTED` cố tình được thiết kế chụp nhiều Tấm Ảnh (statement snapshots). Lỗi "Đọc không lặp lại" (Non-repeatable read) là hành vi hợp lệ được cho phép, không phải do DB bị hư hay sập (corruption).

### Lớp Spring

Cái thẻ `@Transactional(isolation = READ_COMMITTED)` đã làm quá tốt nhiệm vụ tạo 1 Giao dịch vật lý đàng hoàng, nhưng nó Không Thể Đòi Bức Ảnh Xuyên Suốt. Việc bạn gọi method con (propagation `REQUIRED`) nó cũng chỉ bú ké chung cái Giao dịch đang chạy.

### Lớp Hibernate/JPA

Mấy câu Lệnh (Scalar projection) tự kích nổ lệnh SELECT mới thẳng xuống DB. Bộ đệm Cấp 1 (first-level cache) không thèm hòa trộn 2 tập kết quả. Còn lúc Hibernate xả lệnh Ghi (flush/dirty checking), nó cũng bơ luôn việc nhét thêm cái điều kiện (predicate) Kiểm Tra Phiên Bản Cũ vào lệnh INSERT.

### Lớp Mã Nguồn (Application)

Logic ở Code không hề có một "Bản Hợp Đồng" rõ ràng. Đọc lại cái Dòng Quyền Lực (authoritative row) mà KHÔNG HỀ tính toán lại, so sánh thử, hay Đặt Khóa. Làm ăn quá cẩu thả!

## 9. Cú Lừa Bộ Đệm (Persistence context nuance)

Đôi lúc bạn chạy Code thấy 2 lần gọi lại trả về y chang nhau:

```text
Lần 1: findById -> Đối tượng Java (Phiên bản 7)
Lần 2: findById -> Dùng lại Object cũ (Vẫn Phiên bản 7, chẳng thèm gọi SQL)
```

Đừng vội mừng! Đó chỉ là lớp vỏ bọc che đậy (interleaving) của Bộ đệm (Identity Entity). Lỡ ai đó viết câu lệnh Native (native query), gõ ép làm mới (`refresh`), hay mở 1 Giao dịch Khác, Bùm! Phiên bản 8 lòi mặt ra ngay.
Tính Đúng Đắn (Correctness) không bao giờ được phép trông cậy vào Tai Nạn Tình Cờ của Bộ đệm! Nhớ bật log SQL lên mà soi.

Nếu câu truy vấn Entity thực sự được bắn đi, cách Hibernate Merge dữ liệu mới vào Object cũ thế nào còn hên xui tùy chế độ. Viết code phải rành mạch.

## 10. Liệu `REPEATABLE READ` Có Phải Thuốc Tiên?

Nếu ép PostgreSQL xài `REPEATABLE READ`, nó sẽ cấp Bức Ảnh Giao Dịch Xuyên Suốt (transaction snapshot):

```text
SELECT #1 -> Phiên bản 7
Sếp B chốt Phiên bản 8
SELECT #2 -> Vẫn giữ nguyên Phiên bản 7
```

Tuyệt quá, chữa được bệnh "Đọc Không Lặp Lại" cho A rồi!
NHƯNG KHOAN, sếp B VẪN CHỐT SỔ THÀNH CÔNG (vì A chỉ đọc, không ghi đè lên Luật). Nếu sắp xếp tuần tự, câu chuyện sẽ kể lại như vầy:

```text
Ông A duyệt tiền theo Phiên bản 7. Xong xuôi.
Sau đó Sếp B đổi Luật thành Phiên bản 8.
```

Nếu Luật kinh doanh chấp nhận điều kiện ở trên (Duyệt theo hình ảnh lúc đó là xong) -> OK, Thuốc này hiệu nghiệm!
Nhưng nếu Luật công ty là "Lúc Chốt Sổ, Mày Phải Dùng Đúng Cái Luật Tươi Mới Nhất!", thì xin lỗi, `REPEATABLE READ` Một Mình Mày Là Chưa Đủ!

## 11. Đỉnh Cao `SERIALIZABLE`?

Chế độ Serializable Snapshot Isolation của PostgreSQL rất thông minh, tự đi bắt bẻ các Giao dịch chạy lung tung (anomalies) và đạp (abort) mấy cái có cấu trúc phụ thuộc Nguy Hiểm. Nhưng Mức Độ Cách Ly Cao Nhất KHÔNG CÓ NGHĨA LÀ "Bắt thằng Đọc phải Thấy Dữ Liệu Tươi Nhất Của Thằng Vừa Chốt".

Trong vụ này, A Đọc Luật/Ghi Nhật Ký, còn B Cập Nhật Luật. Trật tự "A chạy trước xong B chạy sau" vẫn Hợp Lệ 100%, nên DB tha cho cả 2 đứa cùng chốt (commit). Muốn B và A chém giết nhau rõ ràng, thì phải đặt Khóa Explicit (chặn cửa rành rọt) hoặc thiết kế model hợp lý.

## 12. Hợp Đồng Chống Vỡ Mặt (Revalidation semantics)

Làm rõ liền 3 bản Hợp Đồng Luật Lệ này:

1. **Chụp lúc nào, xử lúc đó (Snapshot-at-evaluation):** Xét duyệt dựa trên Tấm Ảnh Luật đã đọc, khỏi lằng nhằng.
2. **Lúc Cắm Cờ phải Tươi (Current-at-write):** Lúc chốt sổ Ghi Quyết Định, Cái Luật đó phải CHƯA BỊ AI SỬA.
3. **Cấm Ai Khóa Chặn Mõm (Locked-through-commit):** Cấm ai đụng chạm sửa Luật từ lúc Tui bắt đầu xem đến tận lúc Tui chốt sổ xong.

Một câu lệnh Gắn Điều Kiện Đuôi (conditional statement) giải quyết đẹp Hợp Đồng Số 2. Ổ khóa `FOR SHARE` (khóa đọc) lo gọn Hợp Đồng Số 3. Đừng nói mồm chữ "Nhất quán" (consistent) mà không vạch rõ THỜI ĐIỂM nằm ở đâu!

## 13. Sập Cầu Dao Chốt / Xóa Sổ (Commit và rollback)

### Sếp B xù kèo (B rollback)

Luật Phiên bản 8 tan thành mây khói (không visible). Cú SELECT #2 của A lại vui vẻ đọc lại Phiên bản 7; Không hề hấn gì, thế giới hòa bình.

### Lính A xù kèo (A rollback)

Giấy duyệt bị xé (Rollback Decision). Luật Phiên bản 8 của B đã được Chốt Sổ đàng hoàng và sống khỏe. Mọi thứ độc lập.

### Lính A nhanh tay chốt trước B (A commit trước B)

Quyết định Phiên bản 7 của A có thể là hợp lệ nếu soi theo trật tự A chạy trước B, miễn là Lịch sử Đổi Luật và Sổ Nhật Ký chấp nhận chuyện đó.

## 14. Bị Giam Cầm Và Quá Giờ (Timeout và lock wait)

Với cú Lệnh Đọc Chay (plain SELECT) hư đốn, A chả cần chờ B chốt xong làm gì.
Còn nếu lính A ngoan ngoãn mang Ổ Khóa `FOR SHARE` (Xin Phép Khóa), thằng Sếp B đi sau muốn UPDATE sẽ:

- Bị Đứng Hình chờ A chốt xong hoặc bỏ cuộc (commit/rollback).
- Tức quá văng lỗi hết giờ (`lock_timeout`).
- Hoặc dính Chùm Kẹt Xe (deadlock) nếu nhiều Luật bị khóa qua khóa lại không theo thứ tự.

Quá Giờ (Timeout) phải được rẽ nhánh sang logic Từ Chối hoặc Thử Lại đàng hoàng ở Code. Đừng có cố tình nới thời gian timeout lên cả chục giây để lấp liếm sự dốt nát của cái Giao dịch Quá Dài!

## 15. Chiêu Thức Thử Lại (Retry)

Lỗi "Đọc Không Lặp Lại" ở mức `READ COMMITTED` nó chạy lẳng lặng như bóng ma, Đừng Có Mơ CSDL nó ném lỗi (exception) ra cho bạn tự Thử Lại (retry). App phải Tự Động bắt mạch bằng cách so sánh version lệch, hoặc đếm dòng ảnh hưởng (affected-row) = 0.

Quy trình Thử Lại Chuẩn Men:

```text
Thử Lần 1: Chết ngắt, lôi nhau Rollback.
Nghỉ xả hơi một nhịp (có độ trễ / jitter)
Thử Lần 2: Mở một Giao Dịch MỚI TINH tươm.
Tải Lại toàn bộ Luật Lệ tươi mới.
Ngồi xét duyệt tính toán lại từ đầu đến đít.
```

Chỉ đem cái Quyết định sai cũ rích chèn (INSERT) lại là tự vả vào mặt. Mà Thử lại bên trong chính Cái Giao Dịch Cũ Bị Nát cũng chả giúp Dữ liệu tươi lên được đâu.

## 16. Chết Đứng Giữa Đường (Crash behavior)

Nếu App của A Sập Nguồn (crash) trước lúc Chốt Sổ, DB PostgreSQL dẹp mớ rác đó đi, Quyết Định không được lưu. Khỏe!
Nếu sập Sau Khi Chốt Sổ nhưng Chưa Kịp Phản Hồi cho khách, Khách bấm nút Gửi Lại (retry). Mã ID duy nhất (`command_id`) + Cơ chế Bất biến (idempotency) lo vụ Gửi Trùng. Nhưng cái trò đó KHÔNG thể thay thế được cái "Kỷ Luật Chống Trộn Dữ Liệu" nhé!

Bắt buộc Phiên Bản Luật và Số Tiền Duyệt phải được chốt cứng Đóng Đinh cùng 1 chỗ dưới Database, để có cháy rụi cái Server App thì Đội Hậu Kiểm (reconciliation) vẫn còn cái mà dò.

## 17. Chạy 100 Máy Cùng Lúc (Multi-instance)

Ông A chạy ở Server App Số 1, Sếp B ngồi sửa Luật ở Server App Số 2. Mấy cái bùa như `synchronized`, `ReentrantLock` hay Lưu Bộ Đệm Bằng RAM Của Server 1 hoàn toàn phế võ công với Server 2.

Đồn bốt bảo vệ (Authoritative protection) Bắt Buộc phải nằm ở dưới Hầm PostgreSQL:

- Thêm Bảng Lịch Sử Luật (immutable) + Khóa Ngoại (Foreign Key).
- Điều kiện Khắt Khe Lúc Ghi (conditional write).
- Khóa Chặn (row lock).
- Cấp Độ Cô Lập phù hợp để diễn đạt cho trọn nghĩa.

## 18. Mắt Thần Theo Dõi (Observability)

In Log ra đủ các cột này:

```text
Mã_Lệnh (commandId)
Mã_Cửa_Hàng (merchantId)
Số_Tiền (amount)
Phiên_Bản_Đọc (policyRevisionRead)
Phiên_Bản_Lúc_Ghi (policyRevisionWritten)
Quyết_Định (decisionOutcome)
Mức_Cách_Ly_Thực_Tế (effectiveIsolation)
Số_Lần_Thử_Lại (retryAttempt)
```

Giám sát bằng Graph các Món Ăn Chơi:

- Tần suất Lệch Phiên Bản (revision mismatch) / Đếm số lần update hụt (affected-row=0).
- Số Lượng Đứng Chờ Khóa / Thời gian Hết Hạn Khóa (`lock_timeout`).
- Đếm số Lần văng lỗi Trùng Khớp Ổ Khóa (deadlock / serialization failure).
- Tổng Những Cú Quyết Định Lìa Khỏi Lịch Sử.
- Tổng Số Tiền Duyệt Trót Lọt Vượt Quá Hạn Mức Thực Tại (approved amount vượt stored evaluated limit).
- Đo tốc độ chạy của Giao Dịch và áp lực nén lên Bể Kết Nối CSDL (connection pool pressure).

Mấy cái Dấu Vết Câu Lệnh (SQL/trace span) cho 2 cú Đọc SELECT Phải Gắn Cùng Mã Giao Dịch (transaction identifier) để anh em Đội Điều Tra phân biệt được Lỗi Đọc Lại (non-repeatable read) khác với Việc Gọi Lệnh bậy bạ ở 2 Giao dịch chả liên quan gì nhau.
