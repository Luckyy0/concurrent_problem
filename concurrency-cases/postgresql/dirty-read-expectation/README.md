# DB-002 — Dirty read expectation khác PostgreSQL reality

## Tóm tắt

Job `IMPORT-42` có committed progress `20%`. Processor A mở transaction, update
progress lên `80%`, flush SQL nhưng chưa commit. Watchdog B dùng
`@Transactional(isolation = READ_UNCOMMITTED)` và kỳ vọng đọc `80%` để biết job
vẫn khỏe.

PostgreSQL không expose uncommitted tuple version. B vẫn thấy committed `20%`.
Watchdog kết luận sai rằng job stalled và có thể khởi động duplicate recovery.

Invariant:

```text
Mọi coordination/recovery decision phải dựa trên committed durable state.
Không component nào được phụ thuộc vào dirty-read visibility.
Nếu progress cần visible độc lập, nó phải có explicit short commit semantics.
Final success chỉ được công bố sau transaction chứa final state commit.
```

> **Nói ngắn gọn:** request `READ_UNCOMMITTED` không làm PostgreSQL có dirty reads;
> label có thể tồn tại nhưng behavior vẫn committed-only.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Processor A | Update progress trong transaction dài/chưa commit |
| Watchdog B | Đọc progress và quyết định job stalled hay healthy |
| Recovery C | Có thể chạy duplicate work nếu watchdog thấy stale progress |
| `job_run` | Authoritative committed job state |
| PostgreSQL MVCC | Chọn tuple version visible cho B |
| Spring/JDBC | Request transaction isolation cho reader |

Initial committed row:

```text
job_id      = IMPORT-42
status      = RUNNING
progress    = 20
generation  = 7
```

A uncommitted state:

```text
progress = 80
```

B actual read:

```text
progress = 20
```

## Transaction boundary và contention point

A:

```text
BEGIN -> UPDATE progress=80 -> FLUSH -> wait -> COMMIT/ROLLBACK
```

B:

```text
BEGIN requested READ_UNCOMMITTED
  -> plain SELECT
  -> sees committed tuple version 20
COMMIT
```

Contention point là logical row `job_run(IMPORT-42)`, nhưng plain MVCC SELECT không
cần chờ A row lock; nó đọc old committed version. Đây là visibility mismatch, không
phải lock timeout.

## Expected theo broken design và PostgreSQL actual

| | Broken design kỳ vọng | PostgreSQL actual |
| --- | --- | --- |
| Isolation request | Dirty reads allowed | RU behaves as RC |
| Before A commit | B sees 80 | B sees 20 |
| If A rollback | UI/recovery phải “unsee” 80 | 80 chưa từng visible |
| Plain SELECT blocking | Không | Không; đọc old committed version |
| Portability | Assumed uniform | Database-specific semantics |

Nếu A commit, một SELECT mới bắt đầu sau commit thấy `80`. Nếu A rollback, B tiếp
tục thấy `20`.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| dirty read | Đọc data do transaction khác viết nhưng chưa commit |
| aborted version | Tuple version thuộc transaction rollback/abort |
| requested isolation | Level application/JDBC yêu cầu |
| reported label | Tên `transaction_isolation` có thể hiển thị |
| effective semantics | Visibility behavior database thật sự cung cấp |
| committed-only | Chỉ row versions từ committed transactions được visible |
| progress checkpoint | Progress được commit độc lập với processing unit |
| portability assumption | Giả định cùng isolation name có behavior giống mọi DB |

## Điều hướng

- [Broken dirty-read design](broken-code.md)
- [PostgreSQL visibility analysis](analysis.md)
- [Committed coordination and progress solutions](solutions.md)
- [Deterministic visibility experiments](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Isolation levels](../../concepts/isolation-levels.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Watchdog false-positive và duplicate recovery.
- Dashboard/progress stale cho tới commit.
- Port từ database khác làm coordination behavior đổi.
- Developer thêm polling/retry nhưng vẫn chỉ đọc committed old version.
- Long transaction giữ progress invisible lâu hơn và gây operational confusion.
- Nếu dirty read thật sự tồn tại ở DB khác, observer có thể hành động trên state
  sau đó rollback.

## Hướng sửa khuyến nghị

Không dùng uncommitted state làm communication channel. Chọn một contract:

1. observer chấp nhận committed-only progress; hoặc
2. processor publish checkpoint trong short independent transaction với semantics
   rõ `attempt progress`, không phải final success; hoặc
3. dùng atomic lease/heartbeat/state transition cho watchdog coordination.

Đối với long workflow, state machine/outbox/committed checkpoints tốt hơn một
transaction dài.

## Phạm vi

Case chứng minh PostgreSQL dirty-read behavior và portability. Non-repeatable read
sau concurrent commit thuộc DB-003; dirty writes/lost updates thuộc cases khác.
