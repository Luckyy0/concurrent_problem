# DB-002 — Lầm tưởng về Đọc Bẩn (Dirty Read) và Sự Thật Phũ Phàng ở PostgreSQL

## Tóm tắt câu chuyện

Tưởng tượng có một cái Job tên là `IMPORT-42` đang chạy được `20%`. Một thanh niên tên là Processor A đang hì hục cập nhật tiến độ (progress) lên `80%`. Anh này đã đẩy (flush) lệnh SQL xuống DB rồi nhưng **chưa chịu chốt sổ (chưa commit)** vì bận mở một cái Giao dịch (Transaction) dài lê thê.
Cùng lúc đó, một thanh niên khác tên là Watchdog B (đóng vai trò giám sát) nhào vô hỏi thăm xem Job có bị treo không. Thanh niên B này tự tin dùng mức cô lập là `@Transactional(isolation = READ_UNCOMMITTED)` (cho phép đọc bẩn) với niềm tin ngây thơ rằng: "Chắc chắn tao sẽ đọc được con số 80% kia, tao biết mày vẫn đang làm việc!".

Nhưng... đời không như mơ.
PostgreSQL cực kỳ ghét sự bẩn thỉu! Nó kiên quyết **không bao giờ** ném ra cái dữ liệu chưa commit. Thế là B ngớ người khi chỉ nhận được con số `20%`. B kết luận sai bét: "Á à, thằng Job này bị treo rồi!" và lôi ra một con clone khác để chạy lại từ đầu (duplicate recovery).

Nguyên tắc bất di bất dịch (Invariant) ở đây là:

```text
Mọi quyết định sống còn của hệ thống phải dựa trên dữ liệu đã chốt sổ (committed durable state).
Không một thành phần nào được phép sống dựa dẫm vào khả năng Đọc Bẩn (dirty-read).
Nếu bạn muốn báo cáo tiến độ ra ngoài, hãy chốt sổ thật nhanh từng khúc một (short commit).
Tuyệt đối không tung hô "Tôi thành công rồi" khi chưa vứt kết quả cuối cùng xuống mồ (chưa commit).
```

> **Túm cái quần lại:** Gào thét đòi `READ_UNCOMMITTED` ở PostgreSQL vô ích thôi; nó có thể gật đầu nhận cái label đó, nhưng cách hành xử của nó vĩnh viễn là "Chỉ đọc đồ đã chốt" (committed-only).

## Dàn diễn viên và Sân khấu (Actors và shared state)

| Thành phần | Vai trò |
| --- | --- |
| Processor A | Cặm cụi cập nhật tiến độ nhưng giấu giếm trong một cái Transaction chưa commit |
| Watchdog B | Cầm đèn soi xem tiến độ job đang tới đâu để phán quyết nó Khỏe hay Treo |
| Recovery C | Đội cứu hộ, chực chờ nhảy vào làm lại từ đầu nếu ông B báo "Treo" |
| Dữ liệu `job_run` | Nơi lưu trữ trạng thái chân lý cuối cùng (đã commit) |
| PostgreSQL MVCC | Vị trọng tài quyết định B được nhìn thấy phiên bản dữ liệu nào |
| Spring/JDBC | Kẻ đưa tin, mang yêu cầu xin "Đọc bẩn" của B gửi xuống DB |

Lúc đầu, dữ liệu đã chốt dưới DB là:

```text
job_id      = IMPORT-42
status      = RUNNING
progress    = 20
generation  = 7
```

Ông A lén lút sửa (chưa chốt):

```text
progress = 80
```

Ông B vào xem, kết quả đắng lòng:

```text
progress = 20
```

## Khúc cua tử thần (Transaction boundary và contention point)

**Hành trình của A:**

```text
Mở Giao Dịch -> SỬA progress=80 -> XẢ xuống DB -> Ngồi thiền -> CHỐT SỔ (hoặc Hủy)
```

**Hành trình của B:**

```text
Mở Giao Dịch xin ĐỌC_BẨN (READ_UNCOMMITTED)
  -> Lệnh SELECT chay
  -> Thấy con số 20 cũ mèm
Chốt Sổ
```

Điểm nổ ở đây chính là cái dòng trạng thái `job_run(IMPORT-42)`. MVCC của Postgres quá xịn nên lệnh SELECT chay của B chẳng thèm chờ ông A nhả Khóa. Nó lục lại luôn cái phiên bản "đã chốt" cũ rích (20%) cho B xem. Đây là sự "Lệch Pha Góc Nhìn" (visibility mismatch), chứ không phải lỗi kẹt Khóa (lock timeout) đâu nhé.

## Đời Không Trả Cát Xê (Expected theo broken design và PostgreSQL actual)

| | Ảo tưởng của Coder (Broken design) | Sự thật phũ phàng ở Postgres (Actual) |
| --- | --- | --- |
| Xin xỏ Isolation | Chắc chắn DB cho mình Đọc Bẩn! | Nó giả vờ cho nhưng xử lý y như Đọc Chốt (READ_COMMITTED) |
| Trước khi A chốt sổ | B hí hửng nhìn thấy 80 | B chỉ thấy 20 |
| Lỡ ông A Hủy kèo (Rollback)? | UI phải tìm cách "tẩy não" để quên đi con số 80 | 80 chưa từng tồn tại trong mắt ai, chả phải lo |
| Lênh SELECT chay có bị Block không? | Không | Không; nó rảnh rang đi đọc dữ liệu cũ |
| Chạy trên các DB khác nhau? | Đinh ninh DB nào cũng như DB nào | Sai bét, mỗi DB một tính nết rạch ròi |

Nói tóm lại: Nếu A commit xong xuôi, một giao dịch mới mở ra vào xem mới thấy được `80`. Nếu A rollback, B vẫn vui vẻ ngắm con số `20`.

## Tự Điển Thuật Ngữ (Thuật ngữ cần biết)

| Thuật ngữ | Ý nghĩa dân dã |
| --- | --- |
| Đọc bẩn (dirty read) | Tọc mạch đi đọc data của thằng khác khi nó chưa thèm chốt sổ |
| Phiên bản đẻ non (aborted version) | Data của một giao dịch bị Hủy (rollback) nửa chừng |
| Mức cô lập xin xỏ (requested isolation) | Cái Level mà code Java / JDBC của bạn "vái lạy" xin DB cấp cho |
| Nhãn dán bề ngoài (reported label) | Cái tên `transaction_isolation` DB hiện lên cho bạn xem để an ủi |
| Cách hành xử thật (effective semantics) | Cái cách mà Database nó quăng Data vào mặt bạn thật sự |
| committed-only | Tôn chỉ: Chỉ phơi ra ánh sáng những Data đã được chốt sổ |
| Chốt tiến độ (progress checkpoint) | Báo cáo tiến độ riêng biệt, lưu ngay lập tức, không đợi toàn bộ công việc xong |
| Lỗi ảo tưởng đồng bộ (portability assumption) | Việc bạn ngây thơ tin rằng cứ gõ chung 1 tên Isolation Level thì DB nào (MySQL, Postgres...) cũng hành xử y chang nhau |

## Điều hướng

- [Đoạn Code Ảo Tưởng (Broken dirty-read design)](broken-code.md)
- [Mổ xẻ Tầm Nhìn PostgreSQL (PostgreSQL visibility analysis)](analysis.md)
- [Giải pháp báo cáo Tiến Độ Chuẩn (Committed coordination and progress solutions)](solutions.md)
- [Thí nghiệm Đập Tan Ảo Tưởng (Deterministic visibility experiments)](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Isolation levels](../../concepts/isolation-levels.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả rớt nước mắt trên Production

- Ông B (Watchdog) bù lu bù loa báo động nhầm, khiến đội Cứu hộ (Recovery) chạy đúp lại việc đã làm, sinh ra rác.
- Màn hình Dashboard báo tiến độ đứng im re, mãi đến lúc chạy xong mới nảy cái vù lên 100%.
- Bê code từ Database khác sang Postgres là ăn đòn ngay, vì luật chơi Đọc Bẩn đã bị thay đổi.
- Các pháp sư ráng code thêm cơ chế "Hỏi lại liên tục" (polling) vô vọng, vì hỏi 100 lần vẫn chỉ ra cục Data cũ xì.
- Thích mở Giao dịch (Transaction) rõ dài, ôm cục Data vô hình quá lâu làm người vận hành ngơ ngác chả hiểu hệ thống đang chạy tới đâu.
- Còn nếu Đọc bẩn CÓ THẬT ở DB khác, bạn sẽ bị hớ nặng vì dám dùng số liệu đó để xử lý logic, xong ông kia Hủy Kèo (Rollback) một cái là bạn ôm một mớ rác!

## Lời Khuyên Từ Quản Gia (Hướng sửa khuyến nghị)

Bỏ ngay cái thói dùng "Dữ liệu chưa chốt" để bắn tin cho nhau! Hãy chọn 1 trong 3 con đường sáng:

1. **Đứng nhìn từ xa:** Chấp nhận rằng chỉ có Data đã chốt mới đáng tin (committed-only progress).
2. **Báo cáo kiểu chim mồi:** Sinh ra những Giao dịch cực ngắn (short independent transaction) để lưu trạng thái "Tôi đang ráng chạy tới đây nè" (attempt progress), mục đích chỉ để vẫy cờ báo tiến độ, chứ không phải chốt kết quả cuối cùng.
3. **Thuê bảo vệ (Lease/Heartbeat):** Dùng cơ chế Nhịp tim hoặc Thuê mướn để phối hợp giám sát, xịn sò hơn nhiều.

Nhớ nhé: Nếu làm chuyện đại sự mất thời gian dài, hãy xẻ nó ra thành các cỗ máy trạng thái (state machine) / outbox hoặc chia nhỏ checkpoint ra mà chốt sổ dần. Đừng bao giờ ôm khư khư một cái Transaction dài cả dặm!

## Phạm vi bài học

Trường hợp này chỉ xoáy vào trò MVCC của PostgreSQL đập tan giấc mộng Đọc Bẩn (dirty-read) và sự khác biệt giữa các DB (portability).
Còn vụ 2 ông cùng chốt rồi đọc thấy Data khác nhau (Non-repeatable read) thì đón xem ở tập DB-003 nhé. Mấy trò Mất Data (Lost update) hay Ghi Bẩn (dirty writes) cũng có sân khấu riêng!
