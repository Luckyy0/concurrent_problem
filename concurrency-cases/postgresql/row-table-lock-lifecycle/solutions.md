# Giải pháp pessimistic, atomic và optimistic

## Mục tiêu thiết kế

Chọn conflict mechanism từ invariant:

```text
Writer cần độc quyền current-state decision?
Operation diễn đạt bằng conditional SQL?
Hay conflict nên fail/retry thay vì block?
Observer có thực sự cần locking read?
```

## Solution 1 — Explicit `PESSIMISTIC_WRITE`

Repository dành riêng cho mutation:

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

Service:

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

Hibernate/PostgreSQL dialect phải sinh locking clause tương đương:

```sql
select tenant_id, quota, revision
from tenant_quota
where tenant_id=:tenantId
for update;
```

A acquires row lock; B incompatible writer/locking reader blocks, timeout hoặc
fails. Lock sống tới outer commit/rollback.

> **Nói ngắn gọn:** query xin lock, transaction giữ lock; thiếu một trong hai thì
> read-modify-write critical section không tồn tại.

### Boundary requirements

- Call đi qua Spring proxy.
- Locking query chạy trong active transaction.
- Không remote I/O/executor wait khi giữ lock.
- Multiple rows được lock theo deterministic key order.
- `lock_timeout`/overall deadline bounded.
- Revalidate state sau lock acquisition.

## Solution 2 — Native `FOR UPDATE` với timeout rõ

Khi dialect/hint mapping cần explicit:

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

Per transaction:

```sql
set local lock_timeout = '500ms';
```

Do not hard-code one timeout for all workloads; fit within end-to-end deadline.
SQLSTATE `55P03` maps to retryable busy/timeout, không map `NOT_FOUND`.

## Solution 3 — Atomic conditional SQL

Nếu business rule nằm trong predicate, bỏ pre-read/long lock window:

```sql
update tenant_quota
set quota = :requestedQuota,
    revision = revision + 1
where tenant_id = :tenantId
  and revision = :expectedRevision
  and :requestedQuota >= 0;
```

Repository returns affected-row:

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

UPDATE acquires row lock. Nếu chờ concurrent writer, PostgreSQL evaluates current
row/predicate according to command semantics; stale revision gives affected-row
`0`. Map to conflict/reload, không overwrite.

Ưu điểm: short lock lifetime, one round-trip mutation. Không phù hợp khi validation
phức tạp cần nhiều rows/side effects.

## Solution 4 — `@Version` thay blocking

Entity:

```java
@Version
@Column(nullable = false)
private long revision;
```

Hibernate:

```sql
update tenant_quota
set quota=?, revision=?
where tenant_id=? and revision=?;
```

Affected-row `0` -> optimistic lock exception. B không block trước decision nhưng
loser fail/retry. Retry nếu safe phải bounded, backoff/jitter, transaction mới và
reload state. Hot row có retry amplification.

Pessimistic lock phù hợp khi conflict thường xuyên/critical section ngắn; optimistic
phù hợp khi conflict hiếm và caller chịu retry/reject.

## Solution 5 — Dashboard giữ plain SELECT

```java
@Transactional(readOnly = true)
public QuotaView dashboard(UUID tenantId) {
    return quotas.findViewByTenantId(tenantId)
        .orElseThrow(QuotaNotFoundException::new);
}
```

Contract:

```text
return last committed quota visible to statement snapshot
never expose uncommitted writer value
do not wait merely because writer holds row lock
```

Nếu dashboard cần read-after-write, route/transaction/version token semantics phải
được thiết kế riêng; locking every observer không phải default.

## Solution 6 — Explicit table lock chỉ khi scope thật sự là table

```sql
begin;
lock table tenant_quota in share row exclusive mode;
-- short operation requiring this exact compatibility
commit;
```

Chọn exact mode từ PostgreSQL compatibility matrix. `ACCESS EXCLUSIVE` chặn cả
plain SELECT và thường phù hợp DDL/very strong maintenance, không phải single
tenant mutation.

Table locks:

- ảnh hưởng unrelated rows;
- tăng wait/deadlock risk;
- giữ tới transaction end;
- phải acquire theo deterministic table order.

## Lock timeout mapping

```java
try {
    return lockedAttempt.changeQuota(command);
} catch (CannotAcquireLockException ex) {
    return QuotaChangeResult.busyRetryable();
}
```

Classifier/cause cần phân biệt:

- `55P03` lock not available/timeout;
- `40P01` deadlock victim;
- query/statement timeout;
- connection acquisition timeout.

Current failed transaction rollback trước retry. Outer coordinator gọi attempt bean
mới; không catch rồi tiếp tục JPA work trong same transaction.

## Failure behavior

- Holder commit: lock release, waiter continues on current state.
- Holder rollback/crash: changes rollback, waiter continues on prior state.
- Waiter timeout: waiter transaction rollback; holder unaffected.
- Deadlock victim: whole victim transaction rollback.
- Client timeout: commit outcome có thể ambiguous; inspect durable revision before
  replay.
- Duplicate command: separate idempotency contract, không giải bằng row lock alone.

## Trade-off comparison

| Cách | Conflict behavior | Read latency | Write contention | Multi-instance |
| --- | --- | --- | --- | --- |
| `FOR UPDATE` | loser blocks/timeout | Locking readers wait | Serialized hot row | Có |
| Atomic conditional SQL | affected-row `0`/short wait | Không pre-read | Short | Có |
| `@Version` | loser fails/retries | Plain reads | Retry under conflict | Có |
| Plain dashboard SELECT | no row-lock wait | Low, committed-stale | Không protect writer | Observer only |
| Table lock | broad block/timeout | Mode-dependent | Broad | Có |
| JVM lock | local wait | Local | Không cross-node | Không |

## Recommendation

Quota writer cần complex current-state validation: `PESSIMISTIC_WRITE` trong short
transaction. Simple revision/predicate mutation: conditional SQL hoặc `@Version`.
Dashboard giữ plain read. Explicit table lock chỉ dùng khi invariant thật sự
table-wide.

## Production checklist

- [ ] SQL generated cho intended lock mode đã được verify.
- [ ] Locking query nằm trong active proxy-created transaction.
- [ ] Lock acquisition và commit/rollback points visible trong code.
- [ ] Không remote I/O sau lock acquisition.
- [ ] Multiple resources có deterministic order.
- [ ] `lock_timeout` nằm trong overall deadline.
- [ ] Lock timeout/deadlock retry dùng transaction mới.
- [ ] Plain readers có committed-staleness contract.
- [ ] `pg_blocking_pids`, lock waits và transaction age được monitor.
- [ ] Tests dùng PostgreSQL thật và bounded coordination.
