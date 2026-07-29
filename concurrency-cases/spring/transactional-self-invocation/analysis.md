# Proxy, flush, commit và interleaving

## Trạng thái ban đầu

```text
Account A = 100
Account B = 100
Total     = 200
Transfer  = A → B, amount 10
```

## Timeline self-invocation bị lỗi

| Bước | Writer T1 | Reader T2 | Database |
| --- | --- | --- | --- |
| 1 | gọi `transfer` qua proxy | | không có service transaction |
| 2 | `this.executeTransfer` | | interceptor không chạy |
| 3 | repository debit mở Tx-D | | A=100, B=100 |
| 4 | debit A rồi Tx-D commit | | A=90, B=100 |
| 5 | dừng ở probe | đọc total | đọc 190 |
| 6 | repository credit mở Tx-C | | |
| 7 | credit B rồi commit | | A=90, B=110 |

Nếu bước 6 fail, Tx-D không còn để rollback; total 190 tồn tại vĩnh viễn.

> **Nói ngắn gọn:** repository vẫn “có transaction”, nhưng đó là hai chiếc hộp
>commit riêng thay vì một hộp bao trọn transfer.

## Timeline proxy đúng

| Bước | Writer T1 | Reader T2 tại READ COMMITTED |
| --- | --- | --- |
| 1 | gọi public transactional entry qua proxy, mở Tx-S | |
| 2 | debit SQL chạy/flush trong Tx-S | vẫn thấy A=100, B=100 |
| 3 | dừng ở probe | total=200 từ committed version cũ |
| 4 | credit trong cùng Tx-S | |
| 5 | Tx-S commit | lần đọc sau thấy A=90, B=110 |

Flush không làm uncommitted debit visible cho reader PostgreSQL thông thường tại
`READ COMMITTED`. Commit chung là external visibility boundary.

## Expected và actual

| Khía cạnh | Intended transaction | Self-invocation |
| --- | --- | --- |
| Atomicity | Debit/credit cùng commit | Mỗi repository method commit riêng |
| Rollback | Credit failure rollback debit | Debit đã commit |
| Reader | State trước hoặc sau transfer | Có thể thấy partial state |
| Probe transaction | Active | Không active giữa repository calls |
| Locks | Giữ tới service commit | Debit row lock release sớm |

## Root cause theo layer

### Spring AOP proxy

Proxy intercept external method call. Call từ target tới `this` là invocation
trực tiếp; annotation metadata không được interceptor đọc ở runtime path đó.
JDK proxy/CGLIB khác implementation nhưng self-invocation limitation của proxy
mode vẫn tồn tại.

### Spring Data repository

Repository là proxy khác. `debit` và `credit` annotated `@Transactional` vẫn được
intercept, nên bug không nhất thiết ném “no transaction”; nó tạo hai valid nhưng
sai-scope transactions.

### Hibernate/JPA

Bulk update thực thi SQL trực tiếp. Persistence context có thể stale nếu cùng
context đã load entity; production repository nên cân nhắc `clearAutomatically`
hoặc tránh đọc entity stale sau bulk update. Vấn đề này tách biệt với proxy
boundary.

### PostgreSQL

MVCC chỉ cung cấp visibility theo từng database transaction. Nó không biết hai
transactions Tx-D/Tx-C thuộc cùng business transfer, nên không thể tự gộp chúng.

## Exception và rollback

Với boundary đúng, unchecked exception đánh dấu rollback theo mặc định. Checked
exception chỉ rollback nếu policy cấu hình phù hợp, ví dụ
`rollbackFor = TransferException.class`. Không catch/swallow lỗi rồi kỳ vọng
proxy tự rollback; nếu cần chuyển lỗi, giữ cause và rollback contract rõ.

Validation không phụ thuộc mutable state có thể chạy trước transaction; balance
check và update phải nằm trong atomic database boundary.

## Concurrency và multi-instance

Transaction database hoạt động qua nhiều application instance vì PostgreSQL là
authoritative boundary. Tuy nhiên, boundary đúng không tự ngăn lost update,
deadlock hoặc write skew giữa nhiều transfer; isolation/locking thuộc `DB-*`.
Local `synchronized` không thay thế service transaction và không mở rộng qua node.

## Timeout, retry và side effect

- Transaction timeout phải phù hợp query/lock timeout; timeout exception rollback
  transaction còn active.
- Không retry self-invocation lỗi; sửa boundary trước.
- Retry transfer cần idempotency và conflict classification.
- Publish event/email sau debit nhưng trước commit có thể lệch database; dùng
  after-commit hook hoặc outbox khi cần durability.
- Process crash giữa hai repository commits của broken code giữ partial state;
  correct single transaction được PostgreSQL rollback nếu chưa commit.

## Quan sát

Log transaction ID/correlation ID và kiểm tra
`TransactionSynchronizationManager.isActualTransactionActive()` ở diagnostic/test,
không dùng nó thay business assertion. Theo dõi partial invariant, rollback,
transaction duration và repository calls không có expected outer transaction.

Kiến thức nền: [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md).
