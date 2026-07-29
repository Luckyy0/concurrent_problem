# DB-005 — Write skew trên invariant nhiều rows

## Tóm tắt

Roster ca đêm `NOC-NIGHT-42` có hai operators đang on-call: Alice và Bob. Mỗi
người gửi request rời ca trong một Spring transaction riêng. Cả hai cùng đọc
`onCallCount = 2`, kết luận người còn lại vẫn trực, rồi update row của chính mình
thành `on_call = false`.

Hai UPDATEs ghi hai rows khác nhau nên không có direct write conflict. Cả hai có
thể commit; final roster không còn operator nào.

Invariant:

```text
Với mỗi active roster:
  count(assignments where on_call = true) >= 1

Initial: Alice=true, Bob=true
Expected: chỉ một request được accepted
Broken final: Alice=false, Bob=false
```

> **Nói ngắn gọn:** mỗi row update riêng lẻ đều hợp lệ, nhưng tổ hợp hai commits
> làm sai quy tắc của cả tập rows; đó là write skew.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Alice transaction A | Đọc roster rồi tắt on-call của Alice |
| Bob transaction B | Đọc cùng roster rồi tắt on-call của Bob |
| `on_call_roster` | Stable parent/roster identity |
| `on_call_assignment` | Một row cho mỗi operator trong roster |
| Hibernate | Dirty-check entity và flush versioned UPDATE |
| PostgreSQL MVCC | Cấp snapshot và lock từng updated row |

Initial committed rows:

```text
roster=NOC-NIGHT-42
Alice: on_call=true, version=0
Bob:   on_call=true, version=0
```

Broken final rows:

```text
Alice: on_call=false, version=1
Bob:   on_call=false, version=1
```

## Transaction boundary và contention point

Mỗi `leaveOnCall()` call đi qua Spring proxy:

```text
BEGIN
  SELECT COUNT(*) WHERE roster_id=? AND on_call=true
  if count <= 1 -> reject
  SELECT own assignment
  set own on_call=false
  UPDATE own row WHERE version=?
COMMIT
```

Non-atomic sequence:

```text
read multi-row invariant -> decide -> update one member row
```

Contention point logic là toàn bộ assignments của roster. Physical writes lại là
Alice row và Bob row; row locks cùng optimistic version predicates không gặp nhau.

## Expected và actual

| | A/Alice | B/Bob | Final |
| --- | --- | --- | --- |
| Snapshot count | 2 | 2 | |
| Decision | leave allowed | leave allowed | |
| Updated row | Alice | Bob | |
| `@Version` affected rows | 1 | 1 | |
| Broken outcome | commit | commit | 0 on-call |
| Required outcome | winner | reject/retry | 1 on-call |

## Isolation nuance

- `READ COMMITTED`: hai COUNT statements có thể cùng chạy trước writes và thấy
  `2`; both commits are possible.
- PostgreSQL `REPEATABLE READ`: mỗi actor giữ stable snapshot thấy `2`; different
  row updates vẫn có thể cùng commit. Stable snapshot không bảo vệ cross-row
  invariant.
- `SERIALIZABLE`: SSI theo dõi predicate dependencies và có thể abort một actor
  với SQLSTATE `40001`; application cần bounded retry.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| write skew | Transactions đọc cùng invariant rồi ghi rows khác nhau làm invariant sai |
| snapshot isolation | Mỗi transaction quyết định từ stable committed snapshot |
| multi-row invariant | Quy tắc phụ thuộc nhiều rows thay vì một row riêng lẻ |
| rw-antidependency | Reader không thấy write sau đó của transaction khác |
| SSI | Serializable Snapshot Isolation của PostgreSQL |
| guard row | Shared row mà mọi invariant-changing transaction phải lock/update |
| serialization failure | Abort retryable với SQLSTATE `40001` |
| optimistic version | Version predicate phát hiện concurrent write trên cùng row |

## Điều hướng

- [Broken versioned-row implementation](broken-code.md)
- [Snapshot and dependency analysis](analysis.md)
- [Guard row, locking and serializable solutions](solutions.md)
- [Deterministic PostgreSQL experiments](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Isolation levels](../../concepts/isolation-levels.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Ca trực không còn người chịu trách nhiệm dù cả hai requests trả success.
- Alert/escalation không có owner; SLA và incident response bị ảnh hưởng.
- Audit từng row không cho thấy failed update hay exception.
- Retry theo duplicate request không sửa được already-committed invariant.
- Nhiều application instances làm JVM-local coordination vô hiệu.
- Reconciliation chỉ phát hiện sau khi roster đã rơi vào unsafe state.

## Hướng sửa khuyến nghị

Với một stable roster parent, ưu tiên lock guard row bằng `FOR UPDATE`, rồi count
và update trong cùng short transaction. Mọi path làm thay đổi invariant phải theo
cùng protocol.

Nếu contention/retry model phù hợp, `SERIALIZABLE` cùng full-transaction retry là
lựa chọn tốt cho predicate invariant phức tạp. Một authoritative `on_call_count`
với conditional decrement cũng diễn đạt invariant thành single-row conflict.

`@Version` trên assignment vẫn hữu ích cho same-row lost update, nhưng không thay
guard/SSI cho invariant nhiều rows.

## Phạm vi

Case này chỉ xét write skew khi transactions update different existing rows.
Phantom inserts/capacity thuộc DB-004. Cơ chế lock tổng quát được đào sâu ở DB-007
và deadlock ordering ở DB-008.
