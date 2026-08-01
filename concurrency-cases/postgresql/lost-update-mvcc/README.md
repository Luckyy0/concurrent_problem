# DB-001 — Vấn Đề Ghi Đè Mất Dữ Liệu (Lost update) Dưới Cơ Chế PostgreSQL MVCC

## 1. Tóm tắt vấn đề

Giả sử tiến trình (Job) `IMPORT-42` đã hoàn thành được `10` đơn vị công việc (units).
Tại cùng một thời điểm:
- Luồng xử lý A (Worker A) báo cáo hoàn thành thêm `3` đơn vị.
- Luồng xử lý B (Worker B) báo cáo hoàn thành thêm `4` đơn vị.

Vì sử dụng Spring Transaction và JPA mặc định, cả hai luồng đều tải thực thể (entity) hiện tại từ cơ sở dữ liệu lên bộ nhớ: "Số lượng ban đầu là `10`".
- Luồng A thực hiện tính toán trong bộ nhớ (JVM): `10 + 3 = 13`.
- Luồng B cũng thực hiện tính toán tương tự: `10 + 4 = 14`.

Khi lưu dữ liệu xuống DB (thông qua cơ chế Hibernate dirty checking flush), cả hai luồng đều phát sinh câu lệnh UPDATE ghi đè giá trị tuyệt đối (absolute values) với điều kiện đơn giản `WHERE job_id = ?`.

Kết quả thực thi:
- Luồng A hoàn tất giao dịch (commit) trước, ghi nhận giá trị `13`.
- Luồng B hoàn tất giao dịch sau, ghi đè lên kết quả của A, sửa thành giá trị `14`.
- Công sức `+3` của luồng A bị mất hoàn toàn (Lost update), mặc dù cả A và B đều không ghi nhận bất kỳ ngoại lệ (exception) nào và đều báo cáo thao tác thành công.

Điều kiện bất biến (Invariant) bị phá vỡ:
```text
Tổng số công việc Đã Xong (final completedUnits)
  = Số ban đầu (initial) + Tổng khối lượng (delta) của mọi lệnh được chấp nhận

Với số ban đầu = 10, khối lượng A = 3, khối lượng B = 4:
Thì kết quả CẦN PHẢI LÀ = 17. (Thực tế hệ thống lại ghi nhận 14!)
```

> **Nói ngắn gọn:** Cơ chế MVCC của PostgreSQL cho phép cả A và B cùng nhìn thấy phiên bản dữ liệu cũ (đã commit). Nếu mã nguồn của bạn tự tính toán con số tuyệt đối trên bộ nhớ ứng dụng rồi đẩy thẳng lệnh UPDATE xuống CSDL, CSDL sẽ không thể tự động gộp (merge) sự thay đổi này.

## 2. Các thành phần tham gia (Actors và shared state)

| Thành phần | Vai trò |
| --- | --- |
| Worker A | Thực thi Giao dịch (Transaction) cộng thêm `3` đơn vị. |
| Worker B | Thực thi Giao dịch cộng thêm `4` đơn vị. |
| Bảng `job_progress` | Nơi lưu giữ trạng thái chính thức (Authoritative counters). |
| Bộ nhớ đệm Hibernate (Persistence context) | Nơi lưu giữ bản nháp (snapshot) và tự động nhận diện thay đổi (dirty-check) để cập nhật số tuyệt đối. |
| PostgreSQL | Hệ quản trị CSDL quản lý ảnh chụp dữ liệu (snapshots), khóa dòng (row locks) và quyết định tầm nhìn dữ liệu khi commit (commit visibility). |

Trạng thái ban đầu (Initial row):
```text
job_id          = IMPORT-42
completed_units = 10
total_units     = 100
```

Trạng thái lỗi cuối cùng (Broken final):
```text
completed_units = 14
Cả hai giao dịch báo cáo thành công!
```

## 3. Ranh giới giao dịch và điểm tranh chấp (Transaction boundary và contention point)

Mỗi lần phương thức `addCompletedUnits()` được gọi, Spring Proxy sẽ khởi tạo một Giao dịch độc lập trên PostgreSQL với mức độ cô lập `READ COMMITTED` (Chỉ Đọc Dữ Liệu Đã Commit).

Chuỗi thao tác không nguyên tử (Non-atomic sequence) gây ra lỗi:
```text
Lấy dữ liệu hiện tại từ DB (SELECT)
  -> Chạy logic tính toán cục bộ trong JVM
  -> Lưu số mới xuống DB CHỈ dựa trên Khóa Chính (UPDATE by primary key)
```

Điểm tranh chấp (Contention point) chính là dòng dữ liệu `job_progress(job_id='IMPORT-42')`. Các lệnh UPDATE từ hai luồng sẽ tranh giành Khóa cấp dòng (row lock), do đó tại thời điểm ghi dữ liệu, chúng buộc phải thi hành tuần tự (serialize về thời gian). Tuy nhiên, việc tuần tự hóa các lệnh ghi đè mang giá trị sai lệch (stale absolute writes) hoàn toàn không bảo vệ được tính cộng dồn (additive invariant) của hệ thống.

## 4. Kỳ vọng theo thiết kế lỗi và Thực tế (Expected và actual)

| Bước thực thi | Luồng A | Luồng B | Kết quả chung (Final) |
| --- | --- | --- | --- |
| Đọc dữ liệu (Read) | 10 | 10 | |
| Tính toán (Calculate) | 13 | 14 | |
| Giao dịch hoàn tất (Commit) | Hoàn tất trước | Hoàn tất sau | |
| Khối lượng kỳ vọng (Expected) | +3 | +4 | 17 |
| Kết quả thực tế (Broken write) | Lưu 13 | Đè thành 14 | 14 |

Toàn bộ quá trình không hề phát sinh Exception, Timeout, hay cảnh báo về số lượng dòng cập nhật (affected-row conflict). Điều này xảy ra do lệnh UPDATE chỉ dựa vào Khóa chính để tìm dòng cần ghi đè, không kèm theo điều kiện xác nhận phiên bản hoặc giá trị cũ (version/old-value predicate).

## 5. Thuật ngữ kỹ thuật cần nắm

| Thuật ngữ | Giải thích |
| --- | --- |
| Lost update (Mất dữ liệu cập nhật) | Kết quả của một giao dịch bị giao dịch đến sau ghi đè âm thầm, làm mất dữ liệu hợp lệ. |
| MVCC | Cơ chế kiểm soát đồng thời đa phiên bản (Multi-Version Concurrency Control) của RDBMS. |
| Statement snapshot | Bức ảnh chụp dữ liệu tại thời điểm chạy từng Câu lệnh, áp dụng cho mức độ `READ COMMITTED`. |
| Read-modify-write | Chuỗi thao tác: Đọc dữ liệu lên bộ nhớ -> Tính toán -> Ghi đè kết quả xuống DB. |
| Dirty checking | Cơ chế của Hibernate tự động theo dõi thay đổi trên Entity và sinh lệnh UPDATE tương ứng. |
| Absolute write | Lệnh ghi áp đặt con số cố định: Ví dụ `SET completed_units = 14`. |
| Atomic delta | Giao việc tính toán cho CSDL bằng thao tác nguyên tử: `SET completed_units = completed_units + 4`. |
| Version predicate | Điều kiện kiểm tra phiên bản dữ liệu: `WHERE version = expected` để phòng ngừa ghi đè mù quáng. |

## 6. Điều hướng tài liệu

- [Đoạn mã lỗi read-modify-write bằng JPA](broken-code.md)
- [Phân tích chi tiết tầm nhìn MVCC và lỗi Mất Dữ Liệu](analysis.md)
- [Các giải pháp xử lý: Cập nhật Nguyên tử, Khóa Lạc Quan và Khóa Bi Quan](solutions.md)
- [Bộ kiểm thử tái hiện lỗi ở PostgreSQL](experiments.md)
- [Giao dịch Nguyên tử trong CSDL (Atomic database operations)](../../concepts/atomic-database-operations.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Các mức độ Cô lập (Isolation levels)](../../concepts/isolation-levels.md)
- [Khóa Lạc Quan (Optimistic locking)](../../concepts/optimistic-locking.md)
- [Khóa Bi Quan (PostgreSQL locks)](../../concepts/postgresql-locks.md)
- [Kiểm thử Tương tranh (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Hậu quả trên môi trường Production

- Biểu đồ và báo cáo tiến độ hiển thị số liệu thấp hơn so với thực tế công việc đã hoàn thành.
- Các bộ kích hoạt chuyển trạng thái (Job completion trigger) có thể không hoạt động hoặc hoạt động trễ do tiến độ không đạt mức 100% như mong đợi.
- Dữ liệu truy vết (Audit Log) từ phía ứng dụng báo cáo thành công, nhưng CSDL lại thiếu hụt số liệu, gây khó khăn cho việc đối soát.
- Quá trình xử lý chênh lệch số liệu (Reconciliation) đòi hỏi nhiều công sức phân tích lại các sự kiện (event logs) để tính toán thủ công con số chính xác.
- Việc sử dụng khóa bộ nhớ (JVM-local lock - synchronized) sẽ vô tác dụng khi hệ thống được triển khai đa máy chủ (multiple application instances).
- Lượng truy cập (load) càng cao, tỷ lệ mất dữ liệu càng lớn mà hệ thống không hề sinh ra bất kỳ tín hiệu cảnh báo lỗi nào.

## 8. Hướng khắc phục khuyến nghị

Nếu logic nghiệp vụ chỉ đơn thuần là cộng dồn số đếm (commutative counter delta), hãy tối ưu bằng cách đẩy phần tính toán xuống cơ sở dữ liệu qua cú pháp SQL Nguyên tử (atomic conditional SQL):

```sql
update job_progress
set completed_units = completed_units + :delta
where job_id = :jobId
  and completed_units + :delta <= total_units; -- Kiểm tra điều kiện giới hạn
```

Nhớ kiểm tra số lượng dòng thay đổi (`affected-row count`) để xác nhận thao tác thành công.
Tuy nhiên, nếu nghiệp vụ phức tạp và liên quan đến việc thay đổi nhiều trường dữ liệu (aggregate mutation), hãy áp dụng cơ chế Khóa Lạc Quan (`@Version`) kết hợp cùng cơ chế Tự động thử lại (bounded retry) trong giao dịch mới. Hoặc sử dụng Khóa Bi Quan (`FOR UPDATE`) nếu hệ thống chấp nhận sự đánh đổi về thời gian chờ (blocking trade-off).

## 9. Phạm vi bài toán

Lỗi Mất Dữ Liệu Cập Nhật (Lost update) là một rủi ro phổ biến trong lập trình tương tranh (generic anomaly). Tài liệu này sử dụng ví dụ về bộ đếm tiến độ để làm rõ nguyên lý. Nếu hệ thống của bạn xử lý các nghiệp vụ nhạy cảm như Giao dịch tài chính (Financial balance) hay Quản lý Tồn kho (ecommerce stock), vui lòng tham khảo các bài `BANK-002` và `ECOM-001` để xem các thiết kế và giải pháp nghiệp vụ chuyên sâu phù hợp hơn.
