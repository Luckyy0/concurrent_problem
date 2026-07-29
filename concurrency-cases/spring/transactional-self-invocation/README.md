# SPR-001 — Self-invocation bỏ qua transactional proxy

## Tóm tắt

Một transfer service gọi `executeTransfer()` trên chính object. Method này có
`@Transactional`, nhưng call không đi qua Spring proxy nên service transaction
không được tạo. Hai repository update tự mở transaction riêng: debit commit trước,
credit commit sau. Reader đồng thời có thể thấy tổng balance tạm thời bị giảm;
nếu credit fail, partial update trở thành vĩnh viễn.

Invariant:

```text
Debit và credit của một transfer phải cùng commit hoặc cùng rollback.
Reader ngoài transaction không được thấy chỉ một nửa transfer đã commit.
Transfer failure không được làm thay đổi tổng balance.
```

> **Nói ngắn gọn:** annotation nằm đúng method chưa đủ; call phải đi qua đúng
>Spring proxy để transaction boundary tồn tại.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| self-invocation | Bean gọi method khác trên chính `this` |
| proxy interception | Call đi qua proxy để transaction advice chạy |
| partial commit | Một phần operation commit còn phần khác chưa/không commit |
| repository transaction | Transaction ngắn do repository proxy tạo cho một method |
| flush | Gửi SQL trong transaction, chưa phải commit |
| rollback boundary | Phạm vi work được hoàn tác cùng nhau |
| external visibility | State transaction khác có thể đọc theo isolation |

## Bối cảnh và transaction boundary

| Thành phần | Giá trị |
| --- | --- |
| Entry | `transfer(command)` không annotated |
| Self call | `this.executeTransfer(command)` |
| Intended boundary | Cả debit + credit |
| Actual boundary | Mỗi repository update một transaction |
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

Mất cân bằng tạm thời/vĩnh viễn, reader ra quyết định trên partial state, event
được publish sai, reconciliation phát hiện muộn và test unit không tái hiện proxy.

## Hướng sửa khuyến nghị

Đặt `@Transactional` trên public entry method được external bean/controller gọi,
hoặc tách transactional operation sang bean riêng và gọi qua bean proxy. Dùng
`TransactionTemplate` khi boundary động/explicit. Không dùng self-injection như
lựa chọn mặc định.

## Phạm vi

Case chỉ xử lý proxy boundary. Lost update, isolation anomaly và database deadlock
thuộc `DB-*`; async thread escape thuộc `SPR-002`.
