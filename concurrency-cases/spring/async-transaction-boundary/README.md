# SPR-002 — Async work thoát khỏi transaction của caller

## Tóm tắt

`OrderPlacementService` tạo order trong một transaction rồi gọi `@Async` processor.
Async method chạy trên executor thread khác nên không tham gia transaction của
caller. Nó có thể query trước outer commit và không thấy order; hoặc commit một
side effect độc lập rồi outer transaction rollback, để lại orphan work.

Case bảo vệ các invariant:

```text
Async work phụ thuộc order chỉ được dispatch sau khi order transaction commit.
Outer rollback không được tạo notification/audit được coi là hợp lệ cho order.
Không truyền EntityManager, managed entity hoặc transaction context qua thread.
Durability requirement phải được nói rõ: local after-commit hay transactional outbox.
```

> **Nói ngắn gọn:** chuyển code sang thread khác cũng chuyển nó ra khỏi chiếc hộp
>transaction hiện tại; `@Async` không phải continuation của database transaction.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| thread-bound transaction | Transaction/resource gắn với thread đang thực thi |
| async boundary | Điểm work được giao sang executor thread khác |
| after-commit work | Work chỉ được schedule sau khi transaction commit thành công |
| orphan side effect | Side effect tồn tại dù business aggregate gốc rollback |
| context propagation | Truyền MDC/security/trace context; không đồng nghĩa truyền transaction |
| transactional event listener | Listener chạy theo phase commit/rollback của transaction publish event |
| local handoff | Giao work vào in-process executor, có thể mất khi process crash |
| transactional outbox | Ghi business state và event record trong cùng database transaction |

## Bối cảnh và transaction boundary

| Thành phần | Giá trị |
| --- | --- |
| Caller | Request thread, outer `@Transactional` |
| Async worker | `orderExecutor` thread |
| Caller transaction | Insert order, commit/rollback ở cuối entry method |
| Worker transaction | Không có, hoặc transaction mới nếu async method annotated |
| Race window | Sau async submit, trước outer commit |
| Database | PostgreSQL `READ COMMITTED` |
| Durability boundary | Local executor không durable |

## Điều hướng

- [Broken code](broken-code.md)
- [Thread/transaction timeline](analysis.md)
- [After-commit và outbox alternatives](solutions.md)
- [PostgreSQL integration experiments](experiments.md)
- [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- async query báo “order not found” theo timing;
- notification/audit/job commit cho order đã rollback;
- lazy-loading exception khi entity đi qua thread;
- future cancel nhưng worker transaction vẫn commit;
- executor rejection sau commit làm local event bị mất;
- trace/security context sai nếu propagation policy không explicit.

## Hướng sửa khuyến nghị

- Work phải cùng atomic boundary: chạy synchronous trong caller transaction.
- Work chỉ được bắt đầu sau commit, chấp nhận mất khi crash: publish domain event
  và xử lý bằng `@TransactionalEventListener(AFTER_COMMIT)`; có thể kết hợp `@Async`.
- Work phải durable/at-least-once: transactional outbox (`MSG-007`).
- Async method cần database writes riêng: mở transaction mới trên worker sau
  after-commit dispatch, không cố truyền transaction caller.

## Phạm vi

Case xử lý local async invocation. Durable event publication, inbox/outbox và
message redelivery thuộc `MSG-007`; executor starvation thuộc `JVM-008`.
