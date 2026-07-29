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

Random primary key chỉ unique physical row. Nó không diễn đạt logical uniqueness
của `(tenant_id, external_reference)`.

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

Code không bất cẩn hiển nhiên: check cho UX/domain outcome và method có
`@Transactional`. Bug là assumption rằng SELECT và later INSERT tạo một atomic
claim.

> **Nói ngắn gọn:** transaction bảo đảm mỗi request tự commit/rollback; nó không
> ngăn transaction khác chen vào giữa check và INSERT.

## Broken schema

```sql
create table work_item (
    work_item_id uuid primary key,
    tenant_id uuid not null,
    external_reference varchar(100) not null,
    status varchar(32) not null
);
```

Không index/constraint nào conflict khi A/B dùng different UUIDs.

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

Plain SELECT không acquire a lock trên absent key. INSERT row/index locks target
different primary keys, nên không conflict.

## Hibernate flush timing

`save()` thường chỉ làm entity managed. Với application-assigned UUID, SQL INSERT
có thể đợi tới flush/commit:

```text
service returns CREATED internally
transaction interceptor commits
Hibernate flushes INSERT
database error may appear here
```

Vì vậy method code có thể không thấy unique exception nếu chỉ gọi `save()`.
`saveAndFlush()`/`EntityManager.flush()` đưa conflict về insert-attempt boundary,
nhưng constraint vẫn phải tồn tại.

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

Sau PostgreSQL `23505`, physical transaction đã abort và Spring thường đánh dấu
rollback-only. SELECT trong catch có thể fail với “current transaction is aborted”,
hoặc method kết thúc bằng `UnexpectedRollbackException`. Catch không hồi sinh
transaction.

## Preconditions tái hiện

1. Business-key unique constraint chưa tồn tại.
2. A/B dùng independent connections/transactions.
3. Cả hai existence checks hoàn tất trước either INSERT commit.
4. Random primary IDs khác nhau.
5. Cùng normalized tenant/reference được gửi.
6. Không outer test transaction che commits.

## Những cách sửa chưa đủ

### Chỉ dùng `@Transactional`

`READ COMMITTED` không reserve absent business key giữa statements.

### Chỉ dùng `synchronized`

Không coordinate multiple pods, batch jobs hoặc direct SQL.

### Chỉ thêm pre-check thứ hai

Vẫn tồn tại window giữa last check và INSERT.

### Chỉ nâng `REPEATABLE READ`

Cả hai stable snapshots có thể tiếp tục thấy key absent. Không thay unique
constraint.

### Catch mọi integrity exception thành duplicate

Có thể che foreign-key, not-null, check hoặc unique constraint khác. Phải classify
SQLSTATE và constraint name.

### `SELECT ... FOR UPDATE` trên missing row

Không có row để lock. Lock table/advisory key có thể serialize nhưng phức tạp hơn
database uniqueness cho invariant này.

### Dùng cache/Redis `SETNX` làm sole authority

Database vẫn cho duplicate nếu cache expires, bị bypass hoặc transaction DB fail.
Single-database uniqueness phải được enforce ở database.
