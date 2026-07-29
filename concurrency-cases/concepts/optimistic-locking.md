# Optimistic locking và version conflict

## Mục đích

Tài liệu định nghĩa cách JPA/Hibernate dùng `@Version` để phát hiện concurrent
write, cách conflict xuất hiện và điều kiện để retry an toàn. Mỗi business case
vẫn phải quyết định operation có được phép retry hay không.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| optimistic locking | Cho phép actors đọc song song, phát hiện conflict khi write |
| version column | Giá trị tăng sau mỗi successful update của aggregate |
| expected version | Version actor đã đọc và đưa vào write predicate |
| stale entity | Entity state được load từ version không còn current |
| affected-row count | Số row database update; `0` báo expected version không match |
| optimistic conflict | Actor khác đã thay aggregate sau snapshot của current actor |
| retry amplification | Nhiều losers retry cùng lúc và tiếp tục tạo conflict |
| retry boundary | Phạm vi phải rollback, tạo lại và reload trước attempt mới |

## Cơ chế SQL

Entity:

```java
@Version
private long version;
```

Hibernate tạo update tương đương:

```sql
update inventory_item
set available = :newAvailable,
    version = :nextVersion
where sku = :sku
  and version = :expectedVersion;
```

Nếu affected-row count bằng `1`, actor thắng và version tăng. Nếu bằng `0`,
Hibernate phát hiện stale state và phát `OptimisticLockException`/Spring-translated
`ObjectOptimisticLockingFailureException`.

> **Nói ngắn gọn:** `@Version` không ngăn hai actor cùng đọc; nó ngăn stale write
> âm thầm ghi đè current state.

## Persistence context sau conflict

EntityManager đã tham gia một failed flush không phải clean snapshot để dùng lại.
Transaction có thể đã `rollback-only`; managed entities vẫn chứa state được tính
từ version cũ. `clear()` chỉ detach entities, không reset transaction outcome.

Một retry đúng phải:

1. để exception thoát khỏi attempt boundary;
2. rollback/close transaction và persistence context cũ;
3. áp dụng bounded backoff/jitter nếu phù hợp;
4. bắt đầu physical transaction mới;
5. reload aggregate và re-evaluate business preconditions;
6. commit hoặc phân loại conflict tiếp theo.

## Retry không luôn an toàn

Retry phù hợp khi operation là deterministic trên fresh state và không để lại
external side effect từ attempt cũ. Không retry mù:

- validation/business rejection;
- duplicate command không có idempotency protection;
- remote side effect không thể xác định outcome;
- deadline đã hết;
- hot-key contention đang tạo retry storm.

Conflict loser có thể trả fail/no-op thay vì retry nếu latency contract hoặc user
intent yêu cầu.

## Throughput và contention

Optimistic locking phù hợp khi conflicts hiếm và critical section không cần giữ
database lock trong thời gian xử lý. Trên hot key, nhiều actors làm work rồi chỉ
một actor commit; retries tăng CPU, queries và database writes.

Các lựa chọn khác gồm atomic conditional SQL, pessimistic locking, partitioning
theo key hoặc serial queue. Chọn theo invariant, contention distribution, latency
và retry safety.

## Kiểm thử

Test cần ép hai transactions load cùng expected version, cho một actor commit,
sau đó cho actor còn lại flush. Assert:

- generated update có version predicate;
- loser nhận optimistic conflict;
- failed attempt rollback;
- retry dùng transaction/persistence context mới;
- aggregate được reload;
- final business state và version đúng.

Không chỉ assert exception type; một retry loop có thể catch đúng exception nhưng
vẫn chạy trong doomed transaction.

## Liên kết

- [SPR-006 — Retry transaction boundary](../spring/retry-transaction-boundary/README.md)
- [LOCK-001 — Optimistic locking with @Version](../locking/optimistic-version-conflict/README.md)
- [LOCK-002 — Bounded optimistic retry](../locking/optimistic-retry-contention/README.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Deadlock và retry an toàn](deadlocks-and-retries.md)
- [Concurrency testing](concurrency-testing.md)
