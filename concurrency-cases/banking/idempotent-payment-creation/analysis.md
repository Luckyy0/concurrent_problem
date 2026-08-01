# Concurrency Analysis

## Initial State
- Bảng `payment_records` trống.
- Không có `UNIQUE` constraint trên `idempotency_key`.
- API Client bị cấu hình timeout là 1 giây. Nó gửi yêu cầu thứ nhất (`req1`), sau 1s chưa thấy kết quả nên nó retry gửi yêu cầu thứ hai (`req2`) với cùng `Idempotency-Key` là `req-idx-777`.

## Expected vs Actual

**Expected**: 
- Chỉ một bản ghi payment với khóa `req-idx-777` được tạo ra.
- Nếu `req1` đang được xử lý, `req2` phải bị từ chối hoặc bị block chờ `req1` xử lý xong và nhận lại kết quả của `req1` (response replay).

**Actual**:
- Hai request chạy song song, pass qua lớp check của Spring Data JPA.
- Hai lệnh `INSERT` thành công vào DB.
- External Payment Gateway bị gọi 2 lần, thực hiện charge tiền 2 lần.

## Root Cause Analysis by Layer

### 1. Application Layer (Spring/Java)
- Việc dùng JPA `Optional<PaymentRecord> existing = repository.findById(...)` gefollow bởi `repository.save(...)` tạo ra lỗ hổng **Time-of-check to time-of-use (TOCTOU)**.
- Khi Thread B thực hiện `check`, kết quả `save` của Thread A vẫn chưa được commit xuống DB nên Thread B hoàn toàn không nhìn thấy.

### 2. Database Layer (PostgreSQL)
- Do thiếu `UNIQUE INDEX`, PostgreSQL coi 2 lệnh `INSERT` chứa cùng giá trị `idempotency_key` là hoàn toàn hợp lệ. Cả hai transaction đều không có lý do gì để trigger lock hay rollback.

## MVCC / Lock Behavior Detail

Trong PostgreSQL (và các database hỗ trợ MVCC), mức cô lập mặc định thường là `Read Committed`.

| Time | Thread A (Tx 1) | Thread B (Tx 2) | PostgreSQL Behavior / MVCC |
|------|-----------------|-----------------|----------------------------|
| 1 | `BEGIN` | | Bắt đầu Tx 1 |
| 2 | | `BEGIN` | Bắt đầu Tx 2 |
| 3 | `SELECT * FROM payment_records WHERE idempotency_key = 'K1'` | | Trả về 0 dòng. Dữ liệu Snapshot tx1. |
| 4 | | `SELECT * FROM payment_records WHERE idempotency_key = 'K1'`| Trả về 0 dòng. Dữ liệu Snapshot tx2. |
| 5 | `INSERT INTO payment_records(idempotency_key) VALUES ('K1')`| | Tạo row R1. Khóa ghi (RowExclusiveLock) trên row R1. Tx 2 không nhìn thấy. |
| 6 | | `INSERT INTO payment_records(idempotency_key) VALUES ('K1')`| Tạo row R2. Khóa ghi trên row R2. Không có conflict vì không có Unique constraint. |
| 7 | `COMMIT` | | Row R1 trở thành public (visible cho tx tương lai). |
| 8 | | `COMMIT` | Row R2 trở thành public. Data Invariant bị phá vỡ. |

## Multi-instance Behavior
Vấn đề này tồi tệ hơn trong môi trường multi-instance (Kubernetes, nhiều pods). 
- Một số nhà phát triển có thể cố gắng sửa lỗi bằng cách dùng `synchronized` trong Java hoặc `ReentrantLock`.
- Giải pháp In-memory lock sẽ thất bại ngay lập tức khi hệ thống scale out (vì Thread A vào Pod 1, Thread B vào Pod 2 - hai JVM khác nhau không share locks).

## Recovery Timeline
Khi dữ liệu rác (duplicate payments) đã được tạo, việc khắc phục rất khó khăn:
1. Xác định các bản ghi bị trùng lặp bằng cách group theo `idempotency_key`.
2. Xác nhận trạng thái cuối cùng với external Payment Gateway.
3. Nếu Gateway bên ngoài cũng charge 2 lần, phải gọi API `Refund` cho 1 giao dịch.
4. Cập nhật trạng thái thủ công trên DB nội bộ để đánh dấu giao dịch trùng lặp là `REFUNDED` hoặc `CANCELLED`.
