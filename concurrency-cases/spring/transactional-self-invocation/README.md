# SPR-001 — Self-invocation bỏ qua transactional proxy

## Tóm tắt

Một transfer service gọi `executeTransfer()` trên chính đối tượng của nó. Method này có
`@Transactional`, nhưng lời gọi không đi qua Spring proxy nên service transaction
không được tạo. Hai repository update tự mở transaction riêng: debit commit trước,
credit commit sau. Luồng đọc đồng thời có thể thấy tổng số dư tạm thời bị giảm;
nếu credit thất bại, việc cập nhật một phần trở thành vĩnh viễn.

Invariant:

```text
Debit và credit của một transfer phải cùng commit hoặc cùng rollback.
Luồng đọc ngoài transaction không được thấy chỉ một nửa transfer đã commit.
Transfer lỗi không được làm thay đổi tổng số dư.
```

> **Nói ngắn gọn:** annotation nằm đúng method chưa đủ; lời gọi phải đi qua đúng
>Spring proxy để transaction boundary tồn tại.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| self-invocation | Bean gọi method khác trên chính `this` |
| proxy interception | Lời gọi đi qua proxy để transaction advice chạy |
| partial commit | Một phần thao tác commit còn phần khác chưa/không commit |
| repository transaction | Transaction ngắn do repository proxy tạo cho một method |
| flush | Gửi SQL trong transaction, chưa phải commit |
| rollback boundary | Phạm vi công việc được hoàn tác cùng nhau |
| external visibility | Trạng thái mà transaction khác có thể đọc theo isolation |

## Bối cảnh và transaction boundary

| Thành phần | Giá trị |
| --- | --- |
| Entry | `transfer(command)` không được annotate |
| Self call | `this.executeTransfer(command)` |
| Intended boundary | Cả debit + credit |
| Actual boundary | Mỗi cập nhật repository là một transaction |
| Reader | Transaction độc lập ở `READ COMMITTED` |
| Database | PostgreSQL |

## Điều hướng

- [Broken code](broken-code.md)
- [Proxy, flush và interleaving](analysis.md)
- [Solutions](solutions.md)
- [PostgreSQL integration experiments](experiments.md)
- [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

Mất cân bằng tạm thời/vĩnh viễn, luồng đọc ra quyết định trên trạng thái một phần, sự kiện
được phát hành sai, đối soát phát hiện muộn và unit test không tái hiện proxy.

## Hướng sửa khuyến nghị

Đặt `@Transactional` trên public entry method được bean/controller bên ngoài gọi,
hoặc tách thao tác transaction sang bean riêng và gọi qua bean proxy. Dùng
`TransactionTemplate` khi boundary động/tường minh. Không dùng self-injection như
lựa chọn mặc định.

## Phạm vi

Case chỉ xử lý proxy boundary. Lost update, isolation anomaly và database deadlock
thuộc `DB-*`; rò rỉ thread async thuộc `SPR-002`.
