# Phân tích mức độ cô lập thực tế (Effective isolation analysis)

## Kỳ vọng và kết quả thật

Khối mã nghiệp vụ (Business code) một mực ao ước rằng hai lần đọc phải xài chung một bản ảnh tĩnh bất biến (stable snapshot):

```text
kỳ vọng: giá lần 1 (firstPrice) = 100, giá lần 2 (secondPrice) = 100
thực tế: giá lần 1 (firstPrice) = 100, giá lần 2 (secondPrice) = 120
```

Cái mác `REPEATABLE_READ` lù lù xuất hiện chễm chệ trên đầu hàm `readTwice()`, nhưng thành phẩm nhào nặn ra lại sực nức mùi ngữ nghĩa (semantics) của PostgreSQL `READ COMMITTED`. Xin thưa, đây không phải lỗi (bug) bộ nhớ đệm (cache) của Hibernate; mà là do giao dịch vật lý (physical transaction) đã lỡ bị hàm vòng ngoài (outer method) khai sinh ở một mức độ cô lập (isolation) yếu xìu.

> **Nói ngắn gọn:** Spring đanh thép phán quyết mức độ cô lập ở ngay cái giây phút búng tay khai sinh ra giao dịch vật lý, chứ nó rảnh đâu mà chạy đi nắn nót xét duyệt lại mỗi khi bước chân qua một cái hàm được gắn annotation bóng lộn.

## Tiến trình tái hiện (Timeline tái hiện)

Giả sử mức giá khởi thủy là `100`:

| Bước | Người đọc (Reader) — Tx-R | Người ghi (Writer) — Tx-W |
| ---: | --- | --- |
| R0 | Hàm `ReportFacade.generate()` thênh thang mở cửa giao dịch DEFAULT | |
| R1 | Vòng trong (Inner REQUIRED) lon ton xách túi gia nhập (join) vào Tx-R | |
| R2 | Lệnh `SELECT price` nhả về `100`; đồng thời hé cửa (mở gate) cho tay writer lọt vào | |
| W1 | | Vênh váo mở một giao dịch hoàn toàn độc lập (độc lập) |
| W2 | | Nã lệnh `UPDATE product SET price = 120` |
| W3 | | Chốt hạ (Commit) |
| R3 | Lệnh `SELECT price` lần hai ngậm ngùi nhả về `120` | |
| R4 | Gửi báo cáo (Commit report) với cặp số đắng ngắt `100/120` | |

Ở cái xứ sở PostgreSQL mang danh `READ COMMITTED`, mỗi một câu lệnh (statement) tự tung tự tác vớt lên một bản ảnh mới toanh (snapshot mới) ngay tại cái thời khắc câu lệnh ấy vừa nhấc chân bắt đầu. Vì kẻ thọc gậy bánh xe W3 ngang nhiên chen ngang ở giữa R2 và R3, nên việc hai cú SELECT hợp thức hóa nhìn thấu hai phiên bản đã chốt (committed versions) hoàn toàn khác nhau là chuyện thường tình ở huyện.

Ví thử như Tx-R mà là `REPEATABLE READ` xịn sò thật sự, thì bản ảnh (snapshot) đã bị ghim chặt đóng đinh (cố định) cho cả giao dịch ngay từ thời khắc câu lệnh truy vấn đầu tiên vớt lên. Khi đó R3 vẫn hiên ngang thấy mặt con số `100`; cái thằng giao dịch độc lập lò dò bắt đầu ở phía sau tấm rèm R4 mới có phước phần thấy được số `120`.

## Ranh giới giao dịch (Transaction boundary) được đúc ra ở xó xỉnh nào?

Quá trình chặn luồng (Luồng interception rút gọn) tóm tắt:

```text
máy khách (client)
  -> đụng phải cửa ải proxy của ReportFacade
       -> xắn tay bắt đầu giao dịch vật lý (begin physical transaction - chuyển thể DEFAULT -> READ COMMITTED)
       -> chui tiếp vào cửa ải proxy của SnapshotQueryService
            -> luật REQUIRED ngửi thấy mùi giao dịch đã tồn tại (finds existing transaction)
            -> lon ton gia nhập (join); dẹp bỏ ý định mở cửa mới (no second begin); chẳng màng nâng cấp cô lập (no isolation upgrade)
            -> hì hục chạy hàm readTwice()
       -> chốt sổ giao dịch vật lý (commit physical transaction)
```

Lớp áo khoác ngoài cùng `@Transactional` (Outer `@Transactional`) chính là nôi khai sinh giao dịch (transaction creation point). Khi sợi dây kết nối JDBC rùng mình bắt đầu giao dịch, viên cai quản giao dịch (transaction manager) đã thẳng tay ụp ngay cái mức độ cô lập (isolation) lên đầu nó rồi. Lớp áo vòng trong (Inner proxy) chỉ tạo ra một ảo ảnh phạm vi giao dịch logic (logical transaction scope) núp bóng bòn rút chung cái giao dịch vật lý kia thôi.

Tuy nhiên, đừng vội mắng oan cái annotation vòng trong là đồ phế thải: nếu máy khách (client) cả gan gọi sỗ thẳng vào cái bean đó lúc chưa hề tồn tại một giao dịch nào, thì nó sẽ oai phong lẫm liệt tự gầy dựng cơ đồ bằng một giao dịch `REPEATABLE READ` chính hiệu. Lối hành xử xoay như chong chóng tùy thuộc vào nẻo đường xâm nhập (call path) — rõ là một dấu hiệu cảnh báo đỏ lừ của lối thiết kế ranh giới trơn trượt chết người.

## Vì cớ gì sự vênh nhau (mismatch) lại hay lấp liếm im ỉm?

Với ngón đòn lan truyền `REQUIRED`, Spring luôn ngả mũ quy phục những đặc tính (characteristics) của một giao dịch vốn đã nghiễm nhiên tồn tại từ trước. Dù cái mác vòng trong (Inner declaration) có gào thét đòi `REPEATABLE_READ`, đòi `readOnly=true` hay kỳ kèo đòi đổi thời gian chờ (timeout), thì sức mấy mà nó tự động hất cẳng thay thế được mấy cái đặc tính xương máu của giao dịch vật lý đã trót phóng lao mở cửa.

Một khi chốt gác `validateExistingTransaction` ngủ quên (không bật), mọi sự vênh váo lệch pha (mismatch isolation) đa phần đều được tặc lưỡi xuê xoa cho thằng em (inner scope) lách luật gia nhập (join). Đánh thức rào gác (Bật validation) sẽ vạch trần biến cái lỗi semantics ỉm ỉm kia thành trái bom `IllegalTransactionStateException` nổ tung trời trước khi câu query kịp hé môi.

Nên nhớ, thanh gươm canh gác (Validation là guardrail), chứ chẳng phải phép tiên nâng cấp (không phải cơ chế upgrade). Đòn đánh chính diện chữa tận gốc (Cách sửa chính) vẫn là phải đắp đúng cái isolation ngay tại nôi khai sinh giao dịch (transaction creation point).

## Đo đạc mức độ cô lập thực tế (Đo effective isolation)

Đừng dại mà bói toán võ đoán (suy luận) cái mức cài đặt thực tế (effective setting) chỉ bằng cách soi annotation hay bới móc log. Câu lệnh trinh sát (query sau) bắt buộc phải luồn lách chạy bằng đúng ngay cái đường truyền JDBC (JDBC connection) và núp kỹ bên trong lòng giao dịch đang nằm trong tầm ngắm:

```sql
select current_setting('transaction_isolation');
```

Quả báo thu về (Kết quả của broken path) là:

```text
read committed
```

Ánh sáng nơi cuối đường hầm (Kết quả mong đợi sau khi sửa outer boundary):

```text
repeatable read
```

Xài bùa `SHOW transaction_isolation` cũng chả ai cấm, nhưng `current_setting(...)` lại khéo léo bợ bợ (thuận tiện hơn) khi rinh về một giá trị duy nhất (scalar) thông qua `JdbcTemplate`.

## Màn kịch của DEFAULT và DataSource (Vai trò của DEFAULT và DataSource)

`Isolation.DEFAULT` là lời thì thầm to nhỏ xúi giục transaction manager móc ruột moi gan cái mặc định (default) của nguồn tài nguyên (resource). Đối với vương quốc PostgreSQL, lẽ thường tình nó là `READ COMMITTED`, thế nhưng lũ ứng dụng (application) cấm chỉ được ngông cuồng coi cái thói quen môi trường (giả định environment) như một khế ước máu nghiệp vụ (business contract).

Giá trị vận hành thực tế (Effective value) có thể dễ dãi bị uốn nắn (chịu ảnh hưởng) từ:

- con cờ `default_transaction_isolation` ở thượng tầng database, role hay mảng session;
- khâu khởi động kết nối của chiếc hồ chứa (connection initialization của pool);
- bàn tay của transaction manager và đám tài xế JDBC driver;
- sự ấn định độc tài (explicit isolation) trên giao dịch lớp áo ngoài (outer transaction);
- và cả một bóng ma giao dịch lù lù có sẵn do kẻ gọi lệnh (caller) vung tay múa may (mở).

Cũng bởi chiếc hồ nối (connection pool) có cái tật hay xài lại kết nối, nên cái trò nghịch dại thay đổi thiết lập session (thay session setting) tùy tiện có thể lén lút tuồn mầm bệnh (leak behavior) văng sang tận yêu cầu khác nếu chiếc hồ kia lười biếng quên dọn dẹp khôi phục (không reset đúng).

## Tiếng gọi từ thâm tâm (Self-invocation) là một con ngõ cụt thảm khốc khác (failure mode khác)

Trong cái kịch bản tự kỷ ên (biến thể self-call):

```java
this.readWithStableSnapshot(productId);
```

Lời thỉnh cầu (call) trượt lốt không thèm bước chân qua cửa ải Spring proxy. Giả dụ cái hàm outer ngó lơ không buồn dính líu giao dịch (không transactional), thế thì annotation bám trên cái cọc phụ (helper) kia sẽ không được đoái hoài xử lý và kết cục là cái giao dịch vĩ đại từng mộng mị (intended transaction) không hề tồn tại. Nguy to hơn là mấy cái hàm Repository nhí nha nhí nhảnh có thể tự tung tự tác mở ra mấy cái giao dịch vụn vặt (ngắn riêng biệt), biến cái ranh giới đã nát nay còn băm vằm tơi bời hơn.

Nhớ khắc cốt ghi tâm phân định rạch ròi (Cần phân biệt):

- **con hoang proxied bean + luật REQUIRED:** annotation có được liếc mắt qua nhưng cam chịu gia nhập (join) vào giao dịch đã an bài (đã có);
- **tiếng gọi thâm tâm (self-invocation):** kẻ đánh chặn (interceptor) đình công (không chạy) dứt khoát không thèm thó ngó cái lời gọi trong nội bộ (inner call).

Hai ngõ cụt đều chỉ về một chân lý bẽ bàng: mức cô lập (isolation) chỉ có giá trị khi lời cầu nguyện (call thực tế) chịu chui lọt qua cửa proxy đúng tại hang ổ nơi giao dịch vật lý được thai nghén (tại nơi physical transaction được tạo).

## Lá bùa `readOnly` chẳng sinh ra được bản ảnh tĩnh đâu (`readOnly` không tạo stable snapshot)

`@Transactional(readOnly = true)` chỉ là lời gợi ý bóng gió/chính sách (hint/policy) về cách hành xử với tác vụ ghi (write behavior) và may ráo có thể vờn quanh làm nhúc nhích chế độ xả dữ liệu (flush mode). Sức mấy mà nó đủ nội công chuyển hóa `READ COMMITTED` lên hẳn `REPEATABLE READ`, chả dư hơi đâu mà khóa dòng dữ liệu (không khóa row) và hoàn toàn bất lực trong việc thề thốt bảo lãnh hai cú SELECT sẽ ngậm chung một bản ảnh (snapshot).

Đồng hạng hẩm hiu, gọi `flush()`, `clear()` hoặc dội gáo nước lạnh (refresh persistence context) chả thấm tháp gì tới việc đẩy mức cô lập dưới database. Mấy cái trò vặt đó chỉ giỏi khuấy đảo cái cách JPA đồng bộ/nhốt entity (đồng bộ/cache entity), chứ đụng làm sao được tới cọng lông cái bản ảnh MVCC snapshot hùng hậu của PostgreSQL.

## `REQUIRES_NEW` có võ nhưng lật bánh tráng ngữ nghĩa (đổi semantics)

Bùa chú Inner `REQUIRES_NEW` hô biến đình chỉ (suspend) cái giao dịch vòng ngoài và lật đật mở toang một giao dịch vật lý mới toanh, nhờ thế mà lệnh `REPEATABLE_READ` mới có cơ may được thị uy sức mạnh thật sự (áp dụng thật). Nhưng hỡi ôi, đính kèm đó là một mớ rắc rối (tạo):

- xé nát ranh giới chốt/hủy (independent commit/rollback boundary);
- thèm khát đẻ thêm một kết nối nữa (thêm một connection) trong lúc kết nối ngoài vẫn bị bắt nhốt ngậm miệng;
- rình rập rước về nguy cơ đứng khóa cổ (chờ lock) chém lộn lẫn nhau giữa vòng trong và vòng ngoài đang bị đình chỉ (suspended outer);
- mầm mống phản nghịch: thành quả vòng trong hiên ngang commit mặc xác cho vòng ngoài cắm đầu rollback;
- bản ảnh (snapshot) bất lực không thể che chở nổi những đống rác (work) đã xả ra ở vòng ngoài (outer transaction).

Vậy nên bùa `REQUIRES_NEW` chỉ đắc đạo khi và chỉ khi đơn vị độc lập (independent unit) là mệnh lệnh sinh tử (requirement), chứ tuyệt đối không phải là miếng võ giẻ rách (không phải mẹo) để “ép cái annotation nó chạy”.

## Tường cao hào sâu (Isolation cao hơn) vẫn có nguy cơ thủng lưới sập bẫy hợp lệ (failure hợp lệ)

Đừng ấu trĩ nghĩ `REPEATABLE READ` hay `SERIALIZABLE` là kim bài miễn tử bảo chứng mọi giao dịch sẽ cán đích commit. PostgreSQL có dư quyền uy bóp nát (abort) giao dịch nếu ngứa mắt phát giác ra xung đột cập nhật (update conflict) hoặc dị tật chuỗi (serialization anomaly); phận ứng dụng (application) phải cúi đầu phân tách mã SQLSTATE `40001` rồi è cổ lặp lại (retry toàn bộ) nguyên mâm khối lệnh vô hại (idempotent unit) bằng một giao dịch mới toanh (transaction mới) nếu lề luật (policy) cho phép.

Thuốc đắng Retry không thể gỡ gạc (giải quyết) căn bệnh SPR-004 nếu khối u giao dịch vẫn ỳ ạch bám rễ ở đáy `READ COMMITTED`. Bài học vỡ lòng là phải dập nắn lại ranh giới (sửa boundary); rồi sau đó mới vẽ hươu vẽ vượn hòng vãn hồi (thiết kế retry cho failure của isolation đã chọn).

## Đa phân thân (Multi-instance) và giới hạn phủ sóng của lỗi này (scope của case)

Đào đâu ra cái khóa lề mề (JVM lock) nào xưng vương xưng bá đủ sức dọn dẹp (sửa) bãi chiến trường này một khi hàng đàn đám máy chủ ứng dụng (nhiều application instances) xông phi đè nghẽn PostgreSQL. Mức cô lập thực thụ (Effective isolation) là thanh bảo kiếm của cơ sở dữ liệu (database transaction property) và quyền uy tỏa khắp chốn (có hiệu lực qua mọi instance) một khi chúng đều đớp chung một mâm (dùng cùng database).

Tai ương này thu gọn mũi nhọn vào thảm cảnh chênh phô cấu hình Spring (Spring configuration mismatch). Cặn kẽ về lost update, write skew, phantom và đòn bẩy chọn khóa/luật ngầm (locking/constraint) xin mời ghé trang DB cases. Một bản ảnh nằm im (stable snapshot) chưa chắc đã hóa giải được đòn đánh lén (invariant có writes); phải lựa mặt gửi vàng chọn isolation theo huyết mạch nghiệp vụ (business invariant), không được mụ mẫm đua đòi theo tên của thứ dị tật lẻ loi (không theo tên anomaly duy nhất).

## Khói lửa dò vết chiến trường thực tế (Dấu hiệu quan sát trong production)

Phải tỉnh táo gài log/nhặt chỉ số (metric) tinh vi (có kiểm soát):

- mặt mũi use case và xưng danh giao dịch ngoại vi (outer transaction name);
- gia phả cơ chế lan truyền/cô lập (propagation/isolation) gào thét ở cửa ải khai báo (public entry);
- xách máy đo độ cô lập thực địa (effective isolation sampled) sục sạo hầm mỏ database lúc cần bốc thuốc (chẩn đoán);
- thẻ bài án phạt SQLSTATE và sổ điểm đếm mạng hồi sinh (retry count);
- ngắm nghía sức rút hồ chứa (pool usage) lúc đụng `REQUIRES_NEW`;
- lách luật băm vằm (identifiers) danh tính báo cáo/phiên bản thay cho thứ dữ liệu nhạy cảm trần trụi (raw sensitive data).

Đừng có hâm dở ngáo ộp đòi soi (query setting) vô tội vạ cho mọi gõ cửa request chỉ hòng đắp vá (bù) cho cái ranh giới nhập nhèm (boundary mơ hồ). Hãy thề thốt xác quyết đinh ninh cắm dùi trong môi trường tích hợp (integration test) và chỉ rinh con mắt thần phán xét (diagnostic sampling) ra phơi sáng lúc mồ hôi sôi máu ngập lối (khi cần).
