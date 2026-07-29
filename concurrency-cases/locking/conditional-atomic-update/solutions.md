# Giải pháp — guarded mutation và explicit outcome

## Mục tiêu thiết kế

Solution phải giữ đồng thời bốn lớp correctness:

1. stock guard và counter mutation là một SQL statement;
2. affected/returned row quyết định `RESERVED` hay `OUT_OF_STOCK`;
3. command claim ngăn replay decrement lần hai;
4. claim, counters, durable outcome và outbox commit/rollback cùng nhau.

## Schema đề xuất

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
    command_id uuid primary key,
    reservation_id uuid unique,
    order_id uuid not null,
    product_id bigint not null,
    quantity integer not null check (quantity > 0),
    outcome varchar(24) not null,
    request_fingerprint varchar(64) not null,
    remaining_available integer,
    remaining_reserved integer,
    created_at timestamptz not null,
    completed_at timestamptz,
    check (outcome in ('PROCESSING', 'RESERVED', 'OUT_OF_STOCK')),
    check (
        outcome <> 'RESERVED'
        or (
            reservation_id is not null
            and remaining_available is not null
            and remaining_reserved is not null
        )
    )
);

create table outbox_event (
    event_id uuid primary key,
    aggregate_type varchar(64) not null,
    aggregate_id varchar(128) not null,
    event_type varchar(128) not null,
    payload jsonb not null,
    created_at timestamptz not null,
    published_at timestamptz
);
```

`PROCESSING` chỉ là state bên trong transaction đang sở hữu command. Service
không có path commit trước khi chuyển sang final outcome.

## Core conditional SQL

```sql
update inventory_item
set available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity,
    revision = revision + 1
where product_id = :productId
  and available_quantity >= :quantity
returning product_id,
          available_quantity,
          reserved_quantity,
          revision;
```

Quantity được validate positive trong Java và schema. Product lifecycle bảo đảm
row tồn tại cho command; vì vậy zero returned rows map `OUT_OF_STOCK`.

## Value objects

```java
public record ReserveStockCommand(
        UUID commandId,
        UUID orderId,
        long productId,
        int quantity,
        String requestFingerprint
) {
    public ReserveStockCommand {
        Objects.requireNonNull(commandId);
        Objects.requireNonNull(orderId);
        Objects.requireNonNull(requestFingerprint);
        if (quantity <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }
    }
}

public record InventoryAfterReserve(
        long productId,
        int available,
        int reserved,
        long revision
) {}
```

Fingerprint được tính từ canonical command payload ở trusted boundary, không lấy
một hash tùy ý do client gửi.

## Command claim DAO

```java
@Repository
public class ReservationCommandDao {
    private final NamedParameterJdbcTemplate jdbc;

    public ReservationCommandDao(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public boolean tryClaim(ReserveStockCommand command, Instant now) {
        int changed = jdbc.update("""
                insert into inventory_reservation (
                    command_id,
                    order_id,
                    product_id,
                    quantity,
                    outcome,
                    request_fingerprint,
                    created_at
                ) values (
                    :commandId,
                    :orderId,
                    :productId,
                    :quantity,
                    'PROCESSING',
                    :fingerprint,
                    :createdAt
                )
                on conflict (command_id) do nothing
                """, new MapSqlParameterSource()
                .addValue("commandId", command.commandId())
                .addValue("orderId", command.orderId())
                .addValue("productId", command.productId())
                .addValue("quantity", command.quantity())
                .addValue("fingerprint", command.requestFingerprint())
                .addValue("createdAt", Timestamp.from(now)));
        return changed == 1;
    }

    public StoredReservation requireExisting(UUID commandId) {
        return jdbc.queryForObject("""
                select command_id,
                       reservation_id,
                       order_id,
                       product_id,
                       quantity,
                       outcome,
                       request_fingerprint,
                       remaining_available,
                       remaining_reserved
                from inventory_reservation
                where command_id = :commandId
                """, Map.of("commandId", commandId), STORED_ROW_MAPPER);
    }

    public void completeReserved(
            UUID commandId,
            UUID reservationId,
            InventoryAfterReserve stock,
            Instant completedAt
    ) {
        int changed = jdbc.update("""
                update inventory_reservation
                set reservation_id = :reservationId,
                    outcome = 'RESERVED',
                    remaining_available = :available,
                    remaining_reserved = :reserved,
                    completed_at = :completedAt
                where command_id = :commandId
                  and outcome = 'PROCESSING'
                """, new MapSqlParameterSource()
                .addValue("commandId", commandId)
                .addValue("reservationId", reservationId)
                .addValue("available", stock.available())
                .addValue("reserved", stock.reserved())
                .addValue("completedAt", Timestamp.from(completedAt)));
        requireExactlyOne(changed);
    }

    public void completeOutOfStock(UUID commandId, Instant completedAt) {
        int changed = jdbc.update("""
                update inventory_reservation
                set outcome = 'OUT_OF_STOCK',
                    completed_at = :completedAt
                where command_id = :commandId
                  and outcome = 'PROCESSING'
                """, Map.of(
                "commandId", commandId,
                "completedAt", Timestamp.from(completedAt)
        ));
        requireExactlyOne(changed);
    }

    private static void requireExactlyOne(int changed) {
        if (changed != 1) {
            throw new IllegalStateException(
                    "expected one command outcome row, got " + changed
            );
        }
    }
}
```

`ON CONFLICT DO NOTHING` dùng unique primary key làm atomic arbiter. Concurrent
duplicate có thể wait transaction owner; sau statement, một SELECT mới ở
`READ COMMITTED` đọc durable outcome đã commit. Nếu owner rollback, contender có
thể trở thành inserter.

## Atomic inventory DAO với `RETURNING`

```java
@Repository
public class InventoryMutationDao {
    private final NamedParameterJdbcTemplate jdbc;

    public InventoryMutationDao(NamedParameterJdbcTemplate jdbc) {
        this.jdbc = jdbc;
    }

    public Optional<InventoryAfterReserve> tryReserve(
            long productId,
            int quantity
    ) {
        List<InventoryAfterReserve> rows = jdbc.query("""
                update inventory_item
                set available_quantity = available_quantity - :quantity,
                    reserved_quantity = reserved_quantity + :quantity,
                    revision = revision + 1
                where product_id = :productId
                  and available_quantity >= :quantity
                returning product_id,
                          available_quantity,
                          reserved_quantity,
                          revision
                """, Map.of(
                "productId", productId,
                "quantity", quantity
        ), (rs, rowNum) -> new InventoryAfterReserve(
                rs.getLong("product_id"),
                rs.getInt("available_quantity"),
                rs.getInt("reserved_quantity"),
                rs.getLong("revision")
        ));

        if (rows.size() > 1) {
            throw new IllegalStateException(
                    "single-product mutation returned multiple rows"
            );
        }
        return rows.stream().findFirst();
    }
}
```

DAO không load `InventoryItem` thành managed entity, nên không có stale object
cùng product trong persistence context. `RETURNING` cung cấp exact post-update
values.

## Bounded lock wait

```java
@ConfigurationProperties("app.inventory.database")
public record InventoryDatabaseTimeouts(
        Duration lockTimeout,
        Duration statementTimeout
) {
    public InventoryDatabaseTimeouts {
        if (!lockTimeout.isPositive()
                || statementTimeout.compareTo(lockTimeout) <= 0) {
            throw new IllegalArgumentException(
                    "statement timeout must exceed positive lock timeout"
            );
        }
    }

    String lockTimeoutValue() {
        return lockTimeout.toMillis() + "ms";
    }

    String statementTimeoutValue() {
        return statementTimeout.toMillis() + "ms";
    }
}

@Component
public class PostgreSqlStatementPolicy {
    private final NamedParameterJdbcTemplate jdbc;
    private final InventoryDatabaseTimeouts timeouts;

    public PostgreSqlStatementPolicy(
            NamedParameterJdbcTemplate jdbc,
            InventoryDatabaseTimeouts timeouts
    ) {
        this.jdbc = jdbc;
        this.timeouts = timeouts;
    }

    public void apply() {
        jdbc.queryForObject("""
                select set_config('lock_timeout', :value, true)
                """, Map.of(
                "value", timeouts.lockTimeoutValue()
        ), String.class);
        jdbc.queryForObject("""
                select set_config('statement_timeout', :value, true)
                """, Map.of(
                "value", timeouts.statementTimeoutValue()
        ), String.class);
    }
}
```

Settings là transaction-local. Giá trị thật phải nằm trong overall request
deadline và được cấu hình theo workload; ví dụ production `750ms/1500ms`.
Register record bằng `@ConfigurationPropertiesScan` hoặc
`@EnableConfigurationProperties`; không nhận raw timeout từ client.

## Transactional worker

```java
@Service
public class InventoryReservationTx {
    private final ReservationCommandDao commands;
    private final InventoryMutationDao inventory;
    private final OutboxRepository outbox;
    private final PostgreSqlStatementPolicy statementPolicy;
    private final EntityManager entityManager;
    private final Clock clock;

    public InventoryReservationTx(
            ReservationCommandDao commands,
            InventoryMutationDao inventory,
            OutboxRepository outbox,
            PostgreSqlStatementPolicy statementPolicy,
            EntityManager entityManager,
            Clock clock
    ) {
        this.commands = commands;
        this.inventory = inventory;
        this.outbox = outbox;
        this.statementPolicy = statementPolicy;
        this.entityManager = entityManager;
        this.clock = clock;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public ReservationResult reserve(ReserveStockCommand command) {
        Instant now = clock.instant();
        statementPolicy.apply();

        if (!commands.tryClaim(command, now)) {
            StoredReservation existing =
                    commands.requireExisting(command.commandId());
            existing.requireSameRequest(command);
            return existing.toResult();
        }

        Optional<InventoryAfterReserve> changed =
                inventory.tryReserve(
                        command.productId(),
                        command.quantity()
                );

        if (changed.isEmpty()) {
            commands.completeOutOfStock(command.commandId(), now);
            return ReservationResult.outOfStock(command.commandId());
        }

        InventoryAfterReserve stock = changed.orElseThrow();
        UUID reservationId = UUID.randomUUID();
        commands.completeReserved(
                command.commandId(),
                reservationId,
                stock,
                now
        );
        outbox.save(OutboxEvent.inventoryReserved(
                UUID.randomUUID(),
                reservationId,
                command,
                stock,
                now
        ));

        entityManager.flush();
        return ReservationResult.reserved(
                command.commandId(),
                reservationId,
                stock.available(),
                stock.reserved()
        );
    }
}
```

JDBC DAOs và JPA outbox phải dùng cùng `DataSource` và Spring transaction manager.
Với `JpaTransactionManager`, JDBC access cần participate cùng thread-bound
connection; integration test phải chứng minh rollback composition trên cấu hình
thật.

`flush()` làm insert/constraint error xuất hiện trước method body return. Success
chỉ tới caller sau transaction interceptor commit.

> **Nói ngắn gọn:** affected row quyết định business result; transaction quyết
> định counter, durable outcome và outbox có cùng tồn tại hay cùng biến mất.

## Map technical failure ngoài transaction

```java
@Component
public class InventoryReservationCoordinator {
    private final InventoryReservationTx worker;
    private final SqlStateClassifier classifier;

    public InventoryReservationCoordinator(
            InventoryReservationTx worker,
            SqlStateClassifier classifier
    ) {
        this.worker = worker;
        this.classifier = classifier;
    }

    public ReservationResult reserve(ReserveStockCommand command) {
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            throw new IllegalStateException(
                    "coordinator must run outside a transaction"
            );
        }

        try {
            return worker.reserve(command);
        } catch (RuntimeException failure) {
            return classifier.lockOrSerializationFailure(failure)
                    .map(kind -> ReservationResult.busy(
                            command.commandId(),
                            kind.name()
                    ))
                    .orElseThrow(() -> failure);
        }
    }
}

@Component
public class SqlStateClassifier {
    public Optional<DatabaseConflict> lockOrSerializationFailure(
            Throwable failure
    ) {
        for (Throwable current = failure;
             current != null;
             current = current.getCause()) {
            if (current instanceof SQLException sql) {
                return switch (sql.getSQLState()) {
                    case "55P03" -> Optional.of(DatabaseConflict.LOCK_TIMEOUT);
                    case "40P01" -> Optional.of(DatabaseConflict.DEADLOCK);
                    case "40001" ->
                            Optional.of(DatabaseConflict.SERIALIZATION_FAILURE);
                    default -> Optional.empty();
                };
            }
        }
        return Optional.empty();
    }
}
```

Exception rời transactional proxy trước khi coordinator map, nên attempt đã
rollback. Không map technical failure thành `OUT_OF_STOCK`, và không swallow
unknown database/programming errors.

Nếu product policy cho retry, dùng transaction mới, stable command ID, reload,
attempt cap, overall deadline và backoff. Không retry unbounded immediate.

## Spring Data JPA affected-count variant

Khi response không cần remaining counters:

```java
public interface InventoryItemRepository
        extends JpaRepository<InventoryItem, Long> {

    @Modifying
    @Query(value = """
            update inventory_item
            set available_quantity = available_quantity - :quantity,
                reserved_quantity = reserved_quantity + :quantity,
                revision = revision + 1
            where product_id = :productId
              and available_quantity >= :quantity
            """, nativeQuery = true)
    int reserveIfEnough(
            @Param("productId") long productId,
            @Param("quantity") int quantity
    );
}
```

Service bắt buộc kiểm tra `changed == 1`. Method contract phải cấm load/save cùng
`InventoryItem` trong persistence context, hoặc chủ động flush/clear/refresh.

`@Modifying(flushAutomatically = true, clearAutomatically = true)` hữu ích khi
boundary sở hữu toàn persistence context, nhưng clear có thể detach unrelated
entities. Không dùng flags để che transaction design mơ hồ.

## Rollback composition

Nếu outbox insert fail:

```text
conditional UPDATE affected 1
→ command marked RESERVED
→ outbox constraint fails
→ transaction rollback
→ inventory counters return to original values
→ command claim/outcome disappears
```

Caller retry cùng command ID có thể claim lại. Test phải assert counter và
reservation table, không chỉ exception type.

## Cancel/release stock

Cancellation là state transition riêng: command phải chứng minh ownership, đổi
reservation từ `RESERVED` sang `CANCELLED` đúng một lần và hoàn lại counters
trong cùng transaction. Một reverse UPDATE chỉ tăng available mà không atomically
claim transition sẽ hoàn stock nhiều lần khi request bị replay.

Schema của case chưa mô hình hóa cancellation fields/outcomes; full protocol cần

## Alternatives

### `PESSIMISTIC_WRITE`

Load row `FOR UPDATE`, validate rồi mutate. Dễ dùng khi decision cần nhiều fields
hoặc child records, nhưng thêm read round trip và giữ lock qua Java decision.

### Optimistic `@Version`

Loser conflict khi flush, sau đó reload/reject/retry. Hợp với aggregate edit và
contention thấp; stock delta đơn giản thường gọn hơn bằng conditional SQL.

### Constraint-only

`CHECK available >= 0` chặn negative row nhưng không trả domain outcome sớm và
không bảo đảm audit counters cross-table reconcile. Dùng như defense in depth.

### `SERIALIZABLE`

Bảo vệ anomaly rộng hơn nhưng tạo `40001` retry contract. Không cần thiết khi
known-row invariant đã nằm trọn trong one-statement predicate.

### Queue/serialized processing

Có thể tạo backpressure/order cho hot product bền vững, nhưng thêm asynchronous
workflow và operational complexity. Chỉ chọn từ load/failure model, không vì SQL
trông “quá đơn giản”.

## So sánh định tính

| Strategy | Round trips | Loser | Retry | Lock lifetime | Multi-instance |
| --- | --- | --- | --- | --- | --- |
| Conditional `UPDATE` | Một mutation | affected `0` | Thường không | Đến Tx end, acquire tại UPDATE | Có |
| `UPDATE RETURNING` | Một mutation/result | zero rows | Thường không | Như trên | Có |
| `FOR UPDATE` + write | Read + write | block rồi reject/timeout | Không bắt buộc | Từ locking read | Có |
| `@Version` | Read + versioned write | conflict lúc flush | Có thể | Writer lock ngắn | Có |
| JVM lock | Tùy | local wait | Không | JVM only | Không |

## Khuyến nghị

Với known inventory row và guard `available >= quantity`, dùng conditional
`UPDATE RETURNING` làm primary mutation. Validate input trước SQL, định nghĩa
zero-row contract, commit durable outcome/outbox cùng counters, và giữ persistence
context tránh xa direct-mutated entity.

Khi hot-row contention bền vững, không chỉ tăng timeouts. Đo no-op/wait/latency và
đánh giá strategy trong `LOCK-005`.

## Production checklist

- [ ] `WHERE` chứa toàn bộ business guard cần cho mutation.
- [ ] `SET` dùng current column values, không stale absolute values.
- [ ] Query cardinality là zero hoặc one cho command.
- [ ] Affected/returned row được map explicit.
- [ ] Zero-row meanings được document và validate.
- [ ] Constraint bảo vệ nonnegative/conservation state.
- [ ] Không load/save managed inventory entity quanh bulk DML.
- [ ] Command ID unique và fingerprint replay được kiểm tra.
- [ ] Outcome/outbox cùng transaction với counter.
- [ ] Lock/statement timeout nằm trong overall deadline.
- [ ] `55P03`, `40P01`, `40001` không bị map thành out-of-stock.
- [ ] Test PostgreSQL thật chứng minh predicate recheck và rollback.
- [ ] Reconciliation so projection với reservation audit.
