# Code lỗi — coi `SERIALIZABLE` là luôn thành công

## Schema tối thiểu

```sql
create table merchant_limit (
    merchant_id bigint primary key,
    limit_amount numeric(19, 2) not null check (limit_amount >= 0)
);

create table credit_reservation (
    reservation_id uuid primary key,
    command_id uuid not null unique,
    merchant_id bigint not null references merchant_limit(merchant_id),
    amount numeric(19, 2) not null check (amount > 0),
    status varchar(16) not null check (status in ('ACTIVE', 'RELEASED'))
);

create index ix_credit_reservation_scope
    on credit_reservation(merchant_id, status);

create table limit_command_decision (
    command_id uuid primary key,
    merchant_id bigint not null,
    requested_amount numeric(19, 2) not null,
    outcome varchar(16) not null check (outcome in ('ACCEPTED', 'REJECTED'))
);

insert into merchant_limit(merchant_id, limit_amount)
values (7, 100.00);

insert into credit_reservation(
    reservation_id, command_id, merchant_id, amount, status
)
values (
    '00000000-0000-0000-0000-000000000060',
    '10000000-0000-0000-0000-000000000060',
    7, 60.00, 'ACTIVE'
);
```

Index làm predicate access rõ và thực tế hơn. SSI vẫn có thể dùng page/relation
`SIReadLock` tùy execution plan; code không được phụ thuộc lock granularity cụ thể.

## Repository đọc predicate

```java
package example.limit;

import java.math.BigDecimal;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

public interface CreditReservationRepository
        extends JpaRepository<CreditReservation, java.util.UUID> {

    @Query(value = """
            select coalesce(sum(amount), 0)
            from credit_reservation
            where merchant_id = :merchantId
              and status = 'ACTIVE'
            """, nativeQuery = true)
    BigDecimal activeTotal(@Param("merchantId") long merchantId);
}
```

Các repository còn lại:

```java
public interface MerchantLimitRepository
        extends JpaRepository<MerchantLimit, Long> {
}

public interface LimitCommandDecisionRepository
        extends JpaRepository<LimitCommandDecision, UUID> {
}
```

Entity chỉ cần ánh xạ đúng các cột trong DDL. `CreditReservation.accepted(...)`
và `LimitCommandDecision.accepted/rejected(...)` là factory tạo entity mới với
cùng `commandId`.

## Attempt có isolation đúng nhưng thiếu failure contract

```java
package example.limit;

import java.math.BigDecimal;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BrokenSerializableLimitService {

    private final MerchantLimitRepository limits;
    private final CreditReservationRepository reservations;
    private final LimitCommandDecisionRepository decisions;

    public BrokenSerializableLimitService(
            MerchantLimitRepository limits,
            CreditReservationRepository reservations,
            LimitCommandDecisionRepository decisions
    ) {
        this.limits = limits;
        this.reservations = reservations;
        this.decisions = decisions;
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public LimitDecision reserve(ReserveLimitCommand command) {
        var replay = decisions.findById(command.commandId());
        if (replay.isPresent()) {
            return LimitDecision.from(replay.orElseThrow());
        }

        BigDecimal limit = limits.findById(command.merchantId())
                .orElseThrow()
                .limitAmount();
        BigDecimal active = reservations.activeTotal(command.merchantId());

        if (active.add(command.amount()).compareTo(limit) > 0) {
            decisions.save(LimitCommandDecision.rejected(command));
            return LimitDecision.rejected(command.commandId());
        }

        reservations.save(CreditReservation.accepted(command));
        decisions.save(LimitCommandDecision.accepted(command));
        reservations.flush();
        decisions.flush();

        return LimitDecision.accepted(command.commandId());
    }
}
```

Method làm đúng decision nếu chạy một mình. Lỗi là API/caller giả định method
luôn trả `LimitDecision`. Với hai attempts đồng thời, exception `40001` có thể
xuất hiện ở query, flush hoặc transaction commit sau khi method body đã tạo
return value.

Nếu exception xảy ra lúc commit, `try/catch` đặt quanh `flush()` bên trong method
cũng không thấy nó; transaction interceptor mới là nơi đang thực hiện commit.

> **Nói ngắn gọn:** isolation annotation có thể đúng nhưng use-case vẫn thiếu
> correctness nếu không định nghĩa loser rollback, retry và exhaustion.

## Controller làm rò lỗi kỹ thuật

```java
@RestController
public class LimitController {

    private final BrokenSerializableLimitService service;

    public LimitController(BrokenSerializableLimitService service) {
        this.service = service;
    }

    @PostMapping("/merchants/{merchantId}/reservations")
    ResponseEntity<LimitDecision> reserve(
            @PathVariable long merchantId,
            @RequestBody ReserveRequest request
    ) {
        return ResponseEntity.ok(service.reserve(request.toCommand(merchantId)));
    }
}
```

Một command nhận HTTP 500 dù PostgreSQL đã rollback sạch. Client có thể retry với
command ID mới, tạo duplicate operation hoặc làm conflict rate tăng.

## Retry sai trong cùng transaction

```java
package example.limit;

import jakarta.persistence.EntityManager;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BrokenLoopRetryService {

    private final SerializableSteps steps;
    private final EntityManager entityManager;

    public BrokenLoopRetryService(
            SerializableSteps steps,
            EntityManager entityManager
    ) {
        this.steps = steps;
        this.entityManager = entityManager;
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public LimitDecision reserve(ReserveLimitCommand command) {
        for (int attempt = 1; attempt <= 3; attempt++) {
            try {
                return steps.decideInsideCurrentTransaction(command);
            } catch (RuntimeException failure) {
                if (!PostgreSqlFailures.isSerializationFailure(failure)) {
                    throw failure;
                }
                entityManager.clear();
                // Sai: database transaction và snapshot cũ vẫn đang failed.
            }
        }
        throw new LimitContentionException(command.commandId());
    }
}
```

Sau `40001`, PostgreSQL transaction ở failed state. Query tiếp theo thường nhận
`25P02` cho tới `ROLLBACK`. `EntityManager.clear()` chỉ detach Java entities; nó
không rollback connection, không tạo snapshot mới và không hoàn tác external
side effect.

Ngay cả khi dùng savepoint, PostgreSQL yêu cầu retry complete transaction cho
serialization failure; retry một fragment không chạy lại logic đã quyết định SQL
và values.

## `@Retryable` và `@Transactional` trên cùng method

```java
@Retryable(
        retryFor = RuntimeException.class,
        maxAttempts = 4
)
@Transactional(isolation = Isolation.SERIALIZABLE)
public LimitDecision reserve(ReserveLimitCommand command) {
    return decide(command);
}
```

Có ba vấn đề:

1. `RuntimeException.class` retry cả validation/programming failures.
2. Retry và transaction advisors trên cùng proxy khiến boundary phụ thuộc advisor
   ordering/configuration.
3. Không có overall deadline, classification theo SQLSTATE hay durable command
   replay.

Nếu transaction advice bao ngoài retry advice, attempts sau chạy trong cùng
transaction đã doomed. Tách coordinator và worker beans làm boundary dễ nhìn và
dễ test hơn.

## Self-invocation làm mất isolation

```java
@Service
public class BrokenFacade {

    public LimitDecision reserve(ReserveLimitCommand command) {
        return serializableAttempt(command); // this.serializableAttempt(...)
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public LimitDecision serializableAttempt(ReserveLimitCommand command) {
        // read → decide → write
    }
}
```

Call nội bộ không đi qua Spring proxy. Nếu không có transaction bên ngoài,
repository calls có thể chạy trong các transaction khác nhau/default
`READ COMMITTED`; nếu đã có outer transaction, annotation inner không nâng
effective isolation.

## External side effect trước commit

```java
if (allowed) {
    reservations.save(CreditReservation.accepted(command));
    notificationClient.sendAccepted(command.commandId()); // không rollback được
    return LimitDecision.accepted(command.commandId());
}
```

SSI có thể abort tại commit sau khi notification đã gửi. Retry gửi lần nữa hoặc
fresh decision có thể thành `REJECTED`, tạo mâu thuẫn giữa database và external
system.

## Điều kiện để tái hiện

1. Seed limit `100` và committed active total `60`.
2. T1/T2 chạy physical transactions riêng với effective isolation
   `serializable`.
3. Cả hai dùng stable snapshot và hoàn tất `SUM=60` trước khi actor nào insert.
4. C1/C2 có command/reservation IDs khác nhau.
5. Barrier mở cho cả hai insert rồi commit.
6. Test bắt exception quanh toàn transaction, kể cả `commit()`.
7. PostgreSQL thật qua Testcontainers; H2 không chứng minh SSI/`SIReadLock`.

## Các cách sửa chưa đủ

- Chỉ tăng isolation nhưng không xử lý `40001`.
- Retry statement cuối thay vì whole transaction.
- Reuse entity/snapshot từ failed attempt.
- Retry vô hạn hoặc retry ngay lập tức không jitter.
- Retry mọi `DataAccessException`.
- Tạo command ID mới ở mỗi attempt.
- Gửi HTTP/message trước commit mà không có outbox.
- Thêm JVM `synchronized`; hai application instances vẫn dùng lock khác nhau.
- Giả định victim luôn là request bắt đầu sau.
- Assert chỉ exception type mà không kiểm tra final total và decisions.
