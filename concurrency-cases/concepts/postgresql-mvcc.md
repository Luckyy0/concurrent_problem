# Cơ chế MVCC và Tầm nhìn dữ liệu (Row visibility) trong PostgreSQL

## Mục tiêu

Tài liệu này cung cấp bộ từ vựng chuẩn xác để chúng ta cùng hiểu về cơ chế lưu nhiều phiên bản (MVCC), các khái niệm như ảnh chụp (snapshot) và tầm nhìn dữ liệu (visibility) trong PostgreSQL. Tuy nhiên, khi đi vào từng bài toán thực tế, bạn vẫn phải trả lời rõ: câu lệnh của bạn đang đọc phiên bản dữ liệu nào, xung đột khóa nào có thể xảy ra và quy tắc nghiệp vụ nào đang bị đe dọa.

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| MVCC | Multi-Version Concurrency Control: Cơ chế siêu phàm của PostgreSQL giúp duy trì cùng lúc nhiều "phiên bản" của một dòng dữ liệu để người đọc không phải chờ người ghi. |
| Phiên bản dòng (`tuple version`) | Bản sao vật lý của một dòng dữ liệu, được gắn thêm các nhãn (metadata) để biết giao dịch nào được phép nhìn thấy nó. |
| Ảnh chụp (`snapshot`) | Quy tắc xác định xem luồng hiện tại được phép nhìn thấy những dữ liệu nào tại một thời điểm cụ thể. |
| Ảnh chụp từng lệnh (`statement snapshot`) | Ở mức `READ COMMITTED`, mỗi khi chạy một câu lệnh SQL mới, Database lại chụp một bức ảnh mới. |
| Ảnh chụp giao dịch (`transaction snapshot`) | Ở mức `REPEATABLE READ`, Database chỉ chụp đúng một bức ảnh từ lúc bắt đầu giao dịch và dùng mãi. |
| Phiên bản đã chốt (`committed version`) | Dữ liệu từ một giao dịch đã được lưu thành công (commit) và sẵn sàng cho người khác đọc. |
| Dữ liệu rác (`dead tuple`) | Dữ liệu cũ rích hoặc đã bị xóa, không còn bức ảnh (snapshot) nào cần nhìn thấy nó nữa. |
| Dọn rác (`vacuum`) | "Cô lao công" của PostgreSQL đi thu gom và dọn dẹp các dữ liệu rác (dead tuple) để giải phóng ổ cứng. |
| Đọc-tính-ghi (`read-modify-write`) | Mô hình kinh điển dễ gây lỗi: Ứng dụng đọc dữ liệu lên RAM, tính toán cộng trừ, rồi ghi đè con số mới xuống Database. |

## Các phiên bản dòng dữ liệu (Tuple versions)

Trong PostgreSQL, khi bạn chạy lệnh UPDATE để sửa một dòng, nó không sửa đè thẳng vào chỗ cũ (in-place). Về bản chất MVCC, nó sẽ **tạo ra một phiên bản hoàn toàn mới**; còn phiên bản cũ vẫn nằm đó để phục vụ cho các giao dịch đến trước (đang giữ ảnh chụp cũ) cho đến khi rác được dọn (cleanup).

Ứng dụng của bạn nhìn vào thì vẫn thấy đó là một dòng duy nhất. Chính cái Ảnh chụp (Snapshot) sẽ đứng ra làm trọng tài quyết định xem bạn được nhìn thấy phiên bản nào.

> **Nói ngắn gọn:** Nhiều giao dịch có thể cùng lúc đọc chung một phiên bản cũ. Cơ chế MVCC giúp người đọc và người ghi không cản đường nhau (blocking), nhưng nó **KHÔNG** tự động gộp (merge) các phép tính logic nghiệp vụ của bạn!

## Mức cách ly `READ COMMITTED` (Mặc định)

Ở mức này, cứ mỗi lần bạn gọi một câu lệnh SQL, PostgreSQL sẽ tạo một ảnh chụp mới:

```text
Lệnh SELECT thứ 1 -> Chụp ảnh S1
... có một giao dịch khác vừa sửa và chốt (commit) dữ liệu ...
Lệnh SELECT thứ 2 -> Chụp ảnh S2 (Và bùm, bạn thấy ngay dữ liệu mới của giao dịch kia)
```

PostgreSQL không bao giờ cho phép bạn đọc dữ liệu rác đang sửa dở (dirty read). Mức `READ UNCOMMITTED` (đọc tự do) ở PostgreSQL thực chất vẫn bị cưỡng ép chạy y hệt `READ COMMITTED`.

Một câu lệnh SELECT bình thường sẽ không khóa dòng dữ liệu. Do đó, luồng A và luồng B có thể cùng đọc lên con số `10`, cùng tự cộng tự trừ trên RAM, rồi lần lượt ghi đè số mới xuống và làm mất dữ liệu của nhau.

## UPDATE, Khóa dòng và Kiểm tra lại điều kiện (Predicate recheck)

Khi gọi UPDATE, PostgreSQL sẽ tự động khóa dòng đó lại (row-level lock). Nếu dòng đó đang bị một giao dịch khác update dở, câu lệnh của bạn sẽ phải đứng chờ (wait). Sau khi đối thủ chạy xong, PostgreSQL sẽ xử lý phiên bản dữ liệu mới nhất và **tự động đánh giá lại** (re-evaluate) xem điều kiện WHERE của bạn còn đúng không.

**Cập nhật tương đối (Atomic delta):** (RẤT AN TOÀN)
```sql
UPDATE job_progress
SET completed_units = completed_units + :delta
WHERE job_id = :jobId;
```
Người đứng chờ sẽ dùng thẳng dữ liệu mới nhất dưới ổ cứng để cộng `delta`, nên dù chạy song song nhiều luồng, phép tính vẫn cộng dồn cực chuẩn (compose).

**Cập nhật tuyệt đối (Application absolute write):** (RẤT DỄ LỖI)
```sql
UPDATE job_progress
SET completed_units = :valueComputedEarlier
WHERE job_id = :jobId;
```
Điều kiện (Predicate) chỉ đối chiếu `job_id`. PostgreSQL không hề biết rằng cái biến `:valueComputedEarlier` của bạn được tính toán từ một dữ liệu "thiu" (stale read) từ 5 phút trước. Nó sẽ cứ thế ghi đè làm mất dữ liệu người khác!

## Các "bóng ma" dị thường (Anomalies)

MVCC và các mức cách ly thường dính đến các loại lỗi:
- Mất dữ liệu (lost update): Do đọc-tính-ghi mà không khóa.
- Đọc không lặp lại (non-repeatable read): Do ảnh chụp bị làm mới liên tục.
- Dữ liệu bóng ma (phantom): Số lượng dòng bị thay đổi do người khác Insert/Delete.
- Sai lệch ghi chéo (write skew): Quy tắc liên quan nhiều dòng/nhiều bảng.
- Lỗi thứ tự tuần tự (serialization failure): Xảy ra khi dùng mức cách ly cao cấp nhất.

Đừng gom mọi thứ lại và gọi chung chung là "Lỗi đa luồng (race condition)". Với mỗi case, hãy phân tích rõ trình tự chạy chen ngang (interleaving), ảnh chụp và điều kiện ghi.

## Các Mức cách ly cao hơn

Mức `REPEATABLE READ` của PostgreSQL sẽ giữ chặt cái ảnh chụp từ đầu đến cuối giao dịch. Nếu nó thấy có luồng khác vừa sửa cùng một dòng, nó sẽ báo lỗi và HỦY (abort) giao dịch của bạn ngay lập tức chứ không thầm lặng ghi đè (silently overwrite). 
Mức cao nhất `SERIALIZABLE` còn theo dõi sát sao mọi sự phụ thuộc để phát hiện các lỗi tính toán rắc rối hơn.

Đổi lên mức cách ly cao hơn nghĩa là bạn đổi cách báo lỗi: Thay vì bắt đứng chờ, nó sẽ ném lỗi (abort/retry). Nhưng nó KHÔNG làm cho nghiệp vụ của bạn tự động chống trùng lặp (idempotent) và bạn vẫn BẮT BUỘC phải mở một giao dịch mới nếu muốn Thử lại (retry).

## Ảnh chụp sống quá lâu (Long-lived snapshots)

Các giao dịch chạy quá lâu (Long transactions) sẽ ngâm các ảnh chụp rất lâu, khiến cô lao công "vacuum" không dám đi thu dọn các dữ liệu rác (dead tuple). Lâu dần rác sẽ ngập ổ cứng và làm chậm hệ thống (operational pressure).
**Cấm kỵ:** Không bao giờ dùng Giao dịch (Transaction) để ôm trọn các lệnh gọi API mạng, gọi I/O hoặc các vòng lặp chờ vô tận chỉ để "giữ cái ảnh chụp".

## Lựa chọn vũ khí phối hợp (Coordination mechanism)

- **SQL cập nhật tương đối (Atomic delta):** Tuyệt vời nhất khi bạn có thể diễn đạt phép tính (+, -) bằng 1 câu SQL thẳng dưới Database.
- **Khóa lạc quan (`@Version`):** Cực kỳ hiệu quả để phát hiện lỗi ghi đè đồ thiu (stale write).
- **Khóa bi quan (`FOR UPDATE`):** Dùng khi bạn bắt buộc phải bắt các luồng phải xếp hàng (blocking) tuần tự.
- **Ràng buộc (Constraint):** Dùng để bảo vệ luật lệ bất chấp mọi thể loại đua xe đa luồng (race).
- **`SERIALIZABLE`:** Dùng khi quy tắc quá phức tạp. Nhớ phải thiết kế số lần thử lại (retry) có giới hạn.

Hãy chọn công cụ dựa trên Quy tắc nghiệp vụ, Tần suất đụng độ (contention) và Cách bạn muốn xử lý lỗi.

## Lời khuyên khi Kiểm thử (Testing)

Khi viết test PostgreSQL bằng Testcontainers, bạn cần:
1. Tạo nhiều giao dịch/kết nối chạy độc lập với nhau.
2. Điều khiển cho các diễn viên (actor) cùng đọc đúng một phiên bản dữ liệu.
3. Cố ý điều khiển thứ tự lưu (commit) của từng diễn viên.
4. Kiểm tra (Assert) xem Database cô lập (isolation) có chuẩn không.
5. Kiểm tra kết quả dữ liệu nghiệp vụ cuối cùng.
6. **Tuyệt đối không dùng H2 In-Memory Database** để chứng minh lỗi MVCC (vì H2 chạy không giống PostgreSQL).

## Theo dõi hệ thống (Quan sát)

Hãy dùng `pg_stat_activity`, `pg_locks`, tuổi thọ giao dịch và các chỉ số SQL (affected-row) để phân tích lỗi. 
PostgreSQL có các cột hệ thống tàng hình (như `xmin`), nó rất hữu ích để kỹ sư điều tra lỗi, nhưng đừng dại dột lấy nó thay thế cho thẻ `@Version` trong JPA của bạn.

## Liên kết tài liệu tham khảo

- [DB-001 — Lost update under MVCC](../postgresql/lost-update-mvcc/README.md)
- [LOCK-001 — Optimistic locking with @Version](../locking/optimistic-version-conflict/README.md)
- [DB-002 — Dirty-read expectations](../postgresql/dirty-read-expectation/README.md)
- [DB-003 — Non-repeatable read](../postgresql/non-repeatable-read/README.md)
- [DB-004 — Phantom capacity check](../postgresql/phantom-capacity-check/README.md)
- [DB-005 — Write skew](../postgresql/write-skew/README.md)
- [DB-007 — Row/table lock lifecycle](../postgresql/row-table-lock-lifecycle/README.md)
- [Isolation levels](isolation-levels.md)
- [PostgreSQL locks](postgresql-locks.md)
- [Optimistic locking](optimistic-locking.md)
- [Concurrency testing](concurrency-testing.md)
