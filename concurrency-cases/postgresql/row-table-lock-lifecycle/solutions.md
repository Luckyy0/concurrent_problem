# Bài Thuốc Chữa Tắc: Ép Khóa Bạo Bệnh (Pessimistic), Mũi Tiêm Nguyên Tử (Atomic) Và Thả Lỏng Mộng Mơ (Optimistic)

## 1. Mạch Máu Của Thiết Kế (Mục tiêu thiết kế)

Bốc Thuốc Trị Đụng Độ (conflict mechanism) Cứ Nã Vạch Định Hướng (invariant) Ra Mà Phán:

```text
Tay cầm dao (Writer) có thực sự cần đè ngửa độc quyền bàn mổ lúc quyết định số phận hiện tại?
Đường dao thi triển (Operation) có thể gói gọn bọc trong câu lách luật điều kiện (conditional SQL) không?
Lúc đập nhau, có nên vứt mẹ dao báo lỗi (fail/retry) thay vì bắt thằng khác đứng ngáp đợi (block)?
Thằng khán giả rảnh háng (Observer) có nhất thiết phải gồng mình đòi đấm ổ khóa để đứng nhìn không (locking read)?
```

## 2. Kê Toa 1 — Tung Lệnh Sắt Máu Tay Bo `PESSIMISTIC_WRITE` (Solution 1 — Explicit `PESSIMISTIC_WRITE`)

Chia Hẳn 1 Tòa Riêng (Repository) Chỉ Phục Vụ Trảm Quyết Chặt Chém Dữ Liệu (mutation):

```java
public interface TenantQuotaRepository
    extends JpaRepository<TenantQuota, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        select q
          from TenantQuota q
         where q.tenantId = :tenantId
        """)
    Optional<TenantQuota> findForUpdate(UUID tenantId);
}
```

Ở Lò Thợ Khách (Service):

```java
@Service
public class LockedQuotaAdministrationService {
    private final TenantQuotaRepository quotas;
    private final QuotaRules rules;

    public LockedQuotaAdministrationService(
        TenantQuotaRepository quotas,
        QuotaRules rules
    ) {
        this.quotas = quotas;
        this.rules = rules;
    }

    @Transactional
    public QuotaChangeResult changeQuota(
        UUID tenantId,
        int requestedQuota
    ) {
        TenantQuota quota = quotas.findForUpdate(tenantId)
            .orElseThrow(QuotaNotFoundException::new);

        rules.validateTransition(quota.getQuota(), requestedQuota);
        quota.changeQuota(requestedQuota);
        quotas.flush();
        return QuotaChangeResult.changed(requestedQuota);
    }
}
```

Phép Màu Của Lò Bát Quái Hibernate/PostgreSQL Ép Phải Đẻ Ra Đúng Miếng Bùa Gài Khóa (locking clause) Y Xì Này Nha:

```sql
select tenant_id, quota, revision
from tenant_quota
where tenant_id=:tenantId
for update;
```

Bác A Ôm Phập Sợi Dây Khóa Vòng (row lock); Trẻ B Có Hùng Hổ Kéo Lệnh Chém Sửa Hay Khóa Cửa (incompatible writer/locking reader) Cũng Đều Phải Đứt Nhịp Nghẽn Chờ (blocks), Mỏi Hơi Chết Tắc (timeout) Hoặc Nổ Lỗi Vỡ Mặt (fails). Án Khóa Kéo Tới Tận Khi Cú Phán Xét Đập Bàn Ngoài Cùng Chốt Ván Hay Hủy Diệt (outer commit/rollback).

> **Sếp chốt lại:** Trấn Câu Query Giữ Hồn Khóa Nhá; Kén Màng Giao Dịch Xoay Vòng Nuôi Khóa; Đứt 1 Trong 2 Chân Kia Đi Thì Cửa Hầm Tử Địa Đọc-Sửa-Viết (read-modify-write critical section) Coi Như Tàn Phế Rỗng Tuếch.

### Ranh Giới Kỷ Luật Thép Cần Nhớ (Boundary requirements)

- Cú Gọi Phải Trượt Bóng Xuyên Tấm Áo Choàng Spring Proxy Mới Tính.
- Lệnh Truy Vấn Mắc Khóa Bắt Buộc Đua Chạy Trong Ruột 1 Cái Transaction Đang Còn Sống Mở (active).
- Cấm Mọi Trò Ỉm Lệnh Chờ Dây Từ Xa Gọi Ngoại Mạng (remote I/O/executor wait) Lúc Tay Đang Siết Giữ Khóa!
- Cần Bắt Nhiều Đứa Row 1 Lúc Thì Phải Nhặt Tuần Tự (deterministic key order) Nhen Đừng Bóc Lộn.
- Nhét Rọ Chết Hẹn `lock_timeout`/Đo Tổng Phút Án Sống (overall deadline) Kín Mõm Hết Ngay.
- Nắm Trúng Đáy Ổ Khóa Xong Là Nhớ Mở Đèn Pin Soi Dò Khám Nghiệm Tái Sinh Xét Số Gấp (Revalidate state) Kịp Nhen.

## 3. Kê Toa 2 — Cộc Lốc Lệnh Trơn `FOR UPDATE` Mắc Khắc Số Hẹn Chết Líp (Solution 2 — Native `FOR UPDATE` với timeout rõ)

Lúc Cái Bàn Thông Ngữ Dialect Hay Hint Hết Phép Đành Xắn Tay Áo Dọng Cốt Tay (explicit):

```java
@Repository
public class QuotaLocks {
    private final JdbcTemplate jdbc;

    public QuotaLocks(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public QuotaSnapshot lock(UUID tenantId) {
        return jdbc.queryForObject(
            """
            select tenant_id, quota, revision
            from tenant_quota
            where tenant_id=?
            for update
            """,
            (rs, rowNum) -> new QuotaSnapshot(
                rs.getObject("tenant_id", UUID.class),
                rs.getInt("quota"),
                rs.getLong("revision")
            ),
            tenantId
        );
    }
}
```

Giới Hạn Bức Kép Ép Mõm (Per transaction):

```sql
set local lock_timeout = '500ms';
```

Cấm Tiệt Cái Kiểu Đúc Khuôn Cháy Đóng Binh (hard-code) Nguyên Bãi Timeout 1 Cỡ Đi Trùm Quàng Sạch Từng Thằng Workload Nhé; Ráp Chuẩn Ướm Lọt Trọn Khe Đáy Hạn Trót Deadline Đuôi Tới Đầu Nhé (end-to-end). Tín Hiệu Nổi Mốc Xù SQLSTATE `55P03` Quy Đuổi Sang Đội Tắc Thở Timeout Rảnh Rỗi Bật Mồi (retryable busy/timeout), CHỨ ÉO THỂ NÀO Tráo Mù Sang Ngõ Cụt Vắng Khách (NOT_FOUND) Được Nhé!

## 4. Kê Toa 3 — Cửa Súng Thép Bắn Trúng Tâm 1 Phát (Solution 3 — Atomic conditional SQL)

Nếu Mớ Luật Chơi Phức Tạp Lại Lọt Thỏm Đủ Ép Nhồi Dưới Háng Mệnh Đề Đuôi (predicate), Rũ Phạch Sạch Quét Đáy Cửa Khởi Đầu Lệnh Chờ Dọc Đo (pre-read/long lock window) Gỡ Trút Mệt Óc Nhé:

```sql
update tenant_quota
set quota = :requestedQuota,
    revision = revision + 1
where tenant_id = :tenantId
  and revision = :expectedRevision
  and :requestedQuota >= 0;
```

Bọn Đất Repository Liệng Bắn Phản Hồi Về Trúng Chỉ Tiêu Xác Chết Rụng Bị Cắt Cứa (affected-row):

```java
@Modifying
@Query(value = """
    update tenant_quota
       set quota=:requestedQuota,
           revision=revision+1
     where tenant_id=:tenantId
       and revision=:expectedRevision
    """, nativeQuery = true)
int changeIfRevisionMatches(
    UUID tenantId,
    int requestedQuota,
    long expectedRevision
);
```

Phát Súng Này (UPDATE) Vẫn Bung Gương Bám Trọn Ô Khóa Nhé (acquires row lock). Mắc Phải Đợi Chờ Oan Gia Xáp Láp Kể Cận Ngó Sửa Đụng Mặt (concurrent writer), Thì Trùm Sò PostgreSQL Đợi Nét Mép Thằng Kẻ Nhả Lệnh Đứt Rồi Tự Bung Màn Khám Rà Khứa Current Row/Tới Háng Lưới Predicate Ngay Lập Tức Dọc Phép Tịch (command semantics); Vướng Phải Áo Cũ Rỉ Trôi (stale revision) Sinh Ngay Phép Thảy Quả Khất Ảo Trượt Không Đụng Vào Cấu (affected-row `0`). Nhớ Map Đẩy Hút Dính Trái Xoắn Giao Thù Rớt Bóng Lấy Trạm Reload Đoạt Rõ Vét Nhé Nhóc, DẠI GÌ CHẾT NÉN Đè Sửa Tẹt Cứa Xác (không overwrite).

Tốt Ánh Nhất Của Nước Cờ Này Là Nhát Ép Tuổi Thọ Xích Mỏng Ngắn Gọn Tiết Nhanh (short lock lifetime), Và Băm Đứt Đường Truông Đúng 1 Tuyến Đo Mạng (one round-trip mutation). Nhưng ÉO Ôm Bấu Nổi Nếu Quả Khám Soi Bệnh Lý (validation) Tụi Nhóc Quá Chồng Chéo Cấu Hút Dính Làng Dòng Rẽ Kép Side Effects Đâu Nhé.

## 5. Kê Toa 4 — Mượn Áo Niềm Tin Lõm Buông Chướng `@Version` Đỡ Gãy Hơn Tắc Đứng (Solution 4 — `@Version` thay blocking)

Mốc Đầu Bức Thực Thể (Entity):

```java
@Version
@Column(nullable = false)
private long revision;
```

Lệnh Hibernate Giăng:

```sql
update tenant_quota
set quota=?, revision=?
where tenant_id=? and revision=?;
```

Vỡ Xương Thụt Lòi Affected-row `0` -> Bắn Thẳng Mác Kéo Lỗi Óc Tưởng Đợi Bị Nhảy Oan Tranh Lệ (optimistic lock exception). Trẻ B Không Bị Lôi Trói Chờ Nắn Bóp Tắc Nén Phán Xét Đâu (không block) Mà Thua Phủi Đít Té Đi Thử Sức Trượt Khác Nhé Ngài Bại Tướng (loser fail/retry). Trò Chơi Cuộn Lộn Lại Retry Muốn Yên Lành Xương Thẳng Cấp Phải Khoanh Khép Án (bounded), Biết Trễ Nhịp Đo Quăng Run Tay (backoff/jitter), Tạo Kéo Rạch Giao Dịch MỚI Cứng Tươi Cóp Vét (transaction mới) Kéo Reload Nhá Sạch Nhé. Phải Kẹp Thận Trọng Vớ Vòng Ác Đứa Nhấp Háo Nóng (Hot row) Đẻ Bão Chập Cuộn Lại (retry amplification) Chết Dở Nhen.

Liều Máu Khóa Gông Tịch Bi Quan Pessimistic Nuôi Kép Đẹp Bền Nếu Loạn Chạm Xảy Nhau Ốc Đám Bọt Giao Tranh Cháy Liên Khúc Thường Xuyên (conflict thường xuyên) Chắn Critical Ngắn Trọn Phút Giữa Cửa Xé Gấp; Áo Lưới Optimistic Kéo Cửa Vui Ướp Nhan Đẹp Rõ Đứt Xui Trầy Trật Hiếm Chút Hỏng Gọn Mép Caller Nuốt Lực Nuốt Cú Oác Nhan (caller chịu retry/reject).

## 6. Kê Toa 5 — Chỗ Bảng Chờ Xem (Dashboard) Nên Yên Phận Xem Chay Thôi (Solution 5 — Dashboard giữ plain SELECT)

```java
@Transactional(readOnly = true)
public QuotaView dashboard(UUID tenantId) {
    return quotas.findViewByTenantId(tenantId)
        .orElseThrow(QuotaNotFoundException::new);
}
```

Giấy Ký Nhận Kép Hứa Mõm Giao Án Nè (Contract):

```text
Chỉ Trích Nhả Ói Khứ Dữ Vừa Đóng Chốt (last committed quota) Trình Xa Nhìn Statement Snapshot Rực Áo Lên Rạp Màn Thôi Nhen Nhóc
Tuyệt Mệnh KHÔNG Vén Tranh Rẽ Oai Trưng Kép Nháp Áo Thằng Uncommitted Nào Điên Phọt Rỉ Đưa Lên Bảng Trái Khung Trát!
ĐÉO Mắc Khờ Đứng Sững Hóc Trọng Rơi Chờ Mỏi Trĩ Thối Trơ Lỳ Vì Có Đứa Kẻ Khác Mắc Cuộn Vây Khóa Row Nhé! 
```

Nếu Bảng Lò Có Ánh Quái Bức Mắt Phải Bục Đi Vớt Rõ Soi Cuộn Mới Méo Điền Test Vành Ngõ Đọc-Đòi-Sau-Sửa (read-after-write), Thì Tuyến Rạch Cuộn Kế Gắn Cửa/Transaction/Phiếu Version (route/transaction/version token semantics) Buộc Tay Viết Lệnh Thiết Phác Cửa Đẹp Riêng Biệt Ra Nhen; Đi Đội Hút Tịch Khóa Oái Chắn Sập Gãy Sạch Toàn Khán Giả Xem Trận Không Bao Giờ Là Vết Mặc Định Đi Đáy (not default) Đâu Đút!

## 7. Kê Toa 6 — Quất Gây Khóa Đứt Cả Bảng (Table Lock) Khi Gốc Ánh Hút Chốt Xóa Mọi Mép Cứng Nguyên Table Đi Đánh Sóng Kép (Solution 6 — Explicit table lock chỉ khi scope thật sự là table)

```sql
begin;
lock table tenant_quota in share row exclusive mode;
-- Đội Khép Đoạt Ráp Nghề Chỉ Tốn Véo Chút Lệ Hẹp Y Đúc Đúng Mode Compatibility Này
commit;
```

Bấm Mạch Chọn Chuẩn Từng Đinh Ốc Cái Mode Trúng Quả Y Chang Từ Đáy Bảng PostgreSQL Compatibility Matrix Á Nhé Sếp. Cây Chém Đứt Làng `ACCESS EXCLUSIVE` Liệt Án Trảm Tịch Giết Khóc Trọn Bọn Lượn Hóng Chay (plain SELECT) Thường Ăn Vào Dòng Ráp Kế Lệnh Hỏa Mù Thép Trống Sửa Đồ Hình (DDL) Hay Kiểu Maintenance Trấn Siết Giữ Nhà (very strong maintenance), ÉO AI Rãnh Đi Ốp Khép Ánh Trọn Table Cho Nhõn Trò Chạm Đổi 1 Đứa Rìa Tenant Nhé Ngớ Ngẩn!

Treo Án Cục Table Locks Đu Đáy Oan Nè:

- Quật Thọt Chết Lây Cả Những Dòng Vô Tội Nhấp Trượt Rìa Rành Rẽ (unrelated rows);
- Phì Mập Tức Trướng Tắc Ngáp Sập Cửa Tịt Ngõ Dính Quái Đạn Tử Thắt (tăng wait/deadlock risk);
- Kéo Ộc Dai Trối Đến Trọn Tuổi Giao Tuyến Ép Xuống Hàng Sinh (transaction end);
- Cấm Trái Nhịp Lộn Áo Trượt Rách Buộc Sút Bát Thòng Thứ Tự Định Mạng Chuẩn Từ Đầu Đuôi (deterministic table order).

## 8. Dịch Gói Báo Tắt Hết Đạn Đo Khóa Đi Kế Tịch (Lock timeout mapping)

```java
try {
    return lockedAttempt.changeQuota(command);
} catch (CannotAcquireLockException ex) {
    return QuotaChangeResult.busyRetryable();
}
```

Bắt Lỗ Sàn Quét Khía Định Rõ Đục Nghẽn Nắn Tìm Cause Tội Rành Nhen:

- `55P03` Tắc Kẽ Lưới Rụng Trắng Vãi Chút Đoạn Timeout Không Chạm Hưởng Lock Giữ Lấy (lock not available/timeout);
- `40P01` Gãy Cổ Thở Oxi Oan Khứa Chết Thay Cắn Đuôi Tử Vòng Sòng (deadlock victim);
- Cứt Rác Chạy Chờ Nhíu Lưới Đoản Lệnh Giây Đi Đứt (query/statement timeout);
- Tịt Ngáp Phịch Áp Kết Nối Hồ Pool Cháy Đội Lụi Thở (connection acquisition timeout).

Giao Tuyến Quấn Gấp Thằng Gãy Liền Đập Dội Rollback Tụt Hết Trước Khi Kéo Án Lôi Re-try Nhé Nhóc! Tay Giật Trấn Gọi (Outer coordinator) Mới Tái Cắm Áp Đưa Cục Attempt Bean Mới Thụt Kéo Cửa Vào Đỉnh Sinh; CHỨ CẤM Có Nhồi Điên Dọng Lọc Níu Lại Catch Nửa Vời Rẽ Nhét Vục Quất Đi Xé Ác Cửa Mạch Đang Gãy Nứt Ở Đít Dưới Cục Đất Oan Nhen Mấy Ngài (Không catch rồi tiếp tục JPA work trong same transaction).

## 9. Hình Tội Hiện Diễn Cái Rủi (Failure behavior)

- Đứa Nắm Trụ Đỉnh Tung Lệnh Đập Hàng Commit (Holder commit): Cởi Khóa Xích Kép Xõa (lock release), Khứa Chờ Dựng Vóc Lại Vút Áo Current State Đi Tiếp Oai Ngực.
- Đứa Nắm Trụ Tự Tử Đẩy Hủy Rollback Hoặc Sập Ác Oan Gãy Điện (Holder rollback/crash): Trôi Đất Bể Trắng Đổi Trượt Vụt, Khứa Waiter Móc Áp Dưới Bám Bóng Trạng Cũ Đỉnh Rìa (prior state) Đứng Đi Tụt Lên Chép Đoạn Báo Ối Cũ Đỉnh Chấp Lại Tức Tròng Ộc Bình Bình Á!
- Bác Lão Waiter Đứng Trĩ Tắt Hẹn Giờ Ép Oải Gây Bọn (Waiter timeout): Án Sập Giao Đứt Chỉ Thằng Đứng Ngáp Đợi Đó Thôi (waiter transaction rollback); Kẻ Nắm Trụ Holder Nhăn Lên Cười Oai Trấn Chả Vướng Tí Lông Chó Nào Cả Vây Rìa Dính Đáy Hưởng Ác Cả Cút Ụp Nhé! (holder unaffected).
- Lệnh Tử Gãy Đuôi Dính Dây Tròng Khóa Kẹp Đóng Án (Deadlock victim): Cả Bộ Đội Ảo Hiện Active Phía Thằng Chết Bị Đánh Chặn (whole victim transaction rollback).
- Ống Kép Dịch Ảo Giữa Client Cúp Điện Rớt Hơi Cổ Ngó Timeout (Client timeout): Dấu Chốt Mệnh Có Bị Chút Nhấp Nhoáng Cháy Ngơ (ambiguous); Phải Lấy Mõm Rắn Bị Đè Trụ Vén Lụi Tục Xét Dòm Xoi Tích Bọc Giáp Mốc Revision Dai Rìa Sóng Dấu Chạm Replay.
- Đám Sóng Độc Nhái Kép Cệnh Lặp Đè Command Dội Sóng (Duplicate command): Lưới Phù Bùa Đẩy Hút Sóng Idempotency Kéo Đứt Thả Sạch Rải Dây Contract Nhé, Cấm Phọt Chút Khóa Khờ Đi Giải Thất Nhảm Vướng Lỗi Tích (không giải bằng row lock alone).

## 10. Chốt Mâm Vạch Kẻ Cân Kép Lãi Thiệt Đọa (Trade-off comparison)

| Miếng Khẩu Võ (Cách) | Hành Xử Va Chạm Lửa Đứt Đụng Mặt Nhau (Conflict behavior) | Thời Tốc Móc Xem Ngó (Read latency) | Nắm Chặn Tạp Sửa Bạo (Write contention) | Ngã Bọn Tung Máy Chia Trại Áp Dây Độc (Multi-instance) |
| --- | --- | --- | --- | --- |
| Lôi `FOR UPDATE` Trấn Khóa | Bại Tướng Đứng Ngáp Đứt Kìm Hơi Chờ (loser blocks/timeout) | Sói Khóa Đứng Sững Hóc Trọng Chờ Ộp (Locking readers wait) | Tụ Ép Rụng Lực Nóng Áp Trói Nẹt Cạnh (Serialized hot row) | Có Luôn Đỉnh |
| Gài Số Cứng Mệnh Đề Gọn (Atomic conditional SQL) | Thụt Sút Dội Oai Khống Dọc `0` Trượt Hoặc Trực Bắn (affected-row `0`/short wait) | Nhẹ Rìa Không Ôm Nợ Hơi Đầu (Không pre-read) | Đục Kép Tiết Dây Véo Ngắn (Short) | Dọn Dễ Hết Ngay (Có) |
| Ánh Bóng Chút Tự Sửa Lấp `@Version` | Tên Lủi Kéo Ọc Đoạn Chướng Đuổi Đít Té Đi Nhé Kéo Thử Vòng (loser fails/retries) | Trơn Trượt Đọc Nhẹ Hều Đọc Phẳng (Plain reads) | Bể Khắc Ngóc Lại Oan Khi Tịt Ngáp Phịch Tranh Đứt Nhấn (Retry under conflict) | Kéo Kính Ồ (Có) |
| Kéo Giả Dashboard Nhép Plain Xem | Đéo Ai Rãnh Đi Nghẹn Lệnh Chờ Row (no row-lock wait) | Quệt Trôi Thơm Máy Cực Vụt (Low, committed-stale) | Chút Hỏng Đừng Chặn Che Dòm Nắn Viết Nhé Oai Nhé (Không protect writer) | Đứa Mò Xem Rạch Đáy Rớt Móc Thôi Nhé (Observer only) |
| Chặn Đuôi Table Làng Table | Quét Áp Dài Độc Chốt Nín Thở Dày Áo (broad block/timeout) | Tùy Sóng Trọng Mode Móc Khóa Mode Gì Chướng Đâm Đuôi (Mode-dependent) | Khóa Lấp Tịt Tướng Khét Nhất Cản Lấp Tịt (Broad) | Có Áo Kín |
| Móc Sóng Dây Mạng App JVM Kéo Bệnh (JVM lock) | Kìm Đất Trấn App Giữ Áo Ở Trong Nhà JVM Kép Nhé (local wait) | Rọt Cửa Ở Khúc Quật Nhà Tranh Mình JVM Kép Local Nhanh Kéo Xéo Local | ÉO Ngóng Mỏi Đâm Cứa Nát Trực Mảng Giao Nhà App Khác Ầy Đút Đầu Ngó Đít (Không cross-node) | Oẳng Nhé Ạ ĐÉO Có Nhá Bịp Sóng |

## 11. Áo Quyết Tâm Phạt Truy Lệnh Gắn Mác (Recommendation)

Cú Đập Vá Thay Máu Sửa Nắn Quota Cần Rãnh Đục Chọc Kiểm Tra Hiện Thực Sống Phức Tạp Rờn Hiện State (complex current-state validation): Phá Chốt Lệnh Trói `PESSIMISTIC_WRITE` Ngắt Đứt Khúc Cụt Bọc (short transaction).
Ốp Sửa Mộc Mạc Dạo Cờ Condition Số Hiện Hạn Cứ Cọ Khớp Lỗ Đinh Predicate Revision (Simple revision/predicate mutation): Quất Chắn Giải Pháp SQL Đính Áp Sóng Bảng Atomic Conditional Kéo Dọc Phép Hoặc Trải Ụp Quật `@Version`.
Khán Bảng Xem Giương Trái Plain Đọc Trơn Sụp Đáy Kéo (dashboard). Table Trùm Làng Explicit Table Khóa Siết Xéo Nguyên Gốc Cứ Rạp Chỉ Dùng Áo Lúc Kẹp Vênh Oai Nhất Thật Sự Cắn Trọn Dập Mạng Tích Đều Toàn Đám (invariant thật sự table-wide).

## 12. Tờ Phạt Kẹp Soát Trực Chiến Giữ Gạch Đầu Độc Săn (Production checklist)

- [ ] Lệnh SQL Tự Trồi Lên Theo Bảng Độc Đúng Cái Bùa Khóa Lock Mode Ngắm Súng Chuẩn (intended lock mode) Đã Bị Vén Rõ Dòm Kiểm Soát Vết Dấu Sạch.
- [ ] Truy Lệnh Đọc Khóa Phải Giãy Đập Rành Trong Ruột Vòng Giao Dịch Đang Sống Cựa Mạch Áp Che Áp Của Thằng Sinh Proxy (active proxy-created transaction).
- [ ] Ranh Giới Kép Ngắm Ô Khóa Đoạt Hàng Dấu Cuộn Vòng Đạp (acquisition) Cùng Đít Cắt Hủy (commit/rollback) Vạch Tịt Hiện Rõ Trực Bắn Cứ Trong Mã Lệnh Kéo Cửa (code).
- [ ] Nghiêm Cấm Kéo Bóng Đi Chầu Mỏi Gối Chờ Nặng Gánh Bọn API Trạm Remote Kéo Dọc Chóp I/O Sau Đít Giữ Bọc Khóa Đất.
- [ ] Túm Mớ Dòng Hàng Dây Chặt Đống Khóa (Multiple resources) Trúng Tích Luật Khớp Bắn Chóp Định Chốt Theo List Đơn Trật Tự Dựng Chuẩn Rõ Kéo Cắm Trụ Thứ Tự Xương Dọc Lập Áo Lệnh Theo Xác (deterministic order).
- [ ] Đồng Hồ Bom Chờ Khóa Oác `lock_timeout` Buộc Lấp Nằm Vừa Trong Khe Khung Hộp Níu Khép Chết Sinh Tổng Thể (overall deadline).
- [ ] Thuốc Đứt Oải Thở Timeout/Đứt Cuộc Lộn Vòng Ngược Deadlock Retry Áp Rõ Trút Buộc Mượn Tay Tung Kéo Transaction Vòng Sinh Mới (transaction mới).
- [ ] Bọn Sáng Chay Không Thèm Độc Xem Rẻ Trơn Thường (Plain readers) Ôm Cáo Tròn Máng Hứa Khớp Cho Giao Tuyến Nhan Lệ Committed-staleness.
- [ ] Treo Đèn Theo Dõi `pg_blocking_pids`, Lập Cửa Án Khóa Đợi Mỏi Đít Kéo Wait Lượt Trục Tích Tuổi Transaction Chày Trọn Đo Dọc Mũi (transaction age).
- [ ] Bãi Test Khởi Rào Buộc Dọng Thuật Sống Trại Khởi Thực (PostgreSQL thật) Lẫn Chốt Mệnh Kẹp Cắn Buộc Trọng Hẹn Đồng Hồ Giờ Đứt Khống Bound (bounded coordination).
