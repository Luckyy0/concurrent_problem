# Giải Pháp Xử Lý Tranh Chấp: Khóa Bi Quan, Lệnh SQL Nguyên Tử và Khóa Lạc Quan (Pessimistic, atomic and optimistic solutions)

## 1. Nguyên tắc thiết kế cốt lõi (Design objectives)

Để xử lý xung đột đồng thời (conflict mechanism) hiệu quả, ứng dụng cần trả lời rõ ràng các câu hỏi liên quan đến đặc tả nghiệp vụ (invariant):

```text
Luồng ghi (Writer) có thực sự cần một khóa bảo vệ độc quyền tại thời điểm kiểm tra điều kiện để cập nhật dữ liệu không?
Thao tác cập nhật (Operation) có thể được gộp thành một câu truy vấn SQL nguyên tử có điều kiện (atomic conditional SQL) không?
Khi phát sinh xung đột, ứng dụng nên báo lỗi ngay/thử lại (fail/retry) hay đưa luồng cập nhật vào trạng thái chờ (block)?
Khán giả chỉ xem (Observer) có yêu cầu khóa dữ liệu (locking read) hay chỉ cần hiển thị phiên bản dữ liệu gần nhất đã commit?
```

## 2. Giải pháp 1 — Áp dụng Khóa Bi Quan `PESSIMISTIC_WRITE` (Solution 1 — Explicit `PESSIMISTIC_WRITE`)

Thiết lập một lớp Repository riêng biệt chuyên dụng cho thao tác bảo vệ dữ liệu (mutation):

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

Triển khai tại tầng Service:

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

Thông qua công cụ JPA (Hibernate), PostgreSQL nhận được câu truy vấn có thêm cờ yêu cầu khóa (locking clause):

```sql
select tenant_id, quota, revision
from tenant_quota
where tenant_id=:tenantId
for update;
```

Khi luồng A xin khóa thành công (row lock), bất kỳ luồng B nào có yêu cầu viết sửa đổi hoặc lấy khóa trên bản ghi đó (incompatible writer/locking reader) đều phải chờ (blocks), cho đến khi luồng A thực thi lệnh `COMMIT` hoặc `ROLLBACK`.

> **Ghi chú quan trọng:** Phải đảm bảo luôn sử dụng từ khóa đúng cho câu truy vấn và luôn giữ ranh giới transaction tồn tại. Thiếu bất kỳ yếu tố nào cũng sẽ khiến đoạn mã bảo vệ bị phá vỡ.

### Các ranh giới bắt buộc (Boundary requirements)

- Cú gọi từ bên ngoài phải đi qua Spring Proxy để kích hoạt Transaction.
- Lệnh truy vấn khóa (`FOR UPDATE`) phải chạy bên trong một Transaction đang mở (active).
- Nghiêm cấm đưa các lời gọi I/O từ xa (remote I/O/executor wait) vào bên trong chu trình đang nắm giữ khóa dòng (row lock).
- Yêu cầu cấp khóa đối với nhiều bản ghi cùng một lúc phải thực hiện tuần tự và nhất quán dựa trên quy luật sắp xếp (deterministic key order) để chặn deadlock vòng.
- Nên thiết lập thông số thời gian chờ `lock_timeout` hoặc một giới hạn chung (overall deadline).
- Khi có khóa thành công, phải thực hiện kiểm chứng lại dữ liệu (Revalidate state) đối với quy tắc nghiệp vụ.

## 3. Giải pháp 2 — Lệnh SQL Nguyên Tử Có Điều Kiện (Solution 2 — Atomic conditional SQL)

Nếu các quy tắc thay đổi không quá phức tạp và có thể chuyển thành các mệnh đề kiểm tra truy vấn SQL (predicate), hãy loại bỏ khoảng thời gian truy vấn chờ khóa dài (long lock window). 

```sql
update tenant_quota
set quota = :requestedQuota,
    revision = revision + 1
where tenant_id = :tenantId
  and revision = :expectedRevision
  and :requestedQuota >= 0;
```

Cấu trúc khai báo Repository có thể trả về chính xác số bản ghi bị tác động (affected-row):

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

Lệnh `UPDATE` này có khả năng độc quyền ở cấp dòng trên bản ghi mục tiêu (acquires row lock). Trong trường hợp tranh chấp với luồng khác trên cùng một bản ghi, luồng của chúng ta sẽ chờ đến khi có kết quả. Khi lấy được khóa, PostgreSQL lập tức thẩm định các điều kiện logic (predicate). Trạng thái phiên bản cũ (stale revision) sẽ khiến số bản ghi bị tác động (affected-row) trả về kết quả là `0` thay vì báo lỗi hệ thống, qua đó ứng dụng không bị ghi đè lên dữ liệu mới của bên kia (no overwrite).

Ưu điểm tuyệt đối của phương pháp này là rút ngắn tối đa thời gian sống của khóa (short lock lifetime), giới hạn chu trình thực hiện chỉ với một vòng khép kín. Giải pháp này không phù hợp khi quy trình xác thực nghiệp vụ (validation) gây tác động đa diện (side effects) không nằm trọn trong lệnh cơ sở dữ liệu.

## 4. Giải pháp 3 — Cơ chế Khóa Lạc Quan `@Version` (Solution 3 — `@Version` thay thế locking)

Cấu hình trường quản lý phiên bản trong Entity:

```java
@Version
@Column(nullable = false)
private long revision;
```

Khi sử dụng Hibernate để lưu thực thể, lệnh SQL sinh ra chứa điều kiện khóa mềm (optimistic locking):

```sql
update tenant_quota
set quota=?, revision=?
where tenant_id=? and revision=?;
```

Khi phát sinh trả về affected-row `0` do không thỏa mãn điều kiện phiên bản (version conflict), Hibernate sẽ ném ngoại lệ `OptimisticLockException`. Giao dịch thất bại này không bị kẹt ở trạng thái block mà kết thúc bằng lỗi. Ứng dụng phải thiết lập mô hình thử lại (retry architecture) bao gồm: Số lần thử lại hạn mức (bounded), độ trễ ngắt nhịp (backoff/jitter), và nạp một Giao dịch Mới (transaction mới) hoàn toàn để tải dữ liệu cập nhật. Cần chú ý nguy cơ quá tải do vòng lặp sinh ra lượng tải thừa (retry amplification) nếu bản ghi thuộc diện tranh chấp liên tục (Hot row).

Khóa Bi Quan (Pessimistic lock) có hiệu suất cao hơn ở những hệ thống tranh chấp cao (high conflict), và Khóa Lạc Quan (Optimistic lock) ưu việt trong môi trường xung đột ngẫu nhiên, cho phép ứng dụng từ chối nhanh và an toàn.

## 5. Giải pháp 4 — Bảng điều khiển (Dashboard) sử dụng lệnh đọc thông thường (Solution 4 — Dashboard plain SELECT)

```java
@Transactional(readOnly = true)
public QuotaView dashboard(UUID tenantId) {
    return quotas.findViewByTenantId(tenantId)
        .orElseThrow(QuotaNotFoundException::new);
}
```

Trách nhiệm và Đặc tả dành cho chức năng theo dõi trạng thái:

- Hệ thống hiển thị luôn đọc bản ghi chốt hiện hành của luồng MVCC mà không thiết lập khóa (last committed quota).
- Mọi yêu cầu giám sát hiển thị phải bỏ qua giá trị tạm thời chưa commit.
- Lệnh truy vấn không sinh ra lỗi kẹt chờ (block) vì tiến trình đọc hoàn toàn tự chủ mà không tranh chấp.

Đối với các báo cáo yêu cầu chuỗi xác thực toàn vẹn theo dữ liệu đọc (read-after-write consistency), hệ thống phải có kiến trúc quản lý token đồng bộ phiên bản. Truy vấn quan sát (Observer) luôn phải giữ ưu điểm tối giản để không ảnh hưởng luồng cập nhật.

## 6. Giải pháp 5 — Mức khóa bảng `ACCESS EXCLUSIVE` trên diện rộng (Solution 5 — Explicit table lock)

```sql
begin;
lock table tenant_quota in access exclusive mode;
-- Logic can thiệp diện rộng
commit;
```

Mức khóa cấp bảng `ACCESS EXCLUSIVE` loại trừ mọi loại khóa khác, ngăn cản toàn bộ quá trình đọc, viết trong toàn hệ thống. Phương án này chỉ được sử dụng trong các kịch bản thực thi sửa đổi hệ thống (DDL operation) hoặc chu trình bảo trì toàn phần (very strong maintenance). Nghiêm cấm dùng cơ chế này để giải quyết các xung đột cục bộ đối với dữ liệu vì:

- Gây ngưng trệ trên các bản ghi dữ liệu riêng biệt không nằm trong luồng xử lý tương tranh (unrelated rows).
- Nguy cơ sinh ra sự cố bế tắc toàn cục.
- Thời gian giữ khóa thường rất lâu theo vòng đời của giao dịch hệ thống.

## 7. Phân tích mã quá hạn khóa (Lock timeout mapping)

```java
try {
    return lockedAttempt.changeQuota(command);
} catch (CannotAcquireLockException ex) {
    return QuotaChangeResult.busyRetryable();
}
```

Để xây dựng một luồng thử lại an toàn, ứng dụng phải ánh xạ chính xác lỗi PostgreSQL sinh ra, tránh nhầm lẫn nguyên nhân cốt lõi:

- `55P03`: Thời hạn giữ khóa đã cạn kiệt nhưng chưa thể nhận tài nguyên từ hệ thống (lock timeout).
- `40P01`: Yêu cầu gây ra bế tắc vòng tròn nên bị hệ quản trị hủy làm nạn nhân (deadlock victim).
- Hết hạn vòng truy vấn giới hạn tĩnh (query/statement timeout).
- Sự cố ngắt tại vùng bộ đệm kết nối (connection acquisition timeout).

Sau khi bắt lỗi cấp cơ sở dữ liệu (`SQLException`), lớp giao dịch bao bọc phía trên đã bị chuyển sang trạng thái chờ rollback. Cơ chế xử lý gọi thử lại (Outer coordinator) bắt buộc phải đóng giao dịch hỏng và tạo lại một Giao Dịch Mới (transaction mới) hoàn toàn. (Tuyệt đối không tiếp tục vòng gọi JPA trên cùng transaction hiện hành).

## 8. Hành vi phục hồi khi gặp sự cố (Failure behavior)

- **Khi giao dịch Holder (Giao dịch đang nắm khóa) COMMIT**: Khóa cấp dòng được giải phóng, và tiến trình đang phải đứng đợi (Waiter) sẽ được nhận khóa, đồng thời đánh giá lại những gì bị tác động từ trạng thái lưu trữ mới nhất.
- **Khi giao dịch Holder bị ROLLBACK hoặc treo ngắt mạng (Crash)**: Dữ liệu chưa hoàn thiện trong cơ sở dữ liệu bị thu hồi. Khóa được giải phóng. Giao dịch Waiter đọc và thao tác dữ liệu từ phiên bản đã commit ngay trước đó.
- **Khi giao dịch Waiter (Giao dịch đang đứng chờ) quá hạn (Timeout)**: Giao dịch Waiter bị hủy bỏ và ứng dụng sẽ xử lý lỗi Timeout riêng rẽ. Giao dịch Holder vẫn tiếp tục thao tác an toàn (unaffected).
- **Tranh chấp khóa vòng tròn (Deadlock)**: Một trong những giao dịch tham gia sẽ bị chọn làm Victim. Toàn bộ tiến trình xử lý trong Giao dịch của Victim bị Rollback lại từ đầu.
- **Sự cố mất kết nối mạng phía Client**: Lệnh được gọi có thể chưa phản hồi kết quả về ứng dụng mặc dù thao tác đã thành công trong cơ sở dữ liệu (ambiguous outcome). Việc có thêm cấu trúc trường mã định danh giao dịch dự phòng (Idempotency Key) là yêu cầu tối quan trọng để kiểm tra kết quả (status replay) trong môi trường phân tán.

## 9. So sánh đặc tả các giải pháp (Trade-off comparison)

| Giải pháp (Mechanism) | Hành vi khi xung đột (Conflict behavior) | Thời gian trễ thao tác đọc (Read latency) | Quản lý tranh chấp khóa (Write contention) | Phù hợp trên cấu hình nhiều máy chủ (Multi-instance) |
| --- | --- | --- | --- | --- |
| Dùng `FOR UPDATE` | Buộc luồng kia phải chờ/timeout | Gây chậm trễ cho các lệnh đọc đồng thời | Tạo thành nút thắt cổ chai cục bộ | Có |
| SQL Update Có điều kiện | Trả về `affected-row 0` ngay lập tức | Nhanh, không cần bảo vệ truy vấn đọc | Rút ngắn thời gian tương tác SQL | Có |
| `@Version` (Khóa Lạc Quan) | Hủy thao tác (loser fails/retries) | Trơn tru (Plain reads) | Tranh chấp cao sinh tải vô hình khi retry | Có |
| Chức năng Đọc (Dashboard) | Không yêu cầu phải đợi khóa dòng | Nhận dữ liệu MVCC cấp thời (stale) | Bỏ qua cơ chế bảo vệ cập nhật | Chỉ áp dụng đọc (Observer) |
| Khóa Bảng (Table Lock) | Tạm khóa mở diện rộng/timeout | Tùy vào loại Lock Mode được chọn | Ranh giới lớn (serialize toàn cục) | Có |
| Khóa `synchronized` | Khóa tạm luồng chờ trên cùng một JVM | Hỗ trợ cục bộ | Cạnh tranh bị ngắt ngay tầng Local Application | Không (Không tương thích cross-node) |

## 10. Khuyến nghị áp dụng (Recommendation)

- Nghiệp vụ cập nhật cần một quá trình kiểm duyệt hiện trạng logic phức tạp (complex current-state validation): Áp dụng khóa bi quan bằng `PESSIMISTIC_WRITE` (short transaction).
- Các cập nhật nguyên tử dựa trên trạng thái (như cấu hình lại điều kiện kiểm tra phiên bản): Chuyển logic khóa lên truy vấn SQL bằng atomic conditional update, hoặc sử dụng hệ thống Khóa Lạc quan với `@Version`.
- Với chức năng tra cứu trạng thái: Chỉ dùng `SELECT` thông thường cho báo cáo, bảng hiển thị dữ liệu (Dashboard).
- Lệnh khóa cấp bảng (`LOCK TABLE`) chỉ áp dụng khi cần thay đổi DDL/bảo trì với cấp độ bao quát (table-wide invariant).

## 11. Danh sách kiểm tra triển khai (Production checklist)

- [ ] Lệnh SQL phát sinh từ Framework ORM thực thi chính xác mức độ bảo vệ yêu cầu (intended lock mode).
- [ ] Mọi yêu cầu lấy khóa thông qua câu truy vấn luôn phải thực hiện trong một luồng Spring Proxy cung cấp môi trường Transaction hợp lệ.
- [ ] Thời điểm khởi tạo lấy khóa và thời điểm kết thúc Transaction cần thiết kế minh bạch.
- [ ] Xóa bỏ 100% các thao tác gọi RPC/API bên thứ 3 trong khi giao dịch đang duy trì trạng thái Khóa cấp dòng (row-level lock).
- [ ] Các tác vụ truy cập sửa đổi theo danh sách lớn (Multiple resources) cần phải sắp xếp phân bổ mã định danh theo đúng cơ chế thuật toán (deterministic order).
- [ ] Hạn ngạch quá hạn cấp khóa (`lock_timeout`) phải nhỏ hơn tổng thời gian cấp phép vận hành tổng thể (overall deadline) của hệ thống.
- [ ] Sau khi trả về Timeout hoặc lỗi Deadlock, thuật toán gọi thử lại (Retry) bắt buộc xử lý tải lại toàn bộ Giao dịch mới (new transaction).
- [ ] Báo cáo/hiển thị chấp nhận tính có thể trễ đồng bộ (committed-staleness).
- [ ] Thiết lập cảnh báo cho các giao dịch quá hạn thời gian xử lý (transaction age) hoặc lượng luồng đình trệ trong trạng thái `pg_blocking_pids`.
- [ ] Thực thi bộ test tự động (test coverage) kiểm chứng luồng hoạt động cấp dòng (bounded coordination) đối với PostgreSQL.
