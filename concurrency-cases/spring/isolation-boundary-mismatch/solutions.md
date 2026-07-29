# Correct boundary and fail-fast options

## Mục tiêu thiết kế

Một giải pháp đúng phải làm rõ bốn điều:

1. public entry nào tạo physical transaction;
2. effective isolation nào bảo vệ toàn bộ business unit;
3. inner call join hay tạo independent transaction;
4. mismatch bị fail-fast hay được chấp nhận có chủ đích.

> **Nói ngắn gọn:** đặt isolation ở nơi transaction bắt đầu, rồi kiểm thử setting
> thật và outcome thật.

## Solution 1 — Đặt isolation trên outer use-case boundary

Đây là lựa chọn mặc định khi toàn bộ report phải dùng một stable snapshot:

```java
@Service
public class ReportFacade {
    private final SnapshotQueryService snapshotQueries;

    public ReportFacade(SnapshotQueryService snapshotQueries) {
        this.snapshotQueries = snapshotQueries;
    }

    @Transactional(
        isolation = Isolation.REPEATABLE_READ,
        readOnly = true
    )
    public PriceReport generate(long productId) {
        return snapshotQueries.readTwice(productId);
    }
}
```

Inner service có thể không annotated nếu chỉ được gọi trong use case này:

```java
@Service
public class SnapshotQueryService {
    private final PriceRepository prices;
    private final SnapshotProbe probe;

    public PriceReport readTwice(long productId) {
        long first = prices.getPrice(productId);
        probe.afterFirstRead();
        long second = prices.getPrice(productId);
        return new PriceReport(first, second);
    }
}
```

Hoặc inner method dùng `MANDATORY` để document rằng caller phải tạo transaction:

```java
@Transactional(propagation = Propagation.MANDATORY, readOnly = true)
public PriceReport readTwice(long productId) {
    // ...
}
```

`MANDATORY` kiểm tra transaction tồn tại nhưng không tự xác nhận isolation. Vì thế
outer boundary và integration test vẫn là source of truth.

### Khi phù hợp

- Một report/use case cần cùng snapshot từ đầu đến cuối.
- Commit/rollback phải bao phủ toàn bộ operation.
- Caller không nên quyết định isolation một cách ngầm định.

### Lưu ý

Public method phải được gọi qua Spring proxy. Không gọi bằng `this.generate(...)`,
không đặt annotation chỉ trên private helper, và không khởi tạo service bằng `new`.

## Solution 2 — Tạo snapshot độc lập bằng `REQUIRES_NEW`

Dùng khi snapshot query thật sự là independent unit:

```java
@Service
public class SnapshotQueryService {

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        isolation = Isolation.REPEATABLE_READ,
        readOnly = true
    )
    public PriceReport readTwice(long productId) {
        // two reads in one independent snapshot
    }
}
```

Call phải đi qua một bean proxy khác. Nếu outer transaction tồn tại, Spring suspend
nó, mở connection/physical transaction mới, rồi resume outer sau khi inner hoàn
tất.

Chỉ chọn cách này khi tất cả điều sau đều đúng:

- independent commit/rollback là requirement;
- inner không cần thấy uncommitted work của outer;
- connection pool có headroom cho nested concurrent calls;
- lock ordering đã được phân tích;
- outer rollback sau đó không cần đảo kết quả inner.

Nếu report chỉ đọc, “commit độc lập” ít gây partial write hơn nhưng vẫn thay
snapshot scope và resource usage.

## Solution 3 — Dùng `TransactionTemplate` cho boundary động

Khi isolation phụ thuộc operation type hoặc flow không phù hợp proxy annotation,
đặt boundary programmatically:

```java
@Service
public class SnapshotReportService {
    private final TransactionTemplate snapshots;
    private final SnapshotQueryService queries;

    public SnapshotReportService(
        PlatformTransactionManager transactionManager,
        SnapshotQueryService queries
    ) {
        this.snapshots = new TransactionTemplate(transactionManager);
        this.snapshots.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW
        );
        this.snapshots.setIsolationLevel(
            TransactionDefinition.ISOLATION_REPEATABLE_READ
        );
        this.snapshots.setReadOnly(true);
        this.queries = queries;
    }

    public PriceReport generate(long productId) {
        return snapshots.execute(status -> queries.readTwice(productId));
    }
}
```

Nếu không cần independent unit, gọi template khi chắc chắn chưa có transaction và
dùng `PROPAGATION_REQUIRED`. Đừng dùng template lồng nhau mà không document
suspend/join semantics.

Ưu điểm là creation point hiển thị trực tiếp trong code. Nhược điểm là transaction
policy trộn vào orchestration và cần test kỹ hơn khi flow có nhiều exit path.

## Solution 4 — Bật validation cho existing transaction

Fail-fast giúp phát hiện inner declaration không tương thích:

```java
@Configuration
class TransactionConfiguration {

    @Bean
    JpaTransactionManager transactionManager(EntityManagerFactory emf) {
        JpaTransactionManager manager = new JpaTransactionManager(emf);
        manager.setValidateExistingTransaction(true);
        return manager;
    }
}
```

Khi outer `READ_COMMITTED` gọi inner REQUIRED `REPEATABLE_READ`, manager từ chối
join thay vì silently chạy ở weaker level.

Trong Spring Boot project thực tế, cần giữ nguyên các customizations đang áp dụng
cho transaction manager (DataSource, JPA dialect, rollback policy, observability).
Nếu Boot đã tạo manager, có thể customize bean hiện hữu thay vì khai báo một bean
thứ hai. Xác nhận chỉ có đúng manager được dùng bởi `@Transactional`.

### Validation không thay thế boundary design

Validation:

- phát hiện conflict;
- không nâng isolation;
- không bảo đảm code không annotated đang chạy trong đúng level;
- có thể làm lộ các mismatch cũ khi rollout.

Nên bật sớm, chạy integration suite, inventory các nested scopes rồi mới rollout
production nếu hệ thống legacy có nhiều implicit joins.

## Solution 5 — Cấu hình datasource default như global policy

Có thể đặt PostgreSQL/session default khi phần lớn transaction cùng cần một level.
Tuy nhiên default là deployment policy, không nên là cách duy nhất diễn đạt một
business-critical invariant.

Nếu dùng, cần:

- cấu hình giống nhau ở mọi environment;
- xác nhận pool reset session state;
- test `Isolation.DEFAULT` cho effective value mong muốn;
- theo dõi throughput, contention và serialization failures;
- vẫn explicit trên boundary có yêu cầu đặc biệt.

Đổi toàn hệ thống từ `READ COMMITTED` sang level cao hơn có thể tăng abort/retry và
giữ snapshot lâu hơn. Không thực hiện như một hot fix thiếu benchmark.

## Khi isolation chưa đủ

Stable snapshot bảo vệ read consistency, nhưng invariant có writes có thể cần:

- unique/check/exclusion constraint;
- atomic conditional update;
- optimistic version;
- `SELECT ... FOR UPDATE`;
- advisory/application ownership phù hợp;
- `SERIALIZABLE` cùng bounded retry.

Ví dụ, hai transaction cùng đọc “còn chỗ” rồi insert có thể vi phạm capacity dù
mỗi transaction đọc một snapshot ổn định. Chọn mechanism theo invariant và write
conflict, không chỉ theo mong muốn “đọc nhất quán”.

## So sánh lựa chọn

| Lựa chọn | Physical boundary | Isolation có hiệu lực? | Trade-off chính |
| --- | --- | --- | --- |
| Isolation trên outer entry | Một transaction cho use case | Có | Boundary rõ; transaction có thể dài |
| Inner REQUIRED | Join outer | Chỉ khi outer tương thích | Dễ phụ thuộc call path |
| Inner REQUIRES_NEW | Transaction độc lập | Có | Suspend, thêm connection, independent outcome |
| `TransactionTemplate` | Explicit theo block | Có khi block tạo transaction | Nhiều orchestration code |
| Validate existing | Không tạo boundary mới | Không upgrade; fail mismatch | Có thể làm lộ lỗi legacy |
| Datasource default | Mọi DEFAULT transaction | Có theo environment | Blast radius lớn, contract kém cục bộ |

## Recommendation cho case này

1. Chuyển `REPEATABLE_READ` lên `ReportFacade.generate()`.
2. Giữ hai SELECT trong cùng outer transaction.
3. Để query service là helper hoặc `MANDATORY`.
4. Bật existing-transaction validation sau khi inventory nested calls.
5. Query effective isolation trong PostgreSQL integration test.
6. Assert report là `100/100` dưới interleaving đã điều phối.
7. Chỉ thêm retry nếu level đã chọn có abort hợp lệ và operation idempotent.

## Production checklist

### Boundary

- [ ] Public entry được gọi qua Spring proxy.
- [ ] Isolation nằm tại physical transaction creation point.
- [ ] Propagation của mọi inner scope được document.
- [ ] `REQUIRES_NEW` chỉ dùng cho independent unit.
- [ ] Không dựa vào self-invocation hoặc private annotation.

### Database

- [ ] PostgreSQL effective isolation được xác nhận.
- [ ] Datasource/session default giống nhau giữa environments.
- [ ] Transaction duration và long-lived snapshots được theo dõi.
- [ ] SQLSTATE `40001` có retry policy bounded nếu cần.
- [ ] Constraints/locking bổ sung bảo vệ write invariant.

### Test và vận hành

- [ ] Test dùng PostgreSQL Testcontainers, không dùng H2 cho MVCC semantics.
- [ ] Interleaving dùng latch/barrier với timeout, không dùng time-based delay.
- [ ] Assert cả setting kỹ thuật và business outcome.
- [ ] Test method không bọc transaction ngoài ý muốn.
- [ ] Pool được sizing cho concurrency và `REQUIRES_NEW`.
- [ ] Log không chứa dữ liệu nhạy cảm.
