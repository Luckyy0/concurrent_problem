# Mổ Xẻ Tầm Nhìn MVCC Của PostgreSQL

## 1. Trạng Thái Khởi Đầu (Initial state)

```text
job_id          = IMPORT-42
status          = RUNNING
progress        = 20
generation      = 7
last commit     = Tx-0 (Giao dịch số 0 đã chốt sổ)
```

Ông A (Processor) và ông B (Watchdog) dùng hai kết nối/giao dịch PostgreSQL độc lập, chả liên quan gì nhau.

## 2. Sự Cực Kỳ Ảo Tưởng Của Coder (Expected theo broken design)

```text
Ông A cập nhật tiến độ lên 80 nhưng chưa thèm chốt (chưa commit).
Ông B tới gõ cửa, xin phép ĐỌC_BẨN (READ_UNCOMMITTED).
Ông B hí hửng đọc được con số 80.
Ông B gật gù: "Job vẫn sống nhăn răng (HEALTHY)!".
```

## 3. Sự Thật Phũ Phàng Ở PostgreSQL (PostgreSQL actual)

```text
Ông A cập nhật tiến độ lên 80 nhưng chưa chốt (chưa commit).
Ông B tới gõ cửa, xin phép ĐỌC_BẨN (READ_UNCOMMITTED).
Ông B cay đắng chỉ nhận được con số 20 cũ mèm (đã commit từ đời nào).
Ông B hoảng hốt: "Chết rồi, Job bị kẹt! Gọi cứu hộ (START_RECOVERY) ngay!".
```

Nếu ông A thấy hối hận và Hủy kèo (Rollback), con số cuối cùng vẫn nằm ở `20`. Nếu ông A chốt sổ `80` *sau khi* B đọc xong, thì chỉ có thằng nào chạy một lệnh SELECT *mới* tinh bắt đầu sau đó mới thấy được số `80`. B lỡ đọc rồi thì ráng mà chịu cũ.

## 4. Dòng Thời Gian Ba Mặt Một Lời (Timeline ba actor)

| Bước | Ông A (Làm việc) — Tx-A | Ông B (Giám sát) — Tx-B | Đội Cứu Hộ C |
| --- | --- | --- | --- |
| T0 | BẮT ĐẦU Giao Dịch | | |
| T1 | SỬA (UPDATE) progress=80 | | |
| T2 | XẢ (FLUSH) xuống DB, Ngâm tiếp | | |
| T3 | | BẮT ĐẦU xin Đọc Bẩn (RU) | |
| T4 | | Chụp ảnh (SELECT) -> Thấy 20 | |
| T5 | | Ra quyết định: Báo Động! | |
| T6 | | CHỐT SỔ (COMMIT) | Bắt đầu chạy trùng lặp (duplicate work) |
| T7 | Tiếp tục mần ăn hoặc lỗi vặt | | |
| T8 | CHỐT SỔ (COMMIT 80) hoặc HỦY | | |

Chốt lại: Ở bước T4, chẳng có miếng Đọc Bẩn nào diễn ra cả! Lỗi nghiệp vụ (Business failure) ở đây là do ông B dùng việc "Không thấy tiến độ tăng" làm bằng chứng "Job bị kẹt", trong khi thực tế ông A vẫn đang hì hục làm việc.

> **Khắc cốt ghi tâm:** "Đọc Dữ Liệu Đã Chốt" và "Tín Hiệu Nhịp Tim (Liveness)" là 2 hợp đồng hoàn toàn khác biệt. Một cái Giao dịch đang mở toang hoác không thể thay thế cho một kênh Nhịp Tim (Heartbeat channel) đàng hoàng được!

## 5. Trò Ảo Thuật Phân Thân Của MVCC (MVCC tuple versions)

Khi ông A chạy lệnh SỬA (UPDATE), DB không đè bẹp dòng cũ, mà đẻ ra một cái bóng (version) mới:

```text
Bóng cũ: progress=20, tạo ra bởi Giao dịch Tx-0 đã chốt sổ.
Bóng mới: progress=80, tạo ra bởi Giao dịch Tx-A đang mấp mé chưa chốt.
```

Bức ảnh chụp (Snapshot) của ông B:
- Thấy bóng cũ `20` rõ mồn một.
- Giao dịch Tx-A chưa chốt nên cái bóng mới `80` tàng hình.
- B không cần phải đọc ba mớ byte lổn nhổn chưa thành hình.
- Lệnh Đọc Chay (plain SELECT) của B rảnh rang đi lướt qua cái bóng cũ mà chả thèm phải chờ đợi ông A nhả Khóa.

Ông A tự sướng thì luôn thấy được dữ liệu của chính mình sửa. Điều đó KHÔNG CÓ NGHĨA ông B được quyền nhìn ké.

## 6. PostgreSQL Chơi Chữ Mức Cô Lập (PostgreSQL isolation mapping)

Chuẩn SQL thế giới cho phép Đọc Bẩn ở mức `READ UNCOMMITTED`. Nhưng ông nội PostgreSQL thì cứng đầu tự cung cấp cơ chế bảo vệ mạnh mẽ hơn: Cứ xin RU thì bố cấp cho RC (`READ COMMITTED` - Chỉ Đọc Đã Chốt).

Có 2 sự thật bạn phải thông não:
1. **Cái Mác Bề Ngoài (reported/requested label):** Cài đặt biến `transaction_isolation` hoặc nhờ JDBC báo cáo, nó vẫn giả bộ nổ là `read uncommitted` cho bạn vui lòng.
2. **Hành Xử Thật Sự (effective visibility semantics):** Đọc Chay vĩnh viễn không thấy những dòng dữ liệu của thằng khác chưa chốt.

Vì thế, code test chuẩn phải là:
```text
Lưu lại cái Nhãn Dán để tiện bề mắng vốn.
Nhưng phải Assert vào Dữ Liệu Thật Sự nhận được để kiểm chứng độ chính xác.
```
Tuyệt đối không viết test lười biếng chỉ đi kiểm tra cái chuỗi chữ `"read uncommitted"` rồi tự sướng suy diễn là DB cho phép Đọc Bẩn.

## 7. Bức Ảnh Chụp Lại (Snapshot) Của Từng Câu Lệnh (Statement snapshot)

Lệnh Đọc Chay của B dùng cách "Chụp ảnh tại thời điểm bắt đầu Câu Lệnh":

```text
Bắt đầu SELECT trước khi A chốt -> Thấy 20 cho đến khi chạy xong dòng SELECT đó.
A chốt sổ xong.
Lệnh SELECT TIẾP THEO bắt đầu chạy -> Hên xui có thể thấy 80.
```

Sự khác biệt khi gõ 2 lệnh SELECT liên tiếp mà ra 2 kết quả "Đã chốt" khác nhau gọi là `Non-repeatable read` (Đọc Không Lặp Lại), sẽ được bóc phốt ở bài DB-003. Đừng nhầm lẫn nó với Đọc Bẩn!

## 8. Đọc Chay (Plain SELECT) Và Khóa Dòng (Row Lock)

Lệnh UPDATE của A ôm chặt cái Khóa Dòng (row-level lock) cho đến tận lúc Chốt/Hủy.
Lệnh Đọc Chay của B:
- Không xin xỏ cái Khóa đối nghịch `FOR UPDATE`.
- Lẻn đi đọc cái bóng cũ đã chốt.
- Tuyệt đối không rảnh háng đứng đợi nhà văn (writer) hiện tại.

Nhưng nếu B chơi hệ Lấy Khóa:
```sql
select *
from job_run
where job_id = :id
for update;
```
Trò này có thể bị Block (đứng đợi) ông A. Sau khi A giải quyết xong:
- A chốt sổ: B húp lấy dòng Dữ liệu Mới.
- A Hủy kèo (rollback): B ngậm ngùi cầm lại dòng `20` cũ kỹ.

Túm lại: "Đọc Lấy Khóa" thì đứng đợi kết quả cuối cùng; nó KHÔNG phơi ra Đồ Chơi Dở Dang (dirty version).

## 9. Hủy Kèo (Rollback) Và Nhặt Rác (Aborted versions)

Chuyện gì xảy ra nếu A Hủy kèo?
```text
Bóng mới 80 đẻ non -> Trở thành đồ phế liệu (aborted/dead).
Bóng cũ 20 -> Tiếp tục là hoa hậu cho mọi người chiêm ngưỡng.
```
Công đoạn quét rác vật lý (Physical cleanup) như vacuum/hint-bit kệ nó, nó thích quét lúc nào thì quét. Nhưng Tầm Nhìn Logic thì có luật thép: B không bao giờ được phép đọc vào cái đống phế liệu kia.

Đó là lý do tại sao dùng Đọc Bẩn cho tiến độ là chơi dao hai lưỡi, ngay cả trên cái Database cho phép Đọc Bẩn: Bạn có thể đưa ra quyết định chết người (recovery) dựa trên một cục Data bị Hủy mất tích không dấu vết!

## 10. Ranh Giới Giữa Việc Xả (Flush) Và Chốt (Commit)

Quy trình xả của Hibernate:
- Soi xem Object có dơ không (dirty-check).
- Gõ lệnh UPDATE vứt xuống DB.
- PostgreSQL lẳng lặng nặn ra một cái bóng chưa chốt.
- Hibernate ôm khư khư cái Khóa dưới DB.
- Và... chưa cho ai khác được xem cái bóng này.

Chỉ khi bạn gõ lệnh Commit thì cái bóng đó mới vinh quang lên ngôi. Dùng hàm `saveAndFlush()` KHÔNG PHẢI là trò bắn pháo hoa báo cáo tiến độ đâu nha!

## 11. Bóc Phốt Từng Tầng (Root cause theo layer)

| Tầng Lớp | Thói Quen | Lời Chê Bai |
| --- | --- | --- |
| Spring | Gửi lệnh xin Isolation qua transaction manager | Mấy cái thẻ Annotation chả có quyền lực đảm bảo các DB khác nhau hành xử giống nhau. |
| JDBC/driver | Chấp nhận cài cái Nhãn Dán Label | Cái Nhãn vớ vẩn không làm thay đổi được luân thường đạo lý MVCC của Server. |
| PostgreSQL | Xin RU anh cho mày ăn RC | Cửa đóng then cài, Đọc Bẩn là tội ác! |
| Hibernate | Đẩy Flush xuống mà không chịu Chốt | Chuyện hiển nhiên của Giao Dịch, đừng đổi thừa! |
| Code của Bạn | Dùng Dữ liệu Ảo ảnh để đòi Giám sát sự sống | Lỗi Thiết Kế Ngang Tầm Thảm Họa (Root design error). |

## 12. Ảo Tưởng Đồng Bộ Khắp Mọi Nơi (Portability)

Cùng một cái tên Isolation, chả có ông trời nào bảo đảm Database này sẽ cho ra lỗi y chang Database kia như bạn kỳ vọng; các nhà làm DB hoàn toàn có thể cài cắm cơ chế xịn hơn để bóp nghẹt các lỗi bẩn.

Nếu bạn thiết kế dựa dẫm vào Đọc Bẩn, thì kể cả đem qua xài cái DB lởm có Đọc Bẩn thật, bạn vẫn dính đòn:
- Đọc phải mớ Data rách nát chưa thành hình (partial/inconsistent).
- Quyết định việc lớn dựa trên cục Data 5 giây sau sẽ bị Hủy bỏ.
- Màn hình người dùng báo tiến độ 80%, F5 cái tuột xuống 20% (đi lùi).
- Các lệnh chạy sửa sai (compensation) đập vỡ hệ thống vì số liệu ảo ma.
- Code Test pass láng o ở máy mình, lên Server sụp hầm ở máy người khác.

Bản hợp đồng xịn xò (Correct contract) là phải dùng Tín hiệu Rõ Ràng (explicit messaging) hoặc Dữ liệu Đã Chốt, chứ không phải đi tìm mỏi mắt một con Database chứa chấp thói Đọc Bẩn của bạn.

## 13. Bài Học Cảnh Giác Cho Người Gác Cổng (Watchdog semantics)

Báo Cáo Tiến Độ và Chốt Sổ Thành Công là 2 thế giới tách biệt:
```text
Cố gắng Nhịp Tim/Tiến Độ (Heartbeat):
  Giao dịch tách biệt tự chốt sổ nhanh gọn lẹ. Có gắn ID, Generation, Hợp Đồng Thuê mướn rõ ràng.

Chốt kết quả cuối (Final status):
  Chỉ vứt vào mả (durable) khi 100% công việc xử lý đã ngon lành.
```

Người Gác Cổng đừng lười biếng chỉ liếc cái % tiến độ. Phải soi:
- Ai là chủ nhân của Hợp Đồng Thuê này? Token/Thế hệ (generation) của nó?
- Lần Nhịp Tim cuối cùng (đã chốt) là mấy giờ?
- Cho phép trễ hẹn bao lâu (Deadline)?
- Trạng thái cuối là Xong chưa (terminal status)?
- Dùng một cú lách Khóa Tranh Giành (atomic claim) để lao vào giải cứu.

Chỉ vì Tiến Độ Cũ Kỹ không tự động bật đèn xanh cho C lao vào Cứu Hộ. C phải thắng được kèo "Chiếm Quyền Giải Cứu" (conditional claim) để tránh thảm cảnh 100 ông C cùng hò dô đi cứu 1 cái Job.

## 14. Tác Hại Của Việc Ngâm Đồ Quá Lâu (Long transaction effect)

Nếu ông A mở đúng 1 cái Transaction ngâm tôm suốt cả ngàn bước tính toán:
- Cái tiến độ cứ tàng hình suốt buổi;
- Khóa, Connection, Ảnh Chụp (Snapshots) kẹt cứng dưới DB ngốn sạch RAM;
- Lỡ tay Hủy Kèo (Rollback) một cái là đổ sông đổ biển mọi công sức tính toán;
- Khoảng thời gian Mù Thông Tin (stale window) chà bá, Watchdog tha hồ hoảng loạn.

Hãy chia mẻ việc (workflow) thành các Trạm Nghỉ (checkpoints) hoặc Máy Trạng Thái (state transitions) để chốt sổ dần. Mỗi Trạm Nghỉ phải có danh phận rõ ràng, đừng vội vỗ ngực xưng tên "Hoàn Thành Chuyến Đi" khi mới tới ngã tư.

## 15. Chờ Lố Giờ Hay Hỏi Lại (Timeout và polling)

Hỏi han liên tục (Polling) thật nhanh chẳng qua là bắn cả tỉ câu SELECT và nhận về đúng một con số `20` cũ rích cho tới khi ông A chịu Chốt (Commit). Set Query Timeout (chờ lố giờ) chả ảnh hưởng gì đến Tầm Nhìn Dữ Liệu cả.

Nếu Watchdog đợi quá sốt ruột (timeout):
- Đừng có nhào vô nhắm mắt chạy đúp lại việc mờ mịt.
- Cố gắng Chiếm Lấy Hợp Đồng (Atomic lease/claim) một cách đàng hoàng.
- Hỏi kỹ lại cái Thế Hệ (generation/status) ngay trong lệnh Ghi Tranh Giành (conditional write).
- Code hàm Xử lý Đúp lại phải Tự Miễn Dịch (idempotent) phòng khi ông A tỉnh dậy đẩy đồ lần 2.

## 16. Kịch Bản Cháy Nổ (Crash behavior)

- A Sụp Bàn Cầu (crash) trước khi chốt: Kết nối đứt/Rollback; Tiến độ muôn đời là 20.
- A Chốt sổ 80 xong lăn đùng ra chết trước khi nhận hồi âm: Lệnh SELECT tiếp theo của Watchdog sẽ vớ được số 80 ngon ơ.
- Watchdog quyết định Giải Cứu xong tự lăn đùng ra chết: Phải lưu vết việc Đòi Quyền (claim) này thật bền bỉ.
- Recovery C mà cũng ngỏm củ tỏi: Phải có cơ chế Hợp Đồng Thuê cho chính C luôn.

Gào thét đòi Đọc Bẩn chả giúp gì được cho bạn trong khâu xử lý Dữ liệu Vô Định hay Nổ Máy ngang xương đâu.

## 17. Ngoại Lệ Về Sinh Số Tự Động (Sequence caveat)

Trò nhảy số đếm tự động (Sequence changes) trong PostgreSQL có luật riêng không chung mâm với trò Ghi Cập Nhật Dòng (Row update). Nó có thể lòi số ra ngoài và không bị Rollback như bình thường. Chớ dại mà lấy tính năng Sequence này ra để bảo kê cho luận điểm "Ở DB có Đọc Bẩn nghen mạy!".

## 18. Nỗi Đau Nhiều Server (Multi-instance)

Nhét một cái Biến Bộ Nhớ để vẫy cờ Nhịp Tim (Flag) ở Máy Chủ A chả có cửa nào báo tin lọt sang Máy Chủ B đang làm Watchdog. Giao tiếp nhiều Server bắt buộc phải dựa vào Dòng dữ liệu chốt sổ của DB, Nhắn Tin bền bỉ (durable message), hoặc kho lưu trữ Hợp Đồng Thuê (Lease store) có Phân Quyền rõ ràng.

Luật chơi Tầm Nhìn MVCC của PostgreSQL là nhất quán cho mọi cái App (instance) nào chọc vào nó. Đừng tưởng nhét vài cái Local Variable (Primitive) vào bộ nhớ JVM là bóp méo được luật của DB!

## 19. Đo Lường Bệnh Tình Sao Cho Trúng? (Observability)

Ghi chép (Log) lại mấy thứ sau:
- Cái Nhãn Cô Lập Cùi Bắp mà bạn Xin xỏ / DB phịa ra; Tên Hãng DB và Phiên Bản DB.
- Con số Tiến Độ / Thế Hệ đã chốt thật sự đọc được.
- Đếm thời gian sống thọ của cái Transaction phía Ông A.
- Đếm tuổi đời Nhịp Tim và Kẻ Nào Đang Sở Hữu Hợp Đồng Thuê.
- Các pha Ra Quyết Định Cứu Hộ / Kết quả Đòi Quyền Giải Cứu của Watchdog.
- Số lượng công sức đập đi xây lại (duplicate recovery).
- Đếm số Lần Chốt / Hủy của mấy cái Trạm Nghỉ tiến độ.

Đừng có Gào Thét Báo Động (Alert) ỏm tỏi chỉ vì cục Entity trong bộ nhớ bự hơn cục Dòng Dữ liệu Đã Chốt. Đó là bản năng Tầm Nhìn (MVCC visibility) hiển nhiên của vũ trụ này, chả phải do Replication Lag trễ nhịp hay Database Bị Mọt Ăn (corruption) đâu nha!
