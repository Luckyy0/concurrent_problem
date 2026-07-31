# Các loại khóa (Locks) trong PostgreSQL và Thời gian giữ khóa

## Mục tiêu

Tài liệu này sẽ cung cấp cho bạn bộ từ vựng chuẩn xác nhất về các loại khóa (khóa dòng - row lock, khóa bảng - table lock), tình trạng chờ khóa (lock wait), quá hạn (timeout) và vòng đời của một cái khóa (lock lifetime) trong cơ sở dữ liệu PostgreSQL. Tuy nhiên, khi vào từng bài toán thực tế, bạn vẫn phải tự phân tích xem mình đang bảo vệ quy tắc gì, câu lệnh nào sẽ lấy khóa và luồng nào sẽ bị chặn lại nhé.

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Khóa cấp độ dòng (`row-level lock`) | Khóa cụ thể một vài dòng (row) dữ liệu để những người khác không nhảy vào sửa cùng lúc. |
| Khóa cấp độ bảng (`table-level lock`) | Khóa liên quan đến cả một bảng; nhưng đừng hiểu nhầm là nó luôn luôn khóa mọi dòng dữ liệu trong đó nhé. |
| Chờ khóa (`lock wait`) | Tình trạng một luồng đến sau phải đứng chôn chân chờ một luồng khác nhả khóa ra. |
| Người giữ khóa (`lock holder`) | Giao dịch đang sung sướng chạy vì đã cầm được khóa trong tay. |
| Người chờ khóa (`waiter`) | Giao dịch tội nghiệp đến chậm, bị chặn lại và đang đứng chờ. |
| Hàng đợi khóa (`lock queue`) | Một đoàn tàu các giao dịch xếp hàng dài chờ đợi phía sau người giữ khóa. |
| Vòng đời của khóa (`lock lifetime`) | Thời gian tính từ lúc chộp được khóa cho tới lúc kết thúc giao dịch (commit hoặc rollback). |
| `lock_timeout` | Hẹn giờ giới hạn xem mình sẽ đứng chờ bao lâu trước khi bỏ cuộc (báo lỗi). |
| `statement_timeout` | Hẹn giờ giới hạn cho toàn bộ thời gian chạy của một câu lệnh SQL. |
| Bộ dò bế tắc (`deadlock detector`) | "Cảnh sát" của PostgreSQL, chuyên đi tìm các vụ kẹt xe thành vòng tròn (luẩn quẩn) và tự động "giết" một giao dịch để giải tỏa. |

## Khóa cấp độ dòng (Row locks)

Các câu lệnh như `SELECT ... FOR UPDATE`, `UPDATE` và `DELETE` sẽ tự động giật lấy khóa dòng (row-level lock) phù hợp.
Khóa này cực kỳ chung thủy: Nó được giữ chặt cho tới khi giao dịch (transaction) kết thúc. Nó **KHÔNG** tự động nhả ra khi hàm Java của bạn kết thúc, hoặc khi bạn đang bận gọi một API của hệ thống khác.

```sql
SELECT *
FROM payment_order
WHERE id = :id
FOR UPDATE;
```

Nếu một giao dịch khác cũng muốn khóa chính dòng này, nó sẽ phải ngoan ngoãn đứng chờ, bị văng lỗi timeout, hoặc xui xẻo nhất là bị hệ thống dò Deadlock tóm cổ làm "nạn nhân".

> **Nói ngắn gọn:** Tuổi thọ của khóa chính là tuổi thọ của Giao dịch (Transaction). Nếu bạn chèn các đoạn code chạy chậm rì (như gọi API) vào giữa câu lệnh SELECT và lệnh COMMIT, bạn đang kéo dài tuổi thọ của khóa ra, kể cả khi đoạn code đó chẳng đụng chạm gì tới Database!

## Tầm nhìn dữ liệu (Visibility) và tình trạng Đứng chờ trong PostgreSQL

Ở mức cách ly mặc định `READ COMMITTED`, người đọc (reader) thường chỉ nhìn thấy phiên bản dữ liệu đã được lưu thành công trước đó. Nếu bạn đang muốn lấy khóa để ghi (locking writer), bạn có thể phải đứng chờ giao dịch trước đó chạy xong. Sau khi chờ xong, dữ liệu và các điều kiện của bạn sẽ được PostgreSQL tự động đánh giá lại.

Hãy nhớ: Một người đứng chờ (waiter) thì vẫn đang cầm khư khư kết nối (connection) tới Database và ngốn tài nguyên của hệ thống. Nếu có quá nhiều người xếp hàng đứng chờ, hồ bơi kết nối (`connection pool`) của bạn sẽ cạn sạch sành sanh trước cả khi CPU của Database kịp quá tải!

## Các chế độ Khóa bảng (Table lock modes)

PostgreSQL có rất nhiều kiểu khóa bảng khi bạn chạy lệnh đổi dữ liệu (DML) hoặc đổi cấu trúc bảng (DDL). 
Thông thường, các lệnh DML (như SELECT, UPDATE) dùng các loại khóa khá "hiền" và chung sống hòa bình với nhau, chúng chỉ thực sự xung đột với các lệnh thay đổi cấu trúc (như ALTER TABLE). Vì vậy, đừng bao giờ suy luận máy móc rằng "Có khóa bảng là mọi query đều bị chặn". Hãy chỉ định rõ tên loại khóa và xem nó có xung đột (compatibility) với ai không.

## Các loại Hẹn giờ (Timeout layers)

Khi hệ thống bị treo, có rất nhiều loại timeout giải quyết các khâu khác nhau. Hãy phân biệt rõ:
- **pool acquisition timeout**: Đợi xin mượn một kết nối (connection) từ hồ bơi (pool) của ứng dụng.
- `lock_timeout`: Đợi cấp khóa dưới Database.
- `statement_timeout`: Thời gian tối đa để chạy xong nguyên một câu lệnh SQL.
- **transaction/application deadline**: Deadline sống còn của toàn bộ quy trình nghiệp vụ (business unit).
- **remote/client timeout**: Thời gian đợi API hoặc mạng phản hồi.

Timeout chỉ giúp bạn kết thúc tình trạng chờ đợi; nó KHÔNG giúp bạn sửa lỗi thiết kế Giao dịch quá to, cũng không giải quyết tận gốc rễ vấn đề quá tải.
Mọi thông số timeout nhỏ phải nằm lọt thỏm bên trong Deadline tổng, và phải chừa lại thời gian để code còn kịp dọn dẹp (cleanup/rollback) và báo kết quả về cho user.

Sau khi PostgreSQL quăng ra lỗi, giao dịch hiện tại đã bị hỏng (aborted state). BẮT BUỘC phải hoàn tác (rollback) sạch sẽ trước khi muốn làm gì tiếp. Nếu có luật cho phép "Thử lại" (Retry), bạn phải mở một Giao dịch hoàn toàn mới.

## Đừng nhầm lẫn: Bế tắc (Deadlock) khác với Chết đói (Starvation)

Deadlock chỉ xảy ra khi có một Vòng luẩn quẩn (A chờ B, B lại chờ A).
Chết đói (Starvation) hay cạn kiệt Pool thì không cần vòng tròn nào cả: Mọi kết nối của bạn đều đang đứng chờ một ông kẹ nào đó đang giữ khóa nhưng chạy quá chậm (hoặc đang kẹt I/O mạng). Hệ thống dò Deadlock của PostgreSQL "bó tay" và sẽ không giải cứu bạn trong trường hợp chết đói này!

## Tuyệt chiêu giảm tuổi thọ của Khóa (Giảm lock lifetime)

Hãy nằm lòng 6 nguyên tắc vàng sau:
1. Đẩy mọi lệnh gọi API (remote I/O) hoặc chờ xử lý (wait) ra KHỎI phạm vi của Giao dịch (Transaction).
2. Chỉ giật lấy khóa (acquire lock) ngay trước khi bạn thực thi câu SQL cập nhật thật ngắn gọn.
3. Luôn kiểm tra lại dữ liệu (revalidate state) ngay sau khi vừa lấy được khóa.
4. Có quy tắc xếp thứ tự (deterministic lock order) thống nhất từ đầu đến cuối khi muốn khóa nhiều dòng (ví dụ khóa từ ID nhỏ đến lớn).
5. Luôn thiết lập giới hạn chờ (bounded timeout/deadline).
6. Tuyệt đối KHÔNG coi việc "Tăng max pool size" là giải pháp duy nhất (nó chỉ làm hàng đợi dài thêm).

## Theo dõi hệ thống (Quan sát)

Trong PostgreSQL, hãy dùng câu lệnh sau để xem ai đang làm gì:

```sql
SELECT pid,
       state,
       wait_event_type,
       wait_event,
       xact_start,
       query_start,
       query
FROM pg_stat_activity
WHERE datname = current_database();
```

Kết hợp `pg_locks`, `pg_blocking_pids(pid)` cùng mã theo dõi (correlation ID) của ứng dụng để ghép đôi Người giữ khóa và Những người đang chờ.
Trên ứng dụng, hãy theo dõi sát sao các chỉ số của Connection Pool: số đang chạy (active), số rảnh rỗi (idle), số đang chờ xin kết nối (pending acquisition), lỗi quá hạn (acquisition timeout) và thời lượng sử dụng.

*Lưu ý bảo mật:* Đừng log các giá trị nhạy cảm (bind values) vào hệ thống! 
Và cuối cùng, nếu bạn thấy rất nhiều kết nối đang ở trạng thái `idle in transaction` kéo dài, đó là tín hiệu khẩn cấp cho thấy Giao dịch của bạn đang bị bọc sai phạm vi (boundary) hoặc thiếu dọn dẹp (cleanup), chứ không hẳn do gọi mạng chậm.

## Môi trường nhiều máy chủ (Multi-instance)

Mỗi máy chủ (application instance) sẽ ôm một cái hồ bơi (Pool) của riêng nó, nhưng tất cả bọn chúng đều đang "uống chung" ngân sách kết nối của một máy chủ PostgreSQL duy nhất!
Khi bạn nhân bản ứng dụng (scale-out), tổng số kết nối tối đa ập xuống Database sẽ tính bằng:

```text
Tổng sức chứa (capacity) = Số lượng máy chủ × Max pool size của mỗi máy chủ
```

Bạn luôn phải trừ hao sức chứa để cho các lệnh bảo trì (migrations, operations) và phòng hờ rủi ro (failover). Việc cấu hình kích cỡ Pool phải đi đôi chặt chẽ với cơ chế kiểm soát đầu vào, độ dài của Giao dịch và sức chịu đựng vật lý của Database.

## Liên kết tài liệu tham khảo

- [LOCK-003 — Pessimistic write lock với FOR UPDATE](../locking/pessimistic-write-for-update/README.md)
- [LOCK-004 — Conditional atomic UPDATE](../locking/conditional-atomic-update/README.md)
- [DB-007 — Row/table lock lifecycle](../postgresql/row-table-lock-lifecycle/README.md)
- [DB-008 — PostgreSQL opposite row order deadlock](../postgresql/opposite-row-order-deadlock/README.md)
- [DB-010 — Concurrent workers với SKIP LOCKED](../postgresql/skip-locked-work-queue/README.md)
- [SPR-007 — Long transaction pool exhaustion](../spring/connection-pool-long-transaction/README.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Deadlock và retry an toàn](deadlocks-and-retries.md)
- [Concurrency testing](concurrency-testing.md)
