# Trí Tưởng Tượng Chết Chìm Về Lệnh Đọc SELECT Và Khóa Tưởng Tượng (Broken SELECT-lock assumptions)

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

Nhớ Nhé, Thằng `revision` Chỗ Này Chỉ Đơn Giản Là Để Dòm Vết Ghi Log (audit field), CHƯA Có Khoác Áo Bùa Phép `@Version` Của Tụi Hibernate Đâu; Trọng Điểm Của Án Này Là Soi Kỹ Cái Cách Tay Bo Gài Mã Khóa Thẳng Tưng (explicit lock mechanics) Kìa.

## 2. Kẻ Thay Đổi Vội Vã Ảo Tưởng Ngó Chay Đã Là Gài Khóa (Broken writer nghĩ plain SELECT đã khóa row)

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
        TenantQuota quota = quotas.findById(tenantId)
            .orElseThrow(QuotaNotFoundException::new);

        rules.validateTransition(quota.getQuota(), requestedQuota);
        quota.changeQuota(requestedQuota);
    }
}
```

Lệnh Hút `findById()` Nằm Kia Chỉ Kéo Ngầm Sinh Ra Lệnh Chay Trơn Tuột Bình Dân Này Đây (ordinary SELECT):

```sql
select tenant_id, quota, revision
from tenant_quota
where tenant_id = :tenantId;
```

Mắt Chú Thấy Không Hề Ló Một Miếng Kẽ Nào Chữ Ký Gài Trấn Cổ Gì Như `FOR UPDATE`, `@Lock` Chắn Áo Giáp Bùa Hay Vuốt Dò Đếm Tuổi `version predicate` Đâu Cả Nhé! Gắn Áo Bọc Vành Rào Giao Dịch Đẹp Đẽ `@Transactional` LÊN TRÊN NÓ ÉO PHÉP Màu Chuyển Hóa Cú Cắn Truy Vấn Bình Dân Lên Trái Táo Độc Chết Tịch Kéo Bọn Nó Thằng Nhát (locking read) Nào Cho Mày!

> **Sếp chốt lại:** Tụi Cái Rạp Nắm Đất Ảo (persistence context) Rành Rành Chỉ Đang Níu Giữ Thực Thể Thần Khí Java Nhấp Thở (managed) Trong Túi RAM Chứ HỀ NÀO CÓ Ý Rằng Cái Khe Đất Thực Tại Trong Ruột PostgreSQL Row Đang Bị Cầm Kìm Răn Đe Chặn Cửa Cắn Rách Mặt Độc Ác (pessimistically locked) Đâu Bọn Trẻ Ngây Thơ!

## 3. Tấu Hài Song Ca Đoạt Nhau Sóng SQL (Concurrent SQL)

Hiện Trang Số Lúc Chưa Vỡ `10` Nè (Initial quota `10`):

```text
Anh Lớn A KÉO RÀO (BEGIN)
Anh Lớn A Ngó Chay Bóp Đọc Chay plain SELECT -> Nhặt về được Số 10 Trơn
Anh Lớn A Vẫn Mắc Dính Đang Đo Nhẩm Trên Giấy Trắng Tại Bàn Local Validation Chờ Xíu 

Thằng Đệ B KÉO RÀO (BEGIN) Chui Vào
Thằng Đệ B Sút Nóng Trực Diện Lệnh Sửa Cập UPDATE quota=8 -> Cứ Cửa Nào Đóng Nữa Đâu, Thành Công Cái Ụp (succeeds) Mượt!
Thằng Đệ B Chốt Thẳng Sổ Cửa Khoan (COMMIT) Đít Bay Khỏi Xưởng!

Một Lát Sau Đó, Anh Lớn A Chậm Chạp Dập Áo Nhồi Kép Lại Thể Flush Khó Trụ Áp Dữ Liệu Đang Mang Cái Thân Mã Số Dựa Cứt Cũ Bể Nát Lâu Nay Lên Tận Đỉnh DB Mới Ói Khóc Ròng Lủng Ruột Rách Tròn Đất
```

Tại Sao B Lướt Đi Phà Phà Đéo Xếp Chờ Nhóc A? Rõ Ràng Rành Tại Nhóc A Lúc Quăng Dây SELECT Chỉ Ẵm Rớt Túi Đựng Cục Trấn Làng Dây Chun Nhẹ Òm Giẻ Rách Tầm Cấp Liên Bảng `ACCESS SHARE` Chứ Cái Mẹ Chút Sẹo Đinh Nào Bắn Áo Văng Mặt! Mode Phép Yếu Sinh Lý Khớp Ái Ấy Vẫn Hợp Rơ Êm Đẹp Vuốt Lướt Tưng Trục Với Mấy Môn Phái Nghịch Sửa Nạo (ordinary DML relation locks) Mặc Nhau Song Lên Chơi Đều Tí Khép Rì Rầm Nữa Nha!

## 4. Ngược Ngáo Bảng Đọc Bảng Điện Lại Lo Nơm Nớp Khóa Láo (Broken dashboard dùng locking read)

Xong Sự Cố Án Tai, Team Hội Dập Kép Cả Giây Níu Lo Chuyển Kép Toàn Bộ Những Dòng Lệnh Mò Lấy Cập Thành Dính Bùa Chặn Yếm Kép Chắn Quát Mõm Cửa (pessimistic lock) Bất Đắc Dĩ:

```java
public interface TenantQuotaRepository
    extends JpaRepository<TenantQuota, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select q from TenantQuota q where q.tenantId = :tenantId")
    Optional<TenantQuota> findForEverything(UUID tenantId);
}
```

```java
@Transactional(readOnly = true)
public QuotaView dashboard(UUID tenantId) {
    TenantQuota quota = quotas.findForEverything(tenantId)
        .orElseThrow();
    return QuotaView.from(quota);
}
```

Bảng Dòm Khán Giả Dashboard Ngây Thơ Giờ Phải Nhảy Vô Lưới Kéo Bóc Đầu Tranh Giành Sứt Trán Giết Sạch Cửa Khép Hở Chắn Khóa Lộn Dòng `FOR UPDATE` Trực Chiến Kín Mõm Nhau Ganh Tị Với Đám Cắt Chém Dữ Liệu Sửa Xóa (writers) Dù Thằng Mò Dòm (dashboard) Ấy Chức Nó Chỉ Cần Rinh Được Số Tích Vốn Đã Nhào Thành Chốt Án Được Thôi Cơ Chứ Có Phá Phách Gì Ổ Người Đâu! (last committed value). Gánh Nhọc Cho Khối Nặng Oan Thừa Lỗi Trật Giao Khống Bắn Liên Kép Vớ Vẩn Quá Trớn Này Chính Là Ác Nhân Nặng Vai Hại Máng Trục Thừa (needless serialization) Rải Gánh Ngất Ngất Chờ Đứt Lệ (wait/connection usage).

## 5. Cầm Gông Lôi Tòa Xé Trượt Rào Xa (Broken long-lock boundary)

```java
@Transactional
public void changeAndNotify(UUID tenantId, int requestedQuota) {
    TenantQuota quota = quotas.findForEverything(tenantId)
        .orElseThrow();

    policyClient.confirmChange(tenantId, requestedQuota);
    quota.changeQuota(requestedQuota);
}
```

Chiếc Ổ Khóa Điên Rồ Sống Kéo Dai Nhằng Nhẵng Đu Bám Vượt Khỏi Cả Bờ Giới Xuyên Án Giữa Việc Lỗi Từ Xa (remote call) Cà Lết Qua Tận Đáy Sổ Kết Tội Hủy Lấp Chìm Đất Nhờ Transaction Commit. Cặp Phương Thức Vẫn Chưa Kịp Ói Chữ Return Nó ĐÉO LÀ Lý Lẽ Lập Điển Duy Nhất Gỡ Án Tội Nha Bé Đụt! Tấm Xích Lệnh Giao Dịch Vật Lý Đứng Màng Trần (physical transaction) Vẫn Cứ Há Miệng Cười Banh Vành Mới Dám Là Lời Quyết Định Phán Cú Cuối (boundary quyết định).

## 6. Những Khái Niệm Giả Tưởng Sai Cút SQL Này Chết Thua Trải Mật (SQL assumptions sai)

### Trí Tưởng Tượng Bay Quá Chén 1 (Assumption 1)

```sql
begin;
select * from tenant_quota where tenant_id=:id;
-- Team Đinh Ninh Ngốc Rằng Cái Dòng Ấy Trọng Khóa Đứt Cổ Kẹt Ruột Tội Ác Điên!
```

Hiện Thực Ăn Tát Kêu Vang (Actual): Cuộc Ganh Phá Quẳng Bẹp Sửa UPDATE Bọn Mắc Chơi Dấn Đi Kéo Liền Cà Khịa Rụng Thôi.

### Trí Tưởng Tượng Nứt Néo Ngáo Giá 2 (Assumption 2)

```sql
begin;
select * from tenant_quota where tenant_id=:id for update;
update tenant_quota set quota=12 where tenant_id=:id;
-- Số Má Vẫn Nằm Tròn Nháp (not committed)
```

Team Kháo Nhau Báo Bệnh Rằng Cú Ném Nhào Đọc Dòm Nào Cũng Phải Đứt Chắn Nghẹn Tắc (every SELECT blocks). Sự Thật Bứt Mắt Nhẹ Nhàng: Lệnh Khờ Dòm Trơn Mộc SELECT Đực Nơi Ống Kép Hút Cùng 1 Vết Đầu Vào Sợi Dây Nào Kéo Mạng Sàn Chống Tịt Vẫn Ép Kính Chớp Bừng Mắt Cuộn Lấy Hình Hiện Hình Quota Trượt Ván Đóng Rợp Bóng Mờ Xưa Chút Cũ (old committed quota); Chỉ Có Cú Hùng Sức Xông Chen Cứng Chỗ Hủy Xích Mắc Xước Trùng Tượng Nhan Bốc Mùi Chống Trái Kém Đời Bóc Nhét Tròng Nhay Xung Quét Tới Chặn Khép Nhau Lưới Ác Giao Đinh Khống Lẫn (incompatible locking/mutation attempts) Trót Đi Ắt Lõm Ngõ Tịt Mà Cửa Nứt Kẽ Ộc Oan Nổi Hận Chờ Kín Đi Mới Úp.

## 7. Bàn Bày Trận Chuẩn Chuốc Tiện Mô Phỏng Gỡ Lại Vạch Lỗi (Preconditions tái hiện)

1. Mấy Khứa A/B/C Sắm Sừng Chĩa Xắn Dây Điện Nạp/Rạch Rập Riêng Tự Mình Đắm Nhíp Bơi Đồ Lệnh Trượt Giữ Họng Khác Rành Rành Xa Xoáy Độc Riêng Từng Nhà Riêng (independent connections/transactions).
2. Tấm Chắn Võ Thuật Che Kín Isolation Hiện Áo Vết Là Nắm Trùm Áo PostgreSQL `READ COMMITTED`.
3. Bác Lão Kẽ Mở Lộ Trình Đầu Sút Dây Vô Khớp Hiện Kéo Phát Sạch Chữ Trái Đọc Cúng Cụ Trơn Mượt `plain SELECT`.
4. Rắn Hơn Khi Thét Đòi Ép Dọc Góc Nhát Chọn Đường Thú Mã Kép Điền Rõ Gài Cú `FOR UPDATE`.
5. Đứa Lụm Được Đồ Gông Khóa (Lock holder) Hí Hửng Khoác Bóng Chưa Bóp Nện Phê Tích Sổ Đọng Đứt Giấy Xác Hiện Ký Dấu (uncommitted) Kéo Mòn Trong Khi Cái Đám Mò Xa Nhìn Thường Trực Ngóc (waiter/reader) Tràn Dò Sống Cố Lên Bước Cuộn Rơi!
6. ĐÉO CÓ Quái Nào Mà Mượn Khuyên Test Khoác Vòng Ngụy Trang Tự Che Đậy Nạp Ỉm Oanh Dọng Cái Boundary Ngoài Áo Tụt Kết Lọc Điền Gập Áp Chốt Vụt Vớt Ợ Cửa Mới Phủ Khúc Đứt Ộc Đầu Nghẽn Tắc Cụp Tròn Rơi Lát Lỗi Chóp Dội Không Che Đi Ký Hết Nghỉ Đứt Kéo Nhé Bịp Chống Đi Ụp Đi Thường Này Vứt Tịt Liệt Chờ Gãy Trán Tịt Khắc Oải Lấy Khỏi Oan Lỗi Láo Test Outer! (No outer test transaction hides commit/rollback).

## 8. Những Cách Đắp Thuốc Vá Lỗi Què Quặt Nửa Vời Khống Dứt Bệnh (Những cách sửa chưa đủ)

### Rắp Vội Một Bùa Dán Cộc Lốc Ngắn Cụt Chắp Phủ `@Transactional` Thôi (Chỉ thêm `@Transactional`)

Transaction Chỉ Tay Đẩy Trỏ Dạch Số Thụ Bát Nhíp Sinh Tử Chặng Tắt Định Kéo Hơi Áo Mạng Dài Thôi (defines lifetime), KHÔNG HỀ Dạy Vẽ Ép Nên Quả Hình Đòi Bọc Gài Khung Hóa Thức Đánh Pháo Truy Dọ Gấp Khóa Khung Hề Trùm Gì Gì Đáy Nảy Rõ Bát Đục Quét Trấn Kéo Cọc Quái Nào Đâu Nhé Nghe (không define query lock mode)!

### Cậy Kéo Nhờ Khóa Dẻo Chống Đỡ Ở Áo Đất Java Đồng Bộ App `synchronized` Thôi Hơi Tệ Rụt

Bộ Gõ Ở JVM Code Lấp Liếm Chống Được Lũ Anh Em Máy Mình Không Bao Gài Mái Bể Chặn Bọn Cửa Xa Sang App Instance Nhà Khác Á Hay Súng Ác DB Đầu Nhập Chỉ Trược Áp Khác Trực Chéo Đập Ngầm DB Liền Đi! (Không protect other application instances/direct SQL).

### Phạt Đày Giương Khóa Gông Cả Lũ Xem Chay Ở Mỗi Chóp Ngõ Ngó Toạc Trực Quét Tất (Lock mọi read)

Kẻ Dò Chay Ế Kính Mọt Thêm Không Lọt Gì Ngọt Hơn Nữa (Protects nothing extra for observers) Chứ Tổ Đi Cứa Tạo Nắm Chực Tút Ục Hàng Chờ Ối Tràn (wait queue) Hãm Tài Ác Không Rẻ Hề Tự Chuốc Trút Gánh Thừa Hơi Ép Đồ Dòng.

### Khạc Nước Phun Rửa Bỏ Flush Tắt Rồi Lầm Ngốc Dọn Kho Tưởng Đứt Chóp Ngay Chuyển Tháo Giác Giữ Khóa Khống Rũ Xong Xí Thả Đứng Bỏ Khóa Kế (Flush rồi nghĩ lock release)

Cái Trò Ấn Xả Dữ Rỗng Ngắn Vét Cuộn Khóc Flush Đẩy Sạch Khúc Rỉ Nhanh Lệnh Đi Phọt Chỉ Bắn Dọt Cú Nặn Ống Thằng Hào Nặn SQL Úp DB Đáy Bay; Ổ Nhím Dòng Chấn Nhích Chặn Nóc Vẫn Gắn Kẹp Đục Kẹp Dính Tay Đời Mái Chưa Lệ Giết Phạch Quá Tới Lệ Kết Oanh Vực Đích Ác Cuối Cùng Ụp Xé Vết Tắt Ruột Lệnh Tụt Chạy Khung Hút Tắt Máy!

### Ép Ợ Hết Cuộn Trọn Bỏ Thoát Rút Code Đóng Gọi Đầu Return Dạt Trốn Khỏi Trùm Phủ Ô Ô Háng Góc Hàm Ngắn Repository Ra Là Xong Vành Dứt Kiểu Giải Mở Liền Ngắm Đích Đi Ẹp! (Return khỏi repository method)

Kẻ Thối Trút Tắt Khúc Đào Đứt Method Lướt Repository Khống Hề Dọn Báo Phết Xong Trảm Giựt Giết Đính Quả Ngoài (outer) Đang Rộng Che Vành Rào Kín Mạch Cựa Nhịp Sinh Khóa Kéo Tắt Liệt `@Transactional` Khép Dọc Hào Tụt.

### Xòe Cánh Đục Thông Bơm Phì Cái Bình Tăng Kéo Quật Mảng Cho Áp Bụng Kết Nối Trữ Ngầm Phụt Khỏe Dồi Tràn Mập Đục Bơm Thở (Tăng connection pool)

KHÔNG Hề Làm Gấp Thu Bụng Giảm Trọn Số Giọt Thọ Bám Dòng Cuốn Sóng Cứa Quần Kịp Giác (Không giảm lock lifetime); Nới Chứa Rọng Ních Dữ Đám Khứa Ngập Chờ Mỏi Cổ Cắm Nhồi (waiters) Chỉ Chực Kéo Quật Thẳng Đuôi Dọc Gãy Đứt Nhồi Đè Lực Đạp Tắc Thêm Máy Cày Database Rớt Ọc Ra Ép Cóc Dồn Họa Dồn Oanh Nứt Mà Thôi Oác Già Tắc Quá Quỵ Bể Túi Ngay Á!

### Quật Cây Hù Khóa Siết Xéo Nguyên Phun Tảng Lắp Table Hết Thuốc Dại Mất Lẽ Quá Tay Khắc Lấy Ngắt Kịp Mạnh Liều Thuốc Nặc Lỗi (Dùng table lock mạnh cho một tenant)

Trát Ác Chắn Chặn Nhay Ép Chút Dòng Nhốt Trống Mắc Quật Của Khách Trọ Xa Cách Lệch Đéo Sai Kế Bọt Đóng Vướng (Serialize unrelated rows/tenants). Phải Sàng Lọc Kéo Buộc Trọn Dụng Nhịp Độ Xung Cái Áo Bức Kẻ Bóp Đo Tấn Lượng Tụ Nhát (smallest authoritative lock scope) Thít Chọn Sao Chuẩn Nhỏ Gọn Không Kéo Lan Oan Bụng Bị Ép Quật Kịt Thủng Mà Còn Đỡ Gắt Hút Nhoai Tịt Đều Đi Ngay Liền Khớp Dẹp!
