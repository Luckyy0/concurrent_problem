# DB-006 — Unique constraint như atomic concurrency control

## Tóm tắt

Tenant `T-42` gửi hai yêu cầu đồng thời (concurrent requests) để tạo bản ghi work item với cùng external reference `CASE-9001`. Mỗi yêu cầu chạy:

```text
existsByTenantIdAndExternalReference() -> false
save(new WorkItem(...))
```

Hai lệnh SELECT cùng thấy key chưa tồn tại. Nếu schema chỉ áp dụng ràng buộc duy nhất (unique constraint) theo primary key ngẫu nhiên, cả hai lệnh INSERT đều commit và tạo ra hai bản ghi logic (logical records).

Invariant:

```text
Với mỗi (tenant_id, external_reference):
  tồn tại tối đa một work_item.

Hai concurrent attempts cùng key:
  một bên thắng (winner) sẽ tạo row;
  bên thua (loser) nhận kết quả là bản ghi đã tồn tại hoặc trùng lặp (duplicate);
  số lượng row được ghi lại (durable row count) luôn bằng 1.
```

> **Nói ngắn gọn:** không thể khóa một row chưa tồn tại bằng lệnh SELECT thông thường; unique index biến một business key chưa tồn tại thành một quyền sở hữu nguyên tử (atomic claim) trong database.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Request A | Kiểm tra rồi tạo `CASE-9001` |
| Request B | Chạy cùng key trên connection khác |
| `work_item` | Các bản ghi logic có thẩm quyền |
| Hibernate persistence context | Có thể trì hoãn (defer) INSERT tới lúc flush/commit |
| PostgreSQL unique index | Quyết định bên thắng/bên thua (winner/loser) cho business key |
| Spring transaction | Commit hoặc rollback mỗi lần thử (attempt) insert |

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

Điểm tranh chấp (concurrency contention point) là business key chưa tồn tại `(T-42, CASE-9001)`, không phải `work_item_id` ngẫu nhiên. Lệnh kiểm tra tồn tại thông thường không giữ chỗ (reserve) key cho lệnh INSERT sau đó.

Correct boundary:

```sql
unique (tenant_id, external_reference)
```

Khi cả hai lệnh insert nhắm vào cùng một unique key, cơ chế phân xử (arbitration) của PostgreSQL index sẽ giúp một bên thắng. Bên thua có thể chờ kết quả của bên thắng, rồi nhận lỗi `23505`, không làm gì (no-op), hoặc trở thành bên thắng nếu transaction trước đó rollback.

## Expected và actual

| | A | B | Final |
| --- | --- | --- | --- |
| Lệnh kiểm tra (Existence check) | false | false | |
| Lệnh INSERT bị lỗi thiết kế (Broken) | thành công | thành công | 2 rows |
| Ràng buộc duy nhất (Unique constraint) | thắng | chờ rồi trùng lặp/không làm gì (no-op) | 1 row |
| Transaction của bên thắng rollback | rollback | có thể insert thành bên thắng | 1 row |

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| business key | Bộ field định danh bản ghi logic |
| check-then-insert | SELECT kiểm tra rồi INSERT ở statement khác |
| unique constraint | Database rule cấm duplicate key |
| atomic claim | Một lệnh ghi duy nhất quyết định chủ thể sở hữu key |
| upsert | `INSERT ... ON CONFLICT` |
| flush | Hibernate gửi pending SQL tới database |
| SQLSTATE `23505` | PostgreSQL unique-violation signal |
| transaction aborted | Transaction không thể tiếp tục sau SQL error cho tới lúc rollback |

## Điều hướng

- [Broken exists-then-save](broken-code.md)
- [Index, visibility and flush analysis](analysis.md)
- [Constraint, exception mapping and upsert solutions](solutions.md)
- [Deterministic PostgreSQL experiments](experiments.md)
- [Idempotency and uniqueness](../../concepts/idempotency-and-uniqueness.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Hai workers xử lý cùng một phần tử logic (logical item).
- Các tác dụng phụ (side effects)/việc phát hành sự kiện (event publication) ở hạ nguồn bị nhân đôi.
- Các foreign keys/hàng kiểm toán (audit rows) tách về hai IDs ngẫu nhiên.
- Việc dọn dẹp (cleanup) phải chọn ra hàng chuẩn (canonical row) và hợp nhất (merge) các tham chiếu.
- `save()` trả về thành công trước khi flush làm lỗi xuất hiện muộn tại ranh giới commit.
- Triển khai đa phiên bản (Multi-instance deployment) làm cho lock cục bộ trên JVM là không đủ.
- Việc ánh xạ mọi `DataIntegrityViolationException` thành lỗi trùng lặp che giấu các lỗi schema khác.

## Hướng sửa khuyến nghị

1. Triển khai named unique constraint đúng phạm vi nghiệp vụ (business scope).
2. Dùng INSERT làm một quyền sở hữu nguyên tử (atomic claim); loại bỏ bước kiểm tra trước (pre-check) khỏi luồng chính tả (correctness path).
3. Bắt buộc flush trong transaction của insert attempt để xung đột xuất hiện tại ranh giới có thể ánh xạ (map) được.
4. Xử lý ngoại lệ (Catch) ở phía điều phối vòng ngoài (outer coordinator); không truy vấn bản ghi đã tồn tại trong transaction đã bị hủy (aborted).
5. Hoặc dùng `INSERT ... ON CONFLICT DO NOTHING RETURNING id`, rồi đọc bên thắng trong statement/transaction phù hợp.
6. Tách biệt tính duy nhất bền bỉ (durable uniqueness) khỏi hợp đồng API phát lại đầy đủ (full idempotency response-replay contract).

## Phạm vi

Case này chỉ bảo vệ tính duy nhất và tính nguyên tử của quyền sở hữu (claim). Dấu vân tay của yêu cầu (request fingerprint), `IN_PROGRESS`, kết quả được lưu (stored response), thời gian hết hạn (expiry) và phát lại API đầy đủ được triển khai sâu ở `BANK-005`. Thay đổi trùng lặp trên cùng một bản ghi đã tồn tại là một bất biến (invariant) khác.
