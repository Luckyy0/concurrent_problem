# Đoạn Code Thảm Họa: Đọc-Sửa-Ghi Bằng JPA (Broken JPA read-modify-write)

## 1. Thực Thể (Entity) Không Biết Tự Vệ

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
        completedUnits += delta; // Lỗi tiềm ẩn nằm đây! Cộng trên biến Java!
    }

    public int getCompletedUnits() {
        return completedUnits;
    }
}
```

Thực thể này KHÔNG HỀ có vòng bảo vệ `@Version`. Cái logic kiểm tra lỗi (Validation) nó chạy trơn tru trên cái con số Cũ Rích vừa tải lên, chứ KHÔNG PHẢI là con số Mới Nhất Dưới DB ngay lúc lưu.

## 2. Tầng Kho Lưu Trữ (Repository)

```java
public interface JobProgressRepository
        extends JpaRepository<JobProgress, UUID> {
    // Để trống thế này là đủ chết rồi.
}
```

## 3. Tầng Dịch Vụ Cùi Bắp (Broken service)

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
        // BƯỚC 1: ĐỌC
        JobProgress job = progress.findById(jobId)
            .orElseThrow();

        int before = job.getCompletedUnits();
        
        // BƯỚC 2: TÍNH TOÁN TRONG RAM
        job.addCompletedUnits(delta);

        // BƯỚC 3: GHI LẠI (Ảo tưởng là an toàn nhờ @Transactional)
        return new ProgressResult(
            jobId,
            before,
            job.getCompletedUnits()
        );
    }
}
```

Method này chạy qua Proxy của Spring rất chuẩn, mỗi request đẻ ra một Transaction riêng. Lỗi KHÔNG nằm ở cái proxy. Lỗi là do bạn gom 3 bước rời rạc qua lại giữa App và DB vào chung 1 quy trình:

```text
Đọc lên -> Tính toán trong App (JVM) -> Ép Ghi Đè số tuyệt đối
```

## 4. SQL Thằng Hibernate Sinh Ra (SQL Hibernate thực hiện)

Khúc ĐỌC (SELECT):

```sql
select job_id, completed_units, total_units
from job_progress
where job_id = ?;
```

Lúc Chốt Sổ/Xả hàng (flush/commit), thằng Hibernate tự soi lỗi và quăng ra cái lệnh GHI ĐÈ ngây ngô:

```sql
update job_progress
set completed_units = ?, -- Điền số 13, hoặc 14 tuyệt đối vào đây
    total_units = ?
where job_id = ?;      -- Điểm mù chết chóc!
```

Mặc kệ bạn cấu hình động đậy (`@DynamicUpdate`) kiểu gì, cốt lõi là điều kiện lọc (predicate) chỉ dựa vào đúng cái Khóa Chính:

```text
WHERE job_id = ?
```

VÀ HOÀN TOÀN THIẾU VẮNG:

```text
AND version = ? -- (Không có Khóa Lạc Quan)
AND completed_units = old_value -- (Không so sánh giá trị cũ)
```

Nên cả lệnh của ông A và lệnh của ông B chọt xuống đều thấy cập nhật thành công (affected rows = 1). Thằng Hibernate chẳng nhận được tín hiệu cầu cứu nào (conflict signal) để mà báo lỗi.

> **Nói ngắn gọn:** Cái `@Transactional` nó chỉ đảm bảo cái Lệnh UPDATE là Nguyên tử, chứ nó KHÔNG BIẾN 3 BƯỚC Đọc-Tính-Ghi dài dòng của bạn thành 1 Nghiệp vụ Nguyên tử được!

## 5. Hiện Trường Gây Án Cụ Thể (Concrete broken interleaving)

```text
Ban đầu = 10

Ông A gọi SELECT -> Thấy 10
Ông B gọi SELECT -> Cùng Thấy 10

App của ông A nhẩm tính -> Thành 13
App của ông B nhẩm tính -> Thành 14

Giao dịch A phang lệnh: UPDATE SET completed_units = 13 WHERE ...
Giao dịch A CHỐT (COMMIT)

Giao dịch B nhào tới phang: UPDATE SET completed_units = 14 WHERE ...
Giao dịch B CHỐT (COMMIT)

Thực tế = 14, Trong khi Đúng Ra phải là = 17. Mất mẹ nó 3 cái!
```

Dù ông B có vấp phải cái Khóa Dòng (row lock) và phải đứng Đợi ông A chốt xong, thì sau đó B vẫn thản nhiên ốp con số `14` (đã lỡ tính từ lúc đầu) đè lên mọi thứ.

## 6. Biện Hộ: Xài `save()` Có Sửa Được Không? (Không!)

Nếu bạn ép lưu rõ ràng:

```java
job.addCompletedUnits(delta);
progress.save(job);
```

Vẫn y chang! Nó vẫn ôm cái số liệu tính sẵn (absolute state) rớt xuống DB mà không hề có Version. Hàm `save()` không mang lại phép thuật Nguyên Tử nào hết.

## 7. Biện Hộ: Xài `saveAndFlush()` Chắc Ăn Hơn? (Cũng Không!)

```java
progress.saveAndFlush(job);
```

Thiếu `@Version` thì việc Flush này chỉ đẩy cái lệnh Ghi Đè Lỗi sớm hơn một chút thôi. Số dòng thay đổi vẫn là 1; Chả có Exception nào văng ra để bạn tự bắt mà Thử lại (retry).

## 8. Biện Hộ: Dùng Khóa Cục Bộ `synchronized` Trong Java? (Trẻ con!)

```java
public synchronized ProgressResult addCompletedUnits(...) {
    // ...
}
```

Trò này chỉ cấm được các luồng (threads) tranh nhau trên Cùng 1 cái Máy Chủ (JVM). NHƯNG:
- Máy chủ số 2 (Node 2) chạy App của bạn nó xài cái Khóa khác.
- Lỡ ai đó viết 1 Service khác cũng lưu chọt vô cái JobProgress này thì bó tay.
- Mấy cái Job chạy nền chọc thẳng vào SQL cũng không quan tâm Khóa Java.
- Lỡ sập máy khởi động lại là mất hết phối hợp.

Túm lại, Sổ Sách Xịn nằm ở PostgreSQL, thì Luật Chống Sửa Bậy phải được bảo vệ ở cấp độ Database/Transaction.

## 9. Cú Lừa Khi Tự Soi Lỗi Bằng Code (Validation cũng bị stale)

Giả sử ban đầu:
```text
completed = 95
total = 100
Ông A muốn cộng = 3
Ông B muốn cộng = 4
```

Cả 2 ông lấy 95 ra test: `95 + 3 <= 100` (OK!). `95 + 4 <= 100` (OK!). Cả 2 Đều Xanh (pass validation)!
Kết cục Ghi Đè bậy bạ sẽ ra `98` hoặc `99`. Bạn nhìn vào thấy Số Hoàn Thành vẫn Nhỏ Hơn Tổng, ồ tuyệt vời không có lỗi gì hết!
NHƯNG THỰC TẾ, Tổng sức của cả 2 ông dồn vào là `102` (vượt trần). Lỗi Mất Dữ Liệu đã âm thầm giấu nhẹm đi cái sai phạm to đùng, làm Sổ Sách thì đẹp nhưng Thực Tế Đã Nát (accepted-work invariant bị phá).

Bạn thấy đấy, Database Check kiểu `completed_units <= total_units` là cần thiết, nhưng chưa đủ đô để bắt quả tang cái lượng Delta bị đánh cắp.

## 10. Điều Kiện Để Thảm Họa Trổ Bông (Preconditions tái hiện)

- Dùng 2 Giao dịch vật lý riêng biệt với mức `READ COMMITTED`.
- Lệnh Đọc (SELECT) của cả 2 chạy xong trước khi có người chốt sổ đầu tiên.
- Đọc chay mà không ép xin Khóa (Không row lock trên read).
- Không thèm xài bùa `@Version`.
- Điều kiện Cập Nhật (WHERE) không hỏi thăm giá trị cũ/phiên bản cũ.
- Code luôn chạy nuột và trả về Success.
- Đứa nào ghi Đè tuyệt đối sau cùng, đứa đó Thắng.

## 11. Các Giải Pháp Chữa Cháy Nửa Mùa Đừng Dùng (Những cách sửa chưa đủ)

- CHỈ gắn thêm `@Transactional`.
- CHỈ cố đổi gọi hàm `save()` hoặc `saveAndFlush()`.
- CHỈ nhét cái Khóa `synchronized` vô dụng trong RAM.
- CHỈ check lệnh IF `completed <= total` bằng Code Java.
- CHỈ Tăng cấp Cô lập (Isolation level) nhưng làm lơ lỗi bị đạp ra khỏi hàng (serialization failure).
- Cố Thử Lại (Retry) nhưng vẫn nhắm mắt xài lại cái Đối Tượng rác và Giao dịch nát từ trước.
- Sống trong Ảo Tưởng rằng MVCC thần thánh sẽ tự động hợp nhất (merge) dữ liệu giùm bạn. Đừng Mơ!
