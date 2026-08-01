# Lỗi Lập Trình Count-Then-Insert (Broken count-then-insert allocation)

## 1. Cấu trúc Entity Bể Xử Lý (Pool entity)

```java
@Entity
@Table(name = "processing_pool")
public class ProcessingPool {
    @Id
    @Column(name = "pool_id", nullable = false)
    private UUID id;

    @Column(nullable = false)
    private int capacity;

    protected ProcessingPool() {
    }

    public UUID getId() {
        return id;
    }

    public int getCapacity() {
        return capacity;
    }
}
```

## 2. Cấu trúc Entity Cấp Phát (Allocation entity)

```java
@Entity
@Table(
    name = "slot_allocation",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_slot_allocation_request",
        columnNames = {"pool_id", "request_id"}
    )
)
public class SlotAllocation {
    @Id
    @Column(name = "allocation_id", nullable = false)
    private UUID id;

    @Column(name = "pool_id", nullable = false)
    private UUID poolId;

    @Column(name = "request_id", nullable = false)
    private UUID requestId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private AllocationStatus status;

    protected SlotAllocation() {
    }

    public static SlotAllocation active(UUID poolId, UUID requestId) {
        SlotAllocation allocation = new SlotAllocation();
        allocation.id = UUID.randomUUID();
        allocation.poolId = poolId;
        allocation.requestId = requestId;
        allocation.status = AllocationStatus.ACTIVE;
        return allocation;
    }

    public UUID getId() {
        return id;
    }
}
```

Ràng buộc duy nhất `unique(pool_id, request_id)` giúp ngăn chặn việc gửi cùng một request hai lần (idempotency). Tuy nhiên, nếu Request A và Request B là hai yêu cầu độc lập (có ID khác nhau), constraint này sẽ bỏ qua và hoàn toàn vô hại, không thể bảo vệ sức chứa tổng thể.

## 3. Lệnh truy vấn đếm số lượng (Repository predicate query)

```java
public interface SlotAllocationRepository
    extends JpaRepository<SlotAllocation, UUID> {

    @Query("""
        select count(a)
          from SlotAllocation a
         where a.poolId = :poolId
           and a.status = :status
        """)
    long countByPoolAndStatus(
        UUID poolId,
        AllocationStatus status
    );
}
```

## 4. Service triển khai lỗi (Broken service)

```java
@Service
public class PoolAllocationService {
    private final ProcessingPoolRepository pools;
    private final SlotAllocationRepository allocations;

    public PoolAllocationService(
        ProcessingPoolRepository pools,
        SlotAllocationRepository allocations
    ) {
        this.pools = pools;
        this.allocations = allocations;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public AllocationResult allocate(UUID poolId, UUID requestId) {
        ProcessingPool pool = pools.findById(poolId)
            .orElseThrow(PoolNotFoundException::new);

        // BƯỚC 1: Đếm số lượng hiện tại
        long active = allocations.countByPoolAndStatus(
            poolId,
            AllocationStatus.ACTIVE
        );

        // BƯỚC 2: Kiểm tra giới hạn (Quyết định)
        if (active >= pool.getCapacity()) {
            return AllocationResult.full();
        }

        // BƯỚC 3: Lưu mới dữ liệu
        SlotAllocation saved = allocations.save(
            SlotAllocation.active(poolId, requestId)
        );
        return AllocationResult.accepted(saved.getId());
    }
}
```

Đoạn code này trông có vẻ hợp lý và rất phổ biến (realistic): Thao tác kiểm tra sức chứa và thao tác INSERT nằm chung trong một `@Transactional`, kết hợp với constraint unique để chặn request trùng. Tuy nhiên, sai lầm cốt lõi là không có cơ chế khóa (lock) để bảo vệ đoạn logic quyết định dựa trên điều kiện tập hợp (Aggregate Predicate).

> **Nói ngắn gọn:** Áo khoác Transaction chỉ đảm bảo thao tác cấp phát diễn ra nguyên tử (hoặc thành công toàn bộ hoặc thất bại toàn bộ), nhưng nó không ngăn cản hai giao dịch đồng thời cùng nhìn thấy một giá trị đếm `COUNT = 9` rồi cùng ghi đè dữ liệu mới, dẫn đến tổng số thực tế là `11`.

## 5. Các câu lệnh SQL Hibernate thực thi (SQL Hibernate thực thi)

Cả giao dịch A và B đều thực thi tuần tự các câu lệnh sau:

```sql
begin;

select p.pool_id, p.capacity
from processing_pool p
where p.pool_id = :poolId;

select count(*)
from slot_allocation a
where a.pool_id = :poolId
  and a.status = 'ACTIVE';
-- Hai giao dịch đều nhận kết quả là 9

insert into slot_allocation(
    allocation_id,
    pool_id,
    request_id,
    status
)
values (:differentAllocationId, :poolId, :differentRequestId, 'ACTIVE');

commit;
```

Hibernate thường trì hoãn việc gửi lệnh `INSERT` cho đến khi kết thúc giao dịch (thao tác flush/commit). Kể cả khi bạn ép Hibernate thực hiện `flush` sớm, giao dịch khác vẫn không thể thấy được dòng INSERT mới này cho đến khi bạn gọi `commit`. Do đó, điều này không tạo ra rào chắn bảo vệ nào cho điều kiện đếm.

## 6. Trình tự thực thi đan xen thực tế (Concrete interleaving)

```text
A BẮT ĐẦU GIAO DỊCH
B BẮT ĐẦU GIAO DỊCH
A THỰC HIỆN COUNT -> Kết quả 9
B THỰC HIỆN COUNT -> Kết quả 9
A Quyết định chèn A-101
B Quyết định chèn B-202
A Gửi lệnh INSERT A-101 -> Thành công
B Gửi lệnh INSERT B-202 -> Thành công
A COMMIT GIAO DỊCH
B COMMIT GIAO DỊCH
Tổng cộng các dòng ACTIVE thực tế -> 11 
```

Do các dòng dữ liệu INSERT có khóa chính khác biệt, không có bất kỳ xung đột nào ở cấp độ database để bảo vệ quy tắc `COUNT(ACTIVE) <= capacity`.

## 7. Phát hiện bóng ma bằng truy vấn lặp lại (Visible phantom khi query lại)

Trong mức cô lập `READ COMMITTED`, nếu giao dịch thực hiện lệnh truy vấn lần 2, hiện tượng bóng ma có thể xảy ra:

```sql
select count(*) ...; -- Nhận kết quả 9
-- Giao dịch B chèn dữ liệu và commit thành công
select count(*) ...; -- Nhận kết quả 10
```

Bởi vì câu lệnh `SELECT` thứ 2 tạo ra một snapshot mới hơn, nó có thể thấy dữ liệu đã commit (visible phantom). Tuy nhiên, kịch bản code lỗi ở trên không thực hiện việc đếm lại. Nó chỉ sử dụng kết quả đếm một lần duy nhất, ra quyết định và sau đó lập tức thực hiện chèn dữ liệu (Check-Then-Insert) dẫn đến lỗi vi phạm điều kiện.

## 8. Vì sao REPEATABLE READ không giải quyết được vấn đề (Tại sao `REPEATABLE READ` chưa đủ)

Khi cấu hình ở mức `REPEATABLE READ`, snapshot sẽ ổn định xuyên suốt giao dịch:

```text
A thực hiện COUNT #1 -> Kết quả: 9
B thực hiện INSERT và COMMIT 
A thực hiện COUNT #2 -> Kết quả vẫn là: 9
```

Lúc này hai giao dịch vẫn chèn hai dòng dữ liệu độc lập và cùng commit bình thường, không gây ra lỗi vi phạm tương tranh (conflict) trực tiếp:

```text
Snapshot của A: Đếm được 9 -> Chèn INSERT A
Snapshot của B: Đếm được 9 -> Chèn INSERT B
CẢ HAI CÙNG COMMIT THÀNH CÔNG -> Tổng số dòng thực tế vượt ngưỡng (11)
```

Điều này chứng minh rằng "không thấy hiện tượng bóng ma" không có nghĩa là bảo toàn được giới hạn tổng thể (capacity invariant). Cần một cơ chế xử lý tranh chấp (contention) tường minh hơn.

## 9. Điều kiện tái hiện lỗi (Preconditions tái hiện)

1. Sức chứa hiện tại đang ngấp nghé ở ngưỡng `capacity - 1`.
2. Hai giao dịch A và B thực hiện yêu cầu đồng thời.
3. Cả hai giao dịch đều thực hiện xong lệnh `COUNT` trước khi lệnh `INSERT` của luồng kia được commit.
4. Các Allocation ID và Request ID của A và B hoàn toàn khác biệt.
5. Không có cơ chế khóa tường minh (như `FOR UPDATE`) hoặc mức độ cô lập chặn rủi ro (`SERIALIZABLE`).
6. Phương thức đang thực thi không bị giới hạn bởi một cấu hình transaction ở phương thức cha (outer transaction).
7. MVCC của PostgreSQL tuân thủ đúng mức cô lập (Isolation) mặc định của DB.

## 10. Các giải pháp sai lầm thường gặp (Những cách sửa chưa đủ)

### Áp dụng Annotation `@Transactional`

Annotation này không giải quyết việc thiếu tính nguyên tử giữa lệnh quyết định (COUNT) và lệnh chèn dữ liệu (INSERT) nếu không có cơ chế chặn đọc/ghi của Database.

### Ràng buộc Unique Constraint

`Unique(pool_id, request_id)` chỉ nhằm loại bỏ các yêu cầu giống hệt nhau (Duplicate identity), không bảo vệ tổng số dòng trong tập hợp.

### Sử dụng khóa Lock trên các dòng tồn tại (`COUNT(*) FOR UPDATE`)

Lệnh `SELECT COUNT(*) ... FOR UPDATE` không khóa (lock) được phạm vi điều kiện đếm (predicate range), nó chỉ khóa những dòng hiện có lúc truy vấn. Nếu B thêm một dòng mới vào khoảng không đó, khóa này hoàn toàn vô dụng.

### Tăng mức cô lập lên `REPEATABLE READ`

Như phân tích ở trên, Snapshot cô lập của `REPEATABLE READ` không tạo ra xung đột ghi (write conflict) khi hai dòng độc lập được chèn vào.

### Sử dụng biến cục bộ `synchronized`

Phương pháp này chỉ chạy đúng trên một Instance (Local JVM) duy nhất. Nếu triển khai ứng dụng trên nhiều máy chủ (clustering, pods), khóa `synchronized` trở nên vô tác dụng.

### Bổ sung `flush` sớm trước khi trả kết quả

Lệnh `flush` không làm thay đổi ranh giới commit của giao dịch, các giao dịch bên ngoài vẫn không nhìn thấy kết quả cho đến khi `commit` được gọi.

### Kích hoạt Retry tự động (Catch Exception và retry)

Ở mức `READ COMMITTED`, lỗi vi phạm giới hạn này sẽ không sinh ra bất kỳ Exception nào để mà catch. Nếu lạm dụng retry không có Idempotency chuẩn mực, nó thậm chí có thể sinh ra thêm nhiều dòng rác.
