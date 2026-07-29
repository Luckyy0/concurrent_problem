# DB-006 — Unique constraint như atomic concurrency control

## Tóm tắt

Tenant `T-42` gửi hai concurrent requests tạo work item với cùng external reference
`CASE-9001`. Mỗi request chạy:

```text
existsByTenantIdAndExternalReference() -> false
save(new WorkItem(...))
```

Hai SELECTs cùng thấy key chưa tồn tại. Nếu schema chỉ unique theo random primary
key, hai INSERTs đều commit và tạo hai logical records.

Invariant:

```text
Với mỗi (tenant_id, external_reference):
  tồn tại tối đa một work_item.

Hai concurrent attempts cùng key:
  một winner tạo row;
  loser nhận existing/duplicate outcome;
  durable row count luôn bằng 1.
```

> **Nói ngắn gọn:** không thể khóa một row chưa tồn tại bằng plain SELECT; unique
> index biến absent business key thành một atomic database claim.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Request A | Check rồi tạo `CASE-9001` |
| Request B | Chạy cùng key trên connection khác |
| `work_item` | Authoritative logical records |
| Hibernate persistence context | Có thể defer INSERT tới flush/commit |
| PostgreSQL unique index | Quyết định winner/loser cho business key |
| Spring transaction | Commit hoặc rollback mỗi insert attempt |

Broken final:

```text
row id=A, tenant=T-42, external_reference=CASE-9001
row id=B, tenant=T-42, external_reference=CASE-9001
```

## Transaction boundary và contention point

Broken sequence:

```text
BEGIN
  SELECT exists(business_key)
  if false:
    persist entity with random primary key
COMMIT
```

Contention point là absent business key `(T-42, CASE-9001)`, không phải random
`work_item_id`. Plain existence check không reserve key cho later INSERT.

Correct boundary:

```sql
unique (tenant_id, external_reference)
```

Khi both inserts target cùng unique key, PostgreSQL index arbitration làm một
attempt win. Loser có thể chờ winner outcome, rồi nhận `23505`, no-op, hoặc trở
thành winner nếu transaction trước rollback.

## Expected và actual

| | A | B | Final |
| --- | --- | --- | --- |
| Existence check | false | false | |
| Broken INSERT | success | success | 2 rows |
| Unique constraint | winner | wait rồi duplicate/no-op | 1 row |
| Winner rollback | rollback | có thể insert thành winner | 1 row |

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| business key | Bộ field định danh logical record |
| check-then-insert | SELECT kiểm tra rồi INSERT ở statement khác |
| unique constraint | Database rule cấm duplicate key |
| atomic claim | Một write duy nhất quyết định actor sở hữu key |
| upsert | `INSERT ... ON CONFLICT` |
| flush | Hibernate gửi pending SQL tới database |
| SQLSTATE `23505` | PostgreSQL unique-violation signal |
| transaction aborted | Transaction không thể tiếp tục sau SQL error cho tới rollback |

## Điều hướng

- [Broken exists-then-save](broken-code.md)
- [Index, visibility and flush analysis](analysis.md)
- [Constraint, exception mapping and upsert solutions](solutions.md)
- [Deterministic PostgreSQL experiments](experiments.md)
- [Idempotency and uniqueness](../../concepts/idempotency-and-uniqueness.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Hai workers xử lý cùng logical item.
- Downstream side effects/event publication bị nhân đôi.
- Foreign keys/audit rows tách về hai random IDs.
- Cleanup phải chọn canonical row và merge references.
- `save()` trả success trước flush làm lỗi xuất hiện muộn ở commit boundary.
- Multi-instance deployment làm JVM-local lock không đủ.
- Mapping mọi `DataIntegrityViolationException` thành duplicate che lỗi schema khác.

## Hướng sửa khuyến nghị

1. Deploy named unique constraint đúng business scope.
2. Dùng INSERT làm atomic claim; bỏ pre-check khỏi correctness path.
3. Force flush trong insert-attempt transaction để conflict xuất hiện ở boundary
   có thể map.
4. Catch ở outer coordinator; không query existing trong transaction đã abort.
5. Hoặc dùng `INSERT ... ON CONFLICT DO NOTHING RETURNING id`, rồi đọc winner
   trong statement/transaction phù hợp.
6. Tách durable uniqueness khỏi full idempotency response-replay contract.

## Phạm vi

Case này chỉ bảo vệ durable uniqueness và atomic claim. Request fingerprint,
`IN_PROGRESS`, stored response, expiry và full API replay được triển khai sâu ở
`BANK-005`. Duplicate mutation trên cùng existing record là invariant khác.
