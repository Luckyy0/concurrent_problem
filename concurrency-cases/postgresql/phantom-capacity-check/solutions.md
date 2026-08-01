# Giải Pháp Xử Lý Kiểm Tra Giới Hạn (Giải pháp authoritative capacity)

## Mục Tiêu Thiết Kế 

Một giải pháp chuẩn xác (Correct solution) bắt buộc phải biến quá trình "còn sức chứa (capacity)" thành một cuộc xử lý tranh chấp cấp cơ sở dữ liệu (conflict/claim) để PostgreSQL có thể đảm bảo tính toàn vẹn và nguyên tử (enforce atomically). Giao dịch bị từ chối phải nhận được kết quả rõ ràng:

```text
Chấp nhận (accepted), báo đầy (full), trùng lặp (duplicate), lỗi có thể thử lại (retryable conflict) hoặc lỗi hệ thống (technical failure)
```

Kiểm soát sức chứa an toàn (Capacity safety) và ngăn chặn trùng lặp (duplicate-command safety) là hai yêu cầu độc lập và cần hai giải pháp khác nhau.

## Giải Pháp 1 — Sử Dụng Biến Đếm Nguyên Tử (Atomic conditional counter)

Thiết kế thêm biến đếm `used_slots` vào cấp độ cấu hình cha (authoritative parent row):

```sql
alter table processing_pool
    add column used_slots integer not null default 0,
    add constraint ck_processing_pool_capacity
        check (
            capacity >= 0
            and used_slots >= 0
            and used_slots <= capacity
        );
```

Thực hiện chiếm chỗ (Claim) bằng câu lệnh UPDATE đơn kèm theo điều kiện lọc:

```sql
update processing_pool
set used_slots = used_slots + 1
where pool_id = :poolId
  and used_slots < capacity;
```

Cấu hình tầng Repository:

```java
public interface ProcessingPoolRepository
    extends JpaRepository<ProcessingPool, UUID> {

    @Modifying
    @Query(
        value = """
            update processing_pool
               set used_slots = used_slots + 1
             where pool_id = :poolId
               and used_slots < capacity
            """,
        nativeQuery = true
    )
    int claimOne(UUID poolId);

    @Modifying
    @Query(
        value = """
            update processing_pool
               set used_slots = used_slots - 1
             where pool_id = :poolId
               and used_slots > 0
            """,
        nativeQuery = true
    )
    int releaseOne(UUID poolId);
}
```

Cấu trúc tầng Service:

```java
@Service
public class AtomicPoolAllocationService {
    private final ProcessingPoolRepository pools;
    private final SlotAllocationRepository allocations;

    public AtomicPoolAllocationService(
        ProcessingPoolRepository pools,
        SlotAllocationRepository allocations
    ) {
        this.pools = pools;
        this.allocations = allocations;
    }

    @Transactional
    public AllocationResult allocate(UUID poolId, UUID requestId) {
        // Cố gắng khóa và thay đổi biến đếm
        int claimed = pools.claimOne(poolId);
        
        // Nhận kết quả thất bại (affected row = 0)
        if (claimed == 0) {
            return pools.existsById(poolId)
                ? AllocationResult.full()
                : AllocationResult.poolNotFound();
        }

        // Tạo cấp phát mới trong cùng Transaction
        SlotAllocation saved = allocations.saveAndFlush(
            SlotAllocation.active(poolId, requestId)
        );
        return AllocationResult.accepted(saved.getId());
    }
}
```

Cơ chế thực thi đồng thời (Concurrency behavior) khi `used_slots=9`, `capacity=10`:

1. Tiến trình A chạy UPDATE, chiếm được khóa dòng (parent row lock), điều kiện đúng, nâng biến đếm lên `10`.
2. Tiến trình B chạy UPDATE trên cùng dòng đó và phải đợi A.
3. Tiến trình A đóng giao dịch (commit), giải phóng khóa dòng (row lock release).
4. Tiến trình B tiếp tục trên dòng dữ liệu hiện hành; PostgreSQL tự động kiểm tra lại điều kiện lọc (re-evaluate predicate).
5. `10 < 10` trả về false, hệ thống báo (affected-row) `0`; B nhận kết quả `FULL`.

Lưu ý: Bất kỳ vấn đề chèn dữ liệu thất bại nào ở tầng INSERT (Runtime/DataAccess exception) đều sẽ hoàn tác (rollback) quá trình tăng biến đếm. Cần thực hiện `saveAndFlush()` nhằm phát hiện các lỗi Unique Violation ngay trong giới hạn giao dịch, tránh trường hợp bắt lỗi (catch) mà vẫn đẩy thay đổi biến đếm vào Database.

> **Nói ngắn gọn:** Biến đếm chính xác (authoritative counter) không chỉ dùng cho việc hiển thị, mà là một phương pháp kiểm tra điều kiện ngay tại cơ sở dữ liệu. Nó chuyển quá trình cấp phát phức tạp thành 1 cuộc xung đột cập nhật dữ liệu với số kết quả dòng bị tác động (affected-row contract).

### Xử Lý Xả Giới Hạn Chỉ Một Lần (Release)

Tránh truy xuất giá trị trước (load status) rồi mới giảm bộ đếm. Thay vào đó, hãy cập nhật trạng thái có điều kiện (conditionally):

```sql
with released as (
    update slot_allocation
       set status = 'RELEASED'
     where allocation_id = :allocationId
       and status = 'ACTIVE'
    returning pool_id
)
update processing_pool p
   set used_slots = p.used_slots - 1
  from released r
 where p.pool_id = r.pool_id
   and p.used_slots > 0;
```

Nếu số dòng chịu ảnh hưởng (Affected-row) là `1`, việc giải phóng thành công; nếu `0` thì dữ liệu đã được xử lý từ trước. Cách viết gộp này giúp việc chuyển trạng thái và giảm biến đếm luôn đồng bộ và chính xác.

### Đối Soát Đồng Bộ Dữ Liệu (Reconciliation)

Biến đếm (Counter) là một dạng chuẩn hóa đặc biệt, do đó bạn cần các quy trình đối soát định kỳ:

```sql
select p.pool_id,
       p.capacity,
       p.used_slots,
       count(a.*) filter (where a.status = 'ACTIVE') as actual_active
from processing_pool p
left join slot_allocation a on a.pool_id = p.pool_id
group by p.pool_id, p.capacity, p.used_slots
having p.used_slots <>
       count(a.*) filter (where a.status = 'ACTIVE');
```

Hoạt động cấu trúc dữ liệu mới hoặc sửa đổi (Migration) phải diễn ra khi hệ thống không có tác vụ cập nhật đang chạy (write quiescence/lock) hoặc thông qua kỹ thuật ghi hai nguồn (dual-write protocol) để bảo vệ độ chính xác dữ liệu gốc.

## Giải Pháp 2 — Khóa Phụ Thuộc (Lock parent)

Nếu bạn không muốn duy trì một biến đếm, bạn có thể biến một dòng cấp độ cha thành lớp bảo vệ (mutex):

```java
public interface ProcessingPoolRepository
    extends JpaRepository<ProcessingPool, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        select p
          from ProcessingPool p
         where p.id = :poolId
        """)
    Optional<ProcessingPool> findForAllocation(UUID poolId);
}
```

Tương đương với lệnh PostgreSQL:

```sql
select pool_id, capacity
from processing_pool
where pool_id = :poolId
for update;
```

Cấu trúc tầng Service:

```java
@Transactional
public AllocationResult allocateWithParentLock(
    UUID poolId,
    UUID requestId
) {
    // Luôn khóa dòng cha trước
    ProcessingPool pool = pools.findForAllocation(poolId)
        .orElseThrow(PoolNotFoundException::new);

    // Bắt đầu đếm thực tế
    long active = allocations.countByPoolAndStatus(
        poolId,
        AllocationStatus.ACTIVE
    );
    if (active >= pool.getCapacity()) {
        return AllocationResult.full();
    }

    SlotAllocation saved = allocations.saveAndFlush(
        SlotAllocation.active(poolId, requestId)
    );
    return AllocationResult.accepted(saved.getId());
}
```

Trong chiến lược này, tất cả yêu cầu cấp phát, xả slot phải luôn truy cập qua nút thắt cổ chai dòng cha (parent row lock) trước. Giao dịch B sẽ bị chờ cho đến khi giao dịch A thực thi xong và commit, sau đó đếm số lại từ đầu để từ chối yêu cầu. Ưu điểm là giảm độ phức tạp cho CSDL, tuy nhiên tốc độ của bãi chứa (hot pool) sẽ bị giới hạn vì mọi yêu cầu xử lý sẽ diễn ra tuần tự (serialize requests).

## Giải Pháp 3 — Khởi Tạo Sẵn Vị Trí Slot (Pre-created finite slot rows)

Nếu các vị trí của pool là một con số cố định hữu hạn, hãy biến chúng thành các thực thể vật lý:

```sql
create table processing_slot (
    pool_id uuid not null,
    slot_no integer not null,
    allocation_id uuid,
    primary key (pool_id, slot_no),
    unique (allocation_id),
    foreign key (pool_id) references processing_pool(pool_id)
);
```

Yêu cầu lấy vị trí trống:

```sql
with candidate as (
    select pool_id, slot_no
    from processing_slot
    where pool_id = :poolId
      and allocation_id is null
    order by slot_no
    for update skip locked
    limit 1
)
update processing_slot s
set allocation_id = :allocationId
from candidate c
where s.pool_id = c.pool_id
  and s.slot_no = c.slot_no
returning s.slot_no;
```

Mỗi slot bây giờ là một đơn vị dữ liệu khóa được độc lập (lockable/unique capacity unit). Sử dụng tham số `SKIP LOCKED` cho phép hệ thống bỏ qua các slot đang bị nắm giữ và trả về giá trị rỗng nhanh chóng (hoặc kết quả từ chối FULL). 

## Giải Pháp 4 — Cấu Hình Chuẩn SERIALIZABLE 

Sử dụng khi quy định cấp phát nghiệp vụ phức tạp, không thể tập hợp vào biến đếm chung:

```java
@Service
public class SerializableAllocationAttempt {
    private final ProcessingPoolRepository pools;
    private final SlotAllocationRepository allocations;

    @Transactional(
        isolation = Isolation.SERIALIZABLE,
        propagation = Propagation.REQUIRES_NEW
    )
    public AllocationResult run(UUID poolId, UUID requestId) {
        ProcessingPool pool = pools.findById(poolId)
            .orElseThrow(PoolNotFoundException::new);
        long active = allocations.countByPoolAndStatus(
            poolId,
            AllocationStatus.ACTIVE
        );
        if (active >= pool.getCapacity()) {
            return AllocationResult.full();
        }

        SlotAllocation saved = allocations.saveAndFlush(
            SlotAllocation.active(poolId, requestId)
        );
        return AllocationResult.accepted(saved.getId());
    }
}
```

Hỗ trợ vòng lặp lại bên ngoài (Outer retry):

```java
@Service
public class AllocationRetrier {
    private final SerializableAllocationAttempt attempt;
    private final RetryBackoff backoff;

    public AllocationRetrier(
        SerializableAllocationAttempt attempt,
        RetryBackoff backoff
    ) {
        this.attempt = attempt;
        this.backoff = backoff;
    }

    public AllocationResult allocate(UUID poolId, UUID requestId) {
        int maxAttempts = 3;
        for (int number = 1; number <= maxAttempts; number++) {
            try {
                return attempt.run(poolId, requestId);
            } catch (CannotSerializeTransactionException ex) {
                if (number == maxAttempts) {
                    throw ex;
                }
                backoff.pauseWithJitter(number);
            }
        }
        throw new IllegalStateException("Hệ thống lỗi không thể kết nối");
    }
}
```

Các xung đột SSI thường sẽ xảy ra tại tầng flush/commit. Một Giao dịch bị ảnh hưởng sẽ rollback hoàn toàn. Việc yêu cầu thực thi lại (Retry proxy) bắt buộc phải tiến hành thông qua bean tách biệt nhằm đảm bảo cấp `REQUIRES_NEW` giao dịch mới mẻ. Cơ chế chạy lại nên ứng dụng trễ ngẫu nhiên (bounded jitter) để tránh lặp xung đột.

## Giải Pháp 5 — Khóa Ảo Cấp Giao Dịch (Transaction-scoped advisory lock)

Khi không có cấu trúc dòng cha thuận lợi nhưng dữ liệu lại phân loại qua ID rõ ràng:

```sql
select pg_advisory_xact_lock(hashtextextended(:poolId::text, 0));
```

Khóa bảo vệ không liên quan đến quan hệ cơ sở dữ liệu, nên có điểm yếu ở thiết kế phụ thuộc ứng dụng. Mọi người tham gia chỉnh sửa (writers) phải tuân thủ cùng cấu trúc khóa ảo này. Hệ thống tự động gỡ cài đặt khóa tại cuối giao dịch. 

## Tại Sao Ràng Buộc Cơ Bản Không Hiệu Quả (Tại sao constraint thông thường chưa đủ)

Ràng buộc mức Table như `CHECK` không được phép tương tác đếm dữ liệu đa dạng các dòng (cross-row constraint). Ràng buộc `Unique` chỉ chặn mã thông báo (duplicate key) sao chép. Nếu cố dùng Trigger để buộc tổng, bạn vẫn phải duy trì cấu trúc Parent Row hoặc Khóa Ảo ở trên nhằm ngăn chặn tình huống chèn tương tranh.

Sử dụng khóa toàn phần trên CSDL:

```sql
lock table slot_allocation in share row exclusive mode;
```

Điều này gây giới hạn luồng nghiêm trọng do độ tranh chấp quá lớn trên toàn Table hệ thống, cản trở thông lượng (throughput) và dễ dẫn đến Deadlock bất ngờ.

## So Sánh Các Giải Pháp (Trade-off comparison)

| Phương Án | Phạm Vi Bảo Toàn (Correctness boundary) | Giao Dịch Bị Từ Chối (Loser behavior) | Điểm Gây Tranh Chấp (Contention) | Độ Phức Tạp (Complexity) |
| --- | --- | --- | --- | --- |
| Biến đếm (Conditional counter) | Dòng Cha + Kiểm Tra (Parent row + check) | affected-row trả về `0` | Trung bình (Hot pool) | Trung bình (Quản lý Counter/Reconciliation) |
| Khóa dòng cha (Parent `FOR UPDATE`) | Khóa liên kết dữ liệu dòng Cha | Khóa chờ Block, timeout, hoặc FULL | Lớn (Hot pool) | Thấp |
| Khởi tạo sẵn Slot | Dòng vật lý Slot + khóa Unique | Skip/Lỗi không vị trí | Phân tán rộng ở mức Slot | Khá (Quản lý cấu trúc Lifecycle) |
| Cấu Hình `SERIALIZABLE` | Màng lọc cấp Snapshot SSI | Mã lỗi `40001`, sau đó Retry lại | Tái thực thi lại theo conflict | Cao (Xử lý chu trình Retry/Ops) |
| Khóa Ảo (Advisory xact lock) | Khóa từ bộ nhớ quy ước (Numeric key) | Khóa chờ Block/timeout | Nằm ở cấp cấu trúc ID key | Thấp (Dễ bị vượt qua) |

## Hoạt Động Khi Hệ Thống Gặp Trục Trặc (Failure và recovery)

- Lỗi tại dòng INSERT sẽ dẫn tới rollback tự động, và biến đếm (nếu dùng) tự đảo ngược.
- Giao dịch đang giữ khóa sập tiến trình (Crash): PostgreSQL sẽ chủ động dọn dẹp và trả khóa ở cấp độ kết nối.
- Thất bại giao dịch SSI: Tiến trình cũ hoàn toàn bị vứt bỏ, chỉ có thể tạo yêu cầu tái thực thi mới.
- Kết nối Client treo mạng (Timeout): Để tránh tạo lại dòng không mong muốn, ứng dụng nên tái xác nhận dữ liệu thông qua tham số idempotency ID.

## Khuyến Nghị Sử Dụng Dành Cho Dữ Liệu Tập Hợp (Recommendation cho case này)

Đối với hệ thống cấp phát thông lượng đơn giản:

1. Ưu tiên **Bộ đếm có điều kiện (conditional counter)** nếu bạn tự tin quản lý vòng đời và làm reconciliation định kỳ.
2. Dùng **Khóa cấp cha (parent `FOR UPDATE`)** nếu bộ đếm con là cơ sở dữ liệu xác thực (source of truth) và hệ thống của bạn không gặp vấn đề chậm về hiệu năng trên mỗi Pool.
3. Sử dụng cấu trúc **Slot phân bổ cố định (pre-created slots)** khi năng lực tối đa của ứng dụng được giới hạn tự nhiên và không thay đổi liên tục.
4. Triển khai cấu hình cô lập **SERIALIZABLE** khi điều kiện cấp phát dữ liệu yêu cầu vô cùng phức tạp và team vận hành thành thạo quá trình retry luồng.

Đừng bao giờ áp dụng cấu hình `REPEATABLE READ` chỉ vì mong muốn tạm thời che giấu các dòng dữ liệu bóng ma.

## Danh Sách Cần Kiểm Tra Trên Production (Production checklist)

### Điều Kiện Nghiệp Vụ (Invariant)

- [ ] Mọi cấp phát hợp lệ đều đi kèm một sức chứa hữu hình bị hao hụt.
- [ ] Số lượng đang cung cấp hoặc biến đếm không vượt định mức.
- [ ] Các giao dịch trùng lặp ID yêu cầu hoàn toàn bị loại bỏ.
- [ ] Xả Slot chỉ hoàn lại giá trị đúng một lần.

### Quản Lý Giao Dịch (Transactions)

- [ ] Chiếm biến đếm và lưu cấp phát diễn ra chặt chẽ trong một khối giao dịch.
- [ ] Mức cô lập và hiệu ứng đã được thử nghiệm thực tế.
- [ ] Vòng thử lại (Retry) bắt đầu một giao dịch mới nguyên bản.
- [ ] Ghi nhận và xử lý đầy đủ các lỗi ngoại lệ hệ thống như Timeout, Deadlock, và `40001`.
- [ ] Nếu xử lý trên nhiều Pool, thứ tự cấp phát phải tuân thủ nghiêm ngặt (deterministic).

### Vận Hành Hệ Thống (Operations)

- [ ] Lệnh truy vấn kiểm tra dữ liệu đối chiếu chạy tự động theo lịch (Reconciliation).
- [ ] Ghi nhận thống kê cảnh báo lỗi đếm `0`, quá thời gian chờ, lần lặp lại và vượt khung quy định.
- [ ] Không sử dụng cuộc gọi I/O từ xa (như RestAPI/Network) trong suốt quá trình xử lý giao dịch cơ sở dữ liệu (Transaction).
- [ ] Thử nghiệm được khởi động thực tiễn với PostgreSQL Testcontainers và kiểm thử đồng thời có kiểm soát (controlled interleaving).
- [ ] Xây dựng sẵn bộ tài liệu xử lý lỗi khi thông lượng giới hạn bị can thiệp.
