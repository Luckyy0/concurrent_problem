# Khóa bi quan (Pessimistic locking) và `FOR UPDATE`

## Mục tiêu

Khóa bi quan (`pessimistic locking`) là chiến thuật: "Cẩn tắc vô áy náy" – xí chỗ và khóa cứng dữ liệu ở Database ngay từ lúc bắt đầu đọc, trước khi hệ thống kịp đưa ra bất kỳ quyết định tính toán nào. 
Tài liệu này sẽ giải thích cơ chế hoạt động chung; tuy nhiên khi áp dụng thực tế, bạn vẫn phải trả lời rõ: bạn đang khóa dòng nào, bạn muốn bảo vệ quy tắc gì, và nếu phải đứng chờ thì bạn sẽ xử lý lỗi ra sao.

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Khóa bi quan (`pessimistic lock`) | Đòi khóa dữ liệu ngay lập tức vì luôn "bi quan" lo sợ rằng sẽ có luồng khác nhảy vào tranh giành. |
| Đọc kèm khóa (`locking read`) | Một câu truy vấn (query) có tác dụng "2 trong 1": vừa đọc dữ liệu lên, vừa yêu cầu Database khóa dòng đó lại. |
| `PESSIMISTIC_WRITE` | Một cấu hình trong code JPA/Hibernate báo hiệu muốn xin khóa để chuẩn bị ghi (sửa) dữ liệu. |
| `FOR UPDATE` | Một câu lệnh quyền lực trong PostgreSQL, gắn vào cuối câu SELECT để khóa cứng các dòng vừa tìm được. |
| Người giữ khóa (`holder`) | Giao dịch (Transaction) đang chạy nhanh chân chộp được khóa trước. |
| Người chờ khóa (`waiter`) | Giao dịch đến sau, bị chặn lại và phải đứng xếp hàng chờ khóa. |
| Vòng đời của khóa (`lock lifetime`) | Thời gian từ lúc xin được khóa cho tới khi giao dịch đó kết thúc (bằng lệnh commit hoặc rollback). |
| Chờ có giới hạn (`bounded wait`) | Đứng chờ nhưng có hẹn giờ (timeout), quá giờ là bỏ cuộc chứ không chờ mù quáng. |
| Kiểm định lại (`revalidation`) | Sau khi đứng chờ vã mồ hôi mới lấy được khóa, bạn phải đọc lại dữ liệu và kiểm tra lại từ đầu xem điều kiện nghiệp vụ còn đúng không. |
| Thứ tự khóa (`lock order`) | Khi muốn xin khóa nhiều dòng cùng lúc, bạn phải tuân thủ một thứ tự nhất quán (ví dụ từ bé đến lớn) để tránh bị Bế tắc (Deadlock). |

## Cơ chế cốt lõi hoạt động ra sao?

```text
BẮT ĐẦU GIAO DỊCH (BEGIN)
  SELECT kèm lệnh khóa
  → Đòi khóa dòng thành công
  → Đọc trạng thái dữ liệu hiện tại
  → Tính toán logic nghiệp vụ
  → Cập nhật dữ liệu (UPDATE)
KẾT THÚC GIAO DỊCH (COMMIT/ROLLBACK)
  → Database tự động nhả khóa
```

Nếu có một giao dịch khác (người chờ) cũng muốn khóa đúng dòng dữ liệu đó, nó sẽ bị chặn lại (đứng chờ), hoặc báo lỗi ngay lập tức, hoặc bị văng lỗi timeout nếu chờ quá lâu.
**Lưu ý cực kỳ quan trọng:** Sau khi người giữ khóa chạy xong và nhả khóa, người chờ tuyệt đối **không được** dùng cái dữ liệu cũ mèm mà nó lỡ đọc trước đó. Nó phải xài dữ liệu mới toanh vừa được trả về từ lệnh đọc-kèm-khóa và kiểm tra lại toàn bộ logic nghiệp vụ (revalidate).

> **Nói ngắn gọn:** Khóa bi quan đúng nghĩa không chỉ làm cho "người đến sau phải đứng chờ"; mà nó ép người đến sau phải đưa ra quyết định dựa trên cái kết quả mà người đến trước vừa làm xong.

## Lệnh `FOR UPDATE` trong PostgreSQL

```sql
SELECT *
FROM show_seat
WHERE show_id = :showId
  AND seat_no = :seatNo
FOR UPDATE;
```

Lệnh `FOR UPDATE` sẽ khóa chặt các dòng được chọn, hệt như thể bạn sắp sửa lệnh `UPDATE` trên chúng. Bất kỳ lệnh `UPDATE`, `DELETE` hay `SELECT FOR UPDATE` nào khác chạm vào dòng này đều phải đứng chờ giao dịch hiện tại chạy xong. 
Tuy nhiên, lệnh `SELECT` bình thường (không có chữ FOR UPDATE) thì vẫn lấy được dữ liệu cũ ra xem một cách vô tư (nhờ cơ chế MVCC).

- **Khóa tồn tại bao lâu?** Nó sống đến tận lúc kết thúc giao dịch (transaction end), chứ không hề bị nhả ra khi hàm code Repository của bạn chạy xong. Nếu sau khi khóa, code của bạn lại đi gọi API bên ngoài (gửi Email, gọi thẻ tín dụng...), bạn sẽ ngâm cái kết nối Database đó rất lâu và làm cho hàng đợi những người đứng chờ dài dằng dặc.
- **Chỉ khóa cái đang có thật:** Nếu câu SELECT không tìm thấy dòng nào, nó sẽ không khóa cái gì cả. Nó KHÔNG có khả năng chặn người khác thêm mới (insert) một dòng dữ liệu tương tự vào tương lai.

## Khai báo trong JPA/Hibernate

Trong Spring Data JPA, việc xin khóa cực kỳ đơn giản, chỉ cần thêm 1 dòng chữ:

```java
public interface ShowSeatRepository
        extends JpaRepository<ShowSeat, ShowSeatId> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select s from ShowSeat s where s.id = :id")
    Optional<ShowSeat> findForUpdate(@Param("id") ShowSeatId id);
}
```

Hibernate sẽ tự động dịch `@Lock` này thành câu lệnh `FOR UPDATE` chuẩn xác nhất cho PostgreSQL. (Lưu ý: câu SQL thực tế có thể thay đổi tùy phiên bản Hibernate, nên hãy luôn kiểm tra log khi chạy thực tế).

Phạm vi giao dịch (Transaction boundary) phải bao trùm toàn bộ:
```text
Truy vấn xin khóa → Kiểm định lại (revalidate) → Mọi thao tác đổi dữ liệu → Flush (Đẩy xuống DB) → Commit (Chốt sổ)
```
Việc thêm chữ `@Lock` sẽ trở nên vô dụng nếu bạn gọi hàm bị sai (lỗi tự gọi hàm - self-invocation), giao dịch được mở quá rộng, hoặc đối tượng đã lỡ bị tải lên bộ nhớ (cached) từ trước khi bạn gọi hàm xin khóa.

## Các chiến thuật khi phải đứng chờ (Wait policy)

### Bounded wait (Chờ có hẹn giờ)
Chấp nhận đứng chờ người trước một khoảng thời gian ngắn rồi mới kiểm tra lại. Rất phù hợp nếu các luồng làm việc rất nhanh.
Trong PostgreSQL, bạn dùng `lock_timeout` để giới hạn thời gian chờ khóa. Đừng nhầm lẫn nó với:
- `statement_timeout`: Giới hạn thời gian chạy của cả một câu SQL dài.
- transaction/application deadline: Giới hạn của toàn bộ công việc (nhiều lệnh SQL).
- connection acquisition timeout: Thời gian chờ xin một kết nối (connection) từ pool.
- client timeout: Thời gian API chờ phản hồi.
Các thông số này phải được lồng ghép hợp lý, nhớ chừa lại một ít thời gian để còn kịp rollback và báo lỗi cho người dùng.

### `NOWAIT` (Báo lỗi ngay)
Báo lỗi ngay nếu thấy dòng đó đang bị khóa. Rất hợp lý nếu hệ thống của bạn muốn báo ngay chữ `BUSY` (Hệ thống đang bận) cho người dùng. Nhưng đừng dùng nếu người dùng hiểu nhầm lỗi này thành "Dữ liệu không tồn tại".

### `SKIP LOCKED` (Bỏ qua dòng bị khóa)
Thay vì đứng chờ, cứ lấy đại dòng tiếp theo không bị khóa. Cái này cực kỳ lợi hại khi làm hệ thống hàng đợi công việc (work queue - nhiều người cùng chạy vào xí việc). Đừng dùng cái này nếu người dùng chỉ định đích danh muốn mua một chiếc ghế cụ thể.

## Timeout, Bế tắc (Deadlock) và Thử lại (Retry)

Bị văng lỗi quá hạn (timeout) hoặc bế tắc (deadlock) đồng nghĩa với việc Giao dịch đã hỏng. Sau khi nhận lỗi SQL, hãy nhớ để cho giao dịch đó hoàn tác (rollback) sạch sẽ trước khi quyết định làm gì tiếp.

Việc "Thử lại" (Retry) chỉ an toàn khi:
- Lệnh gửi lên có khả năng chạy lại mà không bị trùng lặp (idempotency contract).
- Lần thử mới dùng một kết nối và giao dịch HOÀN TOÀN MỚI.
- Dữ liệu được đọc và kiểm tra lại từ đầu.
- Có giới hạn số lần thử, thời gian thử và thời gian nghỉ (backoff) hợp lý.

**Không được mù quáng thử lại** nếu lỗi đó là lỗi nghiệp vụ (ví dụ: ghế đã có người mua thật rồi) hoặc bạn đang rải try-catch bắt bừa mọi lỗi `DataAccessException`.

## Thứ tự khóa khi lấy nhiều dòng (Multi-row lock order)

Khi một thao tác muốn khóa cùng lúc nhiều dòng:
1. Xóa các ID bị trùng lặp.
2. Chuẩn hóa bằng một khóa cố định.
3. Luôn luôn xin khóa theo CÙNG MỘT THỨ TỰ (ví dụ ID từ nhỏ đến lớn) ở mọi nơi trong code.
4. Đảm bảo khóa đủ số dòng mong muốn rồi mới đi cập nhật.
5. Giữ thời gian chạy thật ngắn.

Xin khóa lung tung không theo thứ tự sẽ dẫn đến Vòng luẩn quẩn (wait-for cycle). Khi đó hệ thống dò Deadlock của PostgreSQL sẽ phải giết một giao dịch làm "nạn nhân". Đừng ỷ lại vào Database, hãy thiết kế code chuẩn ngay từ đầu.

Dùng SQL gom chung lại thường an toàn hơn là bạn viết vòng lặp phụ thuộc request xin từng cái:
```sql
SELECT *
FROM show_seat
WHERE show_id = :showId
  AND seat_no = ANY(:seatNos)
ORDER BY show_id, seat_no   -- Rất quan trọng để khóa theo đúng thứ tự!
FOR UPDATE;
```

## Khóa dòng đã biết và Quy tắc diện rộng

Khóa dòng bi quan (Row locking) cực kỳ hợp lý khi bạn biết chính xác mình đang thao tác trên dữ liệu cố định nào: Tài khoản ngân hàng, Hàng trong kho, Ghế xem phim, dòng guard row.

Nhưng nó **KHÔNG THỂ** tự bảo vệ bạn khỏi các trường hợp:
- Dòng dữ liệu chưa tồn tại.
- Lệnh yêu cầu "không có bản ghi nào thỏa mãn điều kiện".
- Tính toán sức chứa từ một tập hợp dữ liệu (người khác lén Insert thêm dòng mới - phantom inserts).
- Các quy tắc chạy ngang qua nhiều bảng không liên quan (không có giao thức khóa chung).

Khi đó bạn có thể cần dùng các Ràng buộc (unique/check constraint), lệnh SQL có điều kiện, thêm các dòng bảo vệ nhân tạo (stable guard row) hoặc dùng mức cách ly cao nhất (`SERIALIZABLE`).

## Bảng xử lý sự cố (Commit, rollback và crash)

| Người giữ khóa (Holder) bị gì? | Số phận của Người chờ khóa (Waiter/recovery) |
| --- | --- |
| Lưu thành công (Commit) | Người chờ sẽ được lấy khóa, và đọc được dữ liệu MỚI NHẤT do người giữ khóa vừa tạo ra. |
| Bị lỗi và Hoàn tác (Rollback) | Người chờ sẽ được lấy khóa, và vẫn đọc được dữ liệu cũ rích như trước khi có sự cố. |
| Rớt mạng (Connection loss) trước khi Commit | PostgreSQL sẽ tự động báo lỗi hủy giao dịch đó và nhả khóa ra cứu người chờ. |
| Commit xong xuôi nhưng rớt mạng chưa kịp báo về (Response loss) | Dữ liệu dưới Database vẫn là bản mới đã commit. Phải xử lý phục hồi bằng cách dùng tính lũy đẳng (idempotency) hoặc phát lại kết quả cũ (replay). |

**Cảnh báo nghiêm trọng:** Cấp độ ứng dụng (Application) không có câu lệnh API nào để tự gọi hàm mở khóa (`unlock`) cho một dòng dữ liệu. Cách duy nhất để nhả khóa là Kết thúc giao dịch. Nếu code của bạn bị ngâm, treo máy, kết nối (connection leak) sẽ rò rỉ và đây là rủi ro vận hành vô cùng nghiêm trọng!

## Môi trường nhiều máy chủ (Multi-instance)

Khóa dòng của Database là khóa tối thượng: Nó chặn (coordinate) được tất cả các luồng trên mọi máy chủ (App instance) dùng chung cái Database PostgreSQL đó. Các loại khóa nội bộ của Java (JVM-local) hoàn toàn không có khả năng này.

Tuy nhiên, thêm nhiều máy chủ (Scale-out) sẽ làm tình hình tồi tệ hơn vì:
- Càng đông máy chủ nhảy vào tranh một dòng "điểm nóng" (hot row) thì lượng waiter đứng chờ càng dài.
- Tổng số kết nối (connection) vô ích bị ngâm trong lúc đứng chờ sẽ tăng vọt.
- Băng thông sẽ ngập lụt vì hàng loạt lệnh thông báo lỗi (timeout/cancellation traffic).
- Bạn sẽ cần thêm cơ chế kiểm soát số lượng đầu vào (admission control) và theo dõi gắt gao.

Ghi nhớ: Code chạy đúng (Correctness) KHÔNG đồng nghĩa với việc Hệ thống chịu tải tốt (throughput tốt) khi bị tranh chấp liên tục cường độ cao!

## Theo dõi hệ thống (Quan sát)

Khi có sự cố, hãy kết hợp các công cụ này để điều tra:
- Dùng `pg_stat_activity.wait_event_type`, `wait_event`, `xact_start` để xem ai đang chờ.
- Dùng hàm `pg_blocking_pids(pid)` để truy ra mặt mũi kẻ đang cầm khóa.
- Dùng `pg_locks` để vẽ sơ đồ phân tích xem ai đang khóa ai (lock graph).
- Theo dõi mã lỗi SQLSTATE `55P03` (không lấy được khóa/timeout).
- Theo dõi mã lỗi SQLSTATE `40P01` (trở thành nạn nhân của Deadlock).
- Theo dõi các thông số kết nối của Pool (active/pending/acquisition timeout).
- Luôn gắn một mã theo dõi cấp độ ứng dụng (correlation ID) vào log kết hợp với kết quả lỗi để dễ dò đường.

Thông tin về khóa dòng không phải lúc nào cũng hiện nguyên hình dễ hiểu trong `pg_locks`; người đứng chờ (waiter) thường là đang đứng chờ cái ID giao dịch (transaction ID) của kẻ đang giữ khóa.

## Khi nào thì nên chọn Khóa bi quan?

**ƯU TIÊN DÙNG KHI:** Quy trình của bạn có nhiều bước phức tạp cần đọc trạng thái mới nhất, bạn đã biết chính xác dòng dữ liệu đích là dòng nào, tỷ lệ đụng độ (conflict) đủ dày đặc, và bạn hứa sẽ hoàn thành giao dịch (critical section) thật ngắn.

**HÃY SO SÁNH VỚI:**
- Dùng lệnh SQL cập nhật có điều kiện trực tiếp (conditional atomic SQL) nếu bạn chỉ có đúng 1 thao tác duy nhất.
- Dùng Khóa lạc quan (`@Version`) nếu lượng tranh chấp rất thấp.
- Dùng Ràng buộc (Constraint) nếu quy tắc của bạn có thể diễn đạt trực tiếp bằng tính năng của Database.
- Dùng Hàng đợi/Kiểm soát đầu vào (queue/admission control) nếu gặp các điểm nóng (hot key) bị nghẽn bền vững.

**Lời khuyên:** Đừng vội chèn bừa công cụ vào code khi bạn chưa vẽ ra rõ ràng quy tắc bảo vệ (invariant) và số phận của kẻ thua cuộc (loser outcome)!

## Liên kết tài liệu tham khảo

- [LOCK-003 — Pessimistic write lock với FOR UPDATE](../locking/pessimistic-write-for-update/README.md)
- [DB-007 — Row/table lock lifecycle](../postgresql/row-table-lock-lifecycle/README.md)
- [DB-008 — Opposite row order deadlock](../postgresql/opposite-row-order-deadlock/README.md)
- [DB-010 — Work claiming với SKIP LOCKED](../postgresql/skip-locked-work-queue/README.md)
- [SPR-007 — Long transaction và connection pool](../spring/connection-pool-long-transaction/README.md)
- [PostgreSQL locks và lock lifetime](postgresql-locks.md)
- [Deadlock và retry an toàn](deadlocks-and-retries.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
