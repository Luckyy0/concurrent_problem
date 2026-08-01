# Phân Tích Cơ Chế Cấp Khóa, Hiển Thị Dữ Liệu và Vòng Đời Khóa (Lock acquisition, visibility and release analysis)

## 1. Trạng thái khởi điểm (Initial state)

Khởi tạo hệ thống với trạng thái dữ liệu đã được commit:

```text
tenant_quota(T-42)
  quota    = 10
  revision = 5
```

Ba phiên kết nối (actors) được khởi tạo, mỗi phiên giữ một kết nối và một giao dịch vật lý riêng biệt với cơ sở dữ liệu.

## 2. Kịch bản 1 — Lệnh SELECT không giữ khóa (Timeline 1 — Plain SELECT does not reserve row)

| Bước | Luồng A (Holder) | Luồng B (Contender) |
| --- | --- | --- |
| 1 | Thực thi `BEGIN` | |
| 2 | Truy vấn dữ liệu: `SELECT quota` (thấy `10`) | |
| 3 | Xử lý logic tại ứng dụng (local execution) | Thực thi `BEGIN` |
| 4 | | Cập nhật dữ liệu: `UPDATE quota = 8` |
| 5 | | Thực thi `COMMIT` |
| 6 | Trạng thái ở A bị lỗi thời (stale): vẫn là `10` | |

Mở một giao dịch không có nghĩa là cơ sở dữ liệu tự động bảo vệ dữ liệu. Câu lệnh `SELECT` thông thường ở mức cô lập `READ COMMITTED` (và cả ở các mức cô lập cao hơn trong đa số hệ quản trị) không yêu cầu khóa cấp dòng (row-level lock) và do đó không thể ngăn chặn các giao dịch khác tiến hành cập nhật.

## 3. Kịch bản 2 — Khóa tường minh bằng FOR UPDATE (Timeline 2 — Explicit row lock)

| Bước | Luồng A (Holder) | Luồng B (Contender) | Luồng C (Observer) |
| --- | --- | --- | --- |
| 1 | Thực thi `BEGIN` | | |
| 2 | Yêu cầu khóa dòng: `SELECT FOR UPDATE` (đọc `10`) | | |
| 3 | Thực hiện `UPDATE quota = 12` (chưa commit) | | |
| 4 | | Gọi `UPDATE` hoặc `SELECT FOR UPDATE` -> Chuyển sang chờ (waits) | |
| 5 | | | Truy vấn thông thường: `SELECT` -> Trả về bản ghi đã commit (version `10`) |
| 6 | Hoàn tất: Thực thi `COMMIT` | | |
| 7 | Khóa được giải phóng (lock released) | B được cấp khóa và xử lý tiếp (continues) | Lần truy vấn `SELECT` tiếp theo của C sẽ thấy `12` |

> **Nguyên tắc cốt lõi:** Bảng ma trận tương thích khóa (Lock compatibility matrix) xác định xem một luồng yêu cầu khóa có phải chờ (block) hay không. Trong khi đó, tính hiển thị của đa phiên bản (MVCC visibility) quyết định việc truy vấn đọc (reader) có thể nhìn thấy dữ liệu ở phiên bản nào mà không cần sử dụng cơ chế khóa.

## 4. Quá trình lấy khóa của PostgreSQL (Lock acquisition mechanics)

Khi một câu lệnh SQL thực thi, PostgreSQL sẽ tự động yêu cầu các khóa ở cấp bảng (relation/table-level modes) trước, sau đó mới đến các khóa cấp dòng. Cụ thể:

- Truy vấn thông thường (`plain SELECT`): Yêu cầu khóa bảng `ACCESS SHARE`. Khóa này không chặn ai ngoại trừ lệnh đòi hỏi khóa `ACCESS EXCLUSIVE` (như DROP TABLE).
- Lệnh yêu cầu khóa `SELECT ... FOR UPDATE`: Yêu cầu khóa bảng `ROW SHARE`. Ở mức cấp dòng, nó thiết lập khóa trên những bản ghi thỏa mãn điều kiện WHERE.
- Lệnh thao tác dữ liệu (`UPDATE`/`DELETE`/`INSERT`): Yêu cầu khóa bảng `ROW EXCLUSIVE`. Ở mức cấp dòng, nó thiết lập khóa cập nhật lên các bản ghi chịu tác động.

Các thuật ngữ `ROW SHARE` hoặc `ROW EXCLUSIVE` là tên gọi của mức độ khóa cấp bảng (table-level mode names). Sự hiện diện của chúng không có nghĩa là mọi dòng (row) trong bảng đang bị khóa. Ứng dụng phải hiểu sự khác biệt rõ ràng giữa "khóa bảng" và "khóa dòng".

Nếu một luồng gọi lệnh khóa cấp bảng mạnh nhất:

```sql
lock table tenant_quota in access exclusive mode;
```

Lệnh này yêu cầu khóa bảng `ACCESS EXCLUSIVE`. Do mức độ khóa này xung đột với cả `ACCESS SHARE`, nên bất kỳ câu lệnh `SELECT` thông thường nào gọi từ luồng khác cũng sẽ bị chặn lại (wait). Đây là một thiết kế ranh giới khóa cứng (hard block), khác biệt hoàn toàn với cơ chế khóa cấp dòng (row-level lock) linh hoạt.

## 5. Tương thích khóa cấp dòng (Row-lock compatibility)

Khi luồng A giữ khóa cấp dòng `FOR UPDATE` trên dòng dữ liệu X, khóa này sẽ xung đột (conflict) với bất kỳ nỗ lực nào từ luồng khác muốn:

- Thực thi `UPDATE` hoặc `DELETE` trên X.
- Yêu cầu một khóa `FOR UPDATE` khác trên X.
- Yêu cầu một khóa `FOR NO KEY UPDATE` trên X.

Ngược lại, lệnh `FOR UPDATE` của A KHÔNG chặn truy vấn đọc `plain SELECT` của luồng C. Lý do là C không yêu cầu bất kỳ cờ khóa (lock intent) nào; C chỉ dựa vào kiến trúc Snapshot MVCC để đọc phiên bản dữ liệu (tuple version) gần nhất đã được hệ thống commit.

PostgreSQL hỗ trợ các mức khóa dòng yếu hơn, chẳng hạn như `FOR SHARE` và `FOR KEY SHARE`. Chúng cho phép duy trì sự toàn vẹn của khóa ngoại nhưng lại không đảm bảo ngăn chặn các luồng thực hiện `UPDATE` hoặc khóa mạnh hơn. Ứng dụng nên ưu tiên `FOR UPDATE` để đảm bảo độc quyền xử lý (mutation serialization) trên logic kinh doanh (read-modify-write).

## 6. Tính hiển thị MVCC với giao dịch đang mở (MVCC reader vs uncommitted writer)

Trong khi luồng A thực thi lệnh `UPDATE` sửa giá trị thành `12` nhưng chưa hoàn tất giao dịch (uncommitted), PostgreSQL tạo ra một phiên bản bản ghi (tuple version) mới. Lệnh `SELECT` của luồng C sử dụng Snapshot ở mức cô lập `READ COMMITTED` (hoặc cao hơn) sẽ quan sát thấy:

- Phiên bản mới `12` của luồng A -> KHÔNG nhìn thấy (invisible).
- Phiên bản cũ `10` đã commit -> Hiển thị (visible).

Lưu ý rằng C sẽ không đọc phải dữ liệu rác (dirty read). Giá trị `12` sẽ chỉ hiện ra trước các truy vấn khác sau khi luồng A thực thi lệnh `COMMIT`.

## 7. Trạng thái của luồng chờ khóa (Waiter outcomes)

### Khi luồng đang giữ khóa Commit (Holder commit)

Luồng B bị tạm ngưng khi yêu cầu cập nhật bản ghi do luồng A đang giữ khóa. Khi A thực hiện `COMMIT`, luồng B được đánh thức và khóa được nhường cho B. 
Tuy nhiên, tại mức cô lập `READ COMMITTED`, PostgreSQL buộc luồng B phải đánh giá lại (re-evaluate) câu truy vấn của nó dựa trên phiên bản dữ liệu mới cập nhật (latest committed row) của A.

Giả sử câu lệnh của B có sử dụng điều kiện logic (predicate):

```sql
update tenant_quota
set quota = :newQuota
where tenant_id = :id
  and revision = :expectedRevision;
```

Trong lúc chờ, nếu luồng A đã hoàn thành và tăng giá trị `revision`, khi B tiếp tục xử lý, điều kiện `revision = :expectedRevision` sẽ không còn thỏa mãn. Kết quả là số lượng bản ghi bị tác động (affected-row) của B sẽ bằng 0. Luồng B có thể dùng dấu hiệu này để nhận dạng xung đột và thực hiện các logic phục hồi (retry).

### Khi luồng đang giữ khóa Rollback (Holder rollback)

Nếu luồng A quyết định hủy giao dịch (Rollback), những thay đổi (ví dụ: `12`) chưa được commit sẽ bị hệ thống loại bỏ (invisible). Khóa được cấp trên bản ghi bị giải phóng, luồng B được nhận khóa, đọc phiên bản dữ liệu và tiếp tục thực hiện với giá trị cũ `10`.

Việc hệ thống quản lý cơ sở dữ liệu làm mọi thứ tự động đồng nghĩa với việc ứng dụng phải đảm bảo các điểm ranh giới Commit/Rollback. Ứng dụng không được phép hủy hoặc thao túng (release manual) khóa ở giữa một chu kỳ giao dịch trừ khi sử dụng Savepoints, nhưng việc dùng Savepoints trong Spring quản lý giao dịch đòi hỏi quy trình rất tỉ mỉ để không phá vỡ tính nhất quán.

## 8. Chu kỳ sống của khóa (Lock lifetime)

Khóa được lấy kể từ lúc lệnh SQL (có yêu cầu khóa) được gửi đến và cơ sở dữ liệu xử lý xong thao tác cấp khóa.
Khóa đó sẽ được duy trì liên tục cho đến khi quá trình giao dịch vật lý kết thúc hoàn toàn (physical transaction end):

```text
Spring Proxy gọi phương thức bắt đầu giao dịch
  Repository gọi hàm SQL có FOR UPDATE -> Khóa được cấp
  Thực thi các nghiệp vụ xử lý tại Java -> Khóa tiếp tục được giữ
  Hibernate gọi hàm flush() sinh lệnh UPDATE -> Khóa tiếp tục được giữ
Phương thức kết thúc, Spring Proxy xử lý COMMIT/ROLLBACK -> Khóa ĐƯỢC GIẢI PHÓNG
```

Gọi hàm truy xuất có nhãn `readOnly=true` không làm giải phóng sớm các khóa.
Gọi hàm `EntityManager.flush()` cũng không làm nhả khóa; nó chỉ làm trống bộ nhớ đệm để gửi lệnh SQL xuống cơ sở dữ liệu.
Nếu mã nguồn gọi vòng (self-invocation) trong Java một cách trực tiếp mà bỏ qua Spring Proxy, annotation `@Transactional` có thể không hoạt động, khiến các câu lệnh thực thi trong trạng thái `autocommit` (mỗi lệnh SQL là một giao dịch ngắn). Điều này sẽ vô hiệu hóa hoàn toàn mục đích chặn khóa của lệnh `FOR UPDATE`.

## 9. Khóa Cấp Bảng và Tính Phức Tạp (Table Lock Nuances)

Tất cả các truy vấn đều tạo ra mức độ khóa quan hệ cấp bảng (table-level lock). Chẳng hạn, truy vấn `SELECT` thông thường lấy mức khóa `ACCESS SHARE`. Mức khóa yếu này được thiết kế để không xung đột với các lệnh DML (`UPDATE`/`DELETE`), do đó chúng có thể diễn ra song song. Nó chỉ cản trở duy nhất `ACCESS EXCLUSIVE`.

Việc yêu cầu một khóa bảng một cách tường minh (`lock table ... in ... mode`) phải cẩn trọng:

- Lựa chọn đúng cờ khóa (mode) tương thích với kịch bản yêu cầu.
- Lệnh thiết lập khóa ở cấp độ bảng phải tuân thủ thứ tự truy xuất nếu có (acquisition order) để tránh deadlock toàn cục.
- Thiết lập thông số quá hạn (`timeout`) chặt chẽ.
- Hạn chế tối đa ảnh hưởng chéo đến các luồng không liên quan hoặc các tenant khác trong bảng dữ liệu.

Tuyệt đối tránh hiện tượng mập mờ, thiếu tài liệu giải thích (insufficient design review) đối với các lệnh sử dụng `LOCK TABLE`. Quá trình review code phải làm rõ lý do tại sao khóa bảng được dùng, mode nào, và trong bao lâu.

## 10. Giám sát hệ thống khóa PostgreSQL (PostgreSQL observability)

Kiểm tra khóa hiện hữu trên đối tượng bảng (table level):

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

Xem danh sách các giao dịch bị chặn và giao dịch gây chặn:

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

Quan sát các View giám sát của PostgreSQL (như `pg_locks`) CẦN CẨN THẬN. Ở cấp dòng, cơ sở dữ liệu không lưu trữ định danh khóa dòng chi tiết (one row per held lock) vào bảng `pg_locks` để tiết kiệm tài nguyên bộ nhớ. Thay vào đó, khóa cấp dòng chỉ được biểu diễn khi có sự tranh chấp, tức là khi một luồng (waiter) tiến hành chờ đợi (wait on transaction-ID lock) tiến trình đang thực thi. Việc sử dụng kết hợp `pg_stat_activity` và hàm `pg_blocking_pids` là phương thức chuẩn xác nhất để chẩn đoán hệ thống bị nghẽn (contention).

## 11. Xác định nguyên nhân gốc rễ theo phân tầng (Root cause theo layer)

### Tầng 1: Hệ quản trị Database (PostgreSQL)

Các luật kiểm soát tính tương thích khóa và cơ chế hiển thị của MVCC đang hoạt động theo đúng chuẩn thiết kế kỹ thuật (work as designed). Sự sai lầm không xuất phát từ việc cơ sở dữ liệu thực thi lệnh sai, mà ở việc hệ thống yêu cầu một khóa nhưng mong đợi hiệu ứng phong tỏa toàn bộ (pessimistic reservation) từ lệnh đọc không khóa.

### Tầng 2: Môi trường Spring (Transaction Manager)

Sử dụng `@Transactional` xác lập không gian ranh giới (sets boundary); Trong khi hàm của Repository định đoạt chế độ khóa (sets lock mode). Nếu hai thành phần này không hoạt động đồng bộ, thời gian sống (lifetime) của khóa có thể bị cắt bớt hoặc tồn tại quá lâu, gây ra hiện tượng treo kết nối (connection timeout).

### Tầng 3: Trình ánh xạ Hibernate/JPA (ORM)

Quá trình ORM xử lý Persistence Context và Entity Identity hoàn toàn không có cơ chế khóa tự thân tại cấp độ bộ nhớ chia sẻ với cơ sở dữ liệu (no JVM level multi-instance database lock). Lệnh yêu cầu khóa `@Lock` của JPA (Pessimistic lock) phải chuyển ngữ thành từ khóa hợp lệ (SQL locking clause, vd: `FOR UPDATE`) và thực thi tại cơ sở dữ liệu.

### Tầng 4: Logic ứng dụng (Application code)

Phát sinh lỗi do nhà phát triển trộn lẫn các ranh giới: Giao dịch cấp cơ sở dữ liệu, bộ đệm JPA Persistence, và khái niệm "khóa độc quyền", nhưng thiếu khả năng phân định rõ ràng cách MVCC cung cấp phiên bản đọc (visibility) đối lập với cách lệnh ghi áp đặt khóa (locking).

## 12. Quá hạn và Bế tắc (Timeout, deadlock và aborted state)

PostgreSQL sử dụng thông số cấu hình `lock_timeout` để thiết lập thời gian tối đa mà một truy vấn có thể phải chịu đựng để đợi khóa. Khi một truy vấn (query) hết thời gian chờ, hệ thống ném ngoại lệ với mã `55P03`. Giao dịch hiện tại sẽ buộc phải chuyển sang trạng thái bị vô hiệu hóa (aborted transaction) và phải `ROLLBACK`.

Ngược lại, giới hạn `statement_timeout` là tổng thời lượng tối đa cho toàn bộ câu lệnh thực thi mà không phụ thuộc vào trạng thái chờ khóa (wait-time independent). Cả hai cài đặt này nên được xem xét trong bối cảnh ứng dụng để đảm bảo độ khả dụng, ngăn không cho hàng nghìn request bị treo cục bộ.

Khi một tình huống khóa chéo vòng tròn xảy ra (ví dụ: A chờ B, B chờ A), hệ thống sẽ ngắt kết nối một tiến trình được xem là nạn nhân (deadlock victim), trả về mã `40P01`. Ứng dụng phải tự triển khai mô hình phân lập Retry giao dịch từ đầu đối với mã lỗi này để làm sạch bộ nhớ ngữ cảnh và nhận trạng thái dữ liệu (state) mới. Xem hướng dẫn chống khóa ngược chiều tại `DB-008`.

## 13. Hệ quả sự cố máy chủ và Đứt kết nối (Crash và connection loss)

Khi tiến trình ứng dụng Java kết thúc bất ngờ hoặc mạng ngắt, kết nối TCP bị gián đoạn. PostgreSQL sẽ phát hiện ra sự cố đứt gãy mạng, tự động Rollback giao dịch chưa hoàn tất của tiến trình đó và dọn dẹp các khóa.

Tuy nhiên, lỗi treo do chờ đợi I/O mà không có cơ chế Timeout hoặc cơ chế ngắt mạng (keep-alive) không xử lý ngay sẽ gây ra rủi ro trạng thái treo dài lâu (idle in transaction). Những phiên bản giao dịch treo (stuck transaction) sẽ khóa cứng tài nguyên và ảnh hưởng đến cả cơ chế tự động dọn rác (autovacuum), kéo tụt hiệu năng chung. Thiết kế Connection Pool tại tầng ứng dụng bắt buộc phải bổ sung tham số Maximum Lifetime để tái sử dụng hoặc đào thải các phiên giao dịch tồn đọng này.

## 14. Môi trường triển khai đa máy chủ (Multi-instance behavior)

Cơ chế khóa dòng trong PostgreSQL là giải pháp điều phối khóa (coordinate) đáng tin cậy giúp bảo vệ mọi thao tác trên dữ liệu (all connections), không quan trọng yêu cầu SQL đó xuất phát từ cụm máy chủ ứng dụng nào (node A hay node B). 

Việc thay thế điều này bằng từ khóa đồng bộ luồng JVM (`synchronized` ở Java) là giải pháp sai lầm, bởi vì khóa JVM không có khả năng bảo vệ tài nguyên nếu luồng truy vấn được gọi từ hệ thống bên ngoài hoặc từ các PODs riêng biệt của Kubernetes. Cơ sở dữ liệu bắt buộc là công cụ chống xung đột cho các thao tác cấp hệ thống (system-wide invariant).

## 15. Tóm tắt khác biệt: Mong đợi và Thực tế (Expected vs Actual summary)

| Hiểu lầm của lập trình viên (Assumption) | Trạng thái thực tế (Actual) |
| --- | --- |
| Mọi lệnh `SELECT` trong `@Transactional` đều bảo vệ độc quyền (reserve) dòng. | Lệnh `plain SELECT` chỉ đọc bản ghi (MVCC-reads), không phát sinh khóa dòng. |
| Một dòng bị khóa sẽ chặn bất kỳ truy vấn `SELECT` nào. | Truy vấn `plain SELECT` vẫn đọc thành công phiên bản dữ liệu cũ (committed). |
| Gọi `entityManager.flush()` giải phóng khóa dòng hiện tại. | Khóa được giải phóng CHỈ khi ranh giới giao dịch hoàn tất (Commit/Rollback). |
| Sử dụng khóa bảng (`LOCK TABLE`) sẽ đình chỉ mọi yêu cầu truy vấn kể cả `SELECT`. | Yêu cầu phải xác định chính xác loại Mode tương thích, `ACCESS EXCLUSIVE` mới chặn đọc. |
| Timeout ngắn sẽ giúp vượt qua tranh chấp và giữ luồng an toàn. | Hết thời gian (`timeout`) khiến giao dịch hiện tại bị lỗi, ứng dụng phải tự quản lý quy trình Retry. |
| Khóa bằng `synchronized` (JVM) có chức năng tương đương Row Lock DB. | Tính phạm vi của JVM không tác dụng đối với cơ sở dữ liệu khi hệ thống thiết lập đa máy chủ (Scale-out). |
