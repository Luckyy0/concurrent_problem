# Broken exists-then-save

## Entity thiếu business constraint

```java
@Entity
@Table(name = "work_item")
public class WorkItem {
    @Id
    @Column(name = "work_item_id", nullable = false)
    private UUID id;

    @Column(name = "tenant_id", nullable = false)
    private UUID tenantId;

    @Column(name = "external_reference", nullable = false)
    private String externalReference;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private WorkItemStatus status;

    protected WorkItem() {
    }

    public static WorkItem open(UUID tenantId, String externalReference) {
        WorkItem item = new WorkItem();
        item.id = UUID.randomUUID();
        item.tenantId = tenantId;
        item.externalReference = externalReference;
        item.status = WorkItemStatus.OPEN;
        return item;
    }

    public UUID getId() {
        return id;
    }
}
```

Khóa chính ngẫu nhiên chỉ đảm bảo tính duy nhất của hàng vật lý. Nó không diễn đạt tính duy nhất logic (logical uniqueness) của `(tenant_id, external_reference)`.

## Repository

```java
public interface WorkItemRepository extends JpaRepository<WorkItem, UUID> {
    boolean existsByTenantIdAndExternalReference(
        UUID tenantId,
        String externalReference
    );

    Optional<WorkItem> findByTenantIdAndExternalReference(
        UUID tenantId,
        String externalReference
    );
}
```

## Broken service

```java
@Service
public class WorkItemService {
    private final WorkItemRepository workItems;

    public WorkItemService(WorkItemRepository workItems) {
        this.workItems = workItems;
    }

    @Transactional
    public CreateWorkItemResult create(
        UUID tenantId,
        String externalReference
    ) {
        if (workItems.existsByTenantIdAndExternalReference(
            tenantId,
            externalReference
        )) {
            return CreateWorkItemResult.duplicate();
        }

        WorkItem saved = workItems.save(
            WorkItem.open(tenantId, externalReference)
        );
        return CreateWorkItemResult.created(saved.getId());
    }
}
```

Đoạn mã không có lỗi bất cẩn hiển nhiên: đã có thao tác kiểm tra để trả về kết quả cho UX/domain và phương thức có `@Transactional`. Lỗi (bug) nằm ở giả định rằng lệnh SELECT và lệnh INSERT sau đó tạo thành một quyền sở hữu nguyên tử (atomic claim).

> **Nói ngắn gọn:** transaction bảo đảm mỗi yêu cầu tự commit/rollback; nó không ngăn các transaction khác chen vào giữa thao tác kiểm tra và lệnh INSERT.

## Broken schema

```sql
create table work_item (
    work_item_id uuid primary key,
    tenant_id uuid not null,
    external_reference varchar(100) not null,
    status varchar(32) not null
);
```

Không có index/ràng buộc nào gây xung đột khi A và B dùng các UUID khác nhau.

## Concrete interleaving

```text
A BEGIN
B BEGIN
A SELECT EXISTS(T-42, CASE-9001) -> false
B SELECT EXISTS(T-42, CASE-9001) -> false
A persist id=A
B persist id=B
A flush INSERT id=A -> success
B flush INSERT id=B -> success
A COMMIT
B COMMIT
final logical row count -> 2
```

Lệnh SELECT thông thường không xin cấp khóa (acquire a lock) trên một key chưa tồn tại. Các thao tác khóa hàng/index của lệnh INSERT nhắm vào các khóa chính khác nhau, nên không xảy ra xung đột.

## Hibernate flush timing

`save()` thường chỉ đưa entity vào trạng thái được quản lý (managed). Với UUID do ứng dụng tự gán, câu lệnh SQL INSERT có thể phải đợi tới thời điểm flush/commit:

```text
service returns CREATED internally
transaction interceptor commits
Hibernate flushes INSERT
database error may appear here
```

Vì vậy, mã của phương thức có thể không thấy ngoại lệ duy nhất (unique exception) nếu chỉ gọi `save()`. Việc dùng `saveAndFlush()` hoặc `EntityManager.flush()` sẽ đưa xung đột về ranh giới của lần thử insert, nhưng ràng buộc thì vẫn phải tồn tại.

## Broken catch pattern sau khi thêm constraint

```java
@Transactional
public CreateWorkItemResult createOrFind(...) {
    try {
        WorkItem saved = workItems.saveAndFlush(WorkItem.open(...));
        return CreateWorkItemResult.created(saved.getId());
    } catch (DataIntegrityViolationException ex) {
        WorkItem existing = workItems
            .findByTenantIdAndExternalReference(...)
            .orElseThrow();
        return CreateWorkItemResult.existing(existing.getId());
    }
}
```

Sau lỗi `23505` của PostgreSQL, transaction vật lý đã bị hủy và Spring thường đánh dấu trạng thái chỉ-rollback (rollback-only). Lệnh SELECT trong khối catch có thể thất bại với thông báo “current transaction is aborted”, hoặc phương thức kết thúc bằng `UnexpectedRollbackException`. Việc xử lý ngoại lệ (Catch) không làm sống lại transaction.

## Điều kiện tiền đề (Preconditions) để tái hiện

1. Business-key unique constraint chưa tồn tại.
2. A và B dùng các connections/transactions độc lập.
3. Cả hai thao tác kiểm tra tồn tại đều hoàn tất trước khi bất kỳ lệnh INSERT nào được commit.
4. Các khóa chính ngẫu nhiên khác nhau.
5. Cùng một tenant/tham chiếu đã chuẩn hóa (normalized) được gửi đến.
6. Không có transaction kiểm thử bên ngoài (outer test transaction) che khuất các thao tác commit.

## Những cách sửa chưa đủ

### Chỉ dùng `@Transactional`

Mức độ `READ COMMITTED` không giữ chỗ một business key chưa tồn tại giữa các câu lệnh.

### Chỉ dùng `synchronized`

Không điều phối được đa tiến trình (multiple pods), các công việc hàng loạt (batch jobs) hoặc thao tác SQL trực tiếp.

### Chỉ thêm pre-check thứ hai

Vẫn tồn tại khoảng trống thời gian (window) giữa lần kiểm tra cuối cùng và lệnh INSERT.

### Chỉ nâng `REPEATABLE READ`

Cả hai snapshot ổn định đều có thể tiếp tục thấy key chưa tồn tại. Không thể thay thế unique constraint.

### Catch mọi integrity exception thành duplicate

Có thể che giấu các vi phạm khóa ngoại (foreign-key), not-null, ràng buộc kiểm tra (check) hoặc các ràng buộc duy nhất khác. Cần phân loại (classify) SQLSTATE và tên của ràng buộc.

### `SELECT ... FOR UPDATE` trên missing row

Không có hàng nào để khóa. Việc khóa bảng hoặc dùng advisory key có thể giúp tuần tự hóa nhưng phức tạp hơn cơ chế duy nhất của database cho bất biến này.

### Dùng cache/Redis `SETNX` làm sole authority

Database vẫn cho phép dữ liệu trùng lặp nếu bộ nhớ đệm (cache) hết hạn, bị vượt rào (bypass) hoặc transaction của DB thất bại. Tính duy nhất trong một cơ sở dữ liệu (Single-database uniqueness) phải được thực thi tại chính database.
