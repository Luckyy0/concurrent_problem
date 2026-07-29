# Giải pháp canonical ordering và fresh-transaction retry

## Mục tiêu thiết kế

Giải pháp chính có hai lớp độc lập:

```text
prevention: mọi multi-account attempt khóa rows theo cùng canonical order
recovery:  nếu vẫn nhận 40P01, rollback rồi bounded retry trong transaction mới
```

Ordering loại bỏ cycle hai account đã biết. Retry xử lý conflict còn sót lại do
các code path/resource khác và cung cấp outcome rõ khi vượt giới hạn.

## Giải pháp 1 — Khóa account theo stable ID

### Command đầu vào

```java
package example.transfer;

import java.math.BigDecimal;
import java.util.UUID;

public record TransferCommand(
        UUID commandId,
        long fromAccountId,
        long toAccountId,
        BigDecimal amount
) {
    public TransferCommand {
        if (commandId == null) {
            throw new IllegalArgumentException("commandId is required");
        }
        if (fromAccountId == toAccountId) {
            throw new IllegalArgumentException("source equals destination");
        }
        if (amount == null || amount.signum() <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }
    }
}
```

`commandId` chuẩn bị cho idempotency/audit ở banking design. Case này không dùng
nó để tuyên bố exactly-once.

### Repository

```java
package example.transfer;

import jakarta.persistence.LockModeType;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface AccountRepository extends JpaRepository<Account, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select a from Account a where a.id = :id")
    Optional<Account> findByIdForUpdate(@Param("id") long id);
}
```

### Một attempt trong một transaction riêng

```java
package example.transfer;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderedTransferAttempt {

    private final AccountRepository accounts;

    public OrderedTransferAttempt(AccountRepository accounts) {
        this.accounts = accounts;
    }

    @Transactional(
            propagation = Propagation.REQUIRES_NEW,
            isolation = Isolation.READ_COMMITTED
    )
    public TransferReceipt execute(TransferCommand command) {
        long firstId = Math.min(
                command.fromAccountId(),
                command.toAccountId()
        );
        long secondId = Math.max(
                command.fromAccountId(),
                command.toAccountId()
        );

        Account first = lock(firstId);
        Account second = lock(secondId);

        Account source = command.fromAccountId() == first.id()
                ? first
                : second;
        Account destination = command.toAccountId() == first.id()
                ? first
                : second;

        source.debit(command.amount());
        destination.credit(command.amount());
        accounts.flush();

        return new TransferReceipt(
                command.commandId(),
                source.balance(),
                destination.balance()
        );
    }

    private Account lock(long id) {
        return accounts.findByIdForUpdate(id)
                .orElseThrow(() -> new AccountNotFoundException(id));
    }
}
```

`REQUIRES_NEW` phù hợp ở đây vì `OrderedTransferAttempt` là top-level attempt do
một non-transactional coordinator gọi. Không sao chép tùy tiện vào một outer
business transaction: independent commit có thể phá atomicity của outer unit và
tạm cần thêm connection khi suspend transaction.

Hai repository calls có chủ ý. Dựa vào `WHERE id IN (...)` mà không có một
execution/locking order được chứng minh sẽ làm protocol khó audit hơn. Mọi path
dùng chính helper/order này sẽ dễ review.

### Vì sao invariant được bảo vệ?

Cả A→B và B→A đều xin row `101` trước:

```text
T1 acquire 101 ── acquire 202 ── mutate ── commit
T2 wait 101 ───────────────────── acquire 101 ── acquire 202 ── commit
```

T2 chưa giữ `202` khi chờ `101`, nên không tạo chiều chờ ngược. Sau khi giữ đủ
locks, attempt mới validate/mutate; PostgreSQL commit cả hai updates hoặc rollback
toàn bộ.

> **Nói ngắn gọn:** source/destination quyết định nghiệp vụ; stable account ID
> quyết định lock order. Không dùng vai trò nghiệp vụ làm lock order.

## Giải pháp 2 — Retry bên ngoài transaction

Canonical order là biện pháp chính, nhưng production code vẫn phải xử lý
`40P01`. Spring Retry coordinator và transactional worker nằm trên hai beans để
advisor boundary rõ ràng.

### Phân loại SQLSTATE

```java
package example.transfer;

import java.sql.SQLException;

public final class PostgreSqlFailures {

    private PostgreSqlFailures() {
    }

    public static boolean isDeadlock(Throwable failure) {
        for (Throwable current = failure;
                current != null;
                current = current.getCause()) {
            if (current instanceof SQLException sqlException
                    && containsSqlState(sqlException, "40P01")) {
                return true;
            }
        }
        return false;
    }

    private static boolean containsSqlState(
            SQLException exception,
            String expected
    ) {
        for (SQLException current = exception;
                current != null;
                current = current.getNextException()) {
            if (expected.equals(current.getSQLState())) {
                return true;
            }
        }
        return false;
    }
}
```

### Exception dành riêng cho retry policy

```java
package example.transfer;

public final class DeadlockVictimException extends RuntimeException {

    public DeadlockVictimException(Throwable cause) {
        super("PostgreSQL aborted a deadlock victim", cause);
    }
}
```

Bổ sung translation trong attempt worker:

```java
@Transactional(
        propagation = Propagation.REQUIRES_NEW,
        isolation = Isolation.READ_COMMITTED
)
public TransferReceipt execute(TransferCommand command) {
    try {
        return executeLocked(command);
    } catch (RuntimeException failure) {
        if (PostgreSqlFailures.isDeadlock(failure)) {
            throw new DeadlockVictimException(failure);
        }
        throw failure;
    }
}
```

`executeLocked` chứa canonical locking code của Solution 1. Custom exception vẫn
đi ra khỏi transaction interceptor; rollback hoàn tất trước khi exception tới
retry coordinator.

### Retry coordinator không có transaction

```java
package example.transfer;

import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Recover;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;

@Service
public class TransferCoordinator {

    private final OrderedTransferAttempt attempts;

    public TransferCoordinator(OrderedTransferAttempt attempts) {
        this.attempts = attempts;
    }

    @Retryable(
            retryFor = DeadlockVictimException.class,
            maxAttempts = 3,
            backoff = @Backoff(
                    delay = 25,
                    maxDelay = 200,
                    multiplier = 2.0,
                    random = true
            )
    )
    public TransferReceipt transfer(TransferCommand command) {
        return attempts.execute(command);
    }

    @Recover
    TransferReceipt exhausted(
            DeadlockVictimException failure,
            TransferCommand command
    ) {
        throw new TransferTemporarilyUnavailableException(
                command.commandId(),
                failure
        );
    }
}
```

Application cần `@EnableRetry` hoặc equivalent Spring Retry configuration.
Production deadline phải bao trùm cả attempts, backoff và response cleanup;
`maxAttempts` một mình chưa bảo đảm request không vượt deadline.

Không retry:

- insufficient funds hoặc account not found;
- validation failure;
- arbitrary timeout mà policy chưa phân loại;
- ambiguous commit outcome nếu chưa có idempotency/status lookup;
- operation đã phát non-idempotent external side effect.

## Timeout trong phạm vi transaction

Đặt timeout để một lock wait không chiếm toàn bộ request budget:

```sql
select set_config('lock_timeout', '1500ms', true);
select set_config('statement_timeout', '3000ms', true);
```

Tham số thứ ba `true` giới hạn config trong current transaction. Giá trị chỉ là
ví dụ cấu hình cho test/sample; production phải derive từ SLO và overall
deadline, không copy con số mà không đo.

`lock_timeout` trả `55P03`, không phải `40P01`. Timeout là containment, không phải
proof loại bỏ circular wait.

## Triển khai canonical order an toàn

Protocol chỉ đúng khi mọi participant tuân thủ:

1. Inventory tất cả query/path có thể khóa `account`: API, batch, scheduler,
   reconciliation, admin tool và stored procedure.
2. Chọn một stable unique key và comparator duy nhất.
3. Tách business role khỏi lock position.
4. Với hơn hai IDs, sort distinct IDs rồi acquire lần lượt; reject duplicate/
   self-transfer theo domain rule.
5. Nếu operation khóa nhiều loại resource, định nghĩa global hierarchy, ví dụ
   `customer → account → transfer`.
6. Deploy theo cách không để old/new versions dùng hai protocols trái nhau.
7. Giữ `40P01` metrics/retry vì database có thể acquire thêm resources.

Không order theo hash, locale-sensitive string hoặc mutable display number.

## Hành vi khi lỗi

| Thời điểm | Database outcome | Application outcome |
| --- | --- | --- |
| Validation trước transaction | Không có lock | Trả lỗi input, không retry |
| Chờ row vượt `lock_timeout` | Statement lỗi; rollback transaction | Map temporary contention/deadline |
| Deadlock victim `40P01` | Attempt aborted, rollback/release locks | Fresh bounded retry nếu safe |
| Winner commit | Hai balances commit atomically | Trả receipt sau commit |
| Retry exhausted | Không attempt dở dang còn sống | Trả explicit unavailable/retry-later |
| Connection mất lúc commit | Outcome có thể ambiguous | Tra command status/idempotency record |

## External side effects

Không gọi notification/risk HTTP service trong transaction đang giữ hai locks.
Nếu cần phát event sau transfer:

- ghi outbox row trong cùng database transaction;
- publisher gửi sau commit;
- consumer deduplicate bằng event ID;
- retry command dùng idempotency record phù hợp.

Outbox giải quyết atomicity database-state/event-intent, không tự giải quyết mọi
delivery semantic. Chi tiết banking/messaging thuộc case riêng.

## Phương án 1 — Một aggregate/coarse lock

Có thể khóa một row đại diện cho shard/customer trước mọi account rows:

```sql
select id
from account_partition_guard
where id = :guard_id
for update;
```

Một guard order đơn giản hóa proof nhưng serialize nhiều transfer không thật sự
xung đột. Throughput giảm, hot guard tăng latency/pool pressure. Chỉ hợp lý khi
contention thấp và correctness/simplicity quan trọng hơn parallelism.

## Phương án 2 — Serialize bằng queue/owner

Partition commands theo account ownership có thể giảm database conflicts và tạo
backpressure, nhưng transfer chạm hai accounts cần protocol cross-partition.
Queue không tự tạo atomic debit/credit và thêm delivery, crash, ordering,
idempotency complexity. Không chọn chỉ để né một lock order có thể sửa trực tiếp.

## Phương án 3 — Optimistic locking

`@Version` tránh explicit `FOR UPDATE` ở read phase, nhưng transaction cập nhật
hai rows vẫn có thể:

- block ở flush;
- conflict version và retry;
- deadlock nếu SQL updates/other locks xuất theo opposite order.

Hibernate `hibernate.order_updates=true` có thể giảm một số deadlock nhưng không
thay global resource protocol, SQLSTATE handling hay business revalidation.
Optimistic strategy phù hợp khi contention thấp và retry rẻ; không phải drop-in
proof cho hot multi-row transfer.

## Phương án 4 — `SERIALIZABLE`

`SERIALIZABLE` bảo vệ một nhóm anomaly bằng SSI nhưng không hứa mọi transaction
commit, cũng không loại bỏ ordinary lock deadlock. Application phải xử lý cả
`40001` và `40P01` bằng fresh-transaction retry. Xem `DB-009`.

## Phương án 5 — Advisory lock

PostgreSQL advisory lock chỉ an toàn nếu mọi writer dùng đúng key, scope và
unlock semantics. Nó không tự khóa account rows, constraint hoặc code ngoài
protocol. Với invariant nằm trên hai rows, canonical row locking thường nhỏ hơn,
dễ quan sát và được database mutation tự tham gia hơn.

## So sánh trade-off

| Giải pháp | Correctness | Throughput/latency | Retry | Deadlock risk | Vận hành | Multi-instance |
| --- | --- | --- | --- | --- | --- | --- |
| Canonical row order | Mạnh cho resources trong protocol | Song song khi account sets không giao nhau; waiter trên hot row | Vẫn cần bounded fallback | Giảm mạnh cycle đã biết | Cần audit mọi code path | Có, PostgreSQL là boundary |
| Coarse guard row | Dễ chứng minh trong guard scope | Serialize rộng, latency cao khi hot | Ít cycle, vẫn cần failure handling | Thấp nếu guard order thống nhất | Hot-key monitoring | Có |
| Queue/owner | Phụ thuộc partition/protocol | Có backpressure, thêm queue latency | Redelivery/idempotency | Chuyển sang cross-owner risk | Cao | Có |
| Optimistic `@Version` | Detect stale writes | Tốt khi conflict hiếm | Conflict amplification khi hot | Không bảo đảm hết deadlock | Trung bình | Có |
| `SERIALIZABLE` | Mạnh cho serializable invariant | Abort/retry dưới contention | Bắt buộc xử lý `40001` | Vẫn có `40P01` | Cao hơn | Có |
| JVM-local mutex | Chỉ một process | Serialize cục bộ | Không sửa DB conflict | Vẫn xảy ra cross-node | Dễ nhưng sai scope | Không |

## Checklist trước production

### Thứ tự khóa

- [ ] Stable unique resource key và comparator được document.
- [ ] Mọi multi-account path dùng cùng order.
- [ ] Multi-resource-type hierarchy không có chiều ngược.
- [ ] Validation/mutation chỉ diễn ra sau khi giữ đủ required locks.

### Transaction và retry

- [ ] Retry coordinator không có outer transaction.
- [ ] Mỗi attempt qua proxy và mở transaction/persistence context mới.
- [ ] Chỉ `40P01`/outcome đã phê duyệt mới được retry.
- [ ] Attempt cap, jitter/backoff và overall deadline đều có.
- [ ] Exhaustion map thành domain/API outcome rõ.
- [ ] Không reuse entity từ failed attempt.

### Vận hành

- [ ] `lock_timeout` và `statement_timeout` phù hợp latency budget.
- [ ] `pg_stat_database.deadlocks`, SQLSTATE và retry metrics được theo dõi.
- [ ] PostgreSQL deadlock log có correlation nhưng không lộ dữ liệu nhạy cảm.
- [ ] Remote I/O nằm ngoài lock-holding transaction.
- [ ] Ambiguous commit/duplicate có idempotency/status strategy.
- [ ] Regression test dùng PostgreSQL Testcontainers, không dùng H2.
