# LOCK-001 — Optimistic locking với `@Version`

## Tóm tắt

Hai editor cùng mở offer `42`, giá `100`, version `7`. A đổi giá thành `90`; B
đổi thành `80`. Không có version predicate, cả hai `UPDATE` đều affected rows `1`
và B âm thầm ghi đè commit của A.

Thêm `@Version` khiến Hibernate phát SQL:

```sql
update product_offer
set price = 90.00,
    version = 8
where offer_id = 42
  and version = 7;
```

A affected rows `1`. B dùng expected version `7` sau A commit nên affected rows
`0`; Hibernate phát `OptimisticLockException`, Spring thường translate thành
`ObjectOptimisticLockingFailureException`, và transaction loser rollback.

> **Nói ngắn gọn:** `@Version` không chặn concurrent read; nó biến stale write
> thành conflict nhìn thấy được thay vì silent overwrite.

## Actor và trạng thái dùng chung

| Thành phần | Giá trị |
| --- | --- |
| `product_offer` | `offer_id=42`, `price=100`, `version=7` |
| Editor A / App-1 | sửa giá thành `90` |
| Editor B / App-2 | sửa giá thành `80` |
| Authoritative state | PostgreSQL row |

Point of contention là versioned `UPDATE` lúc flush/commit, không phải setter Java
hay `repository.save()`.

## Invariant

```text
Một edit chỉ commit nếu aggregate vẫn ở expected version mà editor đã đọc.

Không stale edit nào được báo thành công sau khi một edit khác đã commit.

Version chỉ do persistence provider cập nhật; application không tự tăng/sửa.
```

Case chọn outcome “conflict và yêu cầu user reload/merge”, không tự retry thao tác
“set price”. Retry mù có thể biến stale intent thành overwrite hợp lệ trên version
mới; bounded retry tổng quát thuộc `LOCK-002`.

## Ranh giới transaction

Mỗi editor submit qua một Spring proxy transaction riêng:

```text
BEGIN
load offer/version
validate expectedVersion từ request
apply domain change
flush versioned UPDATE
COMMIT hoặc optimistic conflict + ROLLBACK
```

Exception có thể xuất hiện lúc API call, flush hoặc commit. Caller chỉ được báo
success sau transaction proxy commit.

## Detached/client boundary

`@Version` trong entity chưa đủ nếu controller bỏ version mà client đã xem:

```text
client B đọc v7
A commit v8
B request không mang expectedVersion
service load current v8 rồi set price=80
update where version=8 thành công
```

Vì vậy edit command/API phải mang `expectedVersion` (hoặc `If-Match` ETag).
Service so sánh nó với entity vừa load; `@Version` tiếp tục bảo vệ race xảy ra sau
load và trước flush.

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong case |
| --- | --- |
| optimistic locking | Cho phép đọc song song, detect conflict khi write |
| version column | Revision do persistence provider quản lý |
| expected version | Version actor/client đã đọc |
| stale entity/edit | State/intent dựa trên revision không còn current |
| affected-row count | `1` là thắng; `0` là version không còn match |
| `OptimisticLockException` | Jakarta Persistence conflict signal |
| flush | Thời điểm Hibernate gửi dirty SQL |
| detached state | Entity/DTO sống ngoài persistence context |

## Điều hướng

- [Code lỗi và stale client boundary](broken-code.md)
- [Timeline MVCC, row lock và conflict](analysis.md)
- [Entity `@Version`, API version và outcome mapping](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [Optimistic locking và version conflict](../../concepts/optimistic-locking.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production

- editor nhận success nhưng thay đổi trước đó biến mất;
- audit/history không phản ánh overwrite hoặc actor intent;
- cache/event phát state không còn current;
- conflict chỉ lộ muộn ở commit và bị map thành generic 500;
- retry trong failed persistence context nhận lỗi tiếp;
- hot aggregate tạo conflict/retry amplification;
- node-local lock không bảo vệ edits từ instance khác.

## Hướng sửa khuyến nghị

1. Thêm cột `version not null` và `@Version` trên aggregate root.
2. Migrate/backfill an toàn trước khi bật mapping.
3. Mang expected version qua HTTP/message command boundary.
4. Flush trong attempt để conflict nằm trong boundary quan sát được; vẫn catch
   exception từ toàn proxy call vì commit cũng có thể phát lỗi.
5. Rollback loser và map conflict thành `409 Conflict`/`412 Precondition Failed`
   theo API contract.
6. Không auto-retry user edit; trả current version/dữ liệu phù hợp để reload/merge.

## Khi phù hợp

Optimistic locking phù hợp khi aggregate row là conflict scope, contention thấp/
vừa và loser có thể retry/reload hoặc yêu cầu user merge. Hot counter/stock delta
thường phù hợp atomic SQL hoặc pessimistic/serialized strategy hơn.

## Phạm vi

Case chỉ xử lý conflict detection. Retry có backoff/jitter/new transaction thuộc
`LOCK-002`; pessimistic `FOR UPDATE` thuộc `LOCK-003`; conditional atomic mutation
thuộc `LOCK-004`.
