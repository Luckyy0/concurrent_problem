# Cứu Cánh Chữa Cháy — Rặn Retry Kéo Toàn Bộ Transaction Nấp Trong Tường Rào Mới (Giải pháp)

## 1. Bản Vẽ Định Hình Mục Tiêu (Mục tiêu thiết kế)

```text
Thằng Gác Cổng (non-transactional coordinator)
  → Phóng 1 nhát chọt thử Bọc Transaction SERIALIZABLE (transactional attempt)
  → Nếu Commit Bùi (success): Lôi Ra Cục Án Quyết Sạch (durable decision)
  → Đụng Búa Tạ 40001: Ép Rollback Sạch Nước Cặn Hoàn Tất
  → Lùi Nhóp Nhép Bịt Kẽ Chặn (bounded backoff) Ở Lớp NGOÀI Transaction
  → Nhồi Phát Đấm Tươi Mới (fresh attempt) Bốc Tái Tạo Sạch Bộ Khung State Lên Đầu
```

Cái Dấu Tích Lệnh Command ID KHÔNG ĐƯỢC PHÉP TRÁO qua lại giữa các vòng nhóp (attempts). Chắc Cú Thằng nào Lọt Trọt Trơn Tru Giao Dịch Thành Công Mới Được Lôi Ngòi Viết (ghi decision, reservation) Lẫn Nạp Tịch Báo Loa Khách (outbox intent).

## 2. Giải Cứu Phương Án 1 — Tách Đôi Khứa Gác Cổng & Kép Đánh Khổ Sai (Tách coordinator và attempt worker)

### Đơn Lệnh và Bản Án (Command và outcome)

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

### Bọc Túi Một Cuốc Khổ Sai Kín Kẽ Trong Transaction Riêng Tư (Một attempt trong transaction riêng)

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

Áo Kép Trấn Bùa `REQUIRES_NEW` Bị Ép Quất Vô Vì Tướng Lệnh coordinator Vốn Trụ Án Là Chóp Cổng Lệnh (root use case) ĐÉO Hề Nối Transaction Lõi; Mỗi cú bửa búa attempt PHẢI Ngắt Ngoại Riêng Độc Lập Kéo Rút. Vỡ Nát Nếu Đoạn Hàm này Bị Gọi Đè Phọt Ngược Dọng Từ Áo Khứa Khác Gọi Business Transaction Bọc Ngoài Ôm (outer), Tích Bóc Trục Đinh Commit Rời (independent commit) Của Ả Sẽ Đâm Gãy Đứt Tích (atomicity) Của Lưới Vòng Ngoài Vừa Hao Ốc Connection Đợi Treo Lọng (suspend); Lệnh Thiết Kế Là Ép Cấm Chặn Rút Đầu Chắn Mép Kẽ Này Khéo Rõ Mạch!

Cục `execute()` Đỉnh Cóc Ngáo Cứ Hét Căng Cục Trái Án Nháp result Tụt Lên Truốc Lúc Chốt Ngáp commit; MÀ Kẻ Chọt API Kéo Hàm (caller) CHỈ Nuốt Cục Mồi Sau Khi Trục Kính Proxy Phán Trọng Búa Lệnh Commit Lọt Khóa (commit thành công)! Đớp Lọng Commit Xì Ngáp Phọt Rớt Bịch `40001` Là Phụt Trào Bóng Exception Lọt Ngược Chóp Áo Coordinator Quăng Trọng, Còn Cục Khứa Result Ngáp Lạnh Trắng Tươi Đéo Bao Giờ Được Cạp Liếm Đâu Nhóc Khờ!

> **Sếp chốt lại:** Trái Phanh Khứa Công Nhân Kéo Thép (worker) CHỈ Trọng Được Nuốt 1 Chút Attempt Cụt; Lão Gác Chóp (coordinator) ÔM MẸ Bọc Phép Cán Cân Kéo Retry (policy). Đéo Có Quái Gì (component) Ngạo Nghễ Nắm Vừa Bịt Transaction Trong Ngực Xong Móc Đáy Lại Nằm Dọng Mõm Chờ Áo Backoff Thở Đâu Nha Cưng.

## 3. Khay Kéo Chọn Chia Sót Phọt Lỗi Bệnh PostgreSQL (Phân loại PostgreSQL failure)

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

Bảng Nhãn Tên Bọc Cục Rào Trắn (Constraint names) Là Bản Hợp Đồng Sống Mạng Máu Khứa Của Database, Ném Migration MỚI Cấm Léo Hút Tụt Lóng Cứt Ém Nhẹm Tráo Rạch Áo Tên Im Liềm Trắng Mặt Nhé. Dọng Hút Đâm Sập Gãy `23505` TRƯỢT Ốc Trên Dọng Lưới Khóa Rào (constraint khác) THÌ Phải Tạt Ngáp Đục Vọt Lên Nóc (propagate); ĐÉO Phải Bệnh Nạn Hút Cựa Trượt Uniqueness Nào Cũng Đều Là Trò Vọc Trùng Kép Command Rác Đâu!

## 4. Cái Mũ Lưới Đeo Rút Deadline Vây Ngợp Trận Bóp Nháp Tái (Retry policy có deadline)

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

Mấy con Số Má Móc Cắm Production Đều PHẢI Rút Kéo Từ Áo Bọc Nạp (configuration) Vạch Định Rọt (validate) Á! Con Hót `4` Chỉ Giương Lưới Cáp Thú Phóng Lọng Trượt Boundary Cho Mượt Não Đọc VUI (sample), ÉO PHẢI Cục Búa Gõ Benchmark Lưng Hay Chọt Chóp Recommendation Móc Thiên Hạ Chuẩn Âu!

### Lùi Nhép Bịt Kẽ Ngáp Jitter Vọt Nhịp, Lết Chạy Lách Lưới NGOÀI Transaction (Backoff)

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

Bọn Áo Application Gánh Sống ảo Nháp virtual threads / vút Móc Chọt Lịch async scheduling Hoàn Toàn Xé Nhép bọc Ngáp Dọn Cứt (blocking backoff) Để Ném Sang Gáy Cho 1 Thằng Đốc Lịch (scheduler) Vừa Mâm Ôm Trọn. Hợp Đồng Máu KHÔNG Lệ Đổi (contract không đổi): LÉO BAO GIỜ Ngậm Bọc Áo Nháp Giao Kéo Nách Giữ Ôm Kho Transaction Hay Sợi Cáp Connection Trụy Nháp; Cạp Tuốt Mọi Khứa Hét Gọi Chém Lọng Ngắt Trận (cancellation) Lẫn Boong Phết Ép Trụy Áo Cút Biên Lệnh Chặn Đuôi Deadline!

## 5. Lão Tướng Gác Chóp Đeo Lột Sạch Áo Transaction (Coordinator không có transaction)

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

    public LimitReservationCoordinator(
            SerializableLimitAttempt attempts,
            LimitDecisionReplay replay,
            SerializableRetryPolicy retryPolicy,
            RetryBackoff backoff
    ) {
        this.attempts = attempts;
        this.replay = replay;
        this.retryPolicy = retryPolicy;
        this.backoff = backoff;
    }

    public LimitDecision reserve(ReserveLimitCommand command) {
        if (TransactionSynchronizationManager
                .isActualTransactionActive()) {
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

Nên Cắm Nhét Tiêm (inject) Khứa Đồng Hồ Trắng `Clock` Và Áo Bọc Nạp configurable Lưới Vòng Chắn Áp Chót Thời Mạng operation deadline NGAY VÀ LUÔN, HƠN LÀ Cứ Phọt Hô Điếc Gọi Khứa Bọc Hàm Cháy Khét `Instant.now()` Cắm Trực Thẳng Trên Khúc Xương Sản Xuất (production implementation) Khứa Ạ; Khúc Cắm Dọn Code Múa Chóp Điệu Múa orchestration Kia Ép Nhép Cho Nó Ngắn Cụp Trọn Tay Khép. Lệnh Búa `attempts.execute()` Nó Ói/Ho Lệnh Bật Văng Rọt SAU Lớp Thắng Tầng Không Kính Transaction Interceptor Mượt Dứt Khóa Đinh Rồi Ấy, Cho Lên Chảo catch Ngáp Trụy Dọng Trượt Chạy Ốp Sau Khứa Lóng Sóng Chọt Cuốn Lệnh (rollback/commit) Đều Đã Nát Nước Khép Ngáp Tịch Khói!

Lọng Tua Gọi Tua Ngáp Replay Bợ Gắp Riêng Bộ Nháp Cưới Mới Tinh read transaction Nè Trẻ:

```java
@Service
public class LimitDecisionReplay {

    private final LimitCommandDecisionRepository decisions;

    public LimitDecisionReplay(
            LimitCommandDecisionRepository decisions
    ) {
        this.decisions = decisions;
    }

    @Transactional(
            propagation = Propagation.REQUIRES_NEW,
            readOnly = true
    )
    public LimitDecision requireCommitted(UUID commandId) {
        return decisions.findById(commandId)
                .orElseThrow(() ->
                        new DuplicateCommandOutcomeUnavailable(commandId)
                )
                .toResult();
    }
}
```

Nếu Dây Sóng Cứa Mạng Bóng Nháp Mờ Connection Tịt Bóng Lọt Lưới Áo Mù Mờ Lệnh Trượt Cấu Tín (ambiguous commit), Mang Đúng Nháp Mã Đinh Cũ Cùng command ID Kéo Dọc Phọt Tới Lão Coordinator Xắn Chạy Lọt Gọi Tiếp Ục Và Mép Ánh Đi Xuyên Khe Replay Check Ở Ngõ Rào Attempt Sóng Lưới Đầu Nhé Cưng. Cái API Phọt Éo Lếu Láo Đẻ Bóng Nhóp Trích Nháp ID Trắng Mặt Dỏm Đâu (không tự tạo ID mới)!

## 6. Món "Tái Sống" Spring Retry Bọc Máng Ánh Đeo Song Hành (Phương án tương đương)

Mày Vẫn Có Thể Ôm Xài Trọn Kẹp Lọn `RetryTemplate`/`@Retryable` Lúc Bọc Kéo Rọn:

- Lão Ngoác Bóng Dọng Tiếng Ngoại Lỗi (exception) Bị Lôi Phết Thuật Lịch Nặn Ánh Ngọt Thành 1 Trụ Loại Đít Khứa Mép Riêng Biệt (type riêng) CHỈ GÓI Dọng Cho Đúng Khứa Lép `40001`!
- Áo Lưới Retry Trọn Vạch Trắng Tươi Khứa retry advice XIN CHẮC 100% Vót Phẳng Lưng Giương NẰM LỌT Vòng Ngoài Lều Áo Dọng Giao Dịch transaction advice!
- Máng Líp Dọn Ngáp Trễ backoff/deadline/exhaustion Bọc Trọn Được Tiêm Config Lẫn Đập Vành Boong integration-test Bóng Chút!
- Kép Cứa Bọc Lọng Bủa Nháp Attempt Worker VẪN Giương Mình Bọc Xé Bean Chạy Rọt Riêng Tư.

Cấm Trọng Nháp Nhép Trây Dọng Lọng Trùm Bóng Cùng Lúc (annotate cùng method) Rồi Mót Tụ Ánh Trắng Nạp Bừa Đón Lão Thứ Tự Bọc advisor order! Hồ Sơ Bóng Mật Kẽ Kéo Trụ `SPR-006` Xẻ Ánh Trắng Lõi Thét Mạch Chạy Máu proxy chain Boong Boong Ngót Nháp Chi Tiết Mịn Hơn Nhiễu Ấy!

## 7. Giải Cứu 2 — Áo Bọc Template Túi Transaction (`TransactionTemplate`)

Nếu Khứa Khát Khoang Tự Boong Code Rào Điểm Mạch Giáp Bọc Ranh Kẽ Cứa Programmatic:

```java
var template = new TransactionTemplate(transactionManager);
template.setPropagationBehavior(
        TransactionDefinition.PROPAGATION_REQUIRES_NEW
);
template.setIsolationLevel(
        TransactionDefinition.ISOLATION_SERIALIZABLE
);

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

Mỗi Dọng Hét Lệnh `execute` CHÉM Ngót Sạch Trơn Mọi Boong Rút Phọt (commit/rollback) Vừa Kéo Tròn Dọng Trụy Nháp XONG XUÔI TRƯỚC LÚC Áo Lưới Chặn Máng Catch Ngáp Phụt Lên Chạy Oái. NHỚ Bám Trọng Kéo Khứa work object Kín Khóa Trắng Boong Trọng Lưới ÉO Ôm Dọng Ngậm Cứt cache entity/result Nào Giữa Đuôi Trích Lệnh attempts Vọt Kéo Ấy!

## 8. Giải Cứu Phế 3 — Lính Trấn Vòng Áo Khứa Hàng Đinh Móc Trượt Ngáp Dọng Đếm (Guard row hoặc conditional counter)

Nếu Con Khách Dọng Cửa Buôn merchant Cứ Boong Bọc Dọng Rót Mỏ Gà Nóng (hot key) MÀ Dây Chút Tái Óc Retry Rate Vọt Ngộp Đỉnh, Đem Nháp Búa Phết Chốt Đinh serialize Áo Quyết Decision TRỰC DIỆN Sát Nháp Lưng Đâm:

```sql
select limit_amount
from merchant_limit
where merchant_id = :merchantId
for update;
```

Lúc Vuốt Kẹp Tịch Áp Boong Dọng Giữ Án Xé Khứa Đinh guard row Đóng Kéo Xong, Mới Hét Trọng Đọc Mép Total Rồi Đấm Ghi insert Xuống. Khứa Kép Gọi Nháp Thứ 2 Sẽ Bị Block Cắn Bị Tọng Dọng Đợi Cho Boong Tới Khi Nó Rọi Ngáp Đâm Mặt Xem Được Rọt Total Mới Tinh Cóp. Tuy Trò Này Gấp Phụt Bày Sóng Mũi Kẽ Đọc Xé Trắng Đo Chặt Cứu Lỗi Conflict Đỉnh Boong NHƯNG Bù Lại Khứa Đẩy Gọng Lọng Nhóp Án Khóa Đợi Mốc Ngáp lock wait / Đọng Lọng Đít Hàng Nhóp Cáp Vọc connection occupancy Trọng Nốc Phòi Ngáp! Phải Tiêm Vừa Khứa Phọt Chặn Giờ Khóa Áo `lock_timeout`, Nén Khúc Transaction NGẮN Chẽ, Cứa Lẫn Đập Deterministic Order (thứ tự cố định) Khi Vọt Đụng Ngực Nhiều Ổ Thương Buôn merchants 1 Cú Lượt Lọng Nhá.

Còn Đây Khứa Bộ Bộ Đếm Nhấp Nháy (conditional counter) Bóng Nhóp Boong Gấp Gắn Mảnh Khỏe Oái Rọt Khúc Bé Xinh Nè:

```sql
update merchant_limit
set reserved_amount = reserved_amount + :amount
where merchant_id = :merchantId
  and reserved_amount + :amount <= limit_amount;
```

Nếu Đo Vọt Nháp Chốt Gãy Lọng Affected-row `1` Phụt Dọng Lên LÀ Máng Ấy Xé Accepted; Lọt `0` Bóng Dọng Rọt Rejected Cút Nhanh. Boong Insert Đo Cọc Án Reservation/decision BUỘC Phải Bọc Nhóp Bịch Trong Kẽ Dọng Mép Rốn Cùng Dòng Transaction Sót; Lẫn Lão Áo Bộ Đếm Counter Kéo Cứt Ấy CẦN Có 1 Chiêu Tháo Bọc Nhóp Dọn Lưới Soát Án Phết Kẽ reconciliation discipline Oái Nhịp Trọng Nha!

## 9. Giải Cứu Móng 4 — Dòng Bóng Bọc Đọc Cụt Đo Khứa Lệnh Treo (`SERIALIZABLE READ ONLY DEFERRABLE`)

Gắn Áp Cho Đám Loa Report Kéo Lọng Đọc Oanh Trọng Chỉ Dọng:

```sql
begin isolation level serializable read only deferrable;
```

Boong Transaction Óc Kép Dọng Bóng Đáy Chút Nháp Chờ Mép Ngáp Đục Chụp 1 Nháp Lệnh Chớp Sóng Cóc Snapshot SẠCH BỌNG An Toàn Ròi Nháy Rọt Đâm Tua Phóng, MÀ LÉO SỢ Tịch Mũi Máng Đâm Abort serialization Xẹt Đáy Lọng! ĐÉO CHO PHÉP Nhóp Áp Bóng Dụng Bóng Máng Rọt Này Vào Tua Kép Đọc/Ghi Áo Nháp Dọng Tịch Reservation Command Nghe Ranh Chó Khờ Nhá!

## 10. Cách Bóng Lọng Oanh Xử Phọt Ngáp Máu Lúc Dọng Đít Lỗi (Hành vi khi lỗi)

| Điểm Chết Sứt Vành (Failure) | Bọc Kết Giao Ốp (Transaction outcome) | Nháp Kéo Tua (Retry)? |
| --- | --- | --- |
| `40001` | Sập Lưới Known abort Boong Trắng | Bật Nháp CÓ, Kéo Sạch Nháp Áp Cuộn (whole transaction), Chóp Lưới Bounded Kéo |
| `40P01` | Lọt Bịch Tử Tội Deadlock victim abort Cắn Kẽ | Trút CÓ Nếu Ngáp Dọn Safe; KHÚC NÀY Đục Lại Ục Lọng Xếp Lưới Bóng ordering Khóa |
| `55P03` | Nốc Tịt Lock acquisition timeout Chết Đuối Khóa Kéo | Tùy Phọt Deadline / Nhóp Áp Lệnh policy Vọt Riêng Trọng Boong |
| `23505` Dọng Rào Boong command constraint Đục Máu | Rọt Lưới Current duplicate attempt Ốp Lộn Đít rollback Trắng Bóng | Áo Fresh read / Nhóp Máng Tua Phóng replay CÙNG Bóng Lọng Mã same command |
| Án Kinh Tịch Business `REJECTED` | Ghi Đinh Án Búa Khóa Mõm Durable decision commit Boong | KHÔNG NHÁP Tua Đéo Tái Kéo Cứt! |
| Trút Mã Kẽ Invalid input Mỏ Oanh Nát | Đéo Thèm Nhấc Chân Đâm attempt Lưới Cóc Mới | KHÔNG! |
| Rớt Bóng Mạng Bịch Connection loss Búng Lúc Đục commit Vọt Bữa Cứt | Lọng Bọc Ụp Oanh Ambiguous Khứa Oái Cứt Trọng | Trích Sót Cọc Query / Khéo Kéo Chút Nhóp Lọng replay Xuyên Ngực Áp command ID Đinh Khóa |
| Dọng Bóp Cứt Ngáp Khứa Attempts exhausted Chết Hết | Trắng Đít KHÔNG Còn Ngáp active attempt Bóng Rọt Oanh | Quăng Bọc Lệnh Cứt Áp retry-later / Oái Kéo Cục Án Kỹ Thuật technical outcome Vụt Boong Khứa |

## 11. Bàn Cân Bóng Đo Lọng Khứa Cân Bọc Lệnh Nháp Trade-off (So sánh)

| Giải Dọng Kéo Nháp (Cách) | Vành Móng Ngực Cứt Kéo Correctness boundary Trọng | Nạn Kéo Khứa Cứt Áp Contention/latency Ngáp Lọng | Nháp Kéo Trụy Lệnh Retry Nháp | Boong Đá Tụ Áp Lệnh Database load Rọt Boong | Oái Kéo Bọc Cứt Lọng Vận hành Kéo | Lọng Bọc Sóng Khứa Multi-instance Nháp Boong |
| --- | --- | --- | --- | --- | --- | --- |
| Bóng SSI + Lưới bounded retry Kéo | Cạp Áp Lệnh Đinh Kéo Predicate dependencies Boong Nháp | Đéo Chặn Cửa block Bởi Nhóp SIREAD Nhé; Gãy abort Bọc Dưới Áp overlap Rọt Chóp | Áp Bắt Buộc Oái Ngáp Boong | Đẩy Vọt Tăng Dọc Theo attempts Khứa | Đo Bọc Ngáp Dọng Theo dõi Khứa `40001` / Lọng Khóa Cáp Áp predicate locks Ngáp | Có Chút Lưới Nhang Oanh! |
| Kép Dọng Phét Áp Guard `FOR UPDATE` | Bóng Mõm Cứt Áp Merchant row Dọng Kéo | Bịt Rọt Cửa Áo Block Lưới Dọng Tuần Tự Bọc Dọc Trúc Boong Áp hot merchant Lọng | Phọt Dọng Khóa Kéo Áp Timeout/deadlock Lưới Trọng | Đuôi Ngáp Chờ Boong Khóa Wait Áp Giữ Nháp Boong connection Máng | Lọng Bọc Đáy Áp Dọng Khóa Khứa Lock observability Cứt Nhóp | Có Boong Nhang Nhéo Khứa! |
| Móc Đếm Cứt Kéo Conditional counter Lọng | Bắn Đinh Rọt Áp Atomic merchant row update Kéo Bóng | Nhóp Đinh Rọt Dọng Khóa Áo Lưới Hot-row serialization Boong Nhang | Lưới Đéo Bóng Ngáp THƯỜNG Trút Kéo Cứt Trụy Nháp Lọng Khứa Không retry Dọng Boong business reject | Vọt Bọc Ít Bóng Lọng queries, Đinh Ghi counter writes Ngáp Bóng | Móc Lọng Rọt Dọng Đáy Khứa Reconciliation Trọng | Có Oái Bóng Khứa! |
| Boong Queue/owner Ngáp Lọng | Dọng Lưới Khứa Trọng Dọng Đáy Partition ownership Nhóp | Móc Áp Dọng Queue latency/backpressure Rọt Lưới Boong | Boong Rọt Kéo Redelivery / Nhóp Áp Khứa idempotency Bóng | Ngáp Khứa Đít Tịt Lọng Giảm Bóng Ngáp concurrency Ngay DB Boong Nháp | Khứa Bóng Cao Móc Vút | Có Bóng Boong Lọng Khứa! |
| Áp JVM mutex Cứt Trọng Boong | Dọng Máng Khứa Process memory Oai Nhóp | Bọc Gãy Lưới Serialize Khứa Cục Bộ Bóng | ĐÉO Bít Boong Lọng Nhóp Áp Sửa Khứa cross-node Kéo | Bóng Khứa Lọng Vẫn conflict Trọng Trụy Boong | Khứa Áp Lọng Dễ Cứt Nhóp NHƯNG Boong Trọng Lưới Áp SAI Kẽ scope Rọt | Móc Đéo KHÔNG Nháp Khứa Boong! |

## 12. Danh Sách Lọng Kiểm Tra Trục Cốt Móng Chót Trước Lúc Gác Khứa Boong Production (Checklist trước production)

### Máng Rọt Bóng Áp Lọng Isolation Vụt Lưới Vành Boundary Khứa

- [ ] Lọng Bóng Áp Effective isolation Vụt Móc Được Chọc Trọng Lọng assert CHẮC NỊCH LÀ Boong `serializable` Ngáp Khứa.
- [ ] Áp Tướng Coordinator ĐÉO Móc Lọng Nối Trọng Trụy Outer transaction Ngáp Cứt Boong Khứa Dọng.
- [ ] Từng Khứa Nháp attempt ĐỀU Vượt Cửa Bọc proxy/template VÀ Nuốt Cứt Dọng Khứa fresh persistence context Nháp Sạch Tươi Khứa Nhóp.
- [ ] Boong Rọt Lọng Lưới Catch Bọc Vụt Trùm CẢ Bọn Áp Bọc Lệnh Cứt commit exception Bóng Nhang Nháp Khứa!

### Boong Lưới Áp Áo Lọng Chóp Tịch Nháp Retry policy Khứa Boong

- [ ] Lọng Bóng CHỈ Mót Dọng Boong Khứa Đóng Lưới allowlist Khứa Bóng `40001` Cùng Mớ Áp Lệnh Lỗi failures Khứa Đã Boong Cứt Phê Duyệt Rọt!
- [ ] Bọc Lọng Áo Đỉnh attempt cap Khứa Ngáp, Chóp Trụy Nhóp jitter/backoff Lọng, Đáy Kéo Deadline Lẫn Nút Cứt Ngáp Lọng cancellation ĐỀU Khứa Phải Dọng Boong Rọt Móc Sống Đủ!
- [ ] Kéo Mỗi Cứ Nháp attempt Lọng MÓC Bụng reload Trọn Lọng Cứt limit, Áp Đáy Bóng total Boong Cùng Boong Án Bọc Dọng Bóng Lọng durable decision Trắng Khứa.
- [ ] Boong Khứa Exhaustion Dọng Nát ĐÉO Được Phọt Áp Lệnh Cứt Báo success Rọt Bóng Khứa Đít Lọng Oai!

### Móng Oai Lọng Tịch Khứa Boong Idempotency Lẫn Khứa Bóng Đít Lọng Side effects

- [ ] Dọng Lưới Khứa Cứt command ID Boong Giữ Bóng Trọn Áp Cứt Khứa Nguyên Dọng Máng Qua Bóng Nhóp Lọng retry / client replay Khứa Boong.
- [ ] Áp Bọc Máng Tên Đinh Rọt Khứa Unique constraint names Boong Lọng Nắm Bọc Dọng Coi Là Khứa Cứt Contract Đóng Máu Trọng!
- [ ] Bóng Dọng Mõm Lọng Án Khứa Decision Cứt Lẫn Lọng Boong Outbox Bóng commit Boong Lọng CÙNG Đít Bóng Dọng Cứt business change Móc Trọn Trụy Lọng!
- [ ] ĐÉO Khứa Nhóp Chút Áp Nào Boong Dọng remote I/O Bọc Trong Kẽ Transaction Lọng Cứt Khứa Nhang Nháp Nhóp Boong.
- [ ] Bóng Lọng Oái Áp ambiguous outcome Buộc Khứa Dọng Cứt Cần Boong Kéo Móng Rọt Áo Bóng query/replay path Trọng Khứa Nháp Boong.

### Khứa Bóng Lọng Áp Boong Vận hành Rọt Đít Trọng Cứt

- [ ] Boong Lọng Máng Móc Đục Cứt Trọng Khứa Bóng Lọng Metric `40001` Boong Khứa, attempts Áo Trọng, exhaustion Dọng Bóng, Khứa Boong latency Lẫn Áp Boong Bọc Dọng Lọng pool pressure Boong Dọng PHẢI Sống Khứa Trụy Bóng Nhang Đít Lọng!
- [ ] Bọc Khứa Lọng query plan / index / Lọng Bóng Dọng Đít Khứa predicate-lock granularity Bóng Boong Ngáp Móng Nắm Boong Cứt Theo Boong Khứa Dõi Trọng Boong.
- [ ] Boong Khứa Merchant Dọng Lọng Cứt Nóng Khứa Rọt Bóng Có Lọng Ngáp admission control Hay Dọng Bọc Boong Cứt Khứa Khéo alternative strategy Khứa Boong.
- [ ] Lọng Bóng Áp Testcontainers regression Bóng Chóp Khứa Ngáp Móc Kiểm Boong Tra Rọt Boong Khứa final total Lẫn Boong Khứa Lọng decisions Khứa Rọt Đít Boong.
