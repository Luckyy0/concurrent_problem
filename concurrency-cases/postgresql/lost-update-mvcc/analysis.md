# Phân Tích Chuyên Sâu: Trò Ảo Thuật MVCC Và Nỗi Đau Mất Dữ Liệu (MVCC and lost-update analysis)

## 1. Hiện trường ban đầu (Initial state)

```text
job_id          = IMPORT-42
completed_units = 10
total_units     = 100
effective isolation = read committed
```

Ông Công Nhân A (Worker A) và Ông Công Nhân B (Worker B) đang hùng hục làm việc ở hai ngả khác nhau (disjoint batches). Số lượng hoàn thành báo về (accepted deltas) lần lượt là `3` và `4`.

## 2. Mong mỏi và Thực tế Phũ phàng (Expected versus actual)

Điều Đáng Lý Phải Xảy Ra (Expected additive invariant):
```text
10 + 3 + 4 = 17 (Chuẩn toán lớp 1)
```

Sự Thật Cay Đắng khi Ông B chốt sổ con số cũ rích cuối cùng:
```text
A báo về: Tui làm từ 10 lên 13 nha sếp!
B báo về: Tui làm từ 10 lên 14 nha sếp!
Kết cục Database chốt số: completed_units=14
```

Nhìn cục bộ, ông nào cũng tưởng mình đã làm tốt nhiệm vụ, không ai thấy lỗi. Nhưng xét trên tổng thể toàn cục (global committed invariant), cái phần công sức `3` của ông A đã "bay màu" không kèn không trống.

## 3. Phân Giải Án Mạng 2 Hung Thủ (Mandatory two-actor timeline)

| Bước | Ông Công Nhân A — Giao dịch Tx-A | Ông Công Nhân B — Giao dịch Tx-B |
| --- | --- | --- |
| T0 | BẮT ĐẦU (BEGIN) | |
| T1 | Chụp ảnh (S-A): Thấy đang là `10` | |
| T2 | | BẮT ĐẦU (BEGIN) |
| T3 | | Chụp ảnh (S-B): Vẫn thấy đang là `10` |
| T4 | Java nhẩm tính: `10 + 3 = 13` | Java nhẩm tính: `10 + 4 = 14` |
| T5 | Ghi thẳng `13`; Ôm khóa cọc dòng này (lock row) | |
| T6 | CHỐT (COMMIT); Dữ liệu `13` mới bắt đầu hiện hình | |
| T7 | | Ghi thẳng `14`; Đè luôn lên `13` (affected 1) |
| T8 | | CHỐT (COMMIT); Tội ác hoàn tất |
| T9 | | Truy vấn ra số `14` cười trừ |

Nếu khúc T5 và lệnh UPDATE của B đụng nhau (overlap):
```text
A đang ôm chặt Khóa Dòng (Row lock).
B đâm lệnh UPDATE vào và... Đứng đợi!
A Chốt sổ xong (commits) nhả khóa.
B giật mình tỉnh dậy, nhảy vào làm tiếp:
B kiểm tra điều kiện `WHERE job_id = ?` thấy vẫn ĐÚNG (true).
B nhắm mắt điền con số 14 vô tri vào (writes parameter 14).
```

Cái Khóa Dòng (Row lock serialization) nó chỉ xếp hàng thứ tự chạy, CHỨ KHÔNG HỀ biết cộng dồn cái công sức (application deltas) của hai bên lại với nhau đâu!

> **Nói ngắn gọn:** Ông B đứng chờ ông A làm xong, không có nghĩa là ông B sẽ THỨC TỈNH và vứt con số `14` đang nhẩm trong đầu đi. Điều kiện Cập nhật (predicate) cũng quá ngu ngơ, không biết hỏi câu: “Ủa, cái dòng này có còn y nguyên như lúc tui mới đọc không vậy?”.

## 4. Máy Ảnh Của Từng Câu Lệnh (Snapshot của từng statement)

Ở cái mức độ `READ COMMITTED` của PostgreSQL, mỗi cú SELECT giống như bấm tách một bức ảnh (statement snapshot):
- Máy ảnh S-A chộp được dòng Đã Chốt là `10`;
- Máy ảnh S-B cũng chộp được dòng là `10` (Vì ông A đã chốt đâu mà thấy);
- Không ông nào nhìn thấy cái đồ dở dang (uncommitted value) của ông kia;
- Ông A Chốt Sổ (commit) thành công, tạo ra một bức ảnh mới có số `13` chễm chệ cho thiên hạ xem;
- NHƯNG ông B cứng đầu, đâu có thèm chạy lại lệnh tính toán logic (business SELECT/calculation) đâu.

Nếu B có tâm, lôi điện thoại ra tự chụp lại hình mới bằng lệnh SELECT sau khi A chốt, thì B sẽ thấy `13`. Nhưng ngặt nỗi cái Entity đang ôm khư khư trong Bộ nhớ đệm (persistence context) của Java thường vẫn là cái cũ kỹ đó, trừ khi bạn đập đầu nó ép load lại (refresh/clear/query). Chân lý là Đừng bao giờ dựa dẫm vào cái thói "Đọc hên xui" để mong Code chạy đúng. Correctness không nên dựa vào accidental second read.

## 5. PostgreSQL MVCC Bị Mù Trạng Thái Nghiệp Vụ (semantic staleness)

Lúc Database nhận được lệnh UPDATE tào lao của B:
```sql
update job_progress
set completed_units = 14,
    total_units = 100
where job_id = :id;
```

Trong mắt PostgreSQL lúc đó, nó thấy gì?
- Ồ, Giao dịch hợp lệ (transaction ok).
- Dòng dữ liệu này có tồn tại (row ok).
- Điều kiện Khóa chính khớp (primary-key predicate match).
- Không vướng cái Ràng buộc nào (no constraint violated).
- Đã sửa thành công 1 dòng (affected-row = 1).

Database LÀM SAO MÀ BIẾT con số `14` đó được tính ra từ con số `10` của đời nảo đời nào? Nó càng không biết ý đồ của bạn là "Cộng thêm 4" (`+4`). Nó chả thấy có xung đột (conflict) gì để phải la toáng lên cho Hibernate/Spring rút lui (rollback).

## 6. Sát Thủ Giấu Mặt (Hibernate dirty checking)

Thằng Hibernate ôm khư khư cái trạng thái lúc mới tải lên:
```text
Lúc tải lên completedUnits = 10
Trạng thái Java hiện tại   = 14
```

Đến lúc xả hàng (flush), công cụ Tự soi lỗi (dirty checking) đẻ ra cái lệnh UPDATE tuyệt đối vô tri đó. Vì không có thẻ bùa `@Version` bảo vệ, Hibernate chỉ biết lấy Khóa Chính (identifier) ra mà đè dữ liệu.

Kể cả bạn có xài `@DynamicUpdate` để bớt cập nhật mấy cột không liên quan, thì nó CŨNG KHÔNG gắn thêm cái Rào Cản chống thiu (stale-state predicate) cho bạn đâu. Nó bảo vệ mấy cột khác thôi, còn cùng cái cột tiến độ đó thì... Nát! (same-column lost update).

## 7. Truy Tìm Root Cause Theo Các Tầng Lớp (Root cause theo layer)

| Tầng (Layer) | Hành Vi | Đổ lỗi cho nó được không? |
| --- | --- | --- |
| Java (JVM) | Ai cũng tính đúng `10 + 3` và `10 + 4` cục bộ | Không. Nó không có bổn phận phân xử với DB |
| Spring | Quản lý Giao dịch chốt/mở rất chuẩn chỉ | Không. Boundary đóng mở chuẩn đâu cản được Ghi đè sai logic |
| Hibernate | Soi lỗi (Dirty-check) và lưu bằng Khóa Chính | CÓ. Thiếu Cấu hình bắt quả tang ghi đè (@Version) |
| PostgreSQL MVCC | Cho phép Đọc mượt, nhưng bắt Ghi xếp hàng | KHÔNG. DB không rảnh để đoán ý đồ cộng dồn của bạn |
| Thiết kế Hệ thống | Ý đồ là `+delta` nhưng Code lại ép ghi Đè Trắng | LỖI TO NHẤT! (Root design error) |

Tội ác chia thành 3 nhịp ngớ ngẩn (Non-atomic operation):
```text
Đọc số liệu Database lên
  -> Quyết định tự làm phép toán ở ngoài mà không giao quyền (outside authoritative write)
  -> Ốp lệnh Ghi Đè Vô Điều Kiện xuống DB (unconditional absolute update)
```

## 8. Khóa Cửa Đứng Đợi (Locks)

Đọc chay (Plain SELECT) thì không có chuyện ôm cái khóa Khóa Dòng `FOR UPDATE` đâu. Chỉ có lệnh UPDATE mới ôm Khóa và giữ chặt nó cho tới khi Chốt Sổ (transaction end).

Cái Khóa đó chỉ đảm bảo Hai ông nội UPDATE không sửa cùng một Dòng trong cùng TÍCH TẮC thôi. Nó KHÔNG BAO GIỜ đảm bảo ông UPDATE thứ 2 sẽ có tâm lấy cái kết quả của ông thứ 1 ra tính lại. Để chơi hệ Chờ Đợi Đứng Đường (blocking solution), bạn phải đớp được cái Khóa ĐÓ TRƯỚC KHI ĐỌC (acquire trước read/calculate).

## 9. Ràng Buộc Cơ Sở Dữ Liệu (Constraint behavior)

Nếu bạn rào cái Schema constraint kiểu:
```sql
check (
    completed_units >= 0
    and completed_units <= total_units
)
```

Trông oai đấy, bảo vệ được số Không bị Lố. NHƯNG con số `14` khốn khổ do lỗi Mất Dữ Liệu kia nó VẪN NẰM TRONG GIỚI HẠN! Constraint của DB chỉ bảo vệ con số hiện hữu, chứ nó không làm thầy bói gọi hồn mấy con số Công Sức (`+3`) không được lưu về (reconstruct accepted deltas).

Muốn hối hận làm lại (reconciliation), bạn phải tự xây cuốn sổ tay (append-only progress event) ghi chép lại, và lúc gộp thì vẫn phải đảm bảo Nguyên tử (atomicity/idempotency).

## 10. Thế Lực Thực Sự Đang Đứng Sau (Effective isolation)

Muốn bắt tận tay day tận trán trong Transaction:
```sql
select current_setting('transaction_isolation');
```

Bài Thí nghiệm thảm họa này sẽ trả về:
```text
read committed
```

Đừng có thấy cái Annotation `@Transactional` rồi tự mãn nghĩ mình đang cô lập (isolation) cực mạnh. Ông datasource/outer boundary có thể bóp méo cái chỉ số đó (effective value) bất cứ lúc nào.

## 11. Đọc Lại Bị Kêu (REPEATABLE READ)

Nếu xài PostgreSQL `REPEATABLE READ`, DB sẽ ôm nguyên một cái máy ảnh bự chụp toàn cảnh (transaction snapshot). Khi tụi nó tranh nhau đè cùng một Dòng, một đứa UPDATE sẽ bị đạp văng ra ngoài với lỗi (serialization failure - SQLSTATE 40001) thay vì lẳng lặng đè số (silently overwrite).

Luật chơi thay đổi hoàn toàn:
```text
Một ông chốt sổ cười tươi
Một ông ăn lỗi vỡ mồm SQLSTATE 40001
App phải Tự Hủy kèo (roll back) và Tự thử lại từ đầu (retry)
```

CẤM được ngậm mồm nuốt lỗi (catch/retry) bên TRONG cái transaction đã nát kia. Xem Case SPR-006 để hiểu thêm về nghệ thuật thử lại.

## 12. Cực Đoan Hoàn Hảo (SERIALIZABLE)

`SERIALIZABLE` là chúa tể chống thảm họa, sẵn sàng Đạp Chết (abort) bất kỳ thằng nào nhảy lộn xộn không thể xếp hàng nối đuôi nhau (serial order). NHƯNG:
- Bị Đạp Chết và Thử Lại (abort/retry) là cơm bữa (expected control path).
- Tất cả các bước trong Transaction phải chịu được việc đập đi xây lại (idempotent/retry-safe).
- Càng đông người tranh nhau, tỉ lệ phải Đập đi làm lại càng cao.
- Vẫn phải giới hạn số lần Thử (attempt/deadline limits).
- Cho nên, nếu chỉ là sửa một cái bộ đếm quèn (single-row additive mutation), thì xài SQL Cập nhật Nguyên tử (atomic SQL) đỡ khổ hơn nhiều.

Đẩy cao mức độ Cô lập (Isolation) là một giải pháp xịn, chứ không phải liều thuốc tiên đa năng cho mọi cái bộ đếm.

## 13. Sống, Chết Và Ngâm Tôm (Commit, rollback và timeout)

Con đường sai lầm:
- Ông A Chốt sống `13`;
- Ông B Chốt đè `14`;
- Không ai Hủy kèo (rollback);
- Khách hàng không nhận lỗi (no exception).

Nếu ông B rớt mạng Ngâm tôm (timeout/rollback) trước khi chốt, thì hên quá, cái `13` của ông A sống sót, phần của ông B coi như chưa xong (delta chưa accepted). Lỡ mà Khách bắt B gọi lại (retry B), B phải có "Chứng minh nhân dân" (idempotency/command identity) để đừng có làm cú đúp dồn thêm công sức khi chuyện cũ chưa rõ ràng.

Dùng SQL Nguyên tử hay Khóa Lạc Quan đều PHẢI biết gom cái số dòng (affected rows) thành Tuyên bố Rõ Ràng: "Xong", "Thử Lại Nhé", hoặc "Cút" (success, retry, rejection).

## 14. Mất Điện (Crash behavior)

- Mất điện TRƯỚC KHI CHỐT: PostgreSQL vứt mẹ hết vào sọt rác (rollback).
- Mất điện SAU KHI CHỐT, chưa kịp báo App: Cái kết quả đã lưu cần phải được ai đó tra cứu (outcome lookup).
- Mất điện KHÔNG PHẢI LÀ THỦ PHẠM GÂY MẤT DỮ LIỆU; Thảm họa sinh ra ngay cả khi mọi thứ Đều Chốt Ngon Lành.
- Muốn đập đi xây lại sổ sách (Rebuild projection), bạn bắt buộc phải có Ghi Nhớ Bền Bỉ mấy cái Tiến Độ Lẻ Tẻ và chống ghi trùng (deduplicated).

## 15. Thử Lại Mù Quáng (Retry behavior)

Hệ thống đang lỗi mà bạn gào thét Gọi Lại (Retry unconditional) thì cũng chẳng cứu vớt nổi cái lệnh đã bị đè. Nó có khi lại phang thêm 1 cục rác nữa hoặc đè thêm phát nữa. Retry chỉ là con người khi:
- Bạn túm được cổ thằng gây Xung đột (conflict);
- Lệnh cũ đã HỦY SẠCH SẼ (rollback hoàn tất);
- Lần Thử Sau phải Lấy Lại Dữ Liệu Tươi Mới (reload current state);
- Chắc chắn Code cùi bắp của bạn chạy lại không bị sinh trùng (command idempotency rõ);
- Có ranh giới Thử rõ ràng (attempt/deadline bounded).

Thường thì xài SQL Nguyên tử thì chả cần Try-Catch cho hai cái `+delta` chạy song song, vì bản thân cái Dòng (row-level) nó đã bắt tụi nó Xếp Hàng và Cộng Ngay Ngắn giùm bạn rồi.

## 16. Chạy Đa Server (Multi-instance)

Nhét chữ `synchronized` vào Code Java App A chả hù dọa được App B ở Server kế bên đâu. Cái Cột/Điều kiện/Phiên Bản (row/predicate/version) ở dưới PostgreSQL mới là Ranh Giới Chia Sẻ duy nhất mà mọi ông thần (instances/direct workers) phải tuân theo.

Dùng mấy cái Khóa Phân Tán (distributed mutex) bên ngoài lằng nhằng rắc rối, mệt tim quản lý (failure/lease/fencing), tự nhiên rước họa vào thân, thua xa trò đập vô Cú SQL Nguyên Tử cho nhanh!

## 17. Giám Sát Sức Khỏe (Observability)

Ghi Đè mất dữ liệu là Sát thủ Không Tiếng Động, Metrics Báo Lỗi là đồ bỏ! Hãy ôm trọn:
- Đếm tổng số Công sức Thực tế và Gom tổng Đóng Góp (accepted command/event count / summed deltas).
- Soi Mức lệch Sổ (projection value/reconciliation drift).
- Số lượng Dòng bị đấm mỗi lần Cập Nhật Có Điều Kiện (affected-row count).
- Tốc độ xảy ra Xung đột Khóa Lạc Quan/Serialization.
- Sự phân chia tài nguyên và Điểm Nóng (hot-key distribution).
- Đọc lén Mức độ Cô Lập lúc chẩn bệnh (effective isolation).
- ID của Command bị Trùng để bắt mấy thằng lầy (duplicate outcome).

Tuyệt đối cấm quăng Data nhạy cảm vô File Log (payload nhạy cảm). Chống đối soát Sổ Sách (Reconciliation invariant) ngon ăn hơn ngàn lần cái kiểu đếm dòng Thành Công nhạt toẹt của SQL.
