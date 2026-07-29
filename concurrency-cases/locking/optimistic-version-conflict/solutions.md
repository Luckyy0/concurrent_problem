# Giải pháp `@Version` và explicit expected-version contract

## Entity versioned

Migration:

```sql
alter table product_offer add column version bigint;
update product_offer set version = 0 where version is null;
alter table product_offer alter column version set not null;
```

Triển khai migration theo chiến lược tương thích old/new application; writer cũ
không increment version sẽ phá protocol.

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

    @Version
    @Column(nullable = false)
    private long version;

    protected ProductOffer() {
    }

    public long version() {
        return version;
    }

    public void changePrice(BigDecimal newPrice) {
        if (newPrice == null || newPrice.signum() < 0) {
            throw new IllegalArgumentException("invalid price");
        }
        price = newPrice;
    }
}
```

Chỉ persistence provider cập nhật `version`.

## Command mang expected version

```java
public record ChangeOfferPrice(
        long offerId,
        long expectedVersion,
        BigDecimal newPrice,
        UUID commandId
) {
}
```

Service:

```java
@Service
public class OfferEditor {

    private final ProductOfferRepository offers;

    public OfferEditor(ProductOfferRepository offers) {
        this.offers = offers;
    }

    @Transactional
    public OfferView changePrice(ChangeOfferPrice command) {
        ProductOffer offer = offers.findById(command.offerId())
                .orElseThrow(() ->
                        new OfferNotFoundException(command.offerId())
                );

        if (offer.version() != command.expectedVersion()) {
            throw new StaleOfferEditException(
                    command.offerId(),
                    command.expectedVersion(),
                    offer.version()
            );
        }

        offer.changePrice(command.newPrice());
        offers.flush();
        return OfferView.from(offer);
    }
}
```

Explicit check phát hiện client đã stale trước transaction. Versioned UPDATE tiếp
tục phát hiện concurrent commit sau load.

> **Nói ngắn gọn:** request version bảo vệ disconnected edit; `@Version` bảo vệ
> race trong database transaction.

## SQL và loser

```sql
update product_offer
set price = :price,
    title = :title,
    version = :version + 1
where offer_id = :id
  and version = :version;
```

Affected rows `0` làm Hibernate ném `OptimisticLockException`; Spring thường
translate thành `ObjectOptimisticLockingFailureException`. Exception phải thoát
transaction method để rollback.

## API mapping

```java
@RestControllerAdvice
public class OfferConflictAdvice {

    @ExceptionHandler({
            StaleOfferEditException.class,
            ObjectOptimisticLockingFailureException.class
    })
    ResponseEntity<ProblemDetail> conflict(RuntimeException failure) {
        ProblemDetail problem = ProblemDetail.forStatus(
                HttpStatus.PRECONDITION_FAILED
        );
        problem.setTitle("Offer changed since it was read");
        problem.setProperty("reloadRequired", true);
        return ResponseEntity
                .status(HttpStatus.PRECONDITION_FAILED)
                .body(problem);
    }
}
```

Nếu API dùng ETag, map `If-Match` thất bại thành `412`; `409` cũng có thể đúng cho
domain command conflict. Không trả stale entity như committed result.

## Flush và outer boundary

Explicit `flush()` giúp conflict xuất hiện trong attempt method, nhưng commit vẫn
là boundary cuối. Controller gọi qua proxy và advice xử lý exception sau rollback.
Không catch/continue cùng persistence context.

## Detached merge

Có thể merge detached entity có version, nhưng DTO + explicit field allowlist và
expectedVersion thường an toàn hơn:

- tránh mass assignment;
- không cascade graph ngoài ý muốn;
- domain validation chạy trên current managed entity;
- conflict response dễ map.

## Bulk/native writers

Nếu bắt buộc native update:

```sql
update product_offer
set price = :newPrice,
    version = version + 1
where offer_id = :id
  and version = :expectedVersion;
```

Phải kiểm tra affected rows và clear/refresh persistence context phù hợp.
JPQL bulk update bypass dirty checking của managed instances; không chạy cạnh
entity writes mà không có protocol/test rõ.

## Không auto-retry user edit

Outcome khuyến nghị:

```text
winner → commit, return new version
loser  → rollback, return conflict/reload-required
```

Nếu command là commutative/deterministic trên fresh state, bounded retry có thể
đúng nhưng phải dùng transaction mới, reload, backoff/jitter, deadline và
idempotency. Đó là scope `LOCK-002`.

## Phương án khác

### Conditional compare-and-set

Khi không dùng managed entity:

```sql
update product_offer
set price = :price,
    version = version + 1
where offer_id = :id
  and version = :expectedVersion
returning version;
```

Cùng correctness contract, mapping thủ công hơn nhưng SQL rõ.

### Pessimistic `FOR UPDATE`

Lock trước read khiến editor sau block rồi thấy current state. Vẫn cần client
expected version để biết form cũ; giữ lock dài hơn và cần timeout/deadlock policy.

### Atomic domain SQL

Với “increase price by 5%” hoặc counter, atomic relative update có thể compose
intent tốt hơn absolute set. Không dùng user-edit example để suy rộng mọi mutation.

### `SERIALIZABLE`

Có thể abort transaction nhưng nặng hơn version predicate cho known aggregate;
application vẫn cần `40001` retry. Không thay client version contract.

## Failure behavior

| Outcome | Database | API/application |
| --- | --- | --- |
| Version match | affected `1`, version tăng | Success + new version |
| Client expected stale | Không UPDATE | `412/409`, reload |
| Race sau load | affected `0`, rollback | Optimistic conflict |
| Lock/statement timeout | Transaction rollback | Technical timeout, không gọi stale |
| Delete cạnh tranh | affected `0` | Domain policy not-found/conflict |
| Crash trước commit | Rollback | Retry/query command status |
| Response mất sau commit | Update durable | Command/audit id để resolve |

## Trade-off

| Cách | Correctness | Contention | Latency | Retry/work waste | Multi-instance |
| --- | --- | --- | --- | --- | --- |
| `@Version` | Detect stale aggregate write | Không lock lúc read | Conflict muộn | Có khi conflict | Có |
| Conditional CAS | Tương đương cho explicit SQL | Không lock lúc read | Conflict ở statement | Thấp/explicit | Có |
| `FOR UPDATE` | Serialize trước decision | Blocking | Wait/timeout | Ít wasted work | Có |
| JVM lock | Chỉ local | Local serialization | Không cover DB | Không đáng tin | Không |

## Checklist trước production

- [ ] Version column `NOT NULL`, mapping `@Version`, không application setter.
- [ ] API/command mang expected version hoặc ETag.
- [ ] Generated SQL có version predicate và increment.
- [ ] Bulk/native writers duy trì cùng protocol.
- [ ] Conflict catch nằm ngoài failed transaction.
- [ ] User edit không auto-retry mù.
- [ ] Response chỉ success sau commit.
- [ ] Conflict metrics không chứa high-cardinality entity ID.
- [ ] PostgreSQL Testcontainers assert affected-row conflict và final version.
