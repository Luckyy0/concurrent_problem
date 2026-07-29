# LOCK-004 — Conditional atomic `UPDATE`

## Tóm tắt

Kho sản phẩm `77` còn `5` đơn vị. Hai checkout khác nhau cùng muốn giữ `4`.
Nếu application đọc tồn kho, kiểm tra rồi ghi absolute values, cả hai có thể cùng
accept trên state `5`; cuối cùng database chỉ còn projection của writer cuối
nhưng đã có hai reservation tổng cộng `8`.

Khi invariant nằm gọn trong một predicate, gửi thẳng business intent tới
authoritative store:

```sql
update inventory_item
set available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
where product_id = :productId
  and :quantity > 0
  and available_quantity >= :quantity;
```

Affected-row count `1` nghĩa là reservation đã thắng. `0` nghĩa là mutation
không được áp dụng; caller không được tạo reservation/outbox như thể đã thành
công.

> **Nói ngắn gọn:** đừng hỏi “còn hàng không?” rồi ghi sau; hãy yêu cầu database
> “chỉ giữ hàng nếu ngay lúc ghi vẫn còn đủ”.

## Actor và trạng thái dùng chung

| Thành phần | Trạng thái ban đầu |
| --- | --- |
| `inventory_item` | Product `77`, available `5`, reserved `0` |
| Command A | Order `A`, quantity `4`, chạy trên App-1 |
| Command B | Order `B`, quantity `4`, chạy trên App-2 |
| `inventory_reservation` | Durable outcome theo command ID |
| `outbox_event` | Chỉ được tạo cho reservation đã commit |

Điểm tranh chấp là `inventory_item(product_id=77)`. Hai commands có ID khác nhau,
nên idempotency không làm chúng trùng nhau; chính conditional mutation bảo vệ
cùng stock row.

## Invariant

Trong phạm vi không có restock/shipment:

```text
available_quantity >= 0

reserved_quantity
  = tổng quantity của các reservation có outcome RESERVED

available_quantity + reserved_quantity = on_hand_quantity ban đầu

Mỗi command ID chỉ tạo một durable outcome.

Caller chỉ nhận RESERVED sau transaction commit.
```

`CHECK (available_quantity >= 0)` là defense in depth. Nó không đủ để phát hiện
lost update làm hai audit rows cùng được accept nhưng counter chỉ phản ánh một.

## Ranh giới transaction

Một `InventoryReservationTx.reserve()` attempt bao gồm:

1. atomically claim command ID hoặc replay durable outcome;
2. chạy conditional `UPDATE ... RETURNING`;
3. nếu zero row, lưu outcome `OUT_OF_STOCK` và không tạo outbox;
4. nếu có row, lưu `RESERVED` cùng remaining quantities và tạo outbox;
5. commit/rollback cả claim, stock mutation và side effects database.

Remote payment, message publish và retry wait nằm ngoài transaction. Nếu bước
sau UPDATE lỗi, rollback phải hoàn nguyên cả stock counters.

## Vì sao concurrent `UPDATE` vẫn đúng?

Ở PostgreSQL `READ COMMITTED`:

```text
Tx-A UPDATE: predicate 5 >= 4 → true → row becomes available=1 → holds lock
Tx-B UPDATE: targets same row → waits
Tx-A COMMIT: releases lock
Tx-B: re-evaluates WHERE on current row → 1 >= 4 is false → affected rows 0
```

Nếu Tx-A rollback, Tx-B re-evaluate trên state `5` và có thể affected rows `1`.
Database expression trừ/cộng dùng current row version; application không gửi
absolute value tính từ stale snapshot.

## Outcome contract

| Database outcome | Domain outcome |
| --- | --- |
| Một row returned / affected rows `1` | `RESERVED` |
| Zero rows với product row được bảo đảm tồn tại | `OUT_OF_STOCK` |
| SQLSTATE `55P03` do lock timeout | `BUSY`, transaction rollback |
| SQLSTATE `40P01`/`40001` | Technical conflict; bounded fresh retry nếu an toàn |
| Constraint/insert/outbox failure sau UPDATE | Rollback toàn attempt |
| Duplicate command cùng fingerprint | Replay durable result, không decrement lại |
| Duplicate command khác fingerprint | `IDEMPOTENCY_MISMATCH` |

Zero affected rows tự nó có thể còn nghĩa “product không tồn tại” hoặc một
predicate khác fail. API phải bảo đảm row tồn tại hoặc định nghĩa rõ cách phân
biệt; không đoán từ `0`.

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong case |
| --- | --- |
| conditional mutation | Chỉ mutate khi business predicate còn đúng |
| atomic `UPDATE` | Predicate check và write là một database statement |
| affected-row count | Số rows thực sự được statement update |
| predicate recheck | PostgreSQL đánh giá lại `WHERE` sau khi chờ concurrent writer |
| current row version | Tuple version mới nhất mà updating command được phép xử lý |
| `RETURNING` | Trả state sau mutation từ chính statement thắng |
| no-op | Statement hợp lệ nhưng không row nào được thay đổi |
| bulk DML | Update trực tiếp, bỏ qua dirty checking của managed entity |
| defense in depth | Constraint bổ sung, không thay thế outcome handling |

## Điều hướng

- [Code read–check–write bị hỏng](broken-code.md)
- [Timeline, row lock và predicate recheck](analysis.md)
- [Spring/JPA/JDBC solution và trade-offs](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [Atomic database operations](../../concepts/atomic-database-operations.md)
- [PostgreSQL locks và lock lifetime](../../concepts/postgresql-locks.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)
- [DB-001 — Lost update dưới MVCC](../../postgresql/lost-update-mvcc/README.md)
- [DB-007 — Row/table lock lifecycle](../../postgresql/row-table-lock-lifecycle/README.md)

## Hậu quả trong production nếu dùng sai

- Accepted reservation quantity lớn hơn stock counters.
- Projection và audit/reservation rows không reconcile.
- Ignore affected rows làm request thua vẫn trả success.
- Bulk DML để lại stale managed entity rồi một flush sau overwrite atomic result.
- Retry cùng command ID nhưng không có durable claim làm decrement nhiều lần.
- Map lock timeout thành `OUT_OF_STOCK` che giấu hot-row contention.
- External publish trước commit tạo event cho transaction đã rollback.
- `synchronized` pass test một node nhưng hỏng khi scale-out.

## Khi nên dùng cách này

Conditional atomic SQL phù hợp khi:

- mutation và guard cùng nằm trên known row;
- rule diễn đạt được trong `WHERE` và `SET`;
- loser có outcome từ affected-row `0`;
- application không cần load aggregate graph để quyết định;
- short statement tốt hơn pre-lock/read round trip hoặc optimistic retry.

Nếu cần decision nhiều bước trên current state, dùng `PESSIMISTIC_WRITE` như
`LOCK-003`. Nếu edit phức tạp và conflict hiếm, cân nhắc `@Version`. Nếu invariant
bao phủ missing rows/predicate rộng, một conditional UPDATE trên một row không đủ.

## Phạm vi

Case tập trung invariant biểu diễn trong một mutation predicate trên known row.
General lost update thuộc `DB-001`; lock mechanics thuộc `DB-007`; strategy dưới
sustained high contention thuộc `LOCK-005`.
