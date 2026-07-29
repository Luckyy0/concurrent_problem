# DB-007 — Row lock, table lock và lock lifetime

## Tóm tắt

Tenant `T-42` có committed quota `10`. Admin A mở transaction, chạy plain
`SELECT`, rồi giữ method đang thực hiện local validation. Team tin SELECT đã khóa
row nên Admin B phải chờ. Thực tế B UPDATE quota thành `8` và commit ngay.

Sau khi thêm `FOR UPDATE`, team lại cho rằng mọi reader sẽ bị block. Nhưng khi A
đang giữ row lock và có uncommitted quota `12`:

- competing writer/locking reader B phải chờ hoặc timeout;
- Dashboard C dùng plain SELECT không chờ và thấy old committed quota `8`;
- locks chỉ release khi A commit/rollback, không phải khi repository method return.

Invariant/contract:

```text
Mọi quota mutation dựa trên current state phải acquire authoritative conflict
trước khi quyết định và giữ nó tới commit/rollback.

Plain observers chỉ thấy committed MVCC state.
Không component nào được hiểu row lock là quyền đọc uncommitted data.
```

> **Nói ngắn gọn:** row lock serialize conflicting mutations, không biến
> PostgreSQL thành database nơi một writer đang khóa thì mọi reader đều đứng chờ.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Admin A | Đọc/điều chỉnh quota trong transaction |
| Admin B | Competing quota writer hoặc locking reader |
| Dashboard C | Plain committed-state observer |
| `tenant_quota` | Authoritative quota row |
| PostgreSQL MVCC | Chọn committed tuple version cho C |
| Spring transaction | Quyết định lock lifetime |

Initial committed row:

```text
tenant_id = T-42
quota     = 10
revision  = 5
```

## Transaction boundaries

Broken assumption:

```text
A: BEGIN -> SELECT quota=10 -> validate -> UPDATE
B: BEGIN -> UPDATE same row -> team expects wait
```

Actual: A plain SELECT không giữ row lock; B có thể update/commit.

Explicit locking:

```text
A: BEGIN -> SELECT ... FOR UPDATE -> UPDATE quota=12 -> wait -> COMMIT
B: UPDATE/FOR UPDATE same row -> waits/fails
C: plain SELECT same row -> returns old committed version
```

Contention point là `tenant_quota(T-42)`. Lock acquire tại locking statement/update
và release ở physical transaction end.

## Lock matrix rút gọn cho case

| A đang giữ | B operation | Expected behavior |
| --- | --- | --- |
| Không row lock, chỉ plain SELECT | UPDATE | B tiếp tục |
| `FOR UPDATE` row lock | UPDATE same row | B waits/timeout |
| `FOR UPDATE` row lock | `FOR UPDATE` same row | B waits/timeout |
| `FOR UPDATE` row lock | plain SELECT | C đọc committed version |
| `ACCESS EXCLUSIVE` table lock | plain SELECT | C waits/timeout |

Bảng này chỉ cover paths của case; PostgreSQL có nhiều row/table lock modes khác.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| row-level lock | Lock coordinate mutation trên row |
| table-level lock mode | Relation lock mode tự động/explicit cho statement |
| lock holder | Transaction đang giữ incompatible lock |
| waiter | Transaction chờ lock holder kết thúc |
| plain SELECT | MVCC read không có locking clause |
| `FOR UPDATE` | Locking read xin exclusive row-level intent phù hợp |
| lock lifetime | Từ acquisition tới commit/rollback |
| `lock_timeout` | Giới hạn thời gian chờ database lock |
| `ACCESS EXCLUSIVE` | Table lock mode chặn cả plain SELECT |

## Điều hướng

- [Broken SELECT-lock assumptions](broken-code.md)
- [Lock acquisition, visibility and release analysis](analysis.md)
- [Pessimistic, atomic and optimistic solutions](solutions.md)
- [Deterministic PostgreSQL lock experiments](experiments.md)
- [Pessimistic locking](../../concepts/pessimistic-locking.md)
- [PostgreSQL locks](../../concepts/postgresql-locks.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Writer dùng stale state vì plain SELECT không reserve row.
- Team thêm JVM lock nhưng multiple nodes vẫn race.
- Transaction giữ `FOR UPDATE` quanh remote I/O làm wait queue/pool exhaustion.
- Dashboard bị serialize không cần thiết nếu dùng locking read theo phản xạ.
- Lock timeout bị map nhầm thành “row không tồn tại”.
- Commit/rollback boundary mơ hồ làm locks giữ lâu hơn code reviewer tưởng.
- Table lock quá mạnh làm unrelated tenants bị block.

## Hướng sửa khuyến nghị

- Writer cần read-modify-write serialization: `PESSIMISTIC_WRITE`/`FOR UPDATE`
  trong short transaction, hoặc atomic conditional SQL khi diễn đạt được.
- Observer: plain SELECT với committed-staleness contract.
- Conflict detection không cần blocking: `@Version` khi retry/reject phù hợp.
- Whole-table/schema operation mới cân nhắc explicit table lock mode.
- Luôn cấu hình bounded timeout, deterministic multi-row order và diagnostics.

## Phạm vi

Case tập trung lock mechanics/lifetime. Claim nhiều jobs bằng `SKIP LOCKED` thuộc
DB-010; lock selection patterns thuộc `LOCK-003`; opposite-order deadlock thuộc
DB-008.
