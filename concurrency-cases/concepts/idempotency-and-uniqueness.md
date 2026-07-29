# Idempotency, uniqueness và atomic claim

## Mục đích

Tài liệu định nghĩa vocabulary chung cho durable uniqueness, duplicate-command
prevention và idempotency lifecycle. Mỗi case vẫn phải nêu business key, stored
outcome, transaction boundary và loser behavior cụ thể.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| business key | Bộ field xác định một logical record, không nhất thiết là primary key |
| unique constraint | Database rule cấm nhiều rows có cùng key |
| atomic claim | Một write quyết định duy nhất actor sở hữu key |
| idempotency key | Client-provided key nhận diện cùng logical command |
| request fingerprint | Hash/canonical representation để phát hiện key bị reuse cho payload khác |
| replay | Trả lại stored outcome thay vì chạy side effect lần nữa |
| in-progress | Claim đã tồn tại nhưng command chưa có final outcome |
| upsert | `INSERT ... ON CONFLICT` với no-op hoặc update semantics |

## Uniqueness khác idempotency

Unique constraint trả lời:

```text
Có tối đa một row cho business key này không?
```

Idempotency contract còn phải trả lời:

```text
Duplicate request nhận response nào?
Payload khác dùng cùng key bị xử lý ra sao?
Attempt đang chạy, failed hoặc expired được replay/retry thế nào?
Side effects nào đã commit?
```

> **Nói ngắn gọn:** uniqueness ngăn row thứ hai; idempotency định nghĩa toàn bộ
> hành vi của request thứ hai.

## Không dùng check-then-insert làm correctness boundary

Broken:

```text
SELECT exists(key) -> false
concurrent SELECT exists(key) -> false
INSERT row A
INSERT row B
```

Plain SELECT không reserve absent key. Correct boundary là unique index/constraint
hoặc atomic claim:

```sql
insert into command_claim(scope_id, idempotency_key, status)
values (:scopeId, :key, 'IN_PROGRESS')
on conflict (scope_id, idempotency_key) do nothing;
```

Affected-row `1` là winner; `0` là existing claim. Mọi application instances dùng
cùng PostgreSQL constraint nên không cần JVM-local lock cho uniqueness.

## Business key design

Key phải đúng scope:

```text
(tenant_id, external_reference)
(account_id, idempotency_key)
(provider, provider_event_id)
```

Một globally unique key khi business chỉ unique per tenant gây false conflicts.
Thiếu tenant/scope lại cho phép duplicate logical records.

Xác định normalization trước insert: case sensitivity, Unicode, whitespace, null
semantics và key rotation. PostgreSQL unique constraint mặc định cho phép nhiều
`NULL` values; dùng `NOT NULL`, expression/partial index hoặc `NULLS NOT DISTINCT`
khi contract yêu cầu.

## Constraint và index

Schema migration là authority:

```sql
alter table work_item
    add constraint uk_work_item_business
    unique (tenant_id, external_reference);
```

JPA `@UniqueConstraint` mô tả mapping/schema generation nhưng không thay migration
đã deploy. Production startup không nên giả định Hibernate tự sửa index.

Unique conflict thường có SQLSTATE `23505`. Exception mapping phải kiểm tra đúng
constraint name/business key, không biến mọi integrity violation thành duplicate.

## PostgreSQL conflict behavior

Hai concurrent INSERTs cùng unique key có thể phải chờ transaction đang sở hữu
conflicting index entry:

- winner commit: loser nhận unique violation hoặc `ON CONFLICT` outcome;
- winner rollback: waiter có thể tiếp tục và trở thành winner;
- lock timeout/deadlock: current transaction rollback/fail theo database rules.

Unique check và row visibility không đồng nhất. Ở `READ COMMITTED`,
`ON CONFLICT DO NOTHING` có thể skip do conflict rồi SELECT statement sau mới thấy
winner đã commit.

## JPA/Hibernate flush boundary

`save()` có thể chỉ đưa entity vào persistence context. Unique violation xuất hiện
khi flush hoặc commit:

```java
repository.saveAndFlush(entity);
```

Sau SQLSTATE `23505`, PostgreSQL transaction đã abort; Spring transaction thường
rollback-only. Không catch exception rồi query existing row trong cùng physical
transaction.

Tách:

1. insert attempt trong transaction riêng;
2. catch/mapping ở outer non-transactional coordinator;
3. read/replay existing result trong transaction mới.

## `ON CONFLICT` choices

### `DO NOTHING`

Phù hợp cho claim:

```sql
insert ... on conflict (scope_id, key) do nothing
returning claim_id;
```

Returned ID có value là winner; empty là loser.

### `DO UPDATE`

Chỉ dùng khi duplicate thật sự có merge/update semantics. No-op update chỉ để lấy
ID có thể tạo tuple version, trigger/audit noise và bloat. Đừng overwrite original
fingerprint/response bằng payload duplicate.

## Idempotency lifecycle

Một durable idempotency record thường cần:

```text
scope/key
request fingerprint
status: IN_PROGRESS | SUCCEEDED | FAILED
resource/response reference
created_at, completed_at, expires_at
owner/attempt metadata nếu recovery cần
```

Contract phải định nghĩa:

- cùng key + cùng fingerprint;
- cùng key + khác fingerprint;
- duplicate khi `IN_PROGRESS`;
- retryable versus terminal failure;
- response storage/reconstruction;
- expiry/cleanup và late retry;
- crash giữa claim, side effect và completion.

Claim và database side effect nên commit cùng transaction khi cùng database.
Cross-system side effects cần inbox/outbox/workflow semantics riêng.

## Failure và recovery

- Insert claim rollback: key chưa được sở hữu.
- Winner crash trước commit: waiter có thể tiếp tục.
- Winner commit nhưng response lost: duplicate đọc stored outcome.
- Attempt stuck `IN_PROGRESS`: recovery dựa trên durable state, timeout và owner
  semantics; không xóa claim mù quáng.
- Cleanup: chỉ xóa khi retention/business replay window cho phép.

## Testing

PostgreSQL Testcontainers test cần:

1. two connections cùng attempt exact business key;
2. start/commit/rollback order được điều phối bằng latch;
3. assert one durable row;
4. assert winner/loser domain outcomes;
5. kiểm tra loser chờ winner commit và winner rollback;
6. kiểm tra constraint name/SQLSTATE mapping;
7. mọi futures/waits bounded.

## Observability

Theo dõi:

```text
claim.created
claim.duplicate
claim.payload_mismatch
claim.in_progress
claim.replayed
unique_violation_by_constraint
claim_wait_duration
stuck_claim_age
```

Không log raw idempotency keys/payload nếu chúng chứa secrets hoặc personal data;
dùng scoped hash/correlation ID phù hợp.

## Liên kết

- [DB-006 — Unique constraint concurrency](../postgresql/unique-constraint-concurrency/README.md)
- [Concurrency testing](concurrency-testing.md)
- [Case catalog](../CONCURRENCY_CASE_LIBRARY.md)
