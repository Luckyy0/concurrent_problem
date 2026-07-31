# Giải phẫu: Xí Chỗ Trước Rồi Mới Suy Nghĩ (FOR UPDATE)

## 1. Mọi Thứ Bắt Đầu Thế Nào? (Initial state)

```text
Cái ghế số "A-10", Suất chiếu "42":
  Trạng thái: AVAILABLE (Đang trống)
  Ai đang giữ: Chả ai cả (null)
  Người giữ đến khi nào: null

Sổ lịch sử (seat_hold): Trống trơn []
```

Giả sử hai máy chủ (App-1 và App-2) nhận lệnh từ hai khách hàng A và B gần như cùng một lúc. Cả hai luồng Giao Dịch đều thề sẽ đặt được cái ghế `A-10` này.

## 2. Kịch Bản Ác Mộng (Timeline code hỏng)

Nếu bạn code ngây thơ bằng câu lệnh SELECT thường:

| Bước | Máy chủ 1 (Khách A) | Máy chủ 2 (Khách B) | Dưới gầm Database |
| --- | --- | --- | --- |
| 1 | Mở Giao Dịch | | |
| 2 | Nhìn thấy ghế `AVAILABLE` | | Chả thèm khóa gì cả! |
| 3 | | Mở Giao Dịch | |
| 4 | | Cũng nhìn thấy ghế `AVAILABLE` | Hai thằng đều thấy giống nhau |
| 5 | Gõ búa: CHẤP NHẬN A | Gõ búa: CHẤP NHẬN B | Hai ông nội tự quyết định ngầm với nhau |
| 6 | Bắt đầu lưu cho A | | Bắt đầu tung Khóa Dòng (Row Lock) |
| 7 | Chốt sổ xong (Commit) | | A đã ghi xong tên vào ghế |
| 8 | | Bắt đầu lưu cho B | B chèn đè tên mình lên tên A! |
| 9 | | Chốt sổ xong (Commit) | Xong phim! 1 ghế 2 chủ! |

Database có cấp Khóa Dòng lúc chạy lệnh `UPDATE`, nhưng nó làm sao biết được cái hàm `isAvailable()` bằng Java của bạn nằm tít trên kia đã chạy và trả về `True` từ đời nào rồi? Code không sai Cú Pháp, nó sai Logic Kinh Doanh.

## 3. Đời Không Như Mơ (Expected và actual)

| Tiêu chí | Kỳ Vọng | Thực Tế (Khi xài SELECT chay) |
| --- | --- | --- |
| Số lượng ĐANG GIỮ (ACTIVE holds) | 1 | 2 (Khóc thét) |
| Liên kết dữ liệu Ghế | Trỏ đúng người đang giữ vé hợp lệ | Trỏ vào ông nào Ghi cuối cùng |
| Báo cho kẻ thất bại | Lỗi `ALREADY_HELD` | Chúc mừng bạn đã mua thành công! (`HELD`) |
| Lịch sử đối soát | 1 lệnh được duyệt | 2 lệnh được duyệt |

## 4. Kịch Bản Cứu Thế (Timeline với `FOR UPDATE`)

Khi xài Khóa Bi Quan:

| Bước | Máy chủ 1 (Khách A) | Máy chủ 2 (Khách B) | Dưới gầm Database |
| --- | --- | --- | --- |
| 1 | Mở Giao Dịch | | |
| 2 | Lấy ghế và Hét lên: **KHÓA!** | | A chính thức cầm Chìa Khóa Dòng |
| 3 | Đọc thấy ghế `AVAILABLE` -> Lưu A | | Khóa vẫn nằm trong tay A |
| 4 | | Lấy ghế và CŨNG Hét: KHÓA! | Khóa bị A cầm rồi -> B BỊ BẮT XẾP HÀNG CHỜ |
| 5 | Đẩy dữ liệu xuống (flush) | Ngáp ruồi đợi... | |
| 6 | Chốt sổ (Commit) | Ngáp ruồi đợi... | Dữ liệu cập nhật xong, Khóa rớt ra |
| 7 | | Lượm Khóa -> Thấy ghế đã HELD | B tỉnh giấc, cầm được Khóa, đọc lại dữ liệu mới nhất |
| 8 | | Thẩm định lại -> Quăng lỗi `ALREADY_HELD` | B phải quay xe từ bỏ |
| 9 | | Hủy Giao Dịch (Rollback) | Trả Khóa |

Điểm mấu chốt: **B bắt buộc phải đọc lại dữ liệu mà A vừa ghi** (Chứ không được bám víu vào cái ảo ảnh lúc nãy nữa). Các tiến trình không còn được phép quyết định cùng lúc nữa.

> **Nói ngắn gọn:** Kẻ đợi khóa không được quyền mang cái quyết định cũ ra dùng; nó chỉ được phép đưa ra phán quyết SAU KHI tận mắt nhìn thấy Kẻ trước mặt đã chốt sổ (commit) hay hủy kèo (rollback).

## 5. Bóng Ma Dữ Liệu ở Chế độ `READ COMMITTED`

Với các câu lệnh đọc bình thường (plain reader), mỗi câu sẽ lấy một bản "Chụp màn hình" (Snapshot) dữ liệu đã được Chốt. Do đó nó mặc kệ cái Khóa `FOR UPDATE`, không ai cấm nó xem data cũ cả.

Nhưng khi xài `SELECT ... FOR UPDATE`, câu chuyện rẽ hướng khác:
1. Nó tìm dòng theo Khóa Chính.
2. Nếu có người đang giữ khóa (incompatible lock), nó phải đứng chờ.
3. Kẻ kia buông khóa xong, PostgreSQL sẽ túm cái Khóa cho nó và tung cho nó dòng dữ liệu **ĐÃ CẬP NHẬT MỚI NHẤT**, (hoặc méo nhả ra dòng nào nếu bị kẻ kia xóa mất).
4. Bạn ôm dòng dữ liệu mới tinh đó về App để tự chạy lại các bước kiểm định (`state`, `hold_until`).

Bởi vì nó truy tìm theo Khóa chính nên lúc nào nó cũng chộp được dòng Dữ liệu (trừ khi dòng đó bị xóa), nhờ vậy mà B bóc trần được sự thật là ghế đã `HELD` sau khi A commit. Nếu A rollback, B sẽ vui vẻ nhận được `AVAILABLE`.

(Nhiều người xúi chuyển sang mức độ cô lập `REPEATABLE READ` nhưng coi chừng! Nó sẽ ném cho bạn rổ lỗi Sập Giao Dịch - serialization failure, thay vì tạo ra cái quy trình "Chờ - và - Thẩm Định lại" đẹp đẽ như chúng ta đang xây dựng).

## 6. Khóa Nào Vừa Bị Giật Vậy?

Lệnh `SELECT ... FOR UPDATE` lấy một cái Khóa bảo kê Cả Bảng (để không ai vô xóa bảng lúc đang xài) và giật cái Khóa Dòng (row-level lock) cho đúng cái Dòng dữ liệu đó. Các giao dịch khác muốn `UPDATE`, `DELETE` hay cũng đòi Khóa Đọc trên Dòng đó đều phải nhăn nhó Đứng Chờ hoặc văng lỗi.

Lưu ý: Khóa Dòng không có nghĩa là "Khóa Cả Bảng", và nó CHẢ THÈM CẢN mấy câu `SELECT` bình thường (người dùng xem Web vẫn load danh sách ghế ào ào). Nó cũng chả rảnh đi khóa những cái ghế Không Tồn Tại.

Khi gắn cờ `PESSIMISTIC_WRITE` trong Hibernate, hãy nhớ xuống tận nơi xem mã SQL mà Hibernate sinh ra là gì. Độ chính xác nằm ở câu SQL chứ đừng tin mù quáng vào mấy cái Annotation. Nhớ là Giao dịch DB phải ôm trọn toàn bộ quá trình Đọc và Ghi.

## 7. Tuổi Thọ Của Một Chiếc Khóa (Lock lifetime)

Khóa sống bao lâu?

```text
MỞ GIAO DỊCH (BEGIN)
  Cài đặt giờ nổ (lock_timeout)
  SELECT ... FOR UPDATE  ← Bắt đầu Xin Khóa ở đây nè!
  Thẩm định lại (revalidate)
  Lưu / Cập nhật / Đẩy xuống DB
CHỐT SỔ HOẶC HỦY KÈO (COMMIT/ROLLBACK) ← Khóa chết ở đây!
```

Khóa **KHÔNG HỀ ĐƯỢC NHẢ RA** khi:
- Bạn vừa chạy xong cái Hàm Repository.
- Object Java bay ra khỏi 1 block code.
- Bạn gọi một cái API gọi điện báo cho User.
- Hibernate xả xong lệnh UPDATE xuống DB nhưng Giao dịch (Transaction) vẫn chưa Chốt.

Tóm lại: Khóa sẽ ôm chân Giao Dịch đến chết. Độ dài Giao dịch chính là giới hạn lý tưởng nhất cho Tuổi thọ của Khóa (không tính đoạn chờ ở cửa).

## 8. Cẩn thận Cái Bóng Của Spring Proxy và Flush

Cái cờ `@Lock` chỉ là vẽ bùa lên hàm Repository. Cái ranh giới Giao Dịch lại do `@Transactional` quyết định, thường nằm ở cái Proxy đứng ngoài cùng:

```text
Thằng Caller gọi API
  → Proxy mở Giao Dịch (BEGIN)
  → Hàm Repository đòi Khóa Dòng
  → Thay đổi dữ liệu Java
  → Hibernate xả SQL (flush)
  → Proxy chốt Giao Dịch dưới DB (COMMIT)
  → Proxy trả kết quả
```

Nếu bạn cố tình gọi tắt (self-invocation) né cái Proxy, lệnh khóa sẽ tạch ngay vì chả có Giao Dịch nào bảo kê.
Cơ chế Dirty checking thường tự nhét lệnh UPDATE vào phút chót. Việc gọi thủ công `flush()` sẽ giúp dồn các lỗi xung đột văng ra ngay trong thân hàm để dễ bề đối phó, nhưng dữ liệu thì chỉ thực sự an bài sau khi Proxy gật đầu Commit.

## 9. Phán Quyết Cuối Cùng: Kẻ Thắng và Người Thua

### Kẻ giữ Khóa (Holder) Chốt sổ thành công (Commit)
Dữ liệu của Khách A biến thành Sự thật rồi nhả Khóa. Khách B được đánh thức, cầm Khóa và nhìn thấy cái ghế đã `HELD`. B ngậm ngùi từ chối vì lỗi Logic. B không được quyền xài lại cái Snapshot ảo ảnh cũ nữa.

### Kẻ giữ Khóa Hủy Giao Dịch (Rollback)
Khách A hủy kèo. Dữ liệu chưa từng tồn tại, ghế vẫn `AVAILABLE`. Khách B tỉnh dậy, cầm Khóa, thấy ghế vẫn y nguyên liền chốt đơn và Commit. Chẳng cần ai đi dọn dẹp Khóa thừa cả.

### Mạng đứt, Rớt mạng, Máy chủ cháy
Nếu máy của A đang cầm Khóa bỗng dưng bốc khói tắt nguồn, PostgreSQL thấy rớt mạng liền tự Hủy Giao Dịch của A và nhả Khóa cho B.
Lưu ý quan trọng: Lỗi mất kết nối (Response mất) KHÔNG ĐỒNG NGHĨA với Giao dịch Hủy. Nếu DB đã Commit xong xuôi rồi đường truyền mới đứt, App phải dùng mã Command ID dò lại xem lệnh đó đã vô chưa, cấm được võ đoán!

## 10. Chờ Mòn Mỏi và Bom Hẹn Giờ (Timeout và aborted)

Thông số `lock_timeout` quy định: "Tao chỉ cho mày đứng chờ xin Khóa chừng này giây thôi". Vượt quá giờ hạn định, DB quăng cục Lỗi `55P03` (`lock_not_available`). Framework sẽ tùy chỉnh tên Lỗi, nên hãy log nguyên cái SQLSTATE lại để biết đó là do Quá Giờ chứ không phải lỗi Logic.

Khi dính lỗi từ Câu SQL, Giao Dịch của bạn coi như BỎ ĐI, đừng hòng làm ăn gì tiếp trong đó:

```text
Xin khóa quá lâu -> Đứt bóng
→ Văng Lỗi bung ra khỏi hàm Giao dịch
→ Spring cắm cờ Phá Sản (Rollback)
→ Cấp trên hốt xác, báo lỗi BUSY cho khách hoặc khởi động lại 1 Giao dịch mới tinh.
```

Đừng nhầm lẫn giữa `lock_timeout` (Giờ chờ Khóa) với `statement_timeout` (Giờ chạy lệnh) hay Giờ Timeout của cả cái Server! Ngân sách Giờ Chờ Khóa phải luôn nhỏ hơn Tổng Giờ cho phép Request tồn tại.

## 11. Chờ Đợi, Không Chờ, hay Bỏ Qua Luôn?

| Chính sách | Cách nó chạy | Xài khi nào? |
| --- | --- | --- |
| **Chờ có Hạn (Bounded wait)** | Đứng chờ rồi kiểm tra lại | Xài khi khách nhắm đích danh 1 tài nguyên, thời gian Giao Dịch cực ngắn |
| **KHÔNG CHỜ (`NOWAIT`)** | Fail ngay nếu bận | Xài khi Caller có phương án Hứng Lỗi rõ ràng |
| **BỎ QUA LUÔN (`SKIP LOCKED`)** | Ai rảnh chờ, lấy dòng khác! | Bọn Worker lấy đồ trong hàng đợi (Ai lấy đồ nào cũng được) |
| **Chờ Tới Mùa Quýt (Unbounded wait)** | Hên xui phụ thuộc thằng cầm Khóa | **Đừng biến nó thành Cấu Hình Mặc Định API** |

Với trò chọn ghế `A-10`, xài `SKIP LOCKED` là bậy. Thằng bạn đang mua, nhẽ ra báo là Đang Có Người Giữ, bạn lại báo "Không Tìm Thấy Ghế Đó", làm khách chửi ầm lên. Xài Bounded Wait hoặc NOWAIT là rõ ràng nhất.

## 12. Khóa Nhiều Dòng Cùng Lúc và Kẹt Xe Kép (Deadlock)

Nếu muốn khóa 2 ghế liền nhau, BẮT BUỘC phải khóa theo đúng 1 thứ tự vĩnh cửu:

```text
Sắp xếp (Ghế ID nhỏ đến lớn)
→ Khóa từng dòng theo thứ tự đó
→ Thẩm định lại toàn bộ
→ Cập nhật toàn bộ
```

Chứ để thằng X khóa `A-10` rồi xin `A-11`, thằng Y khóa `A-11` rồi xin `A-10`, thì hệ thống tạo ra Bẫy Chết (Deadlock). Thám Tử của PostgreSQL sẽ rút súng tỉa 1 thằng (Victim - SQLSTATE `40P01`) và bắt Rollback. Ép thứ tự không đảm bảo diệt 100% Deadlock nhưng nó giảm được rất nhiều.

Chỉ được thử lại Deadlock khi: Giao dịch cũ đã dọn sạch, lệnh an toàn không sợ bị double, và phải quy định số lần/thời gian thử lại chặt chẽ. (Chi tiết xem `DB-008`).

## 13. Lệnh Trùng Lặp Khác Tranh Giành

Cái cờ Unique trên cột `command_id` giúp cản vụ khách spam bấm nút 2 lần tạo ra 2 dòng lịch sử đè nhau.
Khóa Dòng giúp cản hai ông A và B giành chung 1 cái ghế.
Khóa Dòng không hề biết cái vụ ông A bấm gửi `command_id` cho một khách C khác. Hai vũ khí này giải quyết 2 bệnh lý hoàn toàn độc lập.

## 14. Ảo Tưởng Của Việc Đa Máy Chủ (Multi-instance)

Vì chữ `FOR UPDATE` cắm rễ tận dưới PostgreSQL, nên dù bạn chạy App-1 và App-2 riêng rẽ, chúng vẫn xông vào cắn xé nhau tại cửa ngõ DB. Sự bá đạo của nó là dập tắt mấy trò khóa bằng mã Java (`synchronized`, `ReentrantLock`), vì mã Java làm sao ép máy chủ nhà hàng xóm nghe lời?

Chạy càng nhiều App, số thằng ngáp ruồi Đợi Khóa càng cao, Pool càng dễ kẹt.

## 15. Đoàn Tàu Kẹt Bến và Sự "Không Công Bằng"

Ông nào ôm Khóa lâu, sinh ra hàng chờ thê thảm tại cái ghế Hot. Bọn chầu chực sẽ hút cạn lượng Connection của Database. Hơn nữa, PostgreSQL không hề trao cúp công bằng "Ai Xếp Hàng Trước Được Phát Trước". Lỗi Timeout, bị hủy diệt hay lịch chạy của Hệ điều hành đều có thể đảo lộn thứ tự hàng chờ.

Bắt buộc phải giám sát:
- Tốc độ giật Khóa.
- Số lượng văng Lỗi Giờ (`55P03`) và Lỗi Kẹt Chết (`40P01`).
- Thời gian chạy sau khi đã giật được Khóa.
- Cạn kiệt Connection Pool.
- Ai dính `HELD`, ai dính `BUSY`.
- Mấy ông mở Giao dịch xong đi ngủ (idle-in-transaction).

## 16. Lôi Đầu Lên Xem Đứa Nào Cầm Khóa

```sql
select a.pid,
       a.application_name,
       a.state,
       a.wait_event_type,
       a.wait_event,
       a.xact_start,
       pg_blocking_pids(a.pid) as thang_nao_dang_chan_duong,
       a.query
from pg_stat_activity a
where a.datname = current_database()
  and a.application_name like 'lock003-%';
```

Bạn sẽ thấy chúng chờ nhau ở cái Transaction ID Lock. Bảng theo dõi này chỉ dùng để khám bệnh, đừng dại chèn Mật khẩu vô Log nhé.

## 17. Tra Hỏi Tội Lỗi Từng Tầng Lớp (Root cause)

### Lớp Application (Code Nghiệp Vụ)
Chuỗi `Đọc -> Quyết định -> Ghi` bị đứt khúc. 2 ông hùa vào quyết định trên 1 đống data cũ mềm.

### Lớp Spring
Cắm cờ `@Transactional` sai chỗ hoặc cái hàm quá ngắn làm Khóa bị vứt đi trước khi thực sự cần.

### Lớp Hibernate/JPA
Đọc bình thường thì đâu ai nhét giùm chữ `PESSIMISTIC_WRITE`? Quá trình Ghi rà soát lại (dirty checking) lại diễn ra SAU KHI não bạn đã phán xét xong, Khóa tự động đến vào lúc sự đã rồi.

### Lớp PostgreSQL
Chế độ `READ COMMITTED` cho phép thoải mái "Nhìn Về Dĩ Vãng" (đọc data cũ). Lúc nhào vô `UPDATE` nó có Khóa đấy, nhưng Khóa lúc đó thì cái Code Java kia đã quyết cái rụp rồi.

## 18. Ngoài Vùng Phủ Sóng

Trò này chỉ linh nghiệm khi **Bạn Khóa 1 Cái Dòng (Row) Đã Biết Rõ Và Đang Nằm Sờ Sờ Ra Đó**. Nếu yêu cầu của Sếp là "Đảm bảo cả ngày hôm nay không có ai đặt lịch trùng 8 giờ tối", thì bạn không thể Khóa cái dòng "Không có thật" (Phantom row). Lúc đó phải nâng cấp vũ khí lên xài Bảng Ràng Buộc, Khóa nhân tạo, hoặc `SERIALIZABLE`.
