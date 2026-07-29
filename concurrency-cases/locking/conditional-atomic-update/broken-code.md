# Code hỏng — đọc, kiểm tra rồi ghi absolute state

## Schema ban đầu

```sql
create table inventory_item (
    product_id bigint primary key,
    on_hand_quantity integer not null,
    available_quantity integer not null,
    reserved_quantity integer not null,
    revision bigint not null default 0,
    check (on_hand_quantity >= 0),
    check (available_quantity >= 0),
    check (reserved_quantity >= 0),
    check (available_quantity + reserved_quantity = on_hand_quantity)
);

create table inventory_reservation (
    reservation_id uuid primary key,
    command_id uuid not null unique,
    order_id uuid not null,
    product_id bigint not null,
    quantity integer not null check (quantity > 0),
    outcome varchar(24) not null,
    request_fingerprint varchar(64) not null,
    created_at timestamptz not null
);
```

Row `product_id=77` có `on_hand=5`, `available=5`, `reserved=0`.

Các row-level `CHECK` constraints vẫn có thể pass sau lost update: writer cuối để
lại `available=1`, `reserved=4`, trong khi hai reservation rows tổng `8`.

## Entity không có `@Version`

```java
@Entity
@Table(name = "inventory_item")
public class InventoryItem {
    @Id
    private Long productId;

    private int onHandQuantity;
    private int availableQuantity;
    private int reservedQuantity;
    private long revision;

    protected InventoryItem() {}

    boolean canReserve(int quantity) {
        return quantity > 0 && availableQuantity >= quantity;
    }

    void reserve(int quantity) {
        if (!canReserve(quantity)) {
            throw new IllegalStateException("insufficient stock");
        }
        availableQuantity -= quantity;
        reservedQuantity += quantity;
        revision++;
    }
}
```

`revision` chỉ là field bình thường. Không có `@Version`, nên Hibernate không đưa
expected revision vào UPDATE predicate.

## Broken service

```java
@Service
public class BrokenInventoryReservationService {
    private final InventoryItemRepository inventory;
    private final InventoryReservationRepository reservations;
    private final OutboxRepository outbox;
    private final Clock clock;

    public BrokenInventoryReservationService(
            InventoryItemRepository inventory,
            InventoryReservationRepository reservations,
            OutboxRepository outbox,
            Clock clock
    ) {
        this.inventory = inventory;
        this.reservations = reservations;
        this.outbox = outbox;
        this.clock = clock;
    }

    @Transactional
    public ReservationResult reserve(ReserveStockCommand command) {
        InventoryItem item = inventory.findById(command.productId())
                .orElseThrow(ProductNotFoundException::new);

        if (!item.canReserve(command.quantity())) {
            return ReservationResult.outOfStock();
        }

        item.reserve(command.quantity());
        InventoryReservation reservation =
                InventoryReservation.reserved(command, clock.instant());
        reservations.save(reservation);
        outbox.save(OutboxEvent.inventoryReserved(reservation));

        return ReservationResult.reserved(reservation.getReservationId());
    }
}
```

Method có transaction và mọi database writes commit cùng nhau. Vấn đề là
business guard chạy trong Java trên một snapshot cũ, tách khỏi UPDATE.

Hibernate có thể phát SQL tương đương:

```sql
update inventory_item
set available_quantity = 1,
    reserved_quantity = 4,
    revision = 1
where product_id = 77;
```

Cả hai stale entities tính cùng absolute values. Hai UPDATEs đều match primary
key và affected rows `1`.

## Preconditions để tái hiện

- PostgreSQL `READ COMMITTED`.
- Product `77` tồn tại với available `5`.
- Command A/B khác `command_id`, mỗi command quantity `4`.
- Hai plain SELECT hoàn tất trước khi một transaction flush.
- Không có `@Version`, `FOR UPDATE` hoặc guarded UPDATE.
- Reservation/outbox IDs không conflict.

## Race timeline

| Bước | Tx-A | Tx-B |
| --- | --- | --- |
| 1 | SELECT → available `5` | |
| 2 | | SELECT → available `5` |
| 3 | `5 >= 4`, ACCEPT | `5 >= 4`, ACCEPT |
| 4 | insert reservation A/outbox A | insert reservation B/outbox B |
| 5 | UPDATE counters → `1/4` | |
| 6 | commit | |
| 7 | | UPDATE stale counters → `1/4`, commit |

Kết quả:

```text
inventory_item: available=1, reserved=4
RESERVED rows: A quantity=4, B quantity=4, sum=8
```

Không counter âm và mọi CHECK constraint pass, nhưng inventory ledger/projection
đã mâu thuẫn.

> **Nói ngắn gọn:** constraint bảo vệ shape của một row; nó không biết hai
> reservation audits cùng dựa trên stock đã bị tiêu thụ.

## Sai lầm 1 — Tin rằng `@Transactional` làm check atomic

```java
@Transactional
public ReservationResult reserve(ReserveStockCommand command) {
    InventoryItem item = inventory.findById(command.productId()).orElseThrow();
    if (item.getAvailableQuantity() < command.quantity()) {
        return ReservationResult.outOfStock();
    }
    item.reserve(command.quantity());
    return ReservationResult.reserved();
}
```

Transaction tạo commit/rollback boundary. Plain SELECT và later UPDATE vẫn là
hai statements với race window.

## Sai lầm 2 — Conditional UPDATE nhưng bỏ qua affected rows

```java
@Transactional
public ReservationResult reserve(ReserveStockCommand command) {
    inventory.decrementIfEnough(command.productId(), command.quantity());
    reservations.save(InventoryReservation.reserved(command));
    return ReservationResult.reserved();
}
```

Actor thua nhận affected rows `0` nhưng code vẫn tạo durable success. Conditional
SQL chỉ bảo vệ invariant khi return value trở thành business decision.

## Sai lầm 3 — Predicate không chứa business guard

```sql
update inventory_item
set available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
where product_id = :productId;
```

Atomic arithmetic tránh lost update nhưng vẫn cho available âm. Guard
`available_quantity >= :quantity` phải nằm trong cùng statement.

## Sai lầm 4 — Pre-check rồi unconditional delta

```java
if (inventory.currentAvailable(productId) >= quantity) {
    inventory.decrementUnconditionally(productId, quantity);
}
```

Delta update đúng với phép cộng/trừ nhưng pre-check vẫn stale. Predicate phải đi
cùng mutation.

## Sai lầm 5 — Bulk DML rồi dùng managed entity cũ

```java
@Transactional
public void reserve(long productId, int quantity) {
    InventoryItem stale = inventory.findById(productId).orElseThrow();

    int changed = inventory.decrementIfEnough(productId, quantity);
    if (changed == 1) {
        stale.renameForAudit("reserved"); // entity vẫn giữ counters cũ
    }
}
```

JPQL/native bulk DML bypasses persistence context. A later dirty-check flush của
`stale` có thể overwrite counters hoặc trả response từ state cũ, tùy mapping/
dynamic update. Không trộn managed item với direct mutation nếu chưa
clear/refresh có chủ đích.

## Sai lầm 6 — `save()` sau atomic UPDATE

```java
int changed = inventory.decrementIfEnough(productId, quantity);
InventoryItem detachedSnapshot = mapper.fromEarlierRequest(request);
inventory.save(detachedSnapshot);
```

`merge`/flush detached snapshot có thể ghi lại absolute fields và phá kết quả
atomic. UPDATE winner không cần một entity save thứ hai.

## Sai lầm 7 — Map mọi zero row thành `OUT_OF_STOCK`

Affected rows `0` có thể là:

- product row không tồn tại;
- quantity không hợp lệ;
- available không đủ;
- tenant/status predicate không match;
- trigger suppress update.

Case này validate quantity và bảo đảm product row tồn tại trước command flow. Nếu
API cần phân biệt, hãy thiết kế outcome contract thay vì đoán.

## Sai lầm 8 — Map lock timeout thành no-op

```java
try {
    return inventory.decrementIfEnough(productId, quantity) == 1;
} catch (CannotAcquireLockException ignored) {
    return false; // false bị hiểu là OUT_OF_STOCK
}
```

Affected rows `0` là successful statement/no-op. SQLSTATE `55P03` là statement
failure và transaction phải rollback. Hai outcome có retry/telemetry khác nhau.

## Sai lầm 9 — Retry duplicate command như command mới

Nếu response mất sau commit, chạy lại cùng business request với command ID mới sẽ
decrement lần hai. Conditional stock guard không cung cấp idempotency; cần stable
command ID và atomic durable claim.

Ngược lại, unique command ID không serialize hai orders khác nhau trên cùng
product. Hai cơ chế bảo vệ hai invariant.

## Sai lầm 10 — Publish trước commit

```java
int changed = inventory.decrementIfEnough(productId, quantity);
if (changed == 1) {
    kafkaTemplate.send("inventory-reserved", event);
    reservations.save(reservation);
}
```

Database transaction có thể rollback sau send. Ghi outbox trong cùng transaction
rồi publish asynchronous sau commit.

## Sai lầm 11 — JVM lock trong multi-instance

```java
synchronized (localProductMutex(productId)) {
    return transactionalWorker.reserve(command);
}
```

App-1 và App-2 không chia sẻ monitor. Authoritative conditional UPDATE mới là
coordination point cross-node.

## Sai lầm 12 — Dùng replica read để quyết định

Replica tồn kho hữu ích cho display nhưng có thể lag. Nếu write trên primary vẫn
có guard thì correctness được giữ và stale pre-read chỉ làm UX/round trip tệ hơn.
Nếu application bỏ guard vì replica báo đủ, oversell quay lại.

## Dấu hiệu quan sát được

- Sum RESERVED reservation quantity khác `reserved_quantity`.
- Available/reserved counters trông hợp lệ riêng lẻ nhưng reconciliation fail.
- Affected rows `0` tăng mà API vẫn trả `RESERVED`.
- Hibernate persistence context trả stale counters sau native update.
- Duplicate command decrement nhiều lần sau client retry.
- `55P03` bị ghi thành business rejection.
- Bug chỉ xuất hiện khi requests đi qua nhiều application instances.

## Non-atomic sequence cần loại bỏ

```text
SELECT available
→ Java checks available >= quantity
→ Java calculates absolute counters
→ later UPDATE by primary key
```

Thay bằng một statement trong đó `WHERE` là guard và `SET` là business intent,
sau đó bắt buộc xử lý affected-row/returned-row outcome.
