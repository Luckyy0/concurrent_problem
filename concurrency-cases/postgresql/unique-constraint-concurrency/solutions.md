# Giải pháp unique constraint, exception mapping và upsert

## Mục tiêu thiết kế

Database phải thực thi (enforce):

```text
count(tenant_id, external_reference) <= 1
```

Ứng dụng phải ánh xạ (map) một bên thắng/bên thua rõ ràng mà không tái sử dụng transaction đã thất bại.

## Solution 1 — Named unique constraint

Migration:

```sql
alter table work_item
    add constraint uk_work_item_tenant_reference
    unique (tenant_id, external_reference);
```

Ánh xạ thực thể (Entity mapping) để ý định của mã nguồn/schema đều rõ ràng:

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

Công cụ migration hoặc kiểm tra schema mới là nguồn có thẩm quyền trên production; annotation không thay thế DDL đã được triển khai.

> **Nói ngắn gọn:** ràng buộc bảo vệ tính bất biến; mã Java chỉ quyết định cách diễn giải bên thắng và lỗi trùng lặp cho phía gọi (caller).

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

Trình điều phối bên ngoài (Outer coordinator) không có transaction:

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

Transaction thực hiện insert của A đã kết thúc trước khối catch. Trình đọc dữ liệu trùng lặp (Duplicate reader) sử dụng một transaction vật lý/snapshot mới nên không gặp trạng thái bị hủy (aborted state).

Trình phân loại duyệt qua chuỗi nguyên nhân, xác nhận mã PostgreSQL SQLSTATE `23505` và `ServerErrorMessage.getConstraint()`. Nếu siêu dữ liệu của driver không khả dụng, hãy ưu tiên xử lý lỗi theo hướng đóng (fail closed) thay vì ánh xạ mọi lỗi toàn vẹn dữ liệu thành lỗi trùng lặp.

### Hành vi của bên thua

- bên thắng commit: bên thua chờ hoặc nhận lỗi `23505`, sau đó đọc bản ghi đã tồn tại;
- bên thắng rollback: lệnh insert đang chờ có thể thành công và trả về `CREATED`;
- vi phạm ràng buộc khác: lan truyền lỗi (propagate);
- không tìm thấy bản ghi khi đọc: coi đó là một bất thường vận hành/thử lại trong giới hạn, không bịa ra (fabricate) ID.

## Solution 3 — `ON CONFLICT DO NOTHING RETURNING`

Repository dùng `JdbcTemplate` để có ngữ nghĩa trả về (returning semantics) rõ ràng:

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

Lệnh `DO NOTHING` không hủy transaction do lỗi trùng lặp đã được dự kiến. Việc tách biệt transaction đọc vẫn giúp tính hiển thị (visibility) nhất quán giữa mức RC/RR và giữ ranh giới API rõ ràng.

## Đánh đổi khi dùng `DO UPDATE RETURNING`

Một pattern:

```sql
insert into work_item(...)
values (...)
on conflict (tenant_id, external_reference)
do update set external_reference = excluded.external_reference
returning work_item_id;
```

Cách này luôn trả về ID nhưng lệnh UPDATE không có tác dụng thực tế có thể:

- tạo ra một phiên bản tuple mới, gây phình to dữ liệu (bloat);
- kích hoạt các trigger cập nhật hoặc tiến trình kiểm toán (audit);
- xin cấp khóa hàng (row lock) mạnh hơn;
- làm cho trường `updated_at` hoặc cơ chế CDC thay đổi sai ngữ nghĩa.

Chỉ dùng `DO UPDATE` khi dữ liệu trùng lặp có ngữ nghĩa hợp nhất (merge semantics) hợp lệ. Không ghi đè tải trọng hoặc dấu vân tay ban đầu chỉ để lấy ID.

## Dấu vân tay của tải trọng (Payload fingerprint)

Nếu tham chiếu bên ngoài được dùng như một khóa lũy đẳng (idempotency key), hãy lưu thêm dấu vân tay:

```sql
alter table work_item
    add column request_fingerprint varchar(64) not null;
```

Loser đọc existing:

```text
cùng dấu vân tay -> trả về bản ghi hiện tại
khác dấu vân tay -> từ chối vì key được sử dụng lại cho yêu cầu khác (KEY_REUSED_WITH_DIFFERENT_REQUEST)
```

Cách chuẩn hóa hoặc băm (hash) phải ổn định và được đánh phiên bản. Toàn bộ vòng đời của phản hồi/trạng thái thuộc về các trường hợp lũy đẳng sau này; DB-006 không giả định tính duy nhất của hàng là sự phát lại đầy đủ (full replay).

## Chuyển đổi (Migration) khi trên production đã có dữ liệu trùng lặp

Detect:

```sql
select tenant_id, external_reference, count(*)
from work_item
group by tenant_id, external_reference
having count(*) > 1;
```

Rollout:

1. dừng/chuyển hướng các lệnh ghi hoặc triển khai một luồng nguyên tử tạm thời;
2. chọn các hàng chuẩn (canonical rows) và ánh xạ lại các thành phần phụ thuộc theo chính sách nghiệp vụ;
3. xác minh không còn dữ liệu trùng lặp;
4. tạo unique index, thường cân nhắc tùy chọn `CONCURRENTLY` để giảm tình trạng bị khóa;
5. gắn ràng buộc bằng index đã có nếu phù hợp;
6. triển khai mã nguồn để ánh xạ ràng buộc một cách chính xác;
7. giám sát các lỗi `23505`.

Lệnh `CREATE UNIQUE INDEX CONCURRENTLY` không chạy trong một khối transaction thông thường và vẫn thất bại nếu tồn tại các hàng trùng lặp. Sau khi thất bại, phải kiểm tra trạng thái index không hợp lệ.

## Giải pháp thay thế: SERIALIZABLE

Mức độ SSI có thể hủy bỏ tình trạng tranh chấp khi kiểm tra rồi chèn, nhưng tính duy nhất lại là một bất biến trực tiếp:

- ràng buộc hoạt động ở mọi mức độ cô lập;
- bảo vệ khỏi các thao tác SQL trực tiếp hoặc từ các dịch vụ khác;
- tín hiệu xung đột được gắn vào đúng key;
- không cần thử lại (retry) toàn bộ một transaction tùy ý.

Chỉ dùng `SERIALIZABLE` cho các bất biến vị từ (predicate invariants) rộng hơn, không thay thế unique constraint cho các business key chính xác.

## Transaction, quá thời gian chờ (timeout) và sự cố (crash)

- Vi phạm ràng buộc sẽ rollback lần thử của bên thua.
- Quá thời gian chờ khóa/tiến trình bế tắc (deadlock) không được ánh xạ thành lỗi trùng lặp.
- Bên thắng gặp sự cố trước khi commit: bên chờ có thể trở thành bên thắng.
- Bên thắng commit nhưng phản hồi bị mất: yêu cầu tiếp theo sẽ đọc bản ghi đã tồn tại.
- Các tác dụng phụ bên ngoài chỉ xảy ra sau khi đã có quyền sở hữu bền bền hoặc thông qua outbox.
- Không giữ transaction thực hiện insert ở trạng thái mở trong lúc thực hiện I/O từ xa.

## So sánh sự đánh đổi

| Cách | Tính nguyên tử (Atomicity) | Tín hiệu bên thua | Tải trên DB | Độ phức tạp |
| --- | --- | --- | --- | --- |
| Unique + caught `23505` | Key của database | exception + read | Luồng ngoại lệ | Trung bình |
| `DO NOTHING RETURNING` | Key của database | kết quả rỗng | Không có ngoại lệ dự kiến | Trung bình |
| `DO UPDATE RETURNING` | Key của database + update | hàng trả về | Cập nhật thêm/bloat | Nhạy cảm với ngữ nghĩa |
| SERIALIZABLE check/insert | Vị từ SSI | thử lại 40001 | Thử lại transaction | Cao hơn |
| JVM lock | Chỉ trong tiến trình | chờ cục bộ | Không bao quát toàn DB | Sai trong đa phiên bản (multi-instance) |

## Danh sách kiểm tra (Checklist) khi triển khai trên production

- [ ] Phạm vi/chuẩn hóa/ngữ nghĩa null của business key là chính xác.
- [ ] Ràng buộc duy nhất có tên tồn tại trong schema đã triển khai.
- [ ] Xung đột insert bị ép buộc (forced) xảy ra tại ranh giới flush được kiểm soát.
- [ ] Khối Catch nằm ngoài transaction đã thất bại.
- [ ] Trình phân loại (Classifier) kiểm tra mã `23505` và ràng buộc chính xác.
- [ ] `DO UPDATE` chỉ dùng khi có ngữ nghĩa cập nhật thực sự.
- [ ] Sự sai lệch tải trọng đối với bản ghi trùng lặp có kết quả xử lý rõ ràng.
- [ ] Các trường hợp bên thắng rollback, quá thời gian phản hồi và hành vi khi gặp sự cố đều được kiểm thử.
- [ ] Kiểm thử tích hợp đa phiên bản (Multi-instance integration test) xác nhận chính xác chỉ có một hàng bền bỉ.
- [ ] Có sẵn tài liệu hướng dẫn (runbook) dọn dẹp dữ liệu migration/xây dựng index.
