# Effective isolation analysis

## Kỳ vọng và kết quả thật

Business code muốn hai lần đọc dùng cùng một stable snapshot:

```text
expected: firstPrice = 100, secondPrice = 100
actual:   firstPrice = 100, secondPrice = 120
```

`REPEATABLE_READ` xuất hiện trên `readTwice()`, nhưng kết quả thật có semantics của
PostgreSQL `READ COMMITTED`. Đây không phải Hibernate cache bug; physical
transaction đã được outer method tạo với isolation yếu hơn.

> **Nói ngắn gọn:** Spring quyết định isolation khi bắt đầu physical transaction,
> không quyết định lại mỗi khi đi qua một annotated method.

## Timeline tái hiện

Giả sử giá ban đầu là `100`:

| Bước | Reader — Tx-R | Writer — Tx-W |
| --- | --- | --- |
| R0 | `ReportFacade.generate()` mở transaction DEFAULT | |
| R1 | Inner REQUIRED join Tx-R | |
| R2 | `SELECT price` trả `100`; mở gate cho writer | |
| W1 | | Mở transaction độc lập |
| W2 | | `UPDATE product SET price = 120` |
| W3 | | Commit |
| R3 | `SELECT price` lần hai trả `120` | |
| R4 | Commit report với cặp `100/120` | |

Ở PostgreSQL `READ COMMITTED`, mỗi statement lấy snapshot mới tại thời điểm
statement bắt đầu. Vì W3 nằm giữa R2 và R3, hai SELECT hợp lệ khi nhìn thấy hai
committed versions khác nhau.

Nếu Tx-R thật sự là `REPEATABLE READ`, snapshot được cố định cho transaction khi
query đầu tiên thiết lập snapshot. R3 vẫn thấy `100`; transaction độc lập bắt đầu
sau R4 mới thấy `120`.

## Transaction boundary được tạo ở đâu?

Luồng interception rút gọn:

```text
client
  -> ReportFacade proxy
       -> begin physical transaction (DEFAULT -> READ COMMITTED)
       -> SnapshotQueryService proxy
            -> REQUIRED finds existing transaction
            -> join; no second begin; no isolation upgrade
            -> execute readTwice()
       -> commit physical transaction
```

Outer `@Transactional` là transaction creation point. Khi JDBC connection bắt đầu
transaction, transaction manager đã áp dụng isolation. Inner proxy chỉ tạo một
logical transaction scope dùng chung physical transaction.

Annotation inner không vô dụng trong mọi call: nếu client gọi trực tiếp bean đó
khi chưa có transaction, nó sẽ tạo transaction `REPEATABLE READ`. Behavior vì vậy
phụ thuộc call path — một dấu hiệu thiết kế boundary dễ gây lỗi.

## Vì sao mismatch có thể im lặng?

Với propagation `REQUIRED`, Spring thường ưu tiên characteristics của transaction
đang tồn tại. Inner declaration có thể ghi `REPEATABLE_READ`, `readOnly=true` hoặc
timeout khác, nhưng nó không tự động thay characteristics của physical
transaction đã mở.

Khi `validateExistingTransaction` không bật, mismatch isolation thường được chấp
nhận để inner scope join. Bật validation biến lỗi semantics im lặng thành
`IllegalTransactionStateException` trước khi query chạy.

Validation là guardrail, không phải cơ chế upgrade. Cách sửa chính vẫn là đặt
isolation đúng tại transaction creation point.

## Đo effective isolation

Không nên suy luận effective setting chỉ từ annotation hoặc log. Query sau phải
chạy bằng đúng JDBC connection và bên trong đúng transaction đang kiểm tra:

```sql
select current_setting('transaction_isolation');
```

Kết quả của broken path là:

```text
read committed
```

Kết quả mong đợi sau khi sửa outer boundary:

```text
repeatable read
```

`SHOW transaction_isolation` cũng dùng được, nhưng `current_setting(...)` thuận
tiện hơn khi lấy scalar bằng `JdbcTemplate`.

## Vai trò của DEFAULT và DataSource

`Isolation.DEFAULT` bảo transaction manager dùng default của resource. Với
PostgreSQL mặc định thường là `READ COMMITTED`, nhưng application không nên biến
giả định environment thành business contract.

Effective value có thể chịu ảnh hưởng từ:

- `default_transaction_isolation` ở database, role hoặc session;
- connection initialization của pool;
- transaction manager và JDBC driver;
- explicit isolation trên outer transaction;
- một transaction có sẵn do caller mở.

Do connection pool tái sử dụng connection, thay session setting tùy tiện cũng có
thể leak behavior sang request khác nếu pool không reset đúng.

## Self-invocation là một failure mode khác

Trong biến thể self-call:

```java
this.readWithStableSnapshot(productId);
```

call không đi qua Spring proxy. Nếu outer method không transactional, annotation
trên helper không được xử lý và intended transaction không tồn tại. Repository
methods có thể tự mở các transaction ngắn riêng biệt, làm boundary còn rời rạc hơn.

Cần phân biệt:

- **separate proxied bean + REQUIRED:** annotation được đọc nhưng join transaction
  đã có;
- **self-invocation:** interceptor không chạy cho inner call.

Cả hai đều có cùng bài học: isolation chỉ đáng tin khi call thực tế đi qua proxy
tại nơi physical transaction được tạo.

## `readOnly` không tạo stable snapshot

`@Transactional(readOnly = true)` là hint/policy về write behavior và có thể ảnh
hưởng flush mode. Nó không chuyển `READ COMMITTED` thành `REPEATABLE READ`, không
khóa row và không bảo đảm hai SELECT nhìn cùng snapshot.

Tương tự, `flush()`, `clear()` hoặc refresh persistence context không nâng
database isolation. Những thao tác đó thay cách JPA đồng bộ/cache entity, không
thay MVCC snapshot của PostgreSQL.

## `REQUIRES_NEW` có hiệu lực nhưng đổi semantics

Inner `REQUIRES_NEW` suspend outer transaction và mở physical transaction mới, nên
`REPEATABLE_READ` có thể được áp dụng thật. Tuy nhiên nó cũng tạo:

- independent commit/rollback boundary;
- nhu cầu thêm một connection trong lúc outer connection đang được giữ;
- khả năng chờ lock giữa inner và suspended outer;
- kết quả inner có thể commit dù outer rollback;
- snapshot không bao phủ work đã làm ở outer transaction.

Vì vậy `REQUIRES_NEW` chỉ đúng khi independent unit là requirement, không phải
mẹo để “ép annotation chạy”.

## Isolation cao hơn vẫn có failure hợp lệ

`REPEATABLE READ` hoặc `SERIALIZABLE` không có nghĩa mọi transaction sẽ commit.
PostgreSQL có thể abort transaction khi phát hiện update conflict hoặc
serialization anomaly; application phải phân loại SQLSTATE `40001` và retry toàn
bộ idempotent unit bằng transaction mới khi policy cho phép.

Retry không giải quyết SPR-004 nếu transaction vẫn được tạo ở `READ COMMITTED`.
Trước tiên phải sửa boundary; sau đó mới thiết kế retry cho failure của isolation
đã chọn.

## Multi-instance và scope của case

Không có JVM lock nào sửa được case này khi nhiều application instances cùng truy
cập PostgreSQL. Effective isolation là database transaction property và có hiệu
lực qua mọi instance dùng cùng database.

Case này tập trung vào Spring configuration mismatch. Chi tiết về lost update,
write skew, phantom và lựa chọn locking/constraint nằm ở các DB cases. Một stable
snapshot cũng chưa chắc bảo vệ invariant có writes; isolation phải được chọn theo
business invariant, không theo tên anomaly duy nhất.

## Dấu hiệu quan sát trong production

Nên log/metric có kiểm soát:

- use case và outer transaction name;
- propagation/isolation được khai báo ở public entry;
- effective isolation sampled từ database khi chẩn đoán;
- SQLSTATE và retry count;
- pool usage khi có `REQUIRES_NEW`;
- report/version identifiers thay vì raw sensitive data.

Không nên query setting cho mọi request chỉ để bù cho boundary mơ hồ. Xác nhận nó
trong integration test và dùng diagnostic sampling khi cần.

