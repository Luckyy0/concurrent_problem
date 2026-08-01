# Phân Tích Mã Nguồn Lỗi: Ảo Tưởng Về SERIALIZABLE (Broken assumptions)

## 1. Khởi tạo cấu trúc bảng (Schema)

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
```

Chỉ mục (Index) trên `merchant_id` và `status` hỗ trợ tối ưu hóa truy vấn. Hệ thống SSI sẽ áp dụng khóa điều kiện (predicate lock - `SIReadLock`) theo phân tích kế hoạch truy vấn (execution plan), không cố định ở mức dòng (tuple) hay trang (page).

## 2. Truy vấn điều kiện bằng Repository (Predicate Repository)

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

Các chức năng Factory trong Entities (`CreditReservation.accepted(...)`, `LimitCommandDecision.accepted/rejected(...)`) có nhiệm vụ sinh đối tượng, đồng thời đảm bảo bảo toàn mã `commandId`.

## 3. Quản lý Giao dịch thiếu Kế hoạch Lỗi (Incomplete isolation approach)

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

Đoạn mã này cấu hình `@Transactional(isolation = Isolation.SERIALIZABLE)`. Điều này đúng đắn về mặt cơ sở dữ liệu, nhưng bỏ qua kịch bản khi xung đột xảy ra. Trong điều kiện tương tranh cao, lệnh gọi có thể gây ra ngoại lệ `40001`. Lỗi này không phát sinh một cách đồng bộ trong quá trình thực thi hàm (method body), mà có thể phát sinh khi Spring Proxy gọi lệnh `commit()`. Do vậy, lớp ứng dụng (caller) nhận được ngoại lệ, và giao dịch bị hủy bỏ (rollback), dẫn đến sự gián đoạn của hệ thống nghiệp vụ. Việc thiếu cơ chế bắt lỗi và thử lại (retry contract) khiến tính năng này không hoàn chỉnh.

## 4. Xử lý Lỗi tại tầng API (Broken Controller)

```java
@RestController
public class LimitController {

    private final BrokenSerializableLimitService service;

    // ... constructor ...

    @PostMapping("/merchants/{merchantId}/reservations")
    ResponseEntity<LimitDecision> reserve(
            @PathVariable long merchantId,
            @RequestBody ReserveRequest request
    ) {
        return ResponseEntity.ok(service.reserve(request.toCommand(merchantId)));
    }
}
```

Khi lỗi `40001` không được xử lý ở tầng Service, nó sẽ lan truyền lên Controller, tạo ra lỗi HTTP 500. Phản ứng thông thường của Client là tạo một Request mới với mã lệnh mới (new Command ID), dẫn đến nguy cơ nhân đôi dữ liệu.

## 5. Lỗi thử lại trong cùng Transaction (Broken loop retry)

```java
@Service
public class BrokenLoopRetryService {

    private final SerializableSteps steps;
    private final EntityManager entityManager;

    // ... constructor ...

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
                // LỖI NGHIÊM TRỌNG: Giao dịch vật lý bên ngoài đang ở trạng thái lỗi (Failed State).
                // Mọi truy vấn tiếp theo trong vòng lặp này sẽ sinh lỗi 25P02.
            }
        }
        throw new LimitContentionException(command.commandId());
    }
}
```

Giao dịch đã chịu lỗi `40001` không thể tiếp tục thực thi các truy vấn SQL. Gọi `entityManager.clear()` chỉ xóa dữ liệu trong bộ nhớ Java, không giải quyết trạng thái hỏng của Giao dịch cơ sở dữ liệu. Việc thử lại bắt buộc phải được thực hiện trong một Transaction mới hoàn toàn.

## 6. Lỗi sử dụng cấu hình `@Retryable` không chính xác

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

Việc kết hợp hai annotation này tiềm ẩn rủi ro về mặt thứ tự thực thi (advisor order) của Spring AOP. Nếu lớp `@Transactional` bọc bên ngoài lớp `@Retryable`, các vòng thử lại (retry attempts) sẽ chạy bên trong cùng một giao dịch thất bại. Ngoài ra, việc dùng `RuntimeException.class` để kích hoạt thử lại là quá rộng (bao gồm cả lỗi logic lập trình). Cần tách biệt rõ ràng lớp điều phối (Coordinator) và lớp thực thi giao dịch.

## 7. Lỗi thiết kế ranh giới Proxy (Self-invocation loss of isolation)

```java
@Service
public class BrokenFacade {

    public LimitDecision reserve(ReserveLimitCommand command) {
        return serializableAttempt(command); // Gọi nội bộ (Self-invocation)
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public LimitDecision serializableAttempt(ReserveLimitCommand command) {
        // ... logic ...
    }
}
```

Trong Spring, gọi trực tiếp một phương thức `@Transactional` bên trong cùng một đối tượng (this.serializableAttempt) sẽ bỏ qua hoàn toàn cơ chế Proxy. Lệnh thiết lập `SERIALIZABLE` sẽ không được truyền tải xuống cơ sở dữ liệu, khiến giao dịch chạy với mức cô lập mặc định (thường là `READ COMMITTED`), phá vỡ tính đúng đắn của logic.

## 8. Lỗi rò rỉ hiệu ứng phụ (External side effects before commit)

```java
if (allowed) {
    reservations.save(CreditReservation.accepted(command));
    notificationClient.sendAccepted(command.commandId()); // LỖI KIẾN TRÚC
    return LimitDecision.accepted(command.commandId());
}
```

Gửi thông báo (Notification) thông qua hệ thống ngoại vi trước thời điểm Commit là một vi phạm nguyên tắc giao dịch. Nếu cơ sở dữ liệu từ chối giao dịch với mã lỗi `40001`, hệ thống ngoại vi đã nhận được một thông tin sai lệch không thể vãn hồi. Phải sử dụng mô hình Transactional Outbox.

## 9. Điều kiện để tái hiện lỗi (Preconditions for reproduction)

1. Môi trường có dữ liệu mốc: limit `100`, active `60`.
2. Hai kết nối thực thi `SERIALIZABLE` độc lập.
3. Cả hai luồng hoàn tất việc đọc giá trị tổng `SUM=60` trước khi luồng nào đó tiến hành lưu dữ liệu.
4. Mỗi luồng cấp phát một Command ID riêng.
5. Cố gắng thiết lập lệnh `commit()` song song.
6. Thử nghiệm trên cơ sở dữ liệu PostgreSQL thực tế bằng Testcontainers (tránh dùng H2, do H2 không hỗ trợ mô phỏng chính xác SSI).

## 10. Các biện pháp khắc phục chưa đủ chuẩn (Incomplete mitigations)

- Nâng mức cô lập lên `SERIALIZABLE` nhưng không có mã xử lý thử lại (retry block).
- Triển khai thử lại bên trong phạm vi giao dịch.
- Xử lý thử lại mà không tái truy vấn dữ liệu từ đầu (sử dụng lại Entity Cache cũ).
- Thử lại vô hạn mà không có thời gian trễ ngẫu nhiên (jitter/backoff).
- Thực thi Retry với mọi loại ngoại lệ (`DataAccessException`).
- Tạo Command ID mới cho mỗi vòng thử lại thay vì tái sử dụng ID để đảm bảo tính lũy đẳng (Idempotency).
- Duy trì gọi API bên thứ ba (HTTP call) bên trong ranh giới giao dịch đang chờ cấp khóa hoặc có khả năng rollback.
- Sử dụng `synchronized` trên môi trường ứng dụng đa máy chủ (Multi-instance).
- Phụ thuộc hoàn toàn vào ngoại lệ được ném ra để đánh giá thành công của luồng nghiệp vụ thay vì kiểm định tính chính xác của dữ liệu được lưu cuối cùng.
