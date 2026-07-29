# Logical scope, physical transaction và commit timeline

## Trạng thái ban đầu

```text
order 42 status = PENDING
payment_audit     = empty
```

Tx-O bắt đầu và update order 42 thành `PAID` nhưng chưa commit.

## Timeline REQUIRES_NEW partial commit

| Bước | Outer Tx-O | Inner Tx-A | Committed view của reader |
| --- | --- | --- | --- |
| 1 | update order→PAID | | order=PENDING, no audit |
| 2 | suspend, giữ resource/lock | begin bằng connection khác | |
| 3 | | insert PAYMENT_COMPLETED | |
| 4 | | commit Tx-A | order=PENDING + audit=COMPLETED |
| 5 | resume, dừng ở probe | | partial semantic state visible |
| 6 | outer exception, rollback | | order=PENDING + audit vẫn còn |

`REQUIRES_NEW` bảo đảm independent commit; chính guarantee đó phá invariant nếu
record inner tuyên bố kết quả của outer transaction chưa quyết định.

> **Nói ngắn gọn:** propagation chạy đúng kỹ thuật nhưng business semantics chọn
>sai commit boundary.

## Timeline REQUIRED rollback-only

| Bước | Outer logical scope | Inner REQUIRED logical scope | Physical Tx-O |
| --- | --- | --- | --- |
| 1 | begin | | active |
| 2 | gọi bean inner | join Tx-O | active |
| 3 | | runtime exception thoát interceptor | marked rollback-only |
| 4 | catch exception, tiếp tục | | vẫn rollback-only |
| 5 | return, yêu cầu commit | | rollback |
| 6 | outer interceptor | | ném `UnexpectedRollbackException` |

Outer catch xử lý Java control flow, không reset transaction status. Spring ném
exception để caller không tin rằng commit thành công.

## Expected và actual

| Khía cạnh | Intended | Broken propagation |
| --- | --- | --- |
| Success record | Commit cùng/after business outcome | Commit trước outer outcome |
| Outer rollback | Xóa mọi success state | Audit sống sót |
| Optional inner failure | Outcome explicit | Runtime exception mark rollback-only |
| Caller response | Phản ánh commit thật | Có thể catch nhầm và báo success |
| Resource use | Một connection nếu cùng boundary | Outer giữ connection, inner cần connection thứ hai |

## Propagation semantics

### REQUIRED

Nếu có transaction, join cùng physical transaction. Mỗi advised method là logical
scope có rollback rules, nhưng commit chỉ xảy ra một lần ở outer boundary. Inner
rollback-only ảnh hưởng toàn physical transaction.

### REQUIRES_NEW

Suspend outer transaction, tạo physical transaction mới, commit/rollback độc lập,
rồi resume outer. Inner không thấy uncommitted changes của outer theo cách “cùng
transaction”; query có thể thấy committed version cũ hoặc block trên lock.

### NESTED

Thường dùng savepoint trong cùng physical transaction: inner có thể rollback về
savepoint, nhưng outer rollback vẫn rollback tất cả. Support phụ thuộc transaction
manager/driver; `JpaTransactionManager` không mặc định biến mọi JPA operation thành
nested savepoint workflow. Phải integration test stack thực.

## Flush, locks và connection pool

Outer flush có thể giữ row locks khi bị suspend. Inner transaction truy cập cùng
rows có thể tự block chờ outer—outer lại đang chờ inner return, tạo resource wait.
Ngoài ra mỗi concurrent outer `REQUIRES_NEW` có thể cần connection thứ hai; pool
cạn làm mọi outer giữ connection và chờ inner connection. Chi tiết pool exhaustion
thuộc `SPR-007`.

## Exception và rollback rules

Runtime exception vượt transactional interceptor mặc định mark rollback. Checked
exception chỉ theo configured `rollbackFor`. `noRollbackFor` phải dựa trên state
có an toàn để commit, không phải mong muốn “đừng thấy exception”.

Nếu inner tự chuyển expected branch thành value (`RiskOutcome.REJECTED`) mà không
mutation dở/exception, outer có thể quyết định commit/rollback explicit. Nếu lỗi
kỹ thuật đã làm physical transaction không còn tin cậy, không cố cứu bằng catch.

## Retry, idempotency và duplicate

Retry outer sau partial commit có thể insert audit lần hai. Inner independent
record cần unique key như `(operation_id, event_type)` và replay semantics. Nhưng
idempotency chỉ chặn duplicate; nó không sửa record `COMPLETED` sai sự thật.

Không retry `UnexpectedRollbackException` mù quáng; phân loại nguyên nhân,
cleanup transaction cũ và bắt đầu attempt mới ở proxy boundary.

## Multi-instance và distributed boundary

Propagation là local application policy nhưng physical transactions được
PostgreSQL điều phối qua node. Nó không tạo atomicity với remote payment/message
broker. Compensation/saga cho nhiều resource thuộc `DIST-004`; durable event
publication thuộc messaging/outbox cases.

## Crash và observability

Crash sau Tx-A commit nhưng trước Tx-O rollback/commit để lại audit độc lập đúng
như semantics. Theo dõi outer/inner transaction ID, propagation, suspend duration,
connection acquisition, rollback-only/UnexpectedRollback, audit-business mismatch
và duplicate operation ID.

Kiến thức nền: [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md).
