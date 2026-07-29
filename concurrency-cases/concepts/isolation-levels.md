# Isolation level và database visibility

## Mục đích

Tài liệu định nghĩa isolation vocabulary chung cho Spring/PostgreSQL cases. Mỗi
anomaly case vẫn phải mô tả SQL, snapshot và invariant cụ thể.

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| isolation level | Quy tắc transaction quan sát changes của transaction đồng thời |
| dirty read | Đọc data chưa commit của transaction khác |
| non-repeatable read | Cùng row được đọc lại và thấy committed value khác |
| phantom | Predicate query lặp lại thấy tập row thay đổi |
| write skew | Hai transaction đọc cùng constraint rồi update row khác làm constraint sai |
| snapshot | Committed database view mà statement/transaction được phép thấy |
| serialization failure | Database abort transaction vì không thể serialize an toàn |
| effective isolation | Isolation thật của physical transaction đang chạy |

## PostgreSQL levels

- `READ UNCOMMITTED` được xử lý như `READ COMMITTED`.
- `READ COMMITTED`: mỗi statement lấy snapshot mới; hai SELECT trong một
  transaction có thể thấy commits khác nhau.
- `REPEATABLE READ`: transaction dùng một stable snapshot, ngăn non-repeatable
  read/phantom theo MVCC nhưng có thể abort một số write conflict và vẫn cần hiểu
  invariant nhiều row.
- `SERIALIZABLE`: SSI phát hiện dangerous structures và có thể abort với
  serialization failure; application cần safe retry boundary.

> **Nói ngắn gọn:** isolation cao hơn không khóa mọi thứ; nó thay visibility và
>có thể chuyển anomaly thành transaction abort cần retry.

## Snapshot, lock và commit

MVCC snapshot quyết định version được đọc; row locks quyết định writer/blocking.
`SELECT FOR UPDATE` không đồng nghĩa `REPEATABLE READ`, và isolation không thay
unique constraint/atomic update. Flush gửi SQL nhưng commit mới làm version visible
cho transaction khác theo rules.

## Spring effective isolation

Isolation được áp dụng khi transaction manager tạo physical transaction. Inner
`REQUIRED` join transaction có sẵn và không nâng/hạ isolation. `DEFAULT` dùng
DataSource/database default. Self-invocation hoặc non-managed object có thể khiến
annotation không tạo transaction.

Có thể bật `validateExistingTransaction` để Spring fail khi inner declaration
không tương thích transaction đang có, thay vì silently join. `REQUIRES_NEW` tạo
physical transaction/isolation riêng nhưng cũng tạo independent commit boundary.

## Chọn isolation theo invariant

Không chọn mức cao nhất theo phản xạ. Xác định read/write set, anomaly cần cấm,
conflict/retry behavior, lock duration và throughput. Constraint/conditional SQL
thường bảo vệ invariant trực tiếp hơn chỉ nâng isolation.

## Kiểm thử

Dùng PostgreSQL Testcontainers, transaction thật trên connection riêng, latch
đặt writer commit giữa reader statements. Query
`current_setting('transaction_isolation')` để xác nhận effective level, nhưng luôn
assert business outcome/anomaly. Test không được bị outer test transaction che.

## Liên kết

- [SPR-004 — Isolation boundary mismatch](../spring/isolation-boundary-mismatch/README.md)
- [DB-001 — Lost update under MVCC](../postgresql/lost-update-mvcc/README.md)
- [DB-002 — Dirty-read expectations](../postgresql/dirty-read-expectation/README.md)
- [DB-003 — Non-repeatable read](../postgresql/non-repeatable-read/README.md)
- [DB-004 — Phantom capacity check](../postgresql/phantom-capacity-check/README.md)
- [DB-005 — Write skew](../postgresql/write-skew/README.md)
- [DB-009 — Serializable abort and safe retry](../postgresql/serializable-retry/README.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Concurrency testing](concurrency-testing.md)
