# Đoạn Mã Gây Lỗi: Đọc-Sửa-Ghi Bằng JPA (Broken JPA read-modify-write)

## 1. Thực Thể (Entity) Thiếu Cơ Chế Bảo Vệ

```java
@Entity
@Table(name = "job_progress")
public class JobProgress {
    @Id
    private UUID jobId;

    private int completedUnits;
    private int totalUnits;

    protected JobProgress() {
    }

    public void addCompletedUnits(int delta) {
        if (delta <= 0) {
            throw new IllegalArgumentException(
                "Delta must be positive"
            );
        }
        if (completedUnits + delta > totalUnits) {
            throw new IllegalStateException(
                "Completed units exceed total units"
            );
        }
        completedUnits += delta; // Lỗi tiềm ẩn nằm đây! Cộng dồn trên biến bộ nhớ (Java memory).
    }

    public int getCompletedUnits() {
        return completedUnits;
    }
}
```

Thực thể này hoàn toàn không tích hợp cơ chế bảo vệ phiên bản (`@Version`). Các logic kiểm tra giới hạn (Validation) chỉ hoạt động trên con số cũ được tải lên trước đó, KHÔNG PHẢI là số liệu thời gian thực hiện có tại Cơ sở dữ liệu ngay lúc ghi.

## 2. Tầng Kho Lưu Trữ (Repository)

```java
public interface JobProgressRepository
        extends JpaRepository<JobProgress, UUID> {
    // Chỉ sử dụng các phương thức mặc định của JpaRepository mà không thiết lập thêm truy vấn khóa (locking).
}
```

## 3. Lớp Dịch Vụ Với Quy Trình Lỗi (Broken service)

```java
@Service
public class JobProgressService {
    private final JobProgressRepository progress;

    public JobProgressService(JobProgressRepository progress) {
        this.progress = progress;
    }

    @Transactional
    public ProgressResult addCompletedUnits(
        UUID jobId,
        int delta
    ) {
        // BƯỚC 1: TRUY VẤN
        JobProgress job = progress.findById(jobId)
            .orElseThrow();

        int before = job.getCompletedUnits();
        
        // BƯỚC 2: TÍNH TOÁN TRONG BỘ NHỚ (JVM)
        job.addCompletedUnits(delta);

        // BƯỚC 3: GHI LẠI (Sự ảo tưởng về tính an toàn của @Transactional)
        return new ProgressResult(
            jobId,
            before,
            job.getCompletedUnits()
        );
    }
}
```

Mặc dù phương thức này được bao bọc bởi Spring Proxy để cấp phát một Transaction riêng biệt, lỗi hoàn toàn không thuộc về Spring. Rủi ro sinh ra khi ứng dụng gộp 3 bước xử lý tách rời nhau qua lại giữa App và DB vào một quy trình đơn lẻ:

```text
Đọc dữ liệu lên bộ nhớ -> Tính toán bên trong App (JVM) -> Ép Ghi Đè số trị tuyệt đối
```

## 4. Câu Lệnh SQL Mà Hibernate Khởi Tạo (SQL Hibernate thực hiện)

Bước Truy Vấn (SELECT):

```sql
select job_id, completed_units, total_units
from job_progress
where job_id = ?;
```

Khi tiến trình đồng bộ dữ liệu kích hoạt (flush/commit), cơ chế tự động theo dõi thay đổi (dirty checking) của Hibernate sẽ xuất ra một lệnh GHI ĐÈ tuyệt đối:

```sql
update job_progress
set completed_units = ?, -- Điền giá trị 13, hoặc 14 cố định vào đây
    total_units = ?
where job_id = ?;      -- Điểm rủi ro cốt lõi!
```

Dù bạn có chỉ định Hibernate sử dụng `@DynamicUpdate` để cập nhật động các trường, điều kiện lọc (predicate) vẫn chỉ phụ thuộc duy nhất vào Khóa Chính:

```text
WHERE job_id = ?
```

VÀ HOÀN TOÀN BỎ QUA CÁC YẾU TỐ BẢO VỆ:

```text
AND version = ? -- (Khóa Lạc Quan không tồn tại)
AND completed_units = old_value -- (Không đối chiếu giá trị cũ)
```

Điều này dẫn đến hệ quả là mọi lệnh ghi của luồng A hay luồng B đều xác nhận hoàn thành (affected rows = 1). Hibernate không hề nhận được bất kỳ tín hiệu cảnh báo xung đột (conflict signal) nào để kích hoạt luồng xử lý lỗi.

> **Nói ngắn gọn:** Annotation `@Transactional` chỉ đảm bảo tính nguyên tử (Atomicity) cho một Câu lệnh Cập nhật (UPDATE), chứ KHÔNG HỀ biến quy trình 3 bước Đọc-Tính-Ghi trên bộ nhớ ứng dụng của bạn thành một quy trình Nguyên tử.

## 5. Hiện Trường Gây Lỗi Cụ Thể (Concrete broken interleaving)

```text
Dữ liệu ban đầu = 10

Giao dịch A gọi SELECT -> Đọc kết quả 10
Giao dịch B gọi SELECT -> Cùng Đọc kết quả 10

App xử lý của A tính toán -> Đạt 13
App xử lý của B tính toán -> Đạt 14

Giao dịch A gửi lệnh: UPDATE SET completed_units = 13 WHERE ...
Giao dịch A HOÀN TẤT (COMMIT)

Giao dịch B gửi lệnh: UPDATE SET completed_units = 14 WHERE ...
Giao dịch B HOÀN TẤT (COMMIT)

Kết quả lưu trữ = 14, trong khi kết quả toán học phải là = 17. Hệ thống đã mất 3 đơn vị công việc.
```

Ngay cả khi Giao dịch B vướng phải Khóa Dòng (row lock) và phải Đợi Giao dịch A kết thúc, sau thời gian chờ đợi B vẫn sẽ mặc định điền giá trị `14` (do đã tính toán từ giá trị cũ nạp lên) đè lên tất cả.

## 6. Việc Gọi Hàm `save()` Không Có Tác Dụng Khắc Phục

Việc bổ sung thêm phương thức lưu dữ liệu thủ công:

```java
job.addCompletedUnits(delta);
progress.save(job);
```

Vẫn mang lại kết quả lỗi tương tự! Phương thức lưu trữ chỉ đơn giản gửi trạng thái giá trị cố định (absolute state) xuống DB mà không áp dụng Khóa phiên bản. Hàm `save()` không mang lại tính nguyên tử bổ sung nào cho vòng đời ứng dụng.

## 7. Hàm `saveAndFlush()` Cũng Không Mang Lại Hiệu Quả

```java
progress.saveAndFlush(job);
```

Thiếu sót cơ chế `@Version`, việc Flush này chỉ rút ngắn thời gian đẩy Câu lệnh ghi đè xuống CSDL. Thuộc tính dòng bị tác động (affected rows) vẫn là 1; Ứng dụng không ghi nhận Exception nào để tự động phục hồi (retry).

## 8. Khóa Nội Bộ `synchronized` Trong Java Hoàn Toàn Vô Hiệu

```java
public synchronized ProgressResult addCompletedUnits(...) {
    // ...
}
```

Từ khóa này chỉ giới hạn khả năng tương tranh giữa các luồng (threads) trên CÙNG MỘT Máy Chủ (JVM). Tuy nhiên:
- Máy chủ thứ hai (Node 2) trong hệ thống triển khai không chia sẻ Khóa này.
- Các microservice hoặc tác vụ nền (background job) can thiệp thẳng vào CSDL hoàn toàn bỏ qua Java.
- Mất toàn bộ khả năng đồng bộ nếu máy chủ khởi động lại (restart).

Tóm lại, nếu Hệ thống Chân lý dữ liệu được đặt tại PostgreSQL, thì Cơ chế Ngăn ngừa Thay đổi Không Hợp Lệ phải thiết lập tại mức độ Giao dịch / CSDL.

## 9. Cạm Bẫy Của Tính Năng Tự Kiểm Tra (Validation cũng bị stale)

Hãy quan sát trường hợp:
```text
completed = 95
total = 100
Luồng A yêu cầu cộng = 3
Luồng B yêu cầu cộng = 4
```

Cả 2 luồng đều lấy số liệu 95 ra để đối chiếu: `95 + 3 <= 100` (Hợp lệ). `95 + 4 <= 100` (Hợp lệ). Cả hai luồng đều thỏa mãn điều kiện (pass validation)!
Kết cục việc Ghi Đè diễn ra và hệ thống báo `98` hoặc `99`. Khi kiểm tra dữ liệu, số liệu Hoàn Thành nhỏ hơn Tổng (Total), không có lỗi nào hiển thị!
Tuy nhiên trên THỰC TẾ, tổng tiến độ của 2 giao dịch là `102` (vượt quá hạn mức). Sự cố Mất Dữ Liệu vô tình bao che cho việc phân bổ sai (accepted-work invariant bị phá).

Bạn sẽ thấy rằng: Kiểm tra bằng CSDL với `check (completed_units <= total_units)` là cần thiết, nhưng vẫn không đủ khả năng bù đắp hay nhận diện lượng Delta công việc đã thất thoát.

## 10. Điều Kiện Để Lỗi Tái Hiện (Preconditions tái hiện)

- Triển khai 2 Giao dịch độc lập cấu hình ở mức `READ COMMITTED`.
- Truy vấn Đọc (SELECT) của 2 giao dịch hoàn tất trước khi có bất kỳ giao dịch nào thực hiện bước chốt dữ liệu (commit).
- Truy vấn thông thường không sử dụng cơ chế Khóa Dòng (Không row lock trên read).
- Không áp dụng hệ thống bảo vệ phiên bản `@Version`.
- Câu lệnh Cập Nhật (WHERE) bỏ qua thông tin Giá trị cũ/Phiên bản cũ.
- Ứng dụng luôn vận hành trơn tru và hoàn tất (Success).
- Luồng nào cập nhật Ghi đè sau cùng, luồng đó sẽ chiếm dụng kết quả chung cuộc.

## 11. Các Phương Án Sửa Chữa Sai Lầm (Những cách sửa chưa đủ)

- Chỉ gắn Annotation `@Transactional`.
- Chỉ chuyển sang phương thức `save()` hoặc `saveAndFlush()`.
- Chỉ sử dụng cơ chế khóa phân luồng `synchronized` trên bộ nhớ ứng dụng.
- Chỉ thực hiện vòng lặp IF `completed <= total` ở cấp độ Java Code.
- Chỉ Nâng cấp độ Cô lập (Isolation level) nhưng bỏ qua các xử lý bắt ngoại lệ khi bị loại khỏi hàng chờ (serialization failure).
- Cố gắng áp dụng Retry (Thử lại) nhưng vẫn giữ đối tượng lỗi và giao dịch lỗi cũ.
- Phụ thuộc mù quáng vào khả năng MVCC của cơ sở dữ liệu sẽ tự động hợp nhất (merge) phép cộng nghiệp vụ mà không cần chỉ định.
