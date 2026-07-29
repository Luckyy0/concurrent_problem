# DB-003 — Non-repeatable read làm sai business decision

## Tóm tắt

Merchant `M-42` có chính sách tự động duyệt refund với giới hạn `100.00`, revision
`7`. Transaction A đọc policy và quyết định refund `80.00` đủ điều kiện. Trước
khi A ghi decision, transaction B hạ giới hạn xuống `50.00`, tăng revision thành
`8` rồi commit.

A chạy SELECT lần hai ở PostgreSQL `READ COMMITTED`. Statement snapshot mới thấy
revision `8`, nhưng code vẫn giữ kết quả `approved` đã tính từ giới hạn `100.00`.
Decision cuối cùng vì thế mang revision `8` dù amount `80.00` không được policy
revision đó cho phép.

Invariant:

```text
Mỗi refund decision phải được đánh giá từ một policy snapshot nhất quán.

Nếu decision = APPROVED:
  amount <= autoRefundLimit của chính policyRevision được lưu cùng decision.

Không được:
  quyết định bằng revision 7
  nhưng ghi bằng chứng/audit là revision 8.
```

> **Nói ngắn gọn:** hai SELECT trong cùng transaction `READ COMMITTED` không mặc
> nhiên nói về cùng database snapshot; ghép dữ liệu của hai lần đọc có thể tạo
> một decision chưa từng hợp lệ ở bất kỳ thời điểm nào.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Refund evaluator A | Đọc policy, chạy rule và lưu decision |
| Risk administrator B | Thay đổi refund limit của merchant |
| `merchant_refund_policy` | Authoritative current policy |
| `refund_decision` | Audit record của kết quả đã công bố |
| PostgreSQL MVCC | Cấp statement snapshot cho từng SELECT |
| Spring transaction | Giữ hai SELECT và INSERT trong một physical transaction |

Initial committed policy:

```text
merchant_id       = M-42
auto_refund_limit = 100.00
active            = true
revision          = 7
```

Concurrent committed policy:

```text
merchant_id       = M-42
auto_refund_limit = 50.00
active            = true
revision          = 8
```

Broken decision:

```text
amount             = 80.00
outcome            = APPROVED
evaluated_limit    = 100.00
policy_revision    = 8
```

## Transaction boundary và contention point

A đi qua Spring proxy:

```text
BEGIN READ COMMITTED
  SELECT policy                         -> limit=100, revision=7
  decide APPROVED in Java
  ... concurrent B commits ...
  SELECT policy again for audit fields  -> limit=50, revision=8
  INSERT refund_decision                -> APPROVED, revision=8
COMMIT
```

B chạy trong independent transaction:

```text
BEGIN
  UPDATE merchant_refund_policy
     SET auto_refund_limit=50, revision=8
COMMIT
```

Contention point là logical row `merchant_refund_policy(M-42)`. Hai plain SELECT
của A không giữ row lock. B có thể update và commit giữa chúng; INSERT của A lại
ghi row khác nên không có direct write conflict buộc PostgreSQL từ chối.

## Expected và actual

| Bước | Broken assumption | PostgreSQL `READ COMMITTED` |
| --- | --- | --- |
| SELECT đầu của A | Policy ổn định tới hết transaction | Snapshot S1 thấy revision 7 |
| B commit | A không bị ảnh hưởng | Revision 8 trở thành committed |
| SELECT sau của A | Vẫn là cùng policy | Snapshot S2 thấy revision 8 |
| A INSERT | Decision và evidence nhất quán | Decision rev7 bị gắn evidence rev8 |
| Conflict signal | Có exception nếu policy đổi | Không có; A và B có thể cùng commit |

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| non-repeatable read | Cùng logical row được đọc lại và thấy committed value khác |
| statement snapshot | Database view riêng của mỗi statement ở `READ COMMITTED` |
| transaction snapshot | Stable view dùng xuyên transaction ở `REPEATABLE READ` |
| coherent snapshot | Một bộ field cùng thuộc một policy version |
| revalidation | Kiểm tra lại assumption ngay tại authoritative write boundary |
| policy revision | Version nghiệp vụ xác định policy dùng để ra quyết định |
| row lock | Lock giữ updater khác chờ tới commit hoặc rollback |
| serialization order | Thứ tự tuần tự hợp lệ tương đương với concurrent execution |

## Điều hướng

- [Broken two-read decision](broken-code.md)
- [Statement-snapshot analysis](analysis.md)
- [Snapshot, validation and locking solutions](solutions.md)
- [Deterministic PostgreSQL experiments](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Isolation levels](../../concepts/isolation-levels.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Audit record tham chiếu policy revision không thực sự authorize decision.
- Refund vượt current limit vẫn được công bố là auto-approved.
- Reconciliation không thể tái dựng lý do quyết định từ stored evidence.
- Hai lần đọc có thể làm response, event và database row chứa các revision khác
  nhau.
- Lỗi không tạo exception nên dễ bị hiểu nhầm là business logic bug ngẫu nhiên.
- Nhiều application instances làm cửa sổ interleaving rộng hơn; JVM-local lock
  không bảo vệ row dùng chung.

## Hướng sửa khuyến nghị

Trước tiên chọn semantics:

1. Nếu decision chỉ cần nhất quán với policy tại lúc đánh giá, đọc policy một lần,
   truyền một immutable snapshot xuyên suốt và lưu đầy đủ revision/evidence.
   Policy history nên immutable để revision cũ còn audit được.
2. Nếu policy phải còn là current version tại lúc ghi decision, dùng conditional
   validation ở database và xử lý affected-row `0` bằng một transaction mới.
3. Nếu policy update không được phép vượt qua decision đang chạy, đọc row bằng
   `FOR SHARE`/pessimistic read; updater phải chờ hoặc timeout.
4. `REPEATABLE READ` loại non-repeatable read, nhưng không tự biến snapshot cũ
   thành “latest policy at commit”.

Cơ chế phải khớp invariant, không chỉ làm hai SELECT trả về giống nhau.

## Phạm vi

Case này chỉ xét một logical row được đọc lại. Predicate query trả về tập row thay
đổi là phantom và thuộc DB-004. Lost update thuộc DB-001; dirty-read expectation
thuộc DB-002.
