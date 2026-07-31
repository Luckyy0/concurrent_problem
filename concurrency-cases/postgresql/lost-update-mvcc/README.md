# DB-001 — Vấn Đề Ghi Đè Mất Dữ Liệu (Lost update) Dưới Trướng PostgreSQL MVCC

## 1. Tóm tắt câu chuyện đau lòng

Giả sử Job `IMPORT-42` đã cày xong `10` đơn vị (units).
Bỗng nhiên:
- Ông Công Nhân A (Worker A) chạy tới báo: "Sếp ơi, em làm thêm được `3` cái!".
- Ông Công Nhân B (Worker B) cũng chen ngang: "Em làm được `4` cái!".

Vì dùng Spring Transaction và JPA, cả hai ông đều lấy tờ giấy ghi chú (entity) hiện tại ra xem: "À, ban đầu là `10`".
- Ông A tự nhẩm trong đầu (trong JVM): `10 + 3 = 13`.
- Ông B cũng tự nhẩm: `10 + 4 = 14`.

Đến đoạn lưu xuống DB (Hibernate dirty checking flush), cả hai ông đều đẩy lệnh UPDATE theo kiểu ép buộc con số tuyệt đối (absolute values) với điều kiện ngô nghê `WHERE job_id = ?`.

Kết cục:
- Ông A chốt sổ trước, ghi số `13`.
- Ông B chốt sổ sau, đè bẹp lên số `13` của A, sửa thành `14`.
- Công sức `+3` của ông A bốc hơi không dấu vết (Lost update), mặc dù cả A và B đều nhận được thông báo "Lưu thành công, cười tươi nhé!".

Nguyên tắc bất di bất dịch (Invariant) bị phá vỡ:
```text
Tổng số Đã Xong (final completedUnits)
  = Số ban đầu (initial) + Tổng công sức (delta) của mọi lệnh được chấp nhận

Với số ban đầu = 10, công sức A = 3, công sức B = 4:
Thì kết quả CẦN PHẢI LÀ = 17. (Chứ không phải 14!)
```

> **Nói ngắn gọn:** Cơ chế MVCC của PostgreSQL vô tình dung túng cho cả A và B cùng nhìn thấy phiên bản cũ rích (đã chốt). Nếu Code của bạn tự tính toán con số tuyệt đối rồi ốp thẳng xuống DB, thì DB cũng bó tay không biết đường nào mà gộp (merge) số liệu lại giùm bạn đâu.

## 2. Dàn diễn viên và Sân khấu (Actors và shared state)

| Thành phần | Vai trò |
| --- | --- |
| Ông Công Nhân A | Ôm cái Transaction cộng thêm `3` đơn vị. |
| Ông Công Nhân B | Ôm cái Transaction cộng thêm `4` đơn vị. |
| Bảng `job_progress` | Nơi lưu giữ sổ sách chân lý (Authoritative counters). |
| Bộ nhớ đệm Hibernate (Persistence context) | Nơi lưu lén bản nháp (snapshot) và tự động bắt lỗi thay đổi (dirty-check) theo số tuyệt đối. |
| PostgreSQL | Kẻ phán xử: Cung cấp ảnh chụp (snapshots), khóa dòng (row locks) và quyết định tầm nhìn khi chốt sổ (commit visibility). |

Lúc đầu (Initial row):
```text
job_id          = IMPORT-42
completed_units = 10
total_units     = 100
```

Kết cục bi đát (Broken final):
```text
completed_units = 14
Cả hai báo công thành công!
```

## 3. Nơi xảy ra án mạng (Transaction boundary và contention point)

Mỗi lần bạn gọi hàm `addCompletedUnits()`, nó chạy qua Proxy của Spring và đẻ ra một Transaction độc lập hoàn toàn trên PostgreSQL với mức độ `READ COMMITTED` (Chỉ Đọc Hàng Đã Chốt).

Quy trình bóp dái nhau không hề nguyên tử (Non-atomic sequence):
```text
Lấy số hiện tại từ DB (SELECT)
  -> Chạy thuật toán siêu cấp trong JVM (Cộng trừ)
  -> Lưu số mới xuống DB CHỈ bằng Khóa Chính (UPDATE by primary key)
```

Điểm thắt cổ chai (Contention point) chính là cái Dòng `job_progress(job_id='IMPORT-42')`. Các lệnh UPDATE có giành nhau cái Khóa Dòng (row lock), nên lúc ghi tụi nó bắt buộc phải xếp hàng (serialize về thời gian). NHƯNG việc xếp hàng ghi đè những con số đã bị sai lệch (stale absolute writes) thì cũng chẳng bảo vệ được tính cộng dồn (additive invariant) đâu!

## 4. Mong Mỏi vs. Thực Tế Phũ Phàng (Expected và actual)

| Bước | Ông A | Ông B | Chốt sổ (Final) |
| --- | --- | --- | --- |
| Lấy sổ ra xem (Read) | 10 | 10 | |
| Tự tính nhẩm (Calculate) | 13 | 14 | |
| Ai nhanh tay hơn (Commit) | Xong Trước | Xong Sau | |
| Số Đáng Lý Phải Có (Expected) | +3 | +4 | 17 |
| Sự Thật Cay Đắng (Broken write) | Lưu 13 | Đè thành 14 | 14 |

Chả có miếng lỗi Exception, Timeout, hay Báo cáo sai số lượng dòng (affected-row conflict) nào nổ ra cả! Vì lệnh UPDATE chỉ tìm trúng cái ID đó là đè bẹp, không hề có điều kiện kiểm tra phiên bản hay giá trị cũ (version/old-value predicate).

## 5. Từ vựng chém gió (Thuật ngữ cần biết)

| Thuật ngữ | Giải thích |
| --- | --- |
| lost update (Mất cục dữ liệu) | Công sức lưu của một thằng bị thằng đến sau ghi đè âm thầm. Đau! |
| MVCC | Trò ảo thuật phân thân dữ liệu (Multi-Version Concurrency Control) của Database. |
| statement snapshot | Bức ảnh chụp tại thời điểm chạy từng Câu lệnh ở mức `READ COMMITTED`. |
| read-modify-write | Chuỗi hành động: Đọc lên -> Tính trong Code -> Ghi xuống. |
| dirty checking | Trò tự soi xét của Hibernate: Thấy Object bị đổi là đẻ lệnh UPDATE giùm. |
| absolute write | Ép số tuyệt đối: Ví dụ `SET completed_units = 14`. |
| atomic delta | Giao trọng trách cho DB tự tính: `SET completed_units = completed_units + 4`. (Chuẩn bài!) |
| version predicate | Đính kèm điều kiện chống ế: `WHERE version = expected` để bắt quả tang bọn ghi đè. |

## 6. Bản đồ kho báu (Điều hướng)

- [Đoạn Code thảm họa read-modify-write bằng JPA](broken-code.md)
- [Mổ xẻ Tầm nhìn MVCC và lý do Mất Dữ Liệu](analysis.md)
- [Bí kíp giải cứu: Cập nhật Nguyên tử, Khóa Lạc Quan và Khóa Bi Quan](solutions.md)
- [Phòng Thí Nghiệm Đập Tan Ảo Tưởng ở PostgreSQL](experiments.md)
- [Giao dịch Nguyên tử Database (Atomic database operations)](../../concepts/atomic-database-operations.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Mức độ Cô lập (Isolation levels)](../../concepts/isolation-levels.md)
- [Khóa Lạc Quan (Optimistic locking)](../../concepts/optimistic-locking.md)
- [Khóa Bi Quan (PostgreSQL locks)](../../concepts/postgresql-locks.md)
- [Nghệ thuật Viết Test Đa Luồng (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Hậu quả khi lên Production

- Bảng đếm số tiến độ hiển thị số nhỏ hơn số việc thực tế đã làm (Báo cáo láo).
- Trình kích hoạt hoàn thành công việc (Job completion trigger) có nguy cơ ngủ gục hoặc chạy trễ.
- Mở Audit Log / Check response của Worker thì thấy "Báo cáo thành công", nhưng Database lại cãi "Tao không thấy số". Mâu thuẫn!
- Tổ Kiểm toán (Reconciliation) phải hì hục đi rà lại từng dòng Log (event records) để cộng tay lại con số.
- Nhét cái Khóa Cục Bộ (JVM-local lock - synchronized) vào Code chỉ là trò trẻ con vô dụng khi chạy nhiều Server (multiple application instances).
- Lượng truy cập (load) càng cao, tỉ lệ bị đè số (overwrite) càng lớn mà cấm có báo lỗi (error signal) một tiếng nào.

## 8. Hướng sửa cho Người Mới (Khuyến nghị)

Nếu chỉ là cộng dồn số đếm (commutative counter delta), hãy đẩy phần tính toán xuống Database bằng cú SQL Nguyên tử (atomic conditional SQL):

```sql
update job_progress
set completed_units = completed_units + :delta
where job_id = :jobId
  and completed_units + :delta <= total_units; -- Thêm điều kiện tránh lố
```

Nhớ check cái `affected-row count` (số dòng thành công) để biết mà báo lỗi / báo vui.
Nhưng nếu nghiệp vụ của bạn không chỉ là cộng trừ mà rắc rối hơn nhiều (aggregate mutation phức tạp), hãy ôm thẻ bùa `@Version` (Khóa Lạc Quan) kết hợp tự động thử lại (bounded retry) trong Giao dịch mới. Hoặc xài `FOR UPDATE` (Khóa Bi Quan) nếu bạn chấp nhận việc tụi nó đứng xếp hàng (blocking trade-off).

## 9. Phạm vi bài toán

Căn bệnh Mất Dữ Liệu này diễn ra ở khắp mọi nơi (generic anomaly). Nhưng nếu bạn đang xây app Ngân hàng đếm Tiền (Financial balance) hay Bán hàng giữ Kho (ecommerce stock), hãy đọc riêng bài `BANK-002` và `ECOM-001`; vì bài này chỉ mượn ví dụ đếm số để né tránh bớt các mớ bòng bong nghiệp vụ chuyên sâu đó.
