# Broken annotation placement

## Inner REQUIRED không nâng isolation

```java
@Service
public class ReportFacade {
    private final SnapshotQueryService snapshotQueries;

    @Transactional
    public PriceReport generate(long productId) {
        return snapshotQueries.readTwice(productId);
    }
}
```

```java
@Service
public class SnapshotQueryService {
    private final PriceRepository prices;
    private final SnapshotProbe probe;

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public PriceReport readTwice(long productId) {
        long first = prices.getPrice(productId);
        probe.afterFirstRead();
        long second = prices.getPrice(productId);
        return new PriceReport(first, second);
    }
}
```

Call đi qua inner proxy, nhưng propagation mặc định `REQUIRED` join outer physical
transaction. Outer được tạo với `Isolation.DEFAULT`; DataSource/PostgreSQL default
là `READ COMMITTED`. Inner annotation không restart hoặc upgrade transaction.

> **Nói ngắn gọn:** proxy interception có xảy ra, nhưng interceptor thấy transaction
>đã tồn tại và join nó với attributes đã được quyết định trước đó.

## Self-invocation variant

```java
public PriceReport generate(long productId) {
    return this.readWithStableSnapshot(productId);
}

@Transactional(isolation = Isolation.REPEATABLE_READ)
public PriceReport readWithStableSnapshot(long productId) { ... }
```

Nếu entry không transactional và self-call bỏ qua proxy, không có intended service
transaction; repository calls có thể chạy trong transaction riêng. Đây là SPR-001
kết hợp isolation mismatch.

## Datasource default assumption

`@Transactional(isolation = Isolation.DEFAULT)` không có nghĩa một portable fixed
level. Nó dùng transaction manager/DataSource/database default. Environment đổi
`default_transaction_isolation` có thể đổi behavior nếu config không explicit.

## Những cách sửa chưa đủ

- Đặt isolation cao trên private/self-invoked helper.
- Annotate inner REQUIRED và giả định nó override outer.
- Dùng `readOnly=true` và coi đó là stable snapshot.
- Gọi `saveAndFlush`/`EntityManager.clear`; không đổi physical isolation.
- Đổi sang `REQUIRES_NEW` chỉ để annotation có hiệu lực mà không xem independent
  commit, second connection và outer lock interaction.
- Chỉ assert annotation bằng reflection; không đo effective transaction.
- Chọn `SERIALIZABLE` nhưng không xử lý serialization retry.

## Điều kiện tái hiện

Outer physical transaction đã tồn tại ở weaker isolation; validation incompatible
attributes tắt (mặc định thường dùng); writer commit giữa hai statements; reader
logic cần stable snapshot.
