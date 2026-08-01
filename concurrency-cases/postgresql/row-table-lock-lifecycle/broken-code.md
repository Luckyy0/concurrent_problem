# Phân Tích Mã Nguồn Lỗi: Ảo Tưởng Về Lệnh Đọc (Broken SELECT-lock assumptions)

## 1. Khuôn Mẫu Gốc (Entity)

```java
@Entity
@Table(name = "tenant_quota")
public class TenantQuota {
    @Id
    @Column(name = "tenant_id", nullable = false)
    private UUID tenantId;

    @Column(nullable = false)
    private int quota;

    @Column(nullable = false)
    private long revision;

    protected TenantQuota() {
    }

    public void changeQuota(int newQuota) {
        if (newQuota < 0) {
            throw new IllegalArgumentException("quota must be non-negative");
        }
        quota = newQuota;
        revision++;
    }

    public int getQuota() {
        return quota;
    }
}
```

Trường dữ liệu `revision` trong ví dụ này được sử dụng như một trường phục vụ ghi vết kiểm toán (audit field) thuần túy. Nó chưa được cấu hình trở thành công cụ của cơ chế khóa lạc quan (`@Version` của Hibernate). Mục tiêu của phần này là tập trung phân tích những quan niệm sai lệch về cơ chế khóa cấp dòng (explicit row lock).

## 2. Mã lỗi: Sử dụng đọc thông thường thay vì khóa (Broken writer using plain SELECT)

```java
@Service
public class QuotaAdministrationService {
    private final TenantQuotaRepository quotas;
    private final QuotaRules rules;

    public QuotaAdministrationService(
        TenantQuotaRepository quotas,
        QuotaRules rules
    ) {
        this.quotas = quotas;
        this.rules = rules;
    }

    @Transactional
    public void changeQuota(UUID tenantId, int requestedQuota) {
        // LỖI KỸ THUẬT: Đọc dữ liệu không khóa bảo vệ
        TenantQuota quota = quotas.findById(tenantId)
            .orElseThrow(QuotaNotFoundException::new);

        rules.validateTransition(quota.getQuota(), requestedQuota);
        quota.changeQuota(requestedQuota);
    }
}
```

Phương thức `findById()` sẽ sinh ra lệnh SQL truy vấn bình thường (ordinary SELECT):

```sql
select tenant_id, quota, revision
from tenant_quota
where tenant_id = :tenantId;
```

Lệnh `SELECT` này không hề chứa từ khóa yêu cầu bảo vệ độc quyền (như `FOR UPDATE`) hay điều kiện kiểm tra phiên bản (version predicate). Việc đặt annotation `@Transactional` trên dịch vụ (service) thiết lập ranh giới giao dịch, nhưng nó KHÔNG tự động chuyển đổi câu truy vấn thông thường thành truy vấn có khóa.

> **Ghi chú quan trọng:** Persistence Context (bộ đệm của Hibernate) bảo vệ sự thống nhất của đối tượng (managed entity) trong bộ nhớ ứng dụng JVM. Nó hoàn toàn không phản ánh hay áp đặt một trạng thái khóa độc quyền (pessimistic lock) tương ứng xuống mức bản ghi trong cơ sở dữ liệu PostgreSQL.

## 3. Quá trình phát sinh lỗi khi chạy đồng thời (Concurrent SQL timeline)

Giả sử trạng thái quota ban đầu là `10`:

```text
Luồng A khởi tạo giao dịch (BEGIN).
Luồng A thực thi lệnh SELECT thông thường (plain SELECT) -> Nhận giá trị 10.
Luồng A đang thực hiện tính toán (local validation) trên ứng dụng.

Luồng B khởi tạo giao dịch (BEGIN).
Luồng B thực thi lệnh UPDATE cập nhật quota = 8. (Thành công ngay lập tức).
Luồng B thực thi COMMIT và kết thúc.

Luồng A tiếp tục, gọi flush() sinh ra lệnh UPDATE ghi đè giá trị quota = 12 dựa trên dữ liệu cũ.
Luồng A thực thi COMMIT.
Kết quả: Bản cập nhật của Luồng B bị ghi đè không minh bạch (lost update).
```

Nguyên nhân Luồng B không bị chặn:
Luồng A khi gọi lệnh `SELECT` chỉ được hệ thống cấp khóa bảo vệ cấp bảng `ACCESS SHARE`. Mức khóa này không xung đột với các lệnh biến đổi dữ liệu (DML). Do đó, Luồng B hoàn toàn có quyền can thiệp và thực thi lệnh cập nhật trên cùng một dòng.

## 4. Giải pháp lỗi: Đọc bằng khóa tại bảng điều khiển hiển thị (Broken dashboard using locking read)

Sau sự cố, đội ngũ phát triển đôi khi đưa ra quyết định cực đoan: Bổ sung khóa cho toàn bộ các hàm đọc.

```java
public interface TenantQuotaRepository
    extends JpaRepository<TenantQuota, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select q from TenantQuota q where q.tenantId = :tenantId")
    Optional<TenantQuota> findForEverything(UUID tenantId);
}
```

Và sử dụng phương thức này cả trong chức năng báo cáo (Dashboard):

```java
@Transactional(readOnly = true)
public QuotaView dashboard(UUID tenantId) {
    TenantQuota quota = quotas.findForEverything(tenantId)
        .orElseThrow();
    return QuotaView.from(quota);
}
```

Kết quả: Bảng điều khiển (Dashboard), một tính năng chỉ yêu cầu hiển thị dữ liệu (read-only observer) từ phiên bản commit gần nhất, giờ đây lại cố gắng thiết lập khóa bảo vệ (pessimistic lock). Điều này đặt hệ thống truy vấn hiển thị vào tình thế cạnh tranh xung đột trực tiếp với hệ thống xử lý thao tác ghi (writers). Hậu quả là sinh ra trạng thái chờ khóa vô ích (needless serialization) làm suy giảm tài nguyên kết nối của ứng dụng một cách nghiêm trọng.

## 5. Ranh giới giữ khóa quá lớn (Broken long-lock boundary)

```java
@Transactional
public void changeAndNotify(UUID tenantId, int requestedQuota) {
    TenantQuota quota = quotas.findForEverything(tenantId)
        .orElseThrow();

    // LỖI KIẾN TRÚC: Gọi I/O ngoại vi trong phạm vi giao dịch cơ sở dữ liệu
    policyClient.confirmChange(tenantId, requestedQuota);
    quota.changeQuota(requestedQuota);
}
```

Mã nguồn trên đã nới rộng ranh giới giữ khóa cấp dòng thông qua việc thực hiện một lời gọi I/O từ xa (remote call). Giao dịch vật lý (physical transaction) - và cùng với đó là khóa dòng - sẽ kéo dài thời gian tồn tại (lock lifetime) một cách không cần thiết cho đến khi tiến trình mạng hoàn tất. Điều này sẽ làm cạn kiệt số lượng truy cập được phục vụ từ Connection Pool và gây lỗi quá thời gian chờ (timeout) cho các luồng xử lý khác.

## 6. Các quan niệm sai lầm về lệnh SQL (SQL false assumptions)

### Sai lầm 1: SELECT tự động khóa

```sql
begin;
select * from tenant_quota where tenant_id=:id;
-- Nhầm tưởng: Dòng này đã được bảo vệ độc quyền (row-level locked).
```

Thực tế: Truy vấn `SELECT` không thiết lập khóa dòng. Các giao dịch `UPDATE` khác hoàn toàn có thể cập nhật song song.

### Sai lầm 2: SELECT cấm mọi loại đọc

```sql
begin;
select * from tenant_quota where tenant_id=:id for update;
update tenant_quota set quota=12 where tenant_id=:id;
-- (Chưa commit)
```

Nhầm tưởng: Mọi thao tác đọc đều sẽ bị chặn chờ.

Thực tế: Chỉ có các thao tác sửa đổi (`UPDATE`, `DELETE`) hoặc yêu cầu khóa tường minh (`SELECT ... FOR UPDATE`) mới bị chặn (incompatible locking). Một truy vấn đọc thông thường (`plain SELECT`) từ luồng khác vẫn thành công vì nó sử dụng ảnh chụp phiên bản MVCC (old committed quota) chứ không cố gắng xin cấp khóa.

## 7. Các điều kiện tái hiện tranh chấp (Preconditions for reproduction)

1. Thiết lập nhiều luồng hoạt động trên nhiều kết nối/giao dịch độc lập (independent connections/transactions).
2. Thiết lập cơ sở dữ liệu ở mức cô lập mặc định là `READ COMMITTED`.
3. Có luồng thực hiện đọc thông thường `plain SELECT`.
4. Cố tình sử dụng yêu cầu lấy khóa `FOR UPDATE` trong một luồng chỉ định.
5. Luồng sở hữu khóa (Lock holder) kéo dài giao dịch hoặc giữ trạng thái dữ liệu chưa commit, để đánh giá hành vi luồng chờ (waiter).
6. Tuyệt đối không bọc chức năng test trong một transaction lớp ngoài (`@Transactional` ở mức Unit Test), bởi vì giao dịch bao bọc sẽ ngăn chặn việc commit/rollback thực tế đến cơ sở dữ liệu, làm sai lệch cơ chế khóa thực thụ.

## 8. Các phương án khắc phục chưa triệt để (Incomplete mitigations)

### Chỉ bổ sung `@Transactional`

Gắn `@Transactional` định nghĩa khoảng thời gian tồn tại của giao dịch (transaction lifetime), nhưng nó không biến các truy vấn JPA bình thường trở thành truy vấn đòi hỏi khóa (query lock mode).

### Sử dụng khóa cục bộ Java (`synchronized`)

Từ khóa `synchronized` chỉ có hiệu lực trên một đối tượng và giới hạn ở quy mô của một Java Virtual Machine (JVM). Trong môi trường phân tán (multi-instance/Kubernetes), các yêu cầu chuyển trực tiếp đến cơ sở dữ liệu sẽ bỏ qua hoàn toàn khóa cục bộ này.

### Cố gắng ép chặn mọi luồng đọc (Lock mọi read)

Việc yêu cầu hệ thống phải ngăn chặn tất cả các giao dịch đọc không mang lại thêm bất kỳ sự bảo đảm toàn vẹn dữ liệu nào. Khóa đọc chỉ có tác dụng khi quá trình thao tác là chuỗi đọc - kiểm tra - ghi (read-modify-write). Chặn đọc thông thường sẽ khiến hệ thống không thể mở rộng và tăng độ trễ truy vấn.

### Dọn trống bộ đệm (Flush) để giải phóng khóa

Gửi lệnh thao tác dữ liệu sớm thông qua `flush()` chỉ đẩy các lệnh `UPDATE` tới máy chủ cơ sở dữ liệu, nhưng không kết thúc giao dịch vật lý. Khóa cấp dòng sẽ không được giải phóng cho đến khi có lời gọi thực thi `COMMIT` hoặc `ROLLBACK`.

### Khai báo phương thức ngắn để thả khóa nhanh

Kết thúc phương thức của `Repository` không đóng lại chu kỳ khóa nếu giao dịch thực sự nằm ở lớp `Service` đang được bao phủ bởi `@Transactional`. Giao dịch chỉ khép lại khi thoát khỏi đối tượng Proxy ngoài cùng.

### Tăng dung lượng Connection Pool

Tăng lượng kết nối tối đa chỉ đơn giản che đậy sự cố cạn kiệt hàng đợi, nhưng không thu nhỏ được ranh giới thời gian (lock lifetime) của khóa độc quyền. Sự việc sẽ ngày càng trầm trọng do tăng số luồng (waiters) đè nén lên cùng một điểm nghẽn, dẫn đến hao tổn CPU cơ sở dữ liệu.

### Thay bằng Khóa toàn bảng (Table lock)

Ngăn chặn (serialize) sự đồng thời của toàn bộ bảng (tenant_quota) cho một chức năng cập nhật (row-level operation) làm ngưng trệ vô cớ mọi người dùng và dịch vụ khác, phá vỡ khả năng xử lý tương tranh ở mức thiết kế ứng dụng. Phạm vi bảo vệ (lock scope) phải được thiết kế tập trung chính xác vào tài nguyên chịu xung đột.
