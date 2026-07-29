# Rollback classification analysis

## Initial state

```text
payout_request:
  id = P-42
  status = RECEIVED
  amount = 300
  wallet_id = W-7
  beneficiary_id = B-BLOCKED

wallet_account:
  id = W-7
  available_balance = 1000

ledger_entry:
  no PAYOUT_HOLD for P-42
```

Dispatcher chỉ lấy payout có `status = PROCESSING`. Beneficiary registry đã chứa
`B-BLOCKED`, nên policy chắc chắn ném `BeneficiaryRejectedException`.

## Expected versus actual

Expected khi method ném declared business rejection:

```text
payout.status       = RECEIVED
available_balance   = 1000
PAYOUT_HOLD count   = 0
dispatchable        = false
transaction outcome = ROLLED_BACK
```

Broken actual:

```text
payout.status       = PROCESSING
available_balance   = 700
PAYOUT_HOLD count   = 1
dispatchable        = true
transaction outcome = COMMITTED
caller outcome      = BeneficiaryRejectedException
```

Hai outcomes cuối mâu thuẫn nhau. Caller nhìn control-flow failure; các node khác
nhìn durable success state.

## Timeline hai actor

| Bước | Request A — App node 1 | Dispatcher B — App node 2 |
| --- | --- | --- |
| T0 | Proxy mở Tx-A | Poll chưa thấy P-42 |
| T1 | Lock payout và wallet rows | |
| T2 | Set `PROCESSING`, balance `1000 -> 700`, persist hold | |
| T3 | Policy ném checked exception | |
| T4 | Spring rollback rule trả `false`; Hibernate flush nếu cần | |
| T5 | Tx-A commit, release row locks | |
| T6 | | Query mới thấy P-42 `PROCESSING` |
| T7 | Proxy rethrow exception; caller nhận rejected | Claim/dispatch P-42 |

T6 có thể xảy ra trước khi caller xử lý T7 vì commit nằm bên trong interceptor,
trước exception được rethrow ra ngoài proxy.

> **Nói ngắn gọn:** database visibility mở ra ở commit, còn caller chỉ nhận checked
> exception sau commit; dispatcher có thể hành động trên state trái với API outcome.

## Root cause chính xác ở Spring layer

`@Transactional` dùng một rollback rule để quyết định outcome khi method ném
throwable. Default convention:

- rollback với `RuntimeException` và subclass;
- rollback với `Error` và subclass;
- không rollback với checked `Exception` theo mặc định.

`BeneficiaryRejectedException extends Exception`, nên default rule xem nó là
exception không yêu cầu rollback. Transaction interceptor đi vào commit path rồi
rethrow original exception.

Java compiler rule về `throws` và Spring transaction rule là hai hệ thống độc lập.
`throws` chỉ buộc caller handle/declare checked exception; nó không mang metadata
“rollback transaction”.

## Physical transaction và logical scope

Khi `prepare()` không có outer transaction, proxy tạo physical transaction và tự
commit nó sau checked exception.

Khi `prepare()` join outer transaction bằng `REQUIRED`, outcome phụ thuộc toàn bộ
call chain:

- inner không có `rollbackFor`: checked exception không đánh dấu rollback-only;
- outer để cùng checked exception thoát ra với default rule: outer cũng commit;
- outer catch/swallow exception: outer tiếp tục và có thể commit;
- inner có `rollbackFor`: participating transaction được đánh dấu rollback-only;
- outer catch exception rồi cố commit: thường nhận `UnexpectedRollbackException`.

Vì vậy rollback contract phải được đặt ở boundary và exception phải đi qua đúng
proxy. Case SPR-001 vẫn áp dụng cho self-invocation.

## Hibernate flush không phải nguyên nhân

Managed entity mutations có thể chưa tạo SQL ngay. Hibernate thường dirty-check
và flush trước commit; một query hoặc explicit `flush()` cũng có thể flush sớm.

Hai variants:

```text
late flush: checked exception -> commit path -> flush SQL -> COMMIT
early flush: SQL sent -> checked exception -> commit path -> COMMIT
```

Nếu rollback rule đúng:

```text
early flush: SQL sent -> checked exception -> ROLLBACK
```

SQL đã chạy trong transaction vẫn được PostgreSQL undo khi rollback. Vì thế
`saveAndFlush()` không làm state “quá muộn để rollback”, và bỏ nó cũng không sửa
default classification.

## PostgreSQL snapshot và locks

Isolation mặc định `READ COMMITTED` đủ để chứng minh case:

- Tx-A giữ row locks từ pessimistic SELECT/update đến commit hoặc rollback;
- dispatcher transaction trước T5 không thấy uncommitted changes;
- query bắt đầu sau T5 thấy committed `PROCESSING` state;
- nếu Tx-A rollback đúng, dispatcher không bao giờ thấy intermediate mutations;
- PostgreSQL không biết Java exception mang ý nghĩa business gì.

Row lock giải quyết write interleaving trên cùng rows. Nó không quyết định commit
hay rollback và không thể ngăn một state hợp lệ về SQL nhưng sai theo failure
contract trở nên visible.

## Vì sao đây vẫn là concurrency problem?

Một request đơn lẻ đã tạo inconsistent outcome. Concurrency làm hậu quả operational
hơn: dispatcher, retry request, reconciliation job và node khác phản ứng với commit
trước khi con người hiểu exception log.

Shared state không nằm trong JVM; nó nằm ở PostgreSQL. Một `synchronized` quanh
`prepare()` trên App node 1:

- không đổi rollback rule;
- không chặn dispatcher trên App node 2;
- không đảo durable commit;
- không bảo vệ retry đi qua instance khác.

Database transaction outcome và executable-state predicate mới là authoritative
coordination boundary.

## Catching làm thay đổi điều proxy quan sát

Broken variant:

```java
@Transactional(rollbackFor = BeneficiaryRejectedException.class)
public PreparationResult prepare(UUID payoutId) {
    try {
        beneficiaryPolicy.verify(...);
        return PreparationResult.ready();
    } catch (BeneficiaryRejectedException rejected) {
        log.info("Rejected payout {}", payoutId);
        return PreparationResult.rejected();
    }
}
```

Exception không thoát khỏi method, nên interceptor không áp dụng `rollbackFor`.
Nếu code đã mutate executable state trước catch và không sửa lại, transaction vẫn
commit.

Credible choices:

- rethrow để proxy rollback;
- validate trước mutation rồi commit explicit `REJECTED` result;
- mark rollback-only programmatically khi API buộc phải return;
- tách rejected audit record sang một transaction độc lập có chủ đích.

Không vừa swallow exception vừa kỳ vọng annotation thấy nó.

## Exception translation và wrapping

Nếu code wrap checked exception thành `RuntimeException`, default rollback xảy ra.
Nhưng exception hierarchy phải diễn đạt failure contract, không chỉ phục vụ một
framework default.

Ngược lại, wrap một unchecked data-access failure vào checked exception có thể vô
tình đổi rollback thành commit nếu transaction chưa được đánh dấu rollback-only.
Spring-translated `DataAccessException` vốn là unchecked; đừng làm mất signal khi
chuyển abstraction layer.

Ưu tiên type-safe:

```java
rollbackFor = BeneficiaryRejectedException.class
```

String-based class-name rules dễ match rộng hoặc sai khi tên exception trùng một
phần. Nếu dùng shared meta-annotation, integration test phải chứng minh effective
behavior, không chỉ inspect annotation.

## Commit cũng có thể thất bại

Default rule chọn commit không bảo đảm commit thành công. Flush có thể gặp unique
constraint, lock timeout hoặc connection failure; caller khi đó có thể nhận một
transaction/data-access exception khác.

Case này giả định commit thành công để cô lập rollback classification. Remote
timeout/unknown external outcome nằm ngoài scope.

## Retry và duplicate command

Caller thấy checked exception có thể retry P-42 hoặc gửi command mới:

- retry cùng ID gặp `PROCESSING`, không phải initial state;
- retry ID mới có thể giữ tiền lần nữa nếu không có idempotency constraint;
- unique ledger key có thể chặn duplicate ledger nhưng không tự hoàn tác state
  `PROCESSING`;
- retry mù không sửa durable outcome đầu tiên.

Idempotency vẫn cần cho command delivery, nhưng không thay thế rollback rule. Hai
cơ chế bảo vệ hai invariants khác nhau.

## Crash behavior

- Crash trước PostgreSQL commit: connection close làm transaction rollback.
- Crash sau commit nhưng trước API response: state committed, caller không biết
  outcome; đây là ambiguous response case khác.
- Checked exception path ở case này không cần crash: proxy chủ động commit rồi
  rethrow.
- Dispatcher crash sau claim cần recovery/idempotency riêng, không thay đổi root
  cause của SPR-005.

## Observability

Nên ghi nhận cùng correlation identifiers:

- payout ID, command/idempotency key và transaction name;
- domain outcome `READY`/`REJECTED`;
- transaction completion status từ test/diagnostic instrumentation;
- committed payout state và ledger reconciliation result;
- dispatcher claim sau rejection;
- `UnexpectedRollbackException` nếu outer scope swallow một inner rollback.

Không log full beneficiary/bank data. Alert quan trọng là mismatch giữa rejected
business outcome và committed executable/financial state, không chỉ exception
count.
