# Ranh giới transaction trong Spring

## Mục đích

Tài liệu định nghĩa cách Spring tạo transaction boundary qua proxy, propagation,
flush/commit và thread context. Các case cụ thể áp dụng mà không lặp lại nền tảng.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| transactional proxy | Object trung gian intercept method call và mở/đóng transaction |
| self-invocation | Method trong object gọi method khác trên chính `this`, không đi qua proxy |
| transaction boundary | Phạm vi work cùng commit hoặc rollback |
| propagation | Quy tắc join, tạo mới hoặc chạy ngoài transaction |
| flush | Đồng bộ persistence context thành SQL; chưa đồng nghĩa commit |
| commit | Làm transaction changes có hiệu lực theo database visibility rules |
| rollback-only | Transaction đã bị đánh dấu không thể commit |
| thread-bound context | Transaction/resource được gắn với thread hiện tại |

## Proxy interception

Với proxy mode mặc định, `@Transactional` có hiệu lực khi caller gọi method qua
Spring-managed proxy. `this.method()` là Java call trực tiếp nên interceptor
không chạy. Method visibility/proxy type và việc bean có thật sự do Spring quản
lý cũng thuộc contract.

> **Nói ngắn gọn:** annotation mô tả policy; proxy interception mới là cơ chế
>thực thi policy đó.

## Propagation và rollback

`REQUIRED` join transaction hiện có hoặc tạo transaction mới khi đi qua
interceptor. `REQUIRES_NEW` suspend transaction ngoài và commit/rollback độc lập.
Chọn propagation theo atomicity boundary, không dùng để “chữa” exception mơ hồ.

Mặc định Spring rollback với unchecked exception; checked exception cần policy
rõ (`rollbackFor`) nếu muốn rollback. Catch/swallow exception bên trong boundary
có thể khiến transaction commit hoặc rollback-only dẫn tới lỗi muộn.

## Flush, commit và visibility

Hibernate có thể flush trước query hoặc commit. Flush gửi SQL tới database nhưng
transaction chưa commit; lock vẫn giữ và reader khác nhìn thấy theo isolation.
Không dùng `saveAndFlush` để thay thế transaction boundary. Khi không có service
transaction, repository proxy có thể tạo nhiều transaction nhỏ và commit từng
bước, làm partial state externally visible.

## Thread và async boundary

Transaction context thường thread-bound. `@Async`, executor task và
`CompletableFuture` trên thread khác không tự join transaction của caller. Không
truyền EntityManager/entity lazy state qua thread rồi kỳ vọng cùng transaction.

## Kiểm thử

Integration test phải gọi bean qua Spring proxy, dùng transaction độc lập cho
reader/writer và assert committed business invariant. Test method `@Transactional`
có thể che self-invocation/commit behavior vì nó bọc cả test trong transaction.

## Liên kết

- [SPR-001 — Transactional self-invocation](../spring/transactional-self-invocation/README.md)
- [SPR-005 — Checked exception rollback](../spring/checked-exception-rollback/README.md)
- [SPR-006 — Retry transaction boundary](../spring/retry-transaction-boundary/README.md)
- [LOCK-002 — Bounded optimistic retry](../locking/optimistic-retry-contention/README.md)
- [SPR-007 — Long transaction pool exhaustion](../spring/connection-pool-long-transaction/README.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
