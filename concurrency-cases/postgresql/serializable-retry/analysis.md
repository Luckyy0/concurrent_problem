# Khám Bệnh Đứt Mạch SSI, Đòn Đau `40001` Và Tuyệt Chiêu Hồi Sinh (Transaction retry)

## 1. Điểm Xuất Phát (Trạng thái ban đầu)

```text
Trần số đo (limit) của bác merchant 7 = 100
Tiền cọc ACTIVE đã chốt ném vào (committed) = 60
Thằng C1 đòi nã = 30
Thằng C2 cũng ngoạm = 30
```

Hai đứa T1 và T2 đều lôi bộ giáp `SERIALIZABLE` ra mặc. Mỗi ông tự mang theo ống thở (connection), bong bóng giao dịch (transaction), cái hộp phép nháp (persistence context) và tờ thẻ bài phân thân (command ID) riêng biệt nhau nhé.

## 2. Đường Ray Tử Mệnh Phải Bước (Timeline bắt buộc)

| Nhịp (Bước) | Bác T1 — Xách Lệnh C1 | Bác T2 — Xách Lệnh C2 |
| ---: | --- | --- |
| 1 | Bật Máy Bơm `BEGIN SERIALIZABLE` | Cắm Ống Bơm `BEGIN SERIALIZABLE` |
| 2 | Móc túi đéo thấy án mạng C1 cũ | Soi ví đéo thấy lịch sử C2 cũ |
| 3 | Đo trần `100`, đếm tổng active `60` | |
| 4 | | Vén váy đếm trần `100`, tổng active `60` |
| 5 | Gật gù phán: `60 + 30 <= 100` Ngon! | Vỗ tay chốt: `60 + 30 <= 100` Xúc! |
| 6 | Nặn cục cọc nháp reservation C1 | Bơm cục cọc nháp reservation C2 |
| 7 | Viết thẻ xăm phán `ACCEPTED` | Kéo tờ lịnh `ACCEPTED` |
| 8 | Bấm dội bùn / Xả chốt (flush/commit) | Bấm dội bùn / Xả chốt (flush/commit) |
| 9 | (Ví dụ) Thoát cửa sướng Commit Dính | Lãnh 1 nhát súng `40001`, cuộn vỡ mồm (rollback) |
| 10 | Đẩy tổng active trồi lên móc `90` | Trẻ trâu hồi sinh fresh retry đong vội được `90`, sợ té đái ghim thẻ `REJECTED` |

Ông Cảnh Sát PostgreSQL có quyền nã súng bốc hơi (abort) thằng T1 thả thằng T2 cũng được, chả có gì cấm. Lệnh bắn cũng có thể phát nổ ngay từ trước khi mày rặn nút `COMMIT`! Trình viết code/test chuẩn bài (correct) đéo bao giờ đi đánh đu đo độ hên xui coi "Ai chết - Ai sống" hay là soi cái Lỗi nó văng ra trúng ngay dòng Java số mấy đâu nhen nhóc!

> **Sếp chốt lại:** Tấm lưới SSI không hề thiên vị đánh dấu "Đứa tới sau ráng chịu" làm con rối loser; Nó chỉ cần giữ đinh một điều: Tổng đàn transaction đã nhét trót (commit) phải khớp nhau chạy thẳng thành một vạch đẹp trật tự đàng hoàng (serial order hợp lệ).

## 3. Mơ Đẹp Và Hiện Thực Vỡ Mộng (Kết quả mong đợi và thực tế)

| Trạm Đo (Khía cạnh) | Sạch Mịn Màng (Đúng) | Sứt Mẻ Tùy Tùng (Broken) |
| --- | --- | --- |
| Lưới Luật (Invariant) | Chốt mốc active cuối = `90` | Máu lỗi `40001` trào tung toé hoặc rặn retry xé ngõ (retry sai) |
| Lệnh C1/C2 | Một thẻ `ACCEPTED`, một cái vé `REJECTED` sau khi tái tạo | Một đứa command tụt hố mất tiêu số phận (mất outcome) |
| Trạm Nạn Nháp (Attempt lỗi) | Bể sạch sẽ tuột về đáy (Rollback toàn bộ) | App lỳ lợm đi cắm tiếp trên cái sình lầy vỡ `25P02` |
| Bảng Cóp Ảo (Snapshot) | Mở màng tươi mới rà ngay ra bóng Thằng Chốt Ván (winner) | Cứ bám cái giẻ rách nháp cũ soi mãi số `60` |
| Vết Loang Lực (Side effect) | Chỉ đẩy loa phát cho quyết định đã chốt dính commit | Thằng chết đứt aborted lại đi khua loa bắn notification ẩu! |

Bảo kê! Nếu tụi bay xách mền `READ COMMITTED` hoặc `REPEATABLE READ` nhét qua cho cái đám đo/sửa này, 2 ổ đẻ C1 C2 đều lọt commit tuốt, kéo cái tổng lủng trần bay lên mịa `120` cmnl! Méo có vụ đè sửa lộn mâm same-row lost update để tụi bay tự tỉnh phát hiện lỗi đâu nha!

## 4. Dấu Ánh Tích Tịch Của Kẻ Cao Thủ `SERIALIZABLE` (Snapshot của `SERIALIZABLE`)

Chiêu `SERIALIZABLE` của PostgreSQL kéo giãn ra 1 màng nháp ổn định (stable transaction snapshot) hệt như món nền cơ bản snapshot isolation. Chú T1 đéo thấy đống cặn C2, và T2 cũng mù mờ chẳng dòm ra hũ cứt C1 trong màng soi của mình.

Điểm thét lọt màng của SSI là nó có con mắt thần soi đường nối (dependency) để chặn đám đã thành mâm (committed history) KHÔNG THỂ trượt ray bừa bãi. Bác ổng KHÔNG hề lấy mọi cú soi lỗ khóa gông lại (predicate read thành blocking lock) và chả ép 2 thằng transaction tự lết bò theo hàng dọc (tuần tự) đâu!

Khối cặn mày đọc trong 1 giao dịch serializable chỉ được vinh danh là đồ chốt hợp lệ SAU KHI chú mày ghim sập nút búa commit thành công. Cái cục Java Result lôi lên trên tay vẫn là đồ ẢO chừng nào mâm commit advice chưa phán xong nhé!

## 5. Mảnh Xích Hút Báo Tín `SIReadLock` (Và theo dõi predicate)

Khi hai phe đâm rạch gọi `SUM ... WHERE merchant_id=7 AND status='ACTIVE'`, Trùm Sò PostgreSQL dán mác xăm đánh dấu bãi đất vừa hóng bằng chốt xích vùng (predicate locks). Xuống rạp `pg_locks`, tụi nó khoác mác `SIReadLock` rải đều kích cỡ từ tép tuple, đến cuốn page hay cả mớ relation rần rần tùy hứng cái plan lôi súng đếm và bùa lock promotion của DB nhen.

Đám xích ma này:

- CHẢ THÈM bắt giặc chặn cửa bọn viết (writer) đứt cổ như `FOR UPDATE` đâu;
- Chỉ dùng để rà đèn xem cú ghi (write) nào dám làm lem luốc cái vùng mà tao vừa đọc hồi nãy;
- Có khi còn ráng neo giữ dù đã sút búa commit, nén hơi nhịn tới lúc phe đám đâm thọc (overlapping read-write) tụt sạch;
- Coi chừng nó còn trùm lọt ra to đùng oái ăm (coarse hơn dự kiến) nếu nẹt ga sequential scan hoặc bộ nhớ nhồi nén predicate-lock kẹt cứng họng (memory pressure)!

Do `SIReadLock` ĐÉO Xích Họng Đứng Chờ (không blocking), đừng có điên lấy cái súng báo giờ `lock_timeout` vô làm công cụ nhốt vòng xoáy xé lộn SSI nhen! Lấy dây nịt Hẹn Giờ Bọn Cử Ứng (Application deadline) với Vòng Lặp Tái Sinh Có Giới Hạn (bounded retry) mới khóa chặt tổng thời gian được!

## 6. Sợi Dây Chằng Chéo Đọc/Ghi (Read/write dependencies)

Kẻ nháp: Ký hiệu `Treader --rw--> Twriter` nghĩa là Mắt Kính Đọc của Mày đã lướt bãi trống trơn mà đéo thấy món cặn bẩn Thằng Kia Ghi, TRONG KHI món rác cặn Thằng Ghi Bắn ra lại vọt bắn trúng tung phét cái mảng vùng (predicate/version) Mày vừa ngắm lúc nãy!

Ở rạp này:

```text
T1 rà vạch active --rw--> T2 nhét kẹp C2 rọt vào vạch đó
T2 rà vạch active --rw--> T1 nhét tụ C1 ọt thẳng vào vạch đó
```

Cả 2 sợi quất nối thành vòng lặp thắt cổ (cycle trực tiếp). Mở rộng hẹp ra, bộ SSI sẽ mò lượn rà bắt "**Lực lượng Nguy Hiểm**" (`dangerous structure`) ôm gọn cái ổ lộn RW-conflicts nách chéo mâm rình đẻ nọc độc (serialization cycle). PostgreSQL nó ỤC súng bắt mày héo cuộn luôn (abort) để dập lửa ngay tắp lự cho dù hệ Application mày ÉO hề cảm thấy cái ngáp chờ khựng nào (blocking lock wait) đâu nha cưng!

Trò Này KHÔNG PHẢI Deadlock Đâu:

| Ngộp Khó Thở Của SSI (serialization conflict) | Tắc Cửa Khóa Gông Bất Tử (Lock deadlock) |
| --- | --- |
| Lưới dính lộn dây giữa snapshot reads và writes | Đâm đầu kẹp họng chết vòng wait-for trên mấy cái mỏ neo incompatible locks |
| Súng xăm `SIReadLock` đéo cản họng writer | Các ông giằng co chờ cái ổ khóa buông (Actor thực sự chờ lock) |
| Vọt lỗi rách áo SQLSTATE `40001` | Lòi máu họng SQLSTATE `40P01` |
| Có thể tung toé hở tại chóp statement/commit | Thần Detector trảm đứt rạch gãy móc xích phá chờ (phá lock wait cycle) |
| Tụi bay nát thây đều cần LỘT XÁC RÃ TRỌN whole-transaction VÁ LẠI MỚI TOANH khi được cho phép | Thằng kia cần lôi đầu ra chỉnh trật tự nắn khoá (canonical lock order) nằm rạch ra trước chứ chưa bàn chuyện retry |

## 7. Sao Thằng Thua Lỗ Phải Đập Đổ Nhảy Xóa Đánh Trận Mới Từ Đầu? (Vì sao loser phải chạy lại toàn bộ logic?)

Lượt nặn đầu mày chốt nháp dính mỏ con số `active=60`. Sau khi thằng Thắng cướp cò (winner commit), mớ số lật vèo lên `90`. Mày mà chỉ lôi hàm `INSERT` móc ra nện thì xài trúng cục dẻ rách quyết định/giá trị xịt cũ gãy mẹ sườn nhà invariant chứ còn gì.

Làm lại đợt Mới phải xả lật chẻ bài thế này:

```text
khám coi có đứa nào nắn phán quyết bền chưa
→ đếm vét lại cái trần limit
→ quét rờ tính lại tổng kho active predicate
→ cạ so mâm số đòi yêu cầu (requested amount)
→ viết xăm giấy phán ACCEPTED/REJECTED
→ sập búa commit
```

PostgreSQL đéo thèm rảnh đi vớt tự dọng (retry) dùm mày, vì máy chủ nó chả biết bộ óc code App mày mò mẫm bắt sql/chọn số nào hay nặn vút external actions mẻ gì. Rút Ván Đánh Xé Lại Toàn Đội Giao Dịch (Whole-transaction retry) là vác nặng đít của thằng gọi đầu (caller) / anh cầm nhịp (coordinator)!

## 8. Hố Sập Giao Dịch Thối (Transaction failed state)

Khi súng nổ khạc đứt Statement/Commit vãi ra cục `40001`:

- Mớ cặn vừa nặn attempted writes tụt lột không thể dính mâm commit;
- Cuộn giao dịch dính chàm đéo được ngó ngàng lôi ra xài nữa;
- Mày đâm thọc quất query tiếp mà chả gỡ rollback thì auto lãnh thẹo `25P02` (doomed transaction);
- Bóng Proxy Spring bị ép giựt bùn sập rollback rồi ném trả chốt thòng lọng connection đi đóng mâm;
- Đám thực thể Managed entities hay đống cặn Result vương vãi từ cú té KHÔNG THỂ bứng mót làm đòn khởi cho cú Đấm Lại được nhé!

Vung chiêu gạt nháp `EntityManager.clear()`, đánh rạch ghim Savepoint hay bao kén Catch rớt ngầm rờ rờ chui bên TRONG hàm đéo hề đẻ bọc physical transaction tươi mới nào nhen! Chuyện Lui Bước Ngáp Chờ (Backoff) Phải Trượt Sau Lúc Lăn Xả Bùn Rollback, nhảy hẳn RA NGOÀI Vòng Transaction, để không còn ôm trói sợi tơ connection/Mác Soi SIREAD ngậm máu của cựu kiếp cũ oái oan!

## 9. Ngõ Nứt Gãy Conflict Đục Xuất (Nơi conflict có thể xuất hiện)

Rạp kén Hibernate nó hay phọt lén SQL dọng db lắm kiểu lắm:

- gọi lệnh cào repository query lòi chực nổ tự dội phân (auto-flush);
- Móc vút bóp nổ explicit `flush()` ăn gạch đá nổ `40001`;
- Véo cú sập Transaction commit nó nắn đâm mớ rác pending changes đi;
- PostgreSQL thì chốt cú vồ đục khét serialization conflict lúc dòm đang bốc statement hoặc canh cửa tàn canh transaction mới báo án!

Vì thế, anh Coordinator Thượng Điền phải buông lưới catch ôm trọn sập bóng móc gọi **QUA SÀNG Tường Proxy Transactional**, CHỨ ĐÉO phải khép nắp chụp lòi tĩ quanh nhõn 1 chỗ gọi hàm repository call. Lão Bắt Án (Classifier) lướt dọc rễ Cause Chain tít tới xương chót tìm cho ra bóng ma `SQLException#getSQLState() == "40001"`; chứ đem so chuỗi chữ Message Text hay Bọc Wrapper Của Lão Spring thì rạch nát độ lủng gãy lừa lọc đéo thể đanh thép sỏi bằng cái lõi hạch SQLSTATE đâu nhóc!

## 10. Trói Áo Lưới Cho Spring Hợp Mạng (Effective isolation trong Spring)

Áo nhãn bùa `@Transactional(isolation = Isolation.SERIALIZABLE)` chỉ thực thi phép mầu khi bộ Manager giăng thét tạo 1 bọc Giao dịch vật lý mới (physical transaction):

- Cú chọc hàm PHẢI đi xuyên tròng mặt áo Spring Proxy;
- Ruột lồng lệnh xé cúc `REQUIRED` chui tụt lấn cướp dồn sập bóng mâm Outer transaction cũ kĩ đã bật, đéo tự ngóc Áo Lưới cách ly vọt lên (không nâng isolation);
- Trò tự xử (Self-invocation) sút bay màu bóng Lệnh Tụ (Advice);
- Ngâm ngó Mũ `DEFAULT` ăn bám gầm đít Datasource/Database default;
- Lớp Test mà trùm mền outer transaction nó nuốt che sạch boundary áo chiến thật thì tạch cmn soi!

Tốt nhất chắp tay kéo mỗi integration test tọng query khám:

```sql
select current_setting('transaction_isolation');
```

Xong assert rạch ròi thấy mặt bóng `serializable` lót trong cái cũi mâm Lượt Ốp nhé!

## 11. Bến Cuối Lệnh Nháp (Commit, rollback và retry outcomes)

### Đứa Cầm Cúp Sống Thọ (Winner commit)

Bọc cọc Reservation và tờ phán Command Decision của đứa sống chốt đinh ngầm gộp dính ngắc (atomically). Bóng mác `SIReadLock` vẫn có thể kìm lún lén giấu trọ ở rốn ruột Database để soi mạch dependency tracking tới khi tụi rễ chằng (overlapping transactions) cút sạch; Đừng có hở ra la lối lu loa "Ối cha ôi Lộ Khóa Dây Bám Đít (leaked blocking lock) Rồi!" nhen!

### Bại Tướng Đổ Sụp (Victim rollback)

Bọc cọc Reservation/Tờ phán hụt của đứa tử sĩ sẽ trôi dạt vô hư vô (biến mất). Nếu đéo làm cú lật mặt Đấm Lại (retry), cái mõm API mày trả về bắt buộc văng lòi Cáo Trạng Rớt Lực Cục Bộ (temporary failure) Rành Rẽ Bọc Mõm; ĐÉO ĐƯỢC mớ miệng láo xạo chém "Tao DUYỆT (ACCEPTED)" nhé!

### Ván Tái Đấu Tươi Rói (Fresh retry)

Bọc Lại ôm vạch ngực tờ Command ID bọc cũ, xé niêm kén Bảng Nháp (snapshot) mới bóng loáng, đong ngắm quả kho active total đội lên `90` và nắn đập sập commit án cút thẻ `REJECTED`. Cái Lỗi cự tuyệt Bọn Trụ Phép Business (Business rejection) KHÔNG Được kéo lôi xài Đấm Lại Ổ Cứt đó hôi tiếp!

### Mỏi Hơi Tuột Dây (Retry exhaustion)

Rớt ngáp sau chạm giới Tụ Đỉnh Kéo Hẹn Cáp Đứt Mạch (attempt cap/deadline), Cáo Lệnh Trưởng Coordinator phải buông gươm ói lỗi `LimitContentionException` hay bung rèm cáo phán 1 cái thẻ gãy-chờ-sau-nhé (stable retry-later outcome). TUYỆT MỆNH KHÔNG chừa chắp vá vứt 1 đuôi Transaction rỉa rách tươm máu đít nằm đó mụ mị. Người Xưng Giục (Caller) bóc mẻ móc soi cái the Command ID soi xem có rặn kéo cú đập Tầng Ngoại Lai Không Cũng Ổn Á!

## 12. Dán Rễ Lì Lợm Và Đứa Con Bơ Vơ Mất Báo Động (Idempotency và ambiguous outcome)

Súng khét `40001` là án tử khựng Đã Được Đoạt Giấy Khám Tử Trực Diện (known abort): Ổ nặn xụp chưa sập commit. Lỗi này bảnh hơn vỡ oái mạng Rớt Ống (connection loss) tụt luốc ngay khúc commit, rặn óc khứa dòm khách (client) ứa đụ ộc chẳng rành (ambiguous outcome).

Kén bùa Thép Chốt Cục Đeo Durable `limit_command_decision(command_id primary key, outcome, ...)` độ đòn rạch ròi cho tụi bay:

- Đập Lại trúng mác Command ID thì trượt vút Replay luôn bảng `ACCEPTED`/`REJECTED` thong dong;
- Sập báo mất mỏ Response sau lúc dập commit mượt KHÔNG Bị đẻ quái thú reservation lọng hai oán khiếu;
- Quỷ Cùng Lúc Tạt Dọng Ảo Lọng Command (concurrent duplicate) đã được mác Xích Thép Unique Constraint phân trần độ gãy;
- Lệnh Thư Rời Outbox Row Bóc Gài Mã Phép cùng thẻ cọc command/event ID vướng lọng đính.

Chóp Vỡ Xích Tích Đụng Sừng `23505` (unique violation) CẤM Có Nặn Trượt Bịt Mắt Dọng Đánh Lại Bừa Mù. Nếu Gãy tại Cọc Trói Lưới `command_id` của đúng thẻ bùa này, gồng tụt xé Rollback lật tung cuộn mới soi đọc mâm Chốt Kiên Định Commit. Nếu gãy cớ Tụ Phép Lưới Khác (constraint khác), nắn xéo búng lội đứt Trúng Y Phóc cục Cáo Bọn (propagate) nhen!

Bùa Idempotency (Lì Lợm Cứng Bọc Lại) KHÔNG TRÁO Ngược Lấp Sập Vai Trò SSI. Tụi Xách Lệnh Command IDs Trái Nhau Vẫn Kêu Khóc Đoạt Nhót Đóng Lưới Phân Phép (serialization) để rào gác trần merchant limit của Ổng!

## 13. Tụt Ngáp Hẹn Dây Và Án Chốt Đo Kẹp (Backoff, jitter và deadline)

Chọc Lại (Retry) sấn sổ Tắp Lự Ngay thì chả khác mẹ gì nhét lộn đôi quỷ vào chung Khung Khớp Chén Cửa Đụng Chờ Nát Xé Nhau Nhé (collision window)! Bản Bố Áo Đo Gồng Phải Lận Hòm Lỗ Trống Này:

- Gõ list thẻ lấp cho mẻ vút `40001` (allowlist);
- Giới Lấp Tổng Đỉnh Trụ (attempt cap);
- Vút dãn trễ nhịp tụt dốc cấp số nhân Exponential Backoff chọc trộn Cứt Rối (random jitter);
- Chốt cúp đuôi Đoạn Đường Sinh Tịch (overall deadline/cancellation);
- Tróc Test Chắn Lại Độ Trì Đo Xét Nghiệp Vụ (business revalidation) Từng Phút Ục Retry;
- Đo Gọng Bộ Bộ Điểm Đo Nhấp Attempt, Chóp Đỉnh Success-after-retry, Rụt Sụp Xương Tịt Trống (exhaustion).

Không có Trò Kê Số Delay Bíp Tuyệt Môn Cái Nào Ngon Tuyệt. Thương Phím Rát Nóng (Hot merchant), Hơi Cổ Transaction Duration, Độ Ngợp Bể Bơi (pool size) Lẫn SLO Kẻ Thù Mới Nắm Quyền Phán Set Kèo Cấu (configuration). Lũ Rặn Khét Lệnh (Retry amplification) Ráp Khéo Nặn Bão Nhảy Loạn Từ 1 Cú Lỗi Nứt Nẻ Tí Cỡ Vỡ Lở Bể Bọng Đáy Xập Bệnh Thở Quá Trớn (overload) Nếu Xẻ Dọc Kéo Trọng Mảng Quá Đông Attempts Nhen Khứa!

## 14. `SERIALIZABLE READ ONLY DEFERRABLE` Máng Phép Rào Thở

Vác Bọn Mò Xem Trượt Dai Ngâm Ngó Tịch Dọc Chỉ Dòm (Long-running report chỉ đọc) Cứ Sút Xé Phục Rút Trọng Giáp Lệ Lên Nhé:

```sql
begin isolation level serializable read only deferrable;
```

Cảnh sát PostgreSQL sẽ tự Đứng Ngáp Ván Kéo Đọi Hít Đất Nhá Ổn Nặng lấy trọn bọc Snapshot Lành Lặn. Trượt đó cuộn Bơm Transaction KHÔNG HỀ Ngớp Dính Nguy Án Lấp Cuốn (abort) Cựa Dọng Khớp Tịch (serialization conflict), Gọt Cắt Bao Nặng Hơi Ráp SSI Lọt Tròng Vỡ. Chóp Trượt `DEFERRABLE` NÀY ĐÉO GIÚP XÉ HỘ Ván Đòi Cắn Kép Sửa Vòng Đọc-Ghi Nghe Kép Reservation Attempt Nhen, Cũng Chả Làm Chó Gì Sửa Kéo Trọng Vút Cựa Sống Sổ Deadline.

## 15. Tuột Kẹp Ngáp Chờ Lũ Bạo (Timeout và deadlock)

Giáp Bọc `SERIALIZABLE` LÉO Hề Thò Tay Bợ Nhấc Giùm Ba Cái Trò Nghẽn Khứ Mỏ Chọt Row/Table Khóa Trịch Nhăn Lệnh Già Oái Bọn Khứa Kia Lại Nhen (ordinary row/table lock waits):

- `40P01`: Khất Oải Giờ Chóp Rặn Tử Trận (deadlock); Tuột đống lấp gỡ xé Rollback nhấc Retry Ổn Gấp Nếu Rành Rẽ, Kéo Tụt Sửa Vành Dây Chóp Kéo Lấp Lệnh (lock order);
- `55P03`: Sập Lưới Kéo Hẹn Đứng Khóa Chờ (lock timeout); Đào Nhổ Mọc Xét Khứa Đứng Ôm Lệnh Lẫn Khắc Kéo Đo Giao Độ Hẹp (latency contract);
- `57014`: Đứt Trụy Khớp Câu Lệnh Móc (statement canceled/timeout);
- `40001`: Trật Vành Lệnh Sóng Sút Gãy Sẽ Đéo Kéo Lịch Nhá Gãy (serialization failure).

Sàng Điểm Xét Bộ Máy Tróc Phân Cáo (Classifier) Nhồi Dọc Metrics Bắt Tay Xé Ác Cáo Chẻ Bẻ Bọn Mũ Này Tươi Sạch Đơn Côi Riêng Từng Bịch. Máng Áo Ngáp Trễ Tụt Lệnh Phép (Backoff policy) Dù Có Gài Ôm Tụ Giao Thở Trọng Nhau Chứ Còn Gốc Bệnh Cắn Khét Oai Nứt Cổ Lệ (root cause/remediation) Cứa Kéo Hoàn Toàn Tạch Đất Trái Dấu Nhé!

## 16. Loa Phóng Ẩu Đỉnh Giới Lệnh Ra Bìa (External side effect)

Quái Cắn Notification Tiếng Rên, Tiêm Bơm HTTP Call Hay Chặn Thả Message Loa Gởi TUYỆT PHÁT CHÓP Bật Trước Mốc Commit Nằm Dọc ĐÓ ĐÉO RÚT LẠI KÉO Rollback Được Vành Sóng Database Nhen Cưng Nhé! Khứa Đệ Một Lần Thử Có Gật Đầu Báo Kép `ACCEPTED` Dọc Phương Tiện Thân Đo Hàm (method body), Phọt Thét Phóng Tin Message Xé Oai Xong Oẳng Nhận Lại Mã Bể Bát `40001` Lúc Dí Búa Đập Lệnh Chót Á (commit)!

Phun Thủng Ngăn Kéo Túi Đất Lệnh Thét Outbox Dưới Rốn Ruột Mâm Thắng (successful transaction) Trọn Bóc Lệnh Gửi Nép Phanh (publish) Sau Mặt Búa Sút Cửa Commit. Kẻ Đứng Ăn Máng (Consumer) Bắt Ép Đụng Xóa Lưới Rập (deduplicate) Theo Móc event/command ID Ớ Ngạc! ÉP TỰ THẮT CỔ Tịt Sút Cửa Dọng Ôm Sứt Khóa Bọc Kép Mở (serializable transaction mở) Chầu Hốc Remote I/O Rẻ Rúng Trắng Đuôi Nhé Khờ!

## 17. Cháy Khét Sụp Lệnh Khứa Lực Ống Tịch (Crash và connection loss)

Rớt Cúp Nguồn Bất Chợt Trước Mặt Búa Commit Thét Rạch DB Trọng Vút Sót Nhóp Rollback Kép Sạch Giao Dịch Cuộn Tụt Quăng Ngầm Cứ Quyết Attempted Báo Trống!
Lọt Rụng Lưới Chạy Gãy Sụp Sau Khi Phía Rốn Server Máng Tụ Đập Búa Chót (server commit) Mà Ngáp Tịt Đứt Méo Ói Ra Chút Trút Bạc Cáo Chạm Hồi Đáp (response) Kéo Đứt Thụt Ambiguous Outcome; Dọng Rụt Mỏ Trấn Lưới Cáo Bạc Re-try Vẫn Ôm Bụng Xăm Cũ Command ID Sút Áp Chốt Kéo Replay Quả Mâm Oai Kéo Cửa (durable decision).

Trường Móc Nếu Não Nát Chết Oai (process chết) Đang Ục Thụ Thở Tịch Trong Lệnh Backoff Chờ Thì Cửa Trọng Sạch Nhẵn Bóng Đéo Có Món DB Giao Dịch Chăng Ảo Dính. Đứa Mới Gõ Kêu Caller / Lão Trịch Bộ Phân Chấp Handler Mệnh Kiên Bọc (durable command handler) Sẽ Nhảy Kéo Tiếp Thẳng Luồng Kéo Nhồi Đo Trái Áo Lệnh Idempotency Bức Chặt Contract Nhá Nhóc!

## 18. Máng Đo Khớp Chảo Tụ Nhóm Ngạc Oanh Xưng Giao Rạch Đều (Multi-instance)

Bộ Sút Lưới Móc SSI Nuôi Ở Ổ Chung Shared PostgreSQL Nên Giăng Chặn Bao Đóng Bọn Bể Nháp Khách Transactions Trục Thích Ộc App-1, Ải Lệnh Tréo App-2 Kéo Luôn Rìa Cán Ốc Phép Líp Công Nhân Lũ Khác Bịt Đeo Trọn Dọc Nhau — TỤC KHI Toàn Máng Cuộn Phép Tịch Update Sóng Đổi Trọng Gắn Đúng Giáp Mắc Rễ Giao Tịch (compatible isolation/protocol).
Móc Phép Khóa Treo Ngầm Chốt Buộc Đứt Lệnh Hàm Rẽ `synchronized` Nó Ôm Nát Đè Có Đứt Mõm Được 1 Bịch JVM Chứ Không Bao Giờ Tráo Ma Đi Chạm Vào Bóng Nháp Ảo Database Snapshots Đâu Nghen Não!

Mũi Xâm Rẽ Hở Một Sọc Ngõ Vút Direct SQL Path Gắn Chạy Xé `READ COMMITTED` Lọt Luồng Xệ Ụp Có Thể Tạch Lóng Mọc Lệnh Lánh Xé Lấp Tịch KHÔNG Rọi Dưới Dù Đeo Áp Che Vững Bảo Chứng Lưới SSI Nhấp Kêu Chạy Ảo Đâu Nhé Ngài. Khẩu Phép Permission, Trọng Chóp Ống Kép Stored Procedure, Sắc Phong Rút Đỉnh Service Ownership Bấm Liệt Giữ Dấu Kéo Migration Tụ Nghiêm Bật Phải Đè Ép Siết Thòng Lọng Khóa Vệt Đâm Sóng Lách Ngõ Trượt Vòng Bẩn Đuôi Nhá Bypass Khống Cho Cái Chóp Mạch Lập Tịch (invariant) Thiết Kế Bền Nhất Nhé Nhóc!

## 19. Đáy Xương Gốc Bệnh Sọc Tung Rọi Kéo Bãi (Nguyên nhân gốc theo từng layer)

| Màng Chém (Layer) | Chân Trụ Thở Rẽ Lọt (Vai trò) |
| --- | --- |
| Lớp Áo Mặc Đầu (Application) | Ục Lệnh Móc Đọc → Trồi Lệnh Chốt Tịch → Trấn Khóa Chặn Sút Gãy Móc Ghi Phép Insert Vút Lên Ục Predicate Dày Nhiều Cựa Bảng Rows; Thiết Sụp Trọng Khúc Rã Hỏng Retry Contract Báo Oai |
| Khung Nhà Ngáp (Spring) | Mặt Áo Giáp Proxy/Trộn Sóng Isolation/Cắm Ranh Giới Bóng Mỏ Bọc Boundary Tự Kéo Phán Đập Xé Ra Bóng Mặt Nặn Cú Rớt Tươi Lặp Oai (fresh attempt) Hay Nghẽn Gãy Cút Ục Dọc Trán Lì |
| Mỏ Kéo Bọng Tịch (Hibernate) | Dấu Móc Giờ Gắn Rớt Chóp Flush Timing Cựa Trọng Quyết Bẻ Sút Hở Xé Khứa Đội Tịch Oác Exception Vọt Lộ Đầu Chạm Lúc Nào |
| Chảo Dầu Lòng Tụ Rễ (PostgreSQL) | Giáp Rà Bắn Áp SSI Trực Khứa Trượt Ác Dò Theo Nắm Rễ Đỉnh Dependencies Bắt Chân Đòi Abort Tụ Vọt Má Lệnh Móc `40001` |
| Sân Phơi Áp Mõm Nháp Nước Đục Ộc Ngõ (JVM local) | Tịt Gáy Đéo Trấn Áp Bùa Gọi Giao Hẹn Chóp Liên Đới Oai Mệnh Điều Hợp (coordinate) Trích Nặn Dạt Bãi Đáy Các Mạng Trạm Bọn Dính Nhiều Kép Gọi Nhau Lên App Instances Trực Xưng |

Lỗ Chết Tội Gốc CẤM CÚT Khẽ Mỏ Đổ Nặng Oan Thằng PostgreSQL "Nổi Cơn Điện Hấp Ngẫu Nhiên Nhả Lệnh Tịch Rollback Láo"! Rớt Đạn Tịch Phán Khóa Xé Trọn Gương Abort Thể Hiện Mũi Nhan Bản Chất Nhập Cứt Xương Đúng Quy Chép Oai Bản Giao Tranh Chút Ngọt (correctness contract) Mót Được Lúc Móc Quyết Nhấp Tịch Bọc Nắm Cờ Oai Mở Lọng Lệnh Trái Đất Óc Chạm Thực Thi Trơn Sáng Nháp Dọc Rọi Lên Lưng Trọng Áo Đỉnh Oai Optimistic Serializable Nhanh Gọn Chút!

## 20. Trọn Khúc Đèn Trăng Kép Chân Trọng Áp (Khả năng quan sát - observability)

Nắn Mạch Rà Bọc Lưới:

- `40001` chẻ lọng đo bọc dọc mâm trượt ngõ Kép Operation Tịch Và Dấu Thét Khứa Chờ Dọc Lên Mã Nón Attempt Number Tự Dọi Nhé;
- Hấp Mạch Oai Ảo Gọi Chốt Khứa Dòng Success-after-retry, Kéo Nhăn Kiệt Đít Sóng Đục Đáy Kẽ Lệnh Vòng Trượt Lụi Bọn Rỗng Exhaustion, Khứa Lệnh Trọng Chóp Lui Tịch Đứt Đít Áo Rớt Giây Mạch Backoff/Deadline;
- Rạch Chẻ Lõi Kén Mạch Áo Nón Tròng Rờ Phép Isolation (effective isolation) Kéo Cứa Tụ Mạch Tuổi Sinh Bóng Tịch (transaction duration);
- Ngắm Soi Vành Rổ Bể Bơm Trạm Nguồn Vực Cửa Pool Khóa Kéo Rờ Trạng Active/Pending Chóp Hốc Thấy Được Trấn Lấp Bọn Gáy Móc Phọt Ụp Tịch Trán Khuếch Đáy Dọc Ánh Oanh Liệt Retry Amplification Độc;
- Mũi Dọc Đâm Gươm Vén Chướng Dọi Áp Sóng Query Plan/Index Changes Phì Kép Cắt Gọt Ảnh Hưởng Trượt Kho Khép Mảng Dây Đo Đục Cục Predicate-lock Granularity;
- Quất Soi `pg_locks.mode = 'SIReadLock'` Móc Đâm Lên Lọng Nấp Dấu Dọn Khám Tịch Móc Khi Soi Mép Viện Vành Điều Tra Án;
- Cuốn Ngắm PostgreSQL Logs Khỏa Hiện Có Hút Sóng Chữ SQLSTATE/Correlation ID Đóng Kép Tịch NHƯNG CẤM Có Nặn Trượt Rạch Ẩu Móc Tuột Phọt Chẻ Oai Ánh Bóng Ốp Rìa Dụng Dấu Lộ Đút Kẽ Đội Bind Data Thông Tin Nhạy Cảm Nghe Khờ Láo!

Số Bọng Đếm `pg_stat_database.deadlocks` Rạch Giao Đo Cứt Chỉ Gảy Tích Rút Đếm Đống Trọng Xác Hút Sóng Deadlocks Ngu Nhá, ĐÉO Gài Oai Nằm Kho Trụng Nhét Trọng SSI Failures Oan Đáy!
Đũa Gắp Bộ Líp Số Đo Kéo Hút Hạng Kép Application Metric/Log Classification Trọng Đạo Tịch Ảo Phép Hiện Trích Vút Điển Soi Oái Gọi Sóng Bảng Nháp Cứ Nối Rễ Trọng Chính Hút Số Nguồn Gốc Phán Ngõ Dọc Đóng Cửa Tịch Tỷ Lệ Tượt Vỡ SSI Serialization Failure Rate Khét Lẹt Oanh Bọn Sóng Ác Rạch Đất Nhé Nhóc Cưng!
