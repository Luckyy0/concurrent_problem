# Giải pháp unique constraint, exception mapping và upsert

## Mục tiêu thiết kế

Database phải enforce:

```text
count(tenant_id, external_reference) <= 1
```

Application phải map one winner/loser rõ ràng mà không reuse failed transaction.

## Solution 1 — Named unique constraint

Migration:

```sql
alter table work_item
    add constraint uk_work_item_tenant_reference
    unique (tenant_id, external_reference);
```

Entity mapping để code/schema intent cùng rõ:

```java
@Entity
@Table(
    name = "work_item",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_work_item_tenant_reference",
        columnNames = {"tenant_id", "external_reference"}
    )
)
public class WorkItem {
    // Same fields/factory as broken-code.md
}
```

Migration tool/schema validation mới là production authority; annotation không
thay deployed DDL.

> **Nói ngắn gọn:** constraint bảo vệ invariant; Java code chỉ quyết định cách
> diễn giải winner và duplicate cho caller.

## Solution 2 — Flush trong transaction riêng, catch bên ngoài

Insert attempt:

```java
@Service
public class WorkItemInsertAttempt {
    private final WorkItemRepository workItems;

    public WorkItemInsertAttempt(WorkItemRepository workItems) {
        this.workItems = workItems;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public UUID insert(UUID tenantId, String externalReference) {
        WorkItem saved = workItems.saveAndFlush(
            WorkItem.open(tenantId, externalReference)
        );
        return saved.getId();
    }
}
```

Reader:

```java
@Service
public class WorkItemReader {
    private final WorkItemRepository workItems;

    public WorkItemReader(WorkItemRepository workItems) {
        this.workItems = workItems;
    }

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        readOnly = true
    )
    public WorkItemView findByBusinessKey(
        UUID tenantId,
        String externalReference
    ) {
        return workItems.findViewByTenantIdAndExternalReference(
            tenantId,
            externalReference
        ).orElseThrow(ExistingWorkItemNotVisibleException::new);
    }
}
```

Outer coordinator không có transaction:

```java
@Service
public class WorkItemCreator {
    private static final String BUSINESS_CONSTRAINT =
        "uk_work_item_tenant_reference";

    private final WorkItemInsertAttempt insertAttempt;
    private final WorkItemReader reader;
    private final UniqueViolationClassifier classifier;

    public WorkItemCreator(
        WorkItemInsertAttempt insertAttempt,
        WorkItemReader reader,
        UniqueViolationClassifier classifier
    ) {
        this.insertAttempt = insertAttempt;
        this.reader = reader;
        this.classifier = classifier;
    }

    public CreateWorkItemResult create(
        UUID tenantId,
        String externalReference
    ) {
        try {
            UUID id = insertAttempt.insert(tenantId, externalReference);
            return CreateWorkItemResult.created(id);
        } catch (DataIntegrityViolationException ex) {
            if (!classifier.isUniqueViolation(
                ex,
                BUSINESS_CONSTRAINT
            )) {
                throw ex;
            }

            WorkItemView existing = reader.findByBusinessKey(
                tenantId,
                externalReference
            );
            return CreateWorkItemResult.existing(existing.id());
        }
    }
}
```

A insert transaction đã kết thúc trước catch. Duplicate reader dùng new physical
transaction/snapshot nên không gặp aborted state.

Classifier traverse cause chain, xác nhận PostgreSQL SQLSTATE `23505` và
`ServerErrorMessage.getConstraint()`. Nếu driver metadata unavailable, fail closed
thay vì map mọi integrity error thành duplicate.

### Loser behavior

- winner commit: loser waits/gets `23505`, then reads existing;
- winner rollback: waiting insert may succeed và return `CREATED`;
- other constraint violation: propagate;
- read existing missing: treat as operational anomaly/retry bounded, không fabricate
  ID.

## Solution 3 — `ON CONFLICT DO NOTHING RETURNING`

Repository dùng `JdbcTemplate` để có returning semantics rõ:

```java
@Repository
public class WorkItemClaims {
    private final JdbcTemplate jdbc;

    public WorkItemClaims(JdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<UUID> tryInsert(
        UUID tenantId,
        String externalReference
    ) {
        List<UUID> ids = jdbc.query(
            """
            insert into work_item(
                work_item_id,
                tenant_id,
                external_reference,
                status
            )
            values (?, ?, ?, 'OPEN')
            on conflict (tenant_id, external_reference) do nothing
            returning work_item_id
            """,
            (rs, rowNum) -> rs.getObject(1, UUID.class),
            UUID.randomUUID(),
            tenantId,
            externalReference
        );
        return ids.stream().findFirst();
    }
}
```

Transactional attempt returns `Optional<UUID>`:

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public Optional<UUID> claim(UUID tenantId, String reference) {
    return claims.tryInsert(tenantId, reference);
}
```

Outer coordinator:

```java
Optional<UUID> created = claimAttempt.claim(tenantId, reference);
if (created.isPresent()) {
    return CreateWorkItemResult.created(created.orElseThrow());
}

WorkItemView existing = reader.findByBusinessKey(tenantId, reference);
return CreateWorkItemResult.existing(existing.id());
```

`DO NOTHING` không abort transaction do expected duplicate. Tách reader transaction
vẫn giúp visibility nhất quán giữa RC/RR và giữ API boundary rõ.

## `DO UPDATE RETURNING` trade-off

Một pattern:

```sql
insert into work_item(...)
values (...)
on conflict (tenant_id, external_reference)
do update set external_reference = excluded.external_reference
returning work_item_id;
```

Nó luôn trả ID nhưng no-op UPDATE có thể:

- tạo new tuple version/bloat;
- chạy update triggers/audit;
- acquire stronger row lock;
- làm `updated_at`/CDC thay đổi sai nghĩa.

Chỉ dùng `DO UPDATE` khi duplicate có legitimate merge semantics. Không overwrite
original payload/fingerprint chỉ để lấy ID.

## Payload fingerprint

Nếu external reference được dùng như idempotency key, lưu fingerprint:

```sql
alter table work_item
    add column request_fingerprint varchar(64) not null;
```

Loser đọc existing:

```text
same fingerprint    -> return existing
different fingerprint -> reject KEY_REUSED_WITH_DIFFERENT_REQUEST
```

Cách canonicalize/hash phải stable và versioned. Full response/status lifecycle
thuộc idempotency cases sau; DB-006 không giả định row uniqueness là full replay.

## Migration khi production đã có duplicates

Detect:

```sql
select tenant_id, external_reference, count(*)
from work_item
group by tenant_id, external_reference
having count(*) > 1;
```

Rollout:

1. dừng/route writes hoặc deploy temporary atomic path;
2. chọn canonical rows và remap dependents theo business policy;
3. verify không duplicates;
4. tạo unique index, thường cân nhắc `CONCURRENTLY` để giảm blocking;
5. attach constraint bằng existing index khi phù hợp;
6. deploy code mapping exact constraint;
7. monitor `23505`.

`CREATE UNIQUE INDEX CONCURRENTLY` không chạy trong ordinary transaction block và
vẫn fail nếu duplicate rows tồn tại. Sau failure phải inspect invalid index state.

## Alternative: `SERIALIZABLE`

SSI có thể abort check-then-insert race, nhưng uniqueness là direct invariant:

- constraint hoạt động ở mọi isolation level;
- bảo vệ direct SQL/other services;
- conflict signal gắn đúng key;
- không cần retry toàn arbitrary transaction.

Dùng `SERIALIZABLE` cho broader predicate invariants, không thay unique constraint
cho exact business key.

## Transaction, timeout và crash

- Constraint violation rollback loser attempt.
- Lock timeout/deadlock không được map thành duplicate.
- Winner crash trước commit: waiter có thể win.
- Winner commit/response lost: next request reads existing.
- External side effects chỉ sau durable claim hoặc qua outbox.
- Không giữ insert transaction open quanh remote I/O.

## Trade-off comparison

| Cách | Atomicity | Loser signal | DB load | Complexity |
| --- | --- | --- | --- | --- |
| Unique + caught `23505` | Database key | exception + read | Exception path | Medium |
| `DO NOTHING RETURNING` | Database key | empty result | Không expected exception | Medium |
| `DO UPDATE RETURNING` | Database key + update | returned row | Extra update/bloat | Semantics-sensitive |
| SERIALIZABLE check/insert | SSI predicate | `40001` retry | Retry transaction | Higher |
| JVM lock | Process only | local wait | Không DB-wide | Incorrect multi-instance |

## Production checklist

- [ ] Business key scope/normalization/null semantics đúng.
- [ ] Named unique constraint tồn tại trong deployed schema.
- [ ] Insert conflict được forced tại controlled flush boundary.
- [ ] Catch nằm ngoài failed transaction.
- [ ] Classifier kiểm tra `23505` và exact constraint.
- [ ] `DO UPDATE` chỉ dùng khi có true update semantics.
- [ ] Duplicate payload mismatch có explicit outcome.
- [ ] Winner rollback, response timeout và crash behavior được test.
- [ ] Multi-instance integration test assert exactly one durable row.
- [ ] Migration cleanup/index build runbook tồn tại.
