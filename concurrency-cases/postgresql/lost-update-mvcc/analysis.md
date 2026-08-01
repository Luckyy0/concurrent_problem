# Phân Tích Chuyên Sâu: Cơ Chế MVCC Và Sự Cố Mất Dữ Liệu Cập Nhật (MVCC and lost-update analysis)

## 1. Trạng thái khởi đầu (Initial state)

```text
job_id          = IMPORT-42
completed_units = 10
total_units     = 100
effective isolation = read committed
```

Luồng Worker A và Worker B đang xử lý hai tập dữ liệu độc lập (disjoint batches). Số lượng hoàn thành báo về (accepted deltas) lần lượt là `3` và `4`.

## 2. Kỳ vọng theo thiết kế và Thực tế (Expected versus actual)

Điều kiện cộng dồn lý tưởng (Expected additive invariant):
```text
10 + 3 + 4 = 17
```

Thực tế xảy ra khi luồng B ghi đè dữ liệu:
```text
A tính toán: Hoàn thành từ 10 lên 13.
B tính toán: Hoàn thành từ 10 lên 14.
Kết quả lưu cuối cùng trên CSDL: completed_units = 14
```

Nhìn từ góc độ cục bộ, mỗi luồng đều đã thực thi chính xác phần việc của mình và không phát sinh ngoại lệ. Tuy nhiên, xét trên tổng thể toàn cục (global committed invariant), phần công sức tương đương `3` đơn vị của luồng A đã bị loại bỏ khỏi hệ thống mà không có bất kỳ cảnh báo nào.

## 3. Dòng Thời Gian Tương Tranh Hai Luồng (Mandatory two-actor timeline)

| Bước | Worker A — Giao dịch Tx-A | Worker B — Giao dịch Tx-B |
| --- | --- | --- |
| T0 | BẮT ĐẦU (BEGIN) | |
| T1 | Ảnh chụp (Snapshot S-A): Thấy dữ liệu là `10` | |
| T2 | | BẮT ĐẦU (BEGIN) |
| T3 | | Ảnh chụp (Snapshot S-B): Nhìn thấy dữ liệu `10` |
| T4 | Tính toán trong JVM: `10 + 3 = 13` | Tính toán trong JVM: `10 + 4 = 14` |
| T5 | Cập nhật số `13`; Chiếm giữ Khóa dòng (row lock) | |
| T6 | HOÀN TẤT (COMMIT); Giá trị `13` chính thức lưu trữ | |
| T7 | | Cập nhật số `14`; Ghi đè trực tiếp (affected 1) |
| T8 | | HOÀN TẤT (COMMIT) |
| T9 | | Truy vấn kết quả ra `14` |

Trường hợp tại bước T5, nếu luồng B cũng thực thi lệnh UPDATE đồng thời (overlap):
```text
A đang chiếm giữ Khóa Dòng (Row lock).
Lệnh UPDATE của B bị chặn và phải chờ đợi (wait).
A hoàn tất giao dịch (commit) và giải phóng khóa.
B tiếp tục quá trình cập nhật:
B kiểm tra điều kiện `WHERE job_id = ?`, xác nhận dữ liệu vẫn tồn tại (true).
B ghi đè con số 14 một cách tuyệt đối (writes parameter 14) mà không cần tính toán lại.
```

Khóa cấp dòng (Row lock serialization) chỉ có chức năng ép buộc các lệnh thi hành theo thứ tự thời gian. Nó KHÔNG HỀ có khả năng tự động hợp nhất (merge) các giá trị tính toán cục bộ (application deltas) từ các luồng khác nhau. 

> **Nói ngắn gọn:** Việc luồng B đứng chờ luồng A giải phóng khóa không đồng nghĩa với việc bộ nhớ của luồng B sẽ tự động cập nhật sự thay đổi mới nhất. Điều kiện cập nhật (predicate) cũng thiếu đi việc đối chiếu tính nguyên vẹn của trạng thái trước khi ghi (ví dụ: `WHERE version = expected`).

## 4. Tầm Nhìn Ảnh Chụp Từng Lệnh (Statement snapshot)

Ở mức độ cô lập `READ COMMITTED` của PostgreSQL, mỗi lệnh truy vấn (SELECT) sẽ sử dụng một ảnh chụp dữ liệu ngay tại thời điểm lệnh đó bắt đầu thực thi (statement snapshot):
- Lệnh SELECT của S-A thấy dữ liệu đã commit là `10`.
- Lệnh SELECT của S-B cũng thấy dữ liệu `10` (Vì luồng A chưa commit).
- Các luồng hoàn toàn không nhìn thấy sự thay đổi dữ liệu (uncommitted value) của nhau.
- Sau khi luồng A commit thành công, một ảnh chụp dữ liệu mới chứa con số `13` mới bắt đầu hiện diện cho các giao dịch khác.
- Tuy nhiên, luồng B sẽ không truy vấn lại dữ liệu để sử dụng cho phép toán của mình.

Nếu luồng B thực hiện lại một truy vấn `SELECT` sau khi A commit, B sẽ đọc được `13`. Tuy nhiên, trong mô hình ORM, thực thể (Entity) thường được bộ nhớ đệm (persistence context) bảo lưu trạng thái cũ, trừ khi có lệnh buộc tải lại rõ ràng (refresh/clear/query). Việc bảo đảm tính toàn vẹn của ứng dụng (Correctness) không nên phụ thuộc vào yếu tố đọc lại dữ liệu một cách ngẫu nhiên.

## 5. Hạn Chế Phân Tích Của MVCC Đối Với Nghiệp Vụ (Semantic staleness)

Khi PostgreSQL tiếp nhận lệnh cập nhật của B:
```sql
update job_progress
set completed_units = 14,
    total_units = 100
where job_id = :id;
```

Cơ sở dữ liệu sẽ phân tích yêu cầu này như sau:
- Giao dịch đang hợp lệ (transaction ok).
- Dòng dữ liệu tương ứng có tồn tại (row ok).
- Điều kiện Khóa chính hoàn toàn khớp (primary-key predicate match).
- Không vi phạm các Ràng buộc cấp CSDL (no constraint violated).
- Đã thực thi cập nhật thành công trên 1 dòng (affected-row = 1).

CSDL không thể phân tích được nguồn gốc của con số `14` (được tính toán từ số `10`), và cũng không biết được nghiệp vụ yêu cầu "Cộng thêm 4" (`+4`). PostgreSQL không phát hiện ra yếu tố xung đột nghiệp vụ (conflict) nào để buộc Hibernate hoặc Spring phải hủy bỏ thao tác (rollback).

## 6. Vấn Đề Với Cơ Chế Tự Phát Hiện Lỗi (Hibernate dirty checking)

Cơ chế Hibernate lưu trữ trạng thái gốc của Entity từ lúc truy vấn:
```text
Trạng thái tải lên ban đầu: completedUnits = 10
Trạng thái đối tượng Java lúc sửa đổi: 14
```

Đến thời điểm đồng bộ dữ liệu (flush), tính năng tự động theo dõi (dirty checking) sinh ra lệnh UPDATE với giá trị tuyệt đối. Do thiếu vắng cấu trúc bảo vệ phiên bản (`@Version`), Hibernate chỉ sử dụng Khóa chính (identifier) làm điều kiện cập nhật.

Ngay cả khi bạn bật tính năng `@DynamicUpdate` (chỉ cập nhật các cột có thay đổi), cơ chế này không tự động thêm các rào cản chống cập nhật rác (stale-state predicate). Nó chỉ giới hạn số lượng cột tham gia thao tác, còn việc ghi đè trên cùng một cột vẫn diễn ra bình thường (same-column lost update).

## 7. Phân Tích Nguyên Nhân Theo Các Lớp (Root cause theo layer)

| Lớp Hệ Thống (Layer) | Đánh Giá Hành Vi | Kết Luận |
| --- | --- | --- |
| Java (JVM) | Tính toán chính xác logic nghiệp vụ (`10 + 3` và `10 + 4`) một cách cục bộ. | Không có lỗi. Việc xử lý đồng bộ thuộc trách nhiệm của hệ thống CSDL. |
| Spring | Quản lý vòng đời Giao dịch hoàn chỉnh và đóng mở đúng cách. | Không có lỗi. Ranh giới giao dịch không ngăn cản được logic ghi đè sai lầm. |
| Hibernate | Theo dõi thay đổi (Dirty-check) và lưu trữ dựa trên Khóa Chính. | LỖI MỘT PHẦN. Thiếu cấu hình chặn cập nhật dữ liệu cũ (`@Version`). |
| PostgreSQL MVCC | Duy trì trạng thái đọc mượt mà, áp đặt lệnh ghi phải xếp hàng. | Không có lỗi. CSDL thực thi đúng lệnh được gửi và không tự suy đoán nghiệp vụ cộng dồn. |
| Thiết kế Ứng dụng | Nghiệp vụ cộng dồn (`+delta`) nhưng lại thực thi theo logic Ghi đè tuyệt đối vô điều kiện. | NGUYÊN NHÂN CHÍNH (Root design error). |

Thao tác gây lỗi bao gồm 3 bước tách rời (Non-atomic operation):
```text
Truy vấn số liệu hiện tại từ DB
  -> Thực hiện tính toán trên bộ nhớ ứng dụng (outside authoritative write)
  -> Ban hành lệnh cập nhật trực tiếp vô điều kiện (unconditional absolute update)
```

## 8. Hành Vi Của Các Loại Khóa (Locks behavior)

Lệnh đọc dữ liệu thông thường (Plain SELECT) không yêu cầu Khóa Dòng (`FOR UPDATE`). Chỉ có lệnh UPDATE mới thiết lập Khóa dòng và nắm giữ nó cho đến khi giao dịch kết thúc (transaction end).

Khóa dòng này chỉ có chức năng đảm bảo hai giao dịch UPDATE không thao tác trên cùng một dòng trong cùng một thời điểm. Nó hoàn toàn không đảm bảo rằng giao dịch thứ hai sẽ tự động lấy được trạng thái sau cùng của giao dịch thứ nhất để tính toán lại. Để thiết lập một giải pháp chờ đợi đồng bộ (blocking solution), ứng dụng phải chiếm giữ Khóa TRƯỚC KHI thực hiện việc đọc dữ liệu (acquire before read/calculate).

## 9. Hạn Chế Của Ràng Buộc CSDL (Constraint behavior)

Nếu bạn khai báo ràng buộc mức Schema:
```sql
check (
    completed_units >= 0
    and completed_units <= total_units
)
```

Ràng buộc này ngăn ngừa lỗi vượt quá định mức dự kiến. Tuy nhiên, giá trị sai lệch `14` trong trường hợp Mất Dữ Liệu vẫn hoàn toàn hợp lệ theo ràng buộc này! Ràng buộc chỉ giới hạn giới hạn hiện tại, chứ không thể phục dựng lại các thông số biến đổi (accepted deltas) đã bị mất.

Để đảm bảo việc đối soát và khôi phục (reconciliation) khả thi, hệ thống cần duy trì nhật ký các tiến độ cục bộ (append-only progress events), đảm bảo tính nguyên tử trong khi tổng hợp dữ liệu (atomicity/idempotency).

## 10. Mức Độ Cô Lập Thực Tế (Effective isolation)

Kiểm tra tham số cấu hình trong Giao dịch:
```sql
select current_setting('transaction_isolation');
```

Trong kịch bản lỗi, kết quả trả về sẽ là:
```text
read committed
```

Không nên phụ thuộc hoàn toàn vào Annotation `@Transactional` để khẳng định hệ thống đang trong trạng thái cô lập an toàn. Cấu trúc mạng hoặc cấu hình DataSource của kết nối có thể điều chỉnh mức cô lập này (effective value) bất cứ lúc nào.

## 11. Xử Lý Xung Đột Với Mức Độ `REPEATABLE READ`

Nếu thiết lập `REPEATABLE READ`, PostgreSQL sẽ bảo lưu ảnh chụp dữ liệu trên phạm vi toàn bộ giao dịch (transaction snapshot). Khi phát hiện có xung đột thay đổi trên cùng một dòng, tiến trình UPDATE thứ hai sẽ bị từ chối và phát sinh lỗi (serialization failure - SQLSTATE 40001) thay vì tiếp tục ghi đè lẳng lặng (silently overwrite).

Cách thức xử lý sẽ thay đổi:
```text
Giao dịch thứ nhất thành công (commit).
Giao dịch thứ hai phát sinh ngoại lệ (SQLSTATE 40001).
Ứng dụng phải thực hiện hủy bỏ (roll back) và chạy lại quy trình từ đầu (retry).
```

Tuyệt đối không sử dụng try-catch để che giấu lỗi và tiếp tục làm việc ngay trên giao dịch đã hỏng. Hãy xem tài liệu SPR-006 để biết cách áp dụng chiến lược thử lại tự động an toàn.

## 12. Đặc Điểm Của Cấp Độ `SERIALIZABLE`

`SERIALIZABLE` là mức độ cô lập nghiêm ngặt nhất, sẵn sàng hủy bỏ (abort) bất kỳ giao dịch nào có nguy cơ vi phạm trình tự thực thi an toàn (serial order). 
Tuy nhiên:
- Quá trình bị Hủy và phải Thử lại (abort/retry) diễn ra rất thường xuyên (expected control path).
- Tất cả quy trình trong giao dịch phải được thiết kế để hoạt động ổn định khi làm lại (idempotent/retry-safe).
- Tần suất xung đột tỷ lệ thuận với số lượng luồng tương tranh.
- Đòi hỏi thiết lập giới hạn cho số lần Thử Lại (attempt/deadline limits).
- Đối với các nghiệp vụ đơn giản như bộ đếm (single-row additive mutation), sử dụng câu lệnh SQL Cập nhật Nguyên tử (atomic SQL) sẽ mang lại hiệu năng cao hơn rất nhiều.

Tăng mức độ Cô lập là một phương án khả thi, nhưng không phải là công cụ thay thế cho mọi vấn đề tương tranh thao tác tính toán.

## 13. Tác Động Của Quá Trình Commit, Rollback và Timeout

Trong các chuỗi hành động sai lầm:
- A hoàn tất ghi nhận (commit) kết quả `13`.
- B hoàn tất ghi đè kết quả `14`.
- Không giao dịch nào bị Hủy (rollback).
- Ứng dụng phản hồi thao tác thành công (no exception).

Nếu giao dịch B bị ngắt kết nối hoặc hết hạn (timeout/rollback) trước khi commit, dữ liệu của A (`13`) được an toàn bảo lưu, công sức của B không được ghi nhận (delta chưa accepted). Khi hệ thống gọi lại B (retry), ứng dụng phải xác thực tính đơn nhất (idempotency/command identity) để tránh thao tác tính dồn kép lặp lại.

Bất kể sử dụng SQL Nguyên tử hay Khóa Lạc Quan, hệ thống phải xác định rõ ràng số lượng dòng bị tác động (affected rows) nhằm quyết định hướng xử lý: "Thành công", "Thử lại", hoặc "Từ chối" (success, retry, rejection).

## 14. Hành Vi Khi Gặp Sự Cố (Crash behavior)

- Máy chủ gián đoạn TRƯỚC KHI COMMIT: PostgreSQL lập tức hủy bỏ giao dịch (rollback) an toàn.
- Máy chủ gián đoạn SAU KHI COMMIT nhưng chưa kịp gửi thông báo về: Kết quả đã ghi nhận và ứng dụng cần cơ chế truy vấn đối soát (outcome lookup).
- Sự cố hỏng hóc KHÔNG PHẢI LÀ NGUYÊN NHÂN gây mất dữ liệu ở trường hợp này; lỗi ghi đè dữ liệu xảy ra ngay cả khi mọi hệ thống đang hoạt động tối ưu.
- Để xây dựng cơ chế phục hồi sổ sách (Rebuild projection), hệ thống cần một hệ quản trị lưu trữ tiến độ độc lập (Event Log) kèm các thuật toán bảo vệ dữ liệu trùng lắp (deduplicated).

## 15. Thử Lại Một Cách An Toàn (Retry behavior)

Việc ra lệnh thử lại tự động (Retry unconditional) trên một thao tác đang gặp lỗi cấu trúc chỉ làm tăng tần suất đè dữ liệu hoặc đưa ra thông tin rác. Cơ chế thử lại chỉ an toàn khi đảm bảo các tiêu chuẩn sau:
- Nắm rõ nguyên nhân gây ra xung đột (conflict signal).
- Giao dịch lỗi phải được DỌN DẸP SẠCH SẼ (rollback hoàn tất).
- Ở lần thử sau, dữ liệu phải được làm mới hoàn toàn (reload current state).
- Thao tác thực thi luôn duy trì tính tự miễn dịch với dữ liệu trùng lặp (command idempotency).
- Xác lập thời hạn truy cập (attempt/deadline bounded).

Thông thường khi áp dụng các cú pháp SQL Cập nhật Nguyên tử, việc triển khai mã khối Try-Catch là không cần thiết, vì CSDL tự xử lý việc đồng bộ và cộng dồn thông tin một cách ngăn nắp (row-level queue).

## 16. Thiết Kế Trong Môi Trường Đa Máy Chủ (Multi-instance)

Sử dụng từ khóa `synchronized` trong Java chỉ có tác dụng nội bộ trong một Máy chủ duy nhất (JVM). Khóa cấp dòng, điều kiện cập nhật, hay phiên bản (row/predicate/version) tại PostgreSQL mới là vùng Ranh Giới Chia Sẻ đáng tin cậy duy nhất cho tất cả các máy chủ cùng vận hành (instances/direct workers).

Xây dựng một hệ thống Khóa Phân Tán (distributed mutex) qua Redis hay Zookeeper sẽ tăng độ trễ và độ phức tạp cấu trúc (failure/lease/fencing). Thao tác cập nhật nguyên tử vào CSDL thường là giải pháp dễ duy trì và ổn định hơn hẳn.

## 17. Giám Sát Và Phát Hiện Lỗi Ẩn (Observability)

Lỗi Ghi Đè Mất Dữ Liệu diễn ra một cách âm thầm, nên các số liệu cảnh báo Exception không đủ để chẩn đoán. Hệ thống cần chú trọng:
- Đếm tổng số lượng công việc hoàn thành với số lần chấp nhận (accepted command/event count / summed deltas).
- Kiểm tra dữ liệu đối soát trên tổng kết sổ sách (projection value/reconciliation drift).
- Số lượng dòng bị tác động (affected-row count) trong mỗi lần Cập nhật Có Điều kiện.
- Tần suất xuất hiện cảnh báo Xung đột Khóa Lạc Quan/Serialization.
- Khả năng xuất hiện Điểm Nóng trên cơ sở dữ liệu (hot-key distribution).
- Rà soát Mức độ Cô lập lúc giám định (effective isolation).
- Ghi nhận ID của các chỉ thị để ngăn chặn hành vi trùng lặp (duplicate outcome).

Tuyệt đối tránh lưu thông tin cá nhân hoặc nghiệp vụ nhạy cảm vào File Log (payload nhạy cảm). Ưu tiên thiết lập quy trình đối soát Sổ Sách (Reconciliation invariant) để mang lại hiệu quả kiểm định cao hơn việc theo dõi kết quả SQL cập nhật thành công đơn thuần.
