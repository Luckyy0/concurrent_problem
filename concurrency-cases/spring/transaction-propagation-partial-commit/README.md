# SPR-003 — Propagation tạo partial-commit workflow

## Tóm tắt

Checkout transaction đổi order từ `PENDING` sang `PAID`. Bên trong, audit bean
dùng `REQUIRES_NEW` ghi `PAYMENT_COMPLETED` và commit độc lập. Nếu outer transaction
thất bại, order rollback về `PENDING` nhưng audit “completed” vẫn tồn tại.

Case cũng phân tích chiều ngược: inner method dùng `REQUIRED`, ném runtime exception
qua proxy và đánh dấu physical transaction rollback-only. Outer catch exception
rồi tiếp tục, nhưng commit cuối vẫn ném `UnexpectedRollbackException`.

Invariant:

```text
Record mô tả business success chỉ tồn tại khi business transaction commit.
Work cần atomicity phải cùng physical transaction.
Work cố ý sống qua rollback phải có semantics trung thực như ATTEMPT/FAILURE.
Caller không được trả success khi transaction đã rollback-only.
```

> **Nói ngắn gọn:** propagation không chỉ quyết định “có transaction hay không”;
>nó quyết định phần nào có thể commit/rollback độc lập.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| logical transaction scope | Phạm vi method/advice mang propagation policy |
| physical transaction | Database transaction/connection thật sự commit hoặc rollback |
| `REQUIRED` | Join physical transaction hiện có hoặc tạo mới |
| `REQUIRES_NEW` | Suspend outer transaction và tạo physical transaction độc lập |
| rollback-only | Physical transaction đã bị đánh dấu không thể commit |
| `UnexpectedRollbackException` | Outer code yêu cầu commit nhưng transaction đã rollback-only |
| partial commit | Một phần workflow commit dù phần khác rollback |
| suspended transaction | Outer transaction tạm ngừng trong khi inner transaction chạy |

## Bối cảnh và boundary

| Thành phần | Giá trị |
| --- | --- |
| Outer | Checkout Tx-O: order PENDING→PAID, inventory/payment state |
| Inner | Audit Tx-A: insert `PAYMENT_COMPLETED` bằng `REQUIRES_NEW` |
| Failure | Outer validation/inventory step ném exception sau audit |
| Actual result | Tx-A commit, Tx-O rollback |
| Database | PostgreSQL |
| Scope | Một application/database transaction model |

## Điều hướng

- [Broken propagation code](broken-code.md)
- [Physical/logical transaction analysis](analysis.md)
- [Correct boundary choices](solutions.md)
- [PostgreSQL integration experiments](experiments.md)
- [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- audit/event nói payment completed cho order còn pending;
- downstream xử lý record đã commit độc lập;
- outer catch inner REQUIRED error nhưng cuối cùng vẫn rollback;
- `REQUIRES_NEW` giữ thêm connection và có thể làm cạn pool;
- inner transaction block trên lock do suspended outer transaction giữ;
- retry tạo duplicate audit nếu không có idempotency.

## Hướng sửa khuyến nghị

Work mô tả cùng business outcome dùng `REQUIRED` và cùng outer transaction, hoặc
publish success sau commit/outbox. Chỉ dùng `REQUIRES_NEW` khi independent commit
là requirement thật; record phải mô tả attempt/failure độc lập, không giả success.
Đừng catch exception từ REQUIRED inner scope rồi trả success; propagate hoặc đổi
expected branch thành explicit outcome không đánh dấu rollback-only.

## Phạm vi

Case không thiết kế distributed saga/compensation (`DIST-004`) và không giải quyết
isolation anomaly (`DB-*`).
