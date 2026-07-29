# Code lỗi — dirty checking không có version predicate

## Entity không có `@Version`

```java
@Entity
@Table(name = "product_offer")
public class ProductOffer {

    @Id
    private Long offerId;

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private String title;

    protected ProductOffer() {
    }

    public void changePrice(BigDecimal newPrice) {
        if (newPrice == null || newPrice.signum() < 0) {
            throw new IllegalArgumentException("invalid price");
        }
        this.price = newPrice;
    }
}
```

DDL:

```sql
create table product_offer (
    offer_id bigint primary key,
    price numeric(19, 2) not null,
    title varchar(200) not null
);
```

## Service read → mutate → flush

```java
@Service
public class BrokenOfferEditor {

    private final ProductOfferRepository offers;

    public BrokenOfferEditor(ProductOfferRepository offers) {
        this.offers = offers;
    }

    @Transactional
    public OfferView changePrice(long offerId, BigDecimal newPrice) {
        ProductOffer offer = offers.findById(offerId)
                .orElseThrow(() -> new OfferNotFoundException(offerId));

        offer.changePrice(newPrice);
        offers.flush();
        return OfferView.from(offer);
    }
}
```

Hibernate phát absolute update tương đương:

```sql
update product_offer
set price = ?,
    title = ?
where offer_id = ?;
```

PostgreSQL serialize physical row updates bằng row lock, nhưng sau A commit, B
vẫn update current row bằng value tính từ stale entity. Predicate chỉ có primary
key nên B affected rows `1`; database không biết edit của B dựa trên revision cũ.

> **Nói ngắn gọn:** row lock ngăn hai writes sửa tuple cùng một microsecond, nhưng
> không phát hiện semantic stale state nếu `WHERE` không có expected version.

## Timeline lỗi

```text
A load price=100
B load price=100
A set 90, flush, commit
B set 80, flush sau A, commit
final price=80; cả A và B đều nhận success
```

Đây là lost update: durable effect của A bị B overwrite.

## `save()` không thêm conflict detection

```java
ProductOffer saved = offers.save(offer);
```

`save()` chọn persist/merge theo entity state; nó không tự thêm version predicate
cho entity không có `@Version`. Conflict semantics đến từ mapping/SQL contract,
không từ tên repository method.

## Chỉ dùng `@DynamicUpdate` chưa đủ

`@DynamicUpdate` có thể giới hạn columns được update, giúp disjoint field edits
ít overwrite nhau. Nó không phát hiện hai actors cùng sửa `price`, và cũng có thể
merge hai edits vốn cần aggregate-level review. Đây là SQL-shape optimization,
không thay optimistic concurrency contract.

## API bỏ mất version client đã đọc

```java
public record ChangePriceRequest(BigDecimal price) {
}

@Transactional
public OfferView changePrice(long id, ChangePriceRequest request) {
    ProductOffer current = offers.findById(id).orElseThrow();
    current.changePrice(request.price());
    return OfferView.from(current);
}
```

Ngay cả sau khi entity được thêm `@Version`, request không mang version. Nếu A đã
commit trước khi B gọi service, B load current version rồi ghi thành công; JPA
không thể biết B đang submit một form cũ.

## Tự tăng version trong application

```java
offer.setVersion(offer.getVersion() + 1);
```

Version do persistence provider quản lý. Application sửa trực tiếp làm behavior
không portable/undefined và có thể phá expected-version predicate.

## Catch conflict rồi tiếp tục cùng transaction

```java
@Transactional
public OfferView editWithRetry(EditOffer command) {
    try {
        return apply(command);
    } catch (ObjectOptimisticLockingFailureException conflict) {
        entityManager.clear();
        return apply(command); // vẫn cùng failed/rollback-only transaction
    }
}
```

`OptimisticLockException` trong persistence context đã join active transaction
đánh dấu transaction rollback. `clear()` chỉ detach entities, không tạo physical
transaction mới. Robust retry thuộc `LOCK-002`.

## Conflict có thể nằm ngoài method body

```java
@Transactional
public OfferView edit(...) {
    offer.changePrice(...);
    return OfferView.from(offer); // commit/flush diễn ra sau return
}
```

Controller không được giả định return value này đã commit. Exception có thể phát
ở auto-flush hoặc commit trong transaction interceptor.

## Local lock sai scope

```java
synchronized (this) {
    editor.changePrice(...);
}
```

App-1 và App-2 có hai monitor khác nhau. Nó giảm concurrency cục bộ nhưng SQL
không có version predicate vẫn cho cross-node overwrite.

## Điều kiện để tái hiện

1. Một committed offer giá `100`.
2. Hai physical transactions/persistence contexts.
3. Cả hai load trước khi actor nào commit.
4. A flush/commit `90`.
5. B flush/commit `80`.
6. Assert cả hai calls success và final price `80`.
7. Dùng PostgreSQL Testcontainers, không outer test transaction.

## Các cách sửa chưa đủ

- Chỉ thêm `@Transactional`.
- Chỉ gọi `saveAndFlush`.
- Chỉ nâng `READ COMMITTED` lên `REPEATABLE READ` mà không xử lý abort.
- Chỉ dùng `@DynamicUpdate`.
- Chỉ thêm version vào response nhưng không yêu cầu request gửi lại.
- Catch mọi conflict rồi auto-retry user intent.
- Dùng timestamp do application clock tự quản lý làm version.
- Thêm JVM mutex cho multi-instance deployment.
- Assert exception nhưng không kiểm tra final state/version/rollback.
