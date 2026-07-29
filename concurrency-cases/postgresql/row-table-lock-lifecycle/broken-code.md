# Broken SELECT-lock assumptions

## Entity

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

`revision` là audit field, chưa phải `@Version`; case tập trung explicit lock
mechanics.

## Broken writer nghĩ plain SELECT đã khóa row

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

`findById()` sinh ordinary SELECT:

```sql
select tenant_id, quota, revision
from tenant_quota
where tenant_id = :tenantId;
```

Không có `FOR UPDATE`, `@Lock` hoặc version predicate. `@Transactional` không tự
biến query thành locking read.

> **Nói ngắn gọn:** persistence context giữ Java entity managed, không có nghĩa
> PostgreSQL row đang được pessimistically locked.

## Concurrent SQL

Initial quota `10`:

```text
A BEGIN
A plain SELECT -> 10
A local validation still running

B BEGIN
B UPDATE quota=8 -> succeeds
B COMMIT

A later flushes absolute quota based on old state
```

B không chờ A vì A chỉ có relation-level `ACCESS SHARE` từ SELECT; mode này tương
thích với ordinary DML relation locks.

## Broken dashboard dùng locking read

Sau incident, team thay mọi read bằng pessimistic lock:

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

Dashboard giờ cạnh tranh `FOR UPDATE` với writers dù chỉ cần last committed value.
Đây là needless serialization, tăng wait/connection usage.

## Broken long-lock boundary

```java
@Transactional
public void changeAndNotify(UUID tenantId, int requestedQuota) {
    TenantQuota quota = quotas.findForEverything(tenantId)
        .orElseThrow();

    policyClient.confirmChange(tenantId, requestedQuota);
    quota.changeQuota(requestedQuota);
}
```

Lock sống xuyên remote call tới transaction commit. Method chưa return không phải
lý do duy nhất; physical transaction vẫn open mới là boundary quyết định.

## SQL assumptions sai

### Assumption 1

```sql
begin;
select * from tenant_quota where tenant_id=:id;
-- team assumes row is locked
```

Actual: competing UPDATE proceeds.

### Assumption 2

```sql
begin;
select * from tenant_quota where tenant_id=:id for update;
update tenant_quota set quota=12 where tenant_id=:id;
-- not committed
```

Team assumes every SELECT blocks. Actual plain SELECT on another connection sees
old committed quota; only incompatible locking/mutation attempts wait.

## Preconditions tái hiện

1. A/B/C use independent connections/transactions.
2. Effective isolation là PostgreSQL `READ COMMITTED`.
3. A first path truly emits plain SELECT.
4. Explicit-lock path emits `FOR UPDATE`.
5. Lock holder remains uncommitted while waiter/reader runs.
6. No outer test transaction hides commit/rollback.

## Những cách sửa chưa đủ

### Chỉ thêm `@Transactional`

Transaction defines lifetime, không define query lock mode.

### Dùng `synchronized`

Không protect other application instances/direct SQL.

### Lock mọi read

Protects nothing extra for observers, nhưng tạo wait queue không cần thiết.

### Flush rồi nghĩ lock release

Flush sends SQL; row lock vẫn giữ tới transaction end.

### Return khỏi repository method

Repository method return không kết thúc outer `@Transactional` method.

### Tăng connection pool

Không giảm lock lifetime; nhiều waiters hơn có thể chỉ làm database pressure tăng.

### Dùng table lock mạnh cho một tenant

Serialize unrelated rows/tenants. Chọn smallest authoritative lock scope.
