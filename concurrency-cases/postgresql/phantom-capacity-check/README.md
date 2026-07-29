# DB-004 — Phantom rows làm vỡ capacity check

## Tóm tắt

Processing pool `P-42` có capacity `10` và đang có `9` allocations ở trạng thái
`ACTIVE`. Request A và B chạy trong hai Spring transactions độc lập. Cả hai cùng
đếm predicate:

```sql
select count(*)
from slot_allocation
where pool_id = :poolId
  and status = 'ACTIVE';
```

A và B đều thấy `9 < 10`, sau đó insert hai allocation rows khác nhau. Cả hai
commits thành công; final active count là `11`.

Invariant:

```text
Với mọi processing pool:
  active allocations <= configured capacity

Với P-42:
  capacity = 10
  initial active = 9
  chỉ một trong hai concurrent requests được accepted
```

> **Nói ngắn gọn:** row locks trên hai INSERT khác nhau không bảo vệ khoảng trống
> “còn một chỗ”; capacity là predicate invariant trên cả một tập rows.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Allocator A | Check capacity rồi tạo allocation `A-101` |
| Allocator B | Check capacity rồi tạo allocation `B-202` |
| `processing_pool` | Chứa configured capacity |
| `slot_allocation` | Chứa các allocation rows và status |
| PostgreSQL MVCC | Cấp snapshot cho predicate query |
| Hibernate | Flush mỗi new entity thành một INSERT độc lập |

Initial committed state:

```text
processing_pool(P-42): capacity=10
ACTIVE allocation rows: 9
```

Broken final state:

```text
A-101 = ACTIVE
B-202 = ACTIVE
ACTIVE allocation rows: 11
both callers received ACCEPTED
```

## Transaction boundary và contention point

Mỗi `allocate()` call đi qua Spring proxy:

```text
BEGIN
  SELECT pool capacity
  SELECT COUNT(*) WHERE pool_id=P-42 AND status=ACTIVE
  decide that one slot remains
  INSERT a new ACTIVE row
COMMIT
```

Non-atomic sequence:

```text
query qualifying set -> compare count with limit -> insert a new qualifying row
```

Contention point không phải allocation ID. Nó là predicate:

```text
all rows where pool_id=P-42 and status=ACTIVE
```

A insert `A-101`, B insert `B-202`; primary key/unique request constraints không
conflict. Plain COUNT cũng không reserve phần capacity còn lại.

## `READ COMMITTED` và `REPEATABLE READ`

| Isolation | Nếu cùng transaction query lại | Capacity race |
| --- | --- | --- |
| `READ COMMITTED` | Có thể thấy row mới sau concurrent commit: visible phantom | Hai actors vẫn có thể count `9`, insert, commit |
| `REPEATABLE READ` | Stable snapshot tiếp tục thấy `9` | Vẫn có thể cùng insert row khác nhau và final `11` |
| `SERIALIZABLE` | SSI theo dõi predicate dependencies | Một transaction có thể bị abort với `40001`; application phải retry |

PostgreSQL `REPEATABLE READ` ngăn visible phantom trong repeated reads, nhưng
stable old snapshot không tự enforce aggregate capacity invariant.

## Expected và actual

| | A | B | Final |
| --- | --- | --- | --- |
| COUNT | 9 | 9 | |
| Decision | ACCEPT | ACCEPT | |
| INSERT | `A-101` | `B-202` | |
| Broken outcome | commit | commit | 11 active |
| Required outcome | một winner | một reject/retry | 10 active |

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| phantom row | Row mới làm kết quả predicate query thay đổi khi đọc lại |
| predicate invariant | Quy tắc áp dụng cho tập rows thỏa một điều kiện |
| check-then-insert | Query rồi quyết định insert bằng state có thể đã stale |
| stable snapshot | Một transaction view không đổi ở `REPEATABLE READ` |
| predicate dependency | Quan hệ read/write trên range hoặc điều kiện truy vấn |
| SSI | Serializable Snapshot Isolation của PostgreSQL |
| serialization failure | Transaction abort với SQLSTATE `40001` |
| authoritative counter | Counter row được update atomically tại database |

## Điều hướng

- [Broken count-then-insert](broken-code.md)
- [MVCC and predicate analysis](analysis.md)
- [Atomic counter, parent lock and serializable solutions](solutions.md)
- [Deterministic PostgreSQL experiments](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Isolation levels](../../concepts/isolation-levels.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Pool nhận nhiều work hơn capacity thật, gây memory/CPU/resource exhaustion.
- Hai callers đều nhận success nên không có loser signal để recovery.
- Dashboard count có thể vượt configured limit dù mỗi allocation row hợp lệ.
- Unique allocation ID không phát hiện aggregate violation.
- Tăng số application instances làm cửa sổ race rộng hơn.
- Manual cleanup có thể release nhầm allocation hoặc double-decrement counter.
- Retry mù quáng có thể tạo thêm rows nếu command idempotency chưa độc lập.

## Hướng sửa khuyến nghị

Chọn representation làm capacity hữu hình tại authoritative boundary:

1. Với một parent row ổn định, dùng atomic conditional counter update rồi insert
   allocation trong cùng transaction.
2. Nếu không muốn counter denormalization, lock parent bằng `FOR UPDATE`, sau đó
   count và insert; mọi writer phải theo cùng protocol.
3. Với finite slots, pre-create slot rows và claim một row bằng
   `FOR UPDATE SKIP LOCKED`.
4. Với predicate phức tạp khó quy về một row/slot, dùng `SERIALIZABLE` cùng
   bounded full-transaction retry.

Unique request constraint vẫn cần cho duplicate command, nhưng đó là invariant
khác capacity.

## Phạm vi

Case này dùng generic processing pool và một predicate capacity. Hotel/room
capacity semantics thuộc `BOOK-001`. Write skew trên multi-row on-call invariant
được tách sang DB-005.
