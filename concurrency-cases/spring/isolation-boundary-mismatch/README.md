# SPR-004 — Isolation annotation không khớp transaction boundary thật

## Tóm tắt

`ReportFacade.generate()` mở outer transaction bằng isolation mặc định PostgreSQL
`READ COMMITTED`. Nó gọi bean `SnapshotQueryService.readTwice()` được annotated
`REPEATABLE_READ`. Vì propagation mặc định là `REQUIRED`, inner method join physical
transaction đã tồn tại; annotation không nâng isolation.

Writer commit price mới giữa hai SELECT. Report nhận `100` rồi `120`, trái kỳ vọng
stable snapshot.

Invariant:

```text
Hai lần đọc tạo cùng report phải dùng snapshot semantics đã công bố.
Effective isolation phải được đặt tại nơi physical transaction được tạo.
Isolation mismatch không được bị silently chấp nhận trong boundary quan trọng.
Test phải xác nhận cả effective setting và business read outcome.
```

> **Nói ngắn gọn:** isolation thuộc transaction, không thuộc riêng đoạn method có
>annotation đẹp nhất.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| effective isolation | Isolation thật của physical transaction hiện tại |
| isolation declaration | Isolation method mong muốn trong `@Transactional` |
| isolation inheritance | Inner REQUIRED dùng isolation của transaction đã có |
| datasource default | Isolation được dùng khi declaration là `DEFAULT` |
| non-repeatable read | Cùng row đọc hai lần thấy hai committed values |
| transaction creation point | Proxy call nơi transaction manager mở physical transaction |
| validation existing transaction | Fail-fast khi inner attributes không tương thích |

## Bối cảnh và boundary

| Thành phần | Giá trị |
| --- | --- |
| Outer entry | `ReportFacade.generate`, `@Transactional` DEFAULT |
| Database default | PostgreSQL `READ COMMITTED` |
| Inner | `readTwice`, annotated `REPEATABLE_READ`, propagation REQUIRED |
| Writer | Transaction khác update price và commit giữa hai reads |
| Effective result | Cả reads chạy trong outer READ COMMITTED |
| Scope | Spring configuration; anomaly chi tiết thuộc DB cases |

## Điều hướng

- [Broken annotation placement](broken-code.md)
- [Effective isolation analysis](analysis.md)
- [Correct boundary and fail-fast options](solutions.md)
- [PostgreSQL integration experiments](experiments.md)
- [Isolation levels](../../concepts/isolation-levels.md)
- [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

Report/reconciliation ghép data từ nhiều committed moments; code review tin nhầm
annotation; datasource default thay đổi giữa environment; inner declaration bị
ignore; nâng bằng `REQUIRES_NEW` vô tình tạo partial commit/resource cost.

## Hướng sửa khuyến nghị

Đặt isolation trên public entry method thật sự tạo transaction và gọi qua Spring
proxy. Bật `validateExistingTransaction` cho boundary cần fail-fast. Chỉ dùng
`REQUIRES_NEW` nếu snapshot/commit độc lập là requirement. Luôn integration test
với PostgreSQL và query effective isolation.

## Phạm vi

Case không giải thích đầy đủ lost update/write skew/phantom; các anomaly nằm trong
`DB-001`–`DB-005`.
