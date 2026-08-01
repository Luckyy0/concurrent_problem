# Giải Pháp Điều Phối Thử Lại: Quản Lý Giao Dịch và SSI (Transaction Retry Solutions)

## 1. Mục tiêu thiết kế (Design Objectives)

Mô hình kiến trúc được chia làm hai lớp:
- **Tầng Điều phối (Coordinator)**: Quản lý vòng đời gọi, xử lý thử lại (retry), tạm ngưng (backoff) và kết thúc khi xảy ra sự cố. Lớp này không nằm trong giao dịch.
- **Tầng Thực thi (Attempt worker)**: Định nghĩa một đơn vị công việc hoàn chỉnh, chạy trong một giao dịch `SERIALIZABLE` độc lập.

Command ID phải được bảo toàn xuyên suốt các lần thử. Quyết định (Decision), bản ghi lưu trữ (Reservation) và thông điệp ngoại vi (Outbox intent) phải được tạo lập cùng nhau trong một lần commit thành công.

## 2. Giải pháp 1 — Phân tách Bộ Điều phối và Lớp Thực thi (Separation of Coordinator and Worker)

### Định nghĩa Đầu vào/Đầu ra (Command and outcome)

```java
package example.limit;

import java.math.BigDecimal;
import java.util.UUID;

public record ReserveLimitCommand(
        UUID commandId,
        long merchantId,
        BigDecimal amount
) {
    public ReserveLimitCommand {
        if (commandId == null) {
            throw new IllegalArgumentException("commandId is required");
        }
        if (amount == null || amount.signum() <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }
    }
}

public record LimitDecision(
        UUID commandId,
        Outcome outcome
) {
    public enum Outcome {
        ACCEPTED,
        REJECTED
    }
}
```

### Lớp Thực thi Giao dịch (Transactional Attempt)

```java
package example.limit;

import java.math.BigDecimal;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class SerializableLimitAttempt {

    private final MerchantLimitRepository limits;
    private final CreditReservationRepository reservations;
    private final LimitCommandDecisionRepository decisions;
    private final OutboxRepository outbox;

    public SerializableLimitAttempt(
            MerchantLimitRepository limits,
            CreditReservationRepository reservations,
            LimitCommandDecisionRepository decisions,
            OutboxRepository outbox
    ) {
        this.limits = limits;
        this.reservations = reservations;
        this.decisions = decisions;
        this.outbox = outbox;
    }

    @Transactional(
            propagation = Propagation.REQUIRES_NEW,
            isolation = Isolation.SERIALIZABLE
    )
    public LimitDecision execute(ReserveLimitCommand command) {
        var existing = decisions.findById(command.commandId());
        if (existing.isPresent()) {
            return existing.orElseThrow().toResult();
        }

        BigDecimal limit = limits.findById(command.merchantId())
                .orElseThrow(() ->
                        new MerchantLimitNotFoundException(command.merchantId())
                )
                .limitAmount();
        BigDecimal active = reservations.activeTotal(command.merchantId());

        LimitDecision result;
        if (active.add(command.amount()).compareTo(limit) > 0) {
            var decision = LimitCommandDecision.rejected(command);
            decisions.save(decision);
            result = decision.toResult();
        } else {
            reservations.save(CreditReservation.accepted(command));
            var decision = LimitCommandDecision.accepted(command);
            decisions.save(decision);
            outbox.save(OutboxEvent.limitReservationAccepted(command));
            result = decision.toResult();
        }

        reservations.flush();
        decisions.flush();
        outbox.flush();
        return result;
    }
}
```

`Propagation.REQUIRES_NEW` bảo đảm rằng phương thức này luôn mở một giao dịch mới mẻ và độc lập. Lệnh này không bao giờ kế thừa từ bất kỳ giao dịch bên ngoài nào, bảo vệ tính an toàn cho toàn bộ vòng đời điều phối thử lại.
Kết quả (`LimitDecision`) chỉ được phản hồi ra sau khi Proxy xử lý Commit hoàn tất.

> **Ghi chú quan trọng:** Tầng điều phối sẽ quyết định quy trình thử lại, không phải thành phần bên trong. Giao dịch cần duy trì ranh giới nhỏ gọn.

## 3. Phân loại Lỗi Cơ sở dữ liệu (PostgreSQL Failure Classification)

```java
package example.limit;

import java.sql.SQLException;
import java.util.Set;
import org.postgresql.util.PSQLException;

public final class PostgreSqlFailures {

    private static final Set<String> COMMAND_CONSTRAINTS = Set.of(
            "limit_command_decision_pkey",
            "credit_reservation_command_id_key"
    );

    private PostgreSqlFailures() {
    }

    public static boolean isSerializationFailure(Throwable failure) {
        return hasSqlState(failure, "40001");
    }

    public static boolean isDeadlock(Throwable failure) {
        return hasSqlState(failure, "40P01");
    }

    public static boolean isDuplicateCommand(Throwable failure) {
        for (Throwable current = failure;
                current != null;
                current = current.getCause()) {
            if (current instanceof PSQLException postgres
                    && "23505".equals(postgres.getSQLState())
                    && postgres.getServerErrorMessage() != null
                    && COMMAND_CONSTRAINTS.contains(
                            postgres.getServerErrorMessage().getConstraint()
                    )) {
                return true;
            }
        }
        return false;
    }

    private static boolean hasSqlState(
            Throwable failure,
            String expected
    ) {
        for (Throwable current = failure;
                current != null;
                current = current.getCause()) {
            if (current instanceof SQLException sql) {
                for (SQLException chained = sql;
                        chained != null;
                        chained = chained.getNextException()) {
                    if (expected.equals(chained.getSQLState())) {
                        return true;
                    }
                }
            }
        }
        return false;
    }
}
```

Kiểm tra tên ràng buộc (Constraint names) giúp ứng dụng nhận biết đây là tình huống một lệnh (Command ID) đã được xử lý (trùng lặp) hay là lỗi vi phạm dữ liệu (Constraint khác).

## 4. Cấu hình Giới hạn Thử lại và Trễ hẹn (Retry Policy & Backoff)

### Quản lý Thời hạn (Deadline and Policy)

```java
package example.limit;

import java.time.Clock;
import java.time.Instant;
import org.springframework.stereotype.Component;

@Component
public final class SerializableRetryPolicy {

    private static final int MAX_ATTEMPTS = 4;
    private final Clock clock;

    public SerializableRetryPolicy(Clock clock) {
        this.clock = clock;
    }

    public boolean canRetry(int completedAttempt, Instant deadline) {
        return completedAttempt < MAX_ATTEMPTS
                && clock.instant().isBefore(deadline);
    }
}
```

### Cơ chế trì hoãn (Backoff and Jitter)

```java
package example.limit;

import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.locks.LockSupport;
import org.springframework.stereotype.Component;

@Component
public final class RetryBackoff {

    private static final Duration INITIAL = Duration.ofMillis(20);
    private static final Duration MAXIMUM = Duration.ofMillis(200);
    private final Clock clock;

    public RetryBackoff(Clock clock) {
        this.clock = clock;
    }

    public void pause(int completedAttempt, Instant deadline) {
        long factor = 1L << Math.min(completedAttempt - 1, 10);
        long baseMillis = Math.min(
                MAXIMUM.toMillis(),
                INITIAL.toMillis() * factor
        );
        long jitterMillis = ThreadLocalRandom.current()
                .nextLong(Math.max(1, baseMillis / 2));
        long remainingMillis = Math.max(
                0,
                Duration.between(clock.instant(), deadline).toMillis()
        );
        long pauseMillis = Math.min(
                baseMillis + jitterMillis,
                remainingMillis
        );

        LockSupport.parkNanos(Duration.ofMillis(pauseMillis).toNanos());
        if (Thread.currentThread().isInterrupted()) {
            throw new RetryInterruptedException();
        }
    }
}
```

Việc tích hợp cơ chế hoãn (Backoff) ngẫu nhiên (Jitter) giúp giảm tải tập trung trên hệ thống cơ sở dữ liệu khi có sự cố tắc nghẽn (concurrency storm). Lưu ý: Logic trì hoãn bắt buộc phải chạy ở bên ngoài giao dịch, giải phóng kết nối cho Connection Pool.

## 5. Tầng Điều Phối Phi Giao Dịch (Non-transactional Coordinator)

```java
package example.limit;

import java.time.Duration;
import java.time.Instant;
import org.springframework.stereotype.Service;
import org.springframework.transaction.support.TransactionSynchronizationManager;

@Service
public class LimitReservationCoordinator {

    private final SerializableLimitAttempt attempts;
    private final LimitDecisionReplay replay;
    private final SerializableRetryPolicy retryPolicy;
    private final RetryBackoff backoff;

    // ... constructor ...

    public LimitDecision reserve(ReserveLimitCommand command) {
        if (TransactionSynchronizationManager.isActualTransactionActive()) {
            throw new IllegalStateException(
                    "Retry coordinator must not join an outer transaction"
            );
        }

        Instant deadline = Instant.now().plus(Duration.ofSeconds(2));
        for (int attempt = 1; ; attempt++) {
            try {
                return attempts.execute(command);
            } catch (RuntimeException failure) {
                if (PostgreSqlFailures.isDuplicateCommand(failure)) {
                    return replay.requireCommitted(command.commandId());
                }
                if (!PostgreSqlFailures.isSerializationFailure(failure)) {
                    throw failure;
                }
                if (!retryPolicy.canRetry(attempt, deadline)) {
                    throw new LimitContentionException(
                            command.commandId(),
                            attempt,
                            failure
                    );
                }
                backoff.pause(attempt, deadline);
            }
        }
    }
}
```

Phương thức điều phối bắt lỗi ở cấp độ proxy Spring (bên ngoài). Bất kỳ lỗi `40001` nào phát sinh từ các phương thức con đều đã hoàn tất quá trình Rollback trước khi chạm đến mảng kiểm tra này.
Chức năng `LimitDecisionReplay` chuyên để phát lại kết quả nếu lệnh bị lặp:

```java
@Service
public class LimitDecisionReplay {
    // ...
    @Transactional(propagation = Propagation.REQUIRES_NEW, readOnly = true)
    public LimitDecision requireCommitted(UUID commandId) {
        return decisions.findById(commandId)
                .orElseThrow(() -> new DuplicateCommandOutcomeUnavailable(commandId))
                .toResult();
    }
}
```

## 6. Giải pháp thay thế 1 — Dùng Spring Retry (`@Retryable`)

Có thể sử dụng thư viện Spring Retry, miễn là tuân thủ các quy tắc:
- Mở rộng ngoại lệ tùy chỉnh riêng để chỉ nhắm vào lỗi SQLSTATE `40001`.
- Cấu hình `@Retryable` bắt buộc phải đặt trên phương thức điều phối (Coordinator), KHÔNG BAO GIỜ đặt bên dưới hoặc đồng cấp với `@Transactional`.
- Cấu hình đẩy đủ Delay (Backoff), Random Jitter và Max Attempts.

## 7. Giải pháp thay thế 2 — Sử dụng `TransactionTemplate`

Áp dụng mẫu xử lý Giao dịch bằng code tĩnh:

```java
var template = new TransactionTemplate(transactionManager);
template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
template.setIsolationLevel(TransactionDefinition.ISOLATION_SERIALIZABLE);

for (int attempt = 1; ; attempt++) {
    try {
        return template.execute(status -> work.execute(command));
    } catch (RuntimeException failure) {
        if (!PostgreSqlFailures.isSerializationFailure(failure)
                || !policy.canRetry(attempt, deadline)) {
            throw failure;
        }
        backoff.pause(attempt, deadline);
    }
}
```

Hàm `template.execute` đảm bảo mọi tiến trình Commit/Rollback khép kín trước khi lỗi phát sinh về vòng lặp điều phối bên ngoài. Lệnh `work.execute` không được phép duy trì trạng thái dữ liệu rác (cache/entity context).

## 8. Giải pháp Cải tiến — Ghi cập nhật nguyên tử (Atomic conditional counter)

Nếu tần suất cập nhật cao (Hot row) gây ra tỷ lệ lỗi (Conflict rate) lớn, hãy chuyển hướng tiếp cận bằng lệnh đếm nguyên tử:

```sql
update merchant_limit
set reserved_amount = reserved_amount + :amount
where merchant_id = :merchantId
  and reserved_amount + :amount <= limit_amount;
```

Trong kịch bản này:
- Nếu thành công (affected row `1`), tức là lệnh được CHẤP NHẬN.
- Nếu không thành công (affected row `0`), tức là lệnh bị TỪ CHỐI.
Mô hình này không sinh lỗi `40001` (serialization failure) và không cần Retry. Phương án này đòi hỏi phải đảm bảo dữ liệu phụ đếm (Counter) tương thích với logic hệ thống tổng quát.

## 9. Hành vi Ứng phó Lỗi (Failure Matrix)

| Nguyên nhân Thất bại | Kết quả Giao dịch | Thao tác Thử lại (Retry) |
| --- | --- | --- |
| Lỗi SSI `40001` | Hủy bỏ thành công (Rollback hoàn toàn) | Tái tạo Transaction mới cùng giới hạn Backoff |
| Deadlock `40P01` | Trở thành Nạn nhân (Rollback) | Thử lại có thể thực hiện nếu an toàn, phải định vị đúng nguyên nhân khóa chéo. |
| Time-out Hàng chờ `55P03` | Lỗi hết thời hạn cấp khóa | Dựa trên Deadline tổng thể để quyết định (Tránh kẹt dồn ứ). |
| Trùng mã định danh `23505` | Hủy thao tác hiện tại | Phát lại quyết định trước đó với Command ID tương ứng (Replay). |
| Lỗi Nghiệp vụ `REJECTED` | Hoàn thành và Ghi (Commit kết quả) | KHÔNG thử lại. |
| Đầu vào sai định dạng | Lỗi dữ liệu trước khi chạy DB | KHÔNG thử lại. |
| Mất Mạng/Thời hạn (Crash) | Kết quả mập mờ, Server có thể đã nhận | Giao quyền cho Client tái phát lệnh (Command ID) để đảm bảo đồng bộ hóa. |

## 10. Đánh giá Ưu nhược điểm (Trade-off Comparison)

| Yếu tố | `SERIALIZABLE` + Retry | SQL Nguyên tử (Atomic SQL) | Dùng Khóa hàng (`FOR UPDATE`) |
| --- | --- | --- | --- |
| **Bảo vệ Dữ liệu** | Cao (Tự động theo dõi predicate) | Cao (Giới hạn logic tại điểm ghi) | Cao (Bảo vệ một bản ghi cụ thể) |
| **Xung đột / Độ trễ** | Không làm nghẽn truy vấn ghi | Rất thấp (Logic đơn giản) | Làm nghẽn hàng chờ, gia tăng độ trễ |
| **Yêu cầu Thử lại** | Bắt buộc xử lý lỗi và Retry | Thường không cần | Không cần |
| **Tải CSDL** | Tăng theo số lần thử lại | Thấp | Tạo gánh nặng do mở khóa chậm chạp |
| **Vận hành (Ops)** | Đòi hỏi quản lý `40001` và tối ưu | Thích hợp môi trường tốc độ cao | Dễ phát sinh Time-out/Deadlock cục bộ |

## 11. Danh Sách Kiểm Tra Khi Triển Khai (Production Checklist)

### Ranh giới Cô lập (Isolation Boundaries)
- [ ] Mức cô lập `SERIALIZABLE` được xác nhận tại thời điểm chạy thông qua kiểm tra biến hệ thống.
- [ ] Tầng Điều phối (Coordinator) nằm hoàn toàn BÊN NGOÀI giao dịch chung.
- [ ] Giao dịch con (Attempt worker) luôn chạy với chế độ `REQUIRES_NEW`.

### Khung Chính sách Thử lại (Retry Policy)
- [ ] Lệnh xử lý thử lại (Retry) BẮT BUỘC có cơ chế Jitter và Exponential Backoff để tránh dội bom máy chủ (Throttling).
- [ ] Cấu hình Giới hạn thử lại (Max attempts) và Thời hạn (Deadline/Cancellation).
- [ ] Khởi tạo hoàn toàn mới dữ liệu bộ nhớ trong mỗi Giao dịch thử lại.

### Tính Lũy Đẳng và Bảo Vệ Dữ Liệu (Idempotency and Side effects)
- [ ] Tái sử dụng Command ID để ngăn hiện tượng trùng lắp yêu cầu.
- [ ] Tái truy vấn bản ghi đã xác định để tránh tính trạng cấp duyệt dư thừa.
- [ ] Mọi hoạt động ngoại vi (như gọi API khác, gửi Email) BẮT BUỘC được thực hiện SAU khi giao dịch Commit thành công.

### Quản trị Vận hành (Observability)
- [ ] Ghi chép dữ liệu hoạt động về tần suất lỗi `40001`, số lần Retry thành công và Thất bại (Exhaustion).
- [ ] Áp dụng các bài test mô phỏng tự động (Testcontainers) để bắt sai sót (Regression testing).
