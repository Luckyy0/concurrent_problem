# Giải Pháp Ngăn Chặn Deadlock: Trật Tự Khóa Chuẩn và Phục Hồi An Toàn (Canonical lock order and safe retries)

Để đảm bảo hệ thống tài chính vận hành nhất quán và không xảy ra tình trạng Deadlock (bế tắc) khi nhiều luồng truy cập đồng thời, hệ thống cần áp dụng kết hợp hai mô hình: **Trật tự khóa chuẩn** (Canonical Locking Order) làm giải pháp gốc rễ và **Thử lại có giới hạn** (Bounded Retry) như một cơ chế phòng vệ thứ cấp.

## 1. Thiết lập Trình tự Khóa Chuẩn (Canonical Locking Order)

Nhiệm vụ của thiết kế này là loại bỏ hoàn toàn khả năng hình thành chu trình chờ (wait-for cycle) tại cấp độ cơ sở dữ liệu bằng cách buộc mọi giao dịch yêu cầu tài nguyên theo một trật tự được sắp xếp sẵn.

```java
package example.transfer;

public record AccountLockOrder(long firstId, long secondId) {

    public static AccountLockOrder of(long leftId, long rightId) {
        if (leftId == rightId) {
            throw new IllegalArgumentException("source equals destination");
        }
        // Trật tự chuẩn: Luôn lấy ID nhỏ trước, ID lớn sau
        return leftId < rightId
                ? new AccountLockOrder(leftId, rightId)
                : new AccountLockOrder(rightId, leftId);
    }
}
```

```java
package example.transfer;

import java.math.BigDecimal;
import java.util.UUID;

public record TransferCommand(
        UUID commandId,
        long fromId,
        long toId,
        BigDecimal amount
) {

    public TransferCommand {
        if (amount.signum() <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }
    }
}
```

```java
package example.transfer;

import java.math.BigDecimal;
import java.util.UUID;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderedTransferAttempt {

    private final AccountRepository accounts;

    public OrderedTransferAttempt(AccountRepository accounts) {
        this.accounts = accounts;
    }

    /**
     * Phương thức chịu trách nhiệm thực thi logic chuyển tiền an toàn.
     * Transaction context sẽ được tạo mới hoàn toàn tại đây nếu được gọi thông qua proxy (Spring AOP).
     */
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public TransferReceipt execute(TransferCommand command) {
        // 1. Áp dụng trật tự khóa chuẩn
        AccountLockOrder order = AccountLockOrder.of(
                command.fromId(),
                command.toId()
        );

        // 2. Yêu cầu khóa theo thứ tự đã sắp xếp
        lock(order.firstId());
        lock(order.secondId());

        // 3. Tải dữ liệu vào persistence context theo chiều chuyển tiền
        Account source = accounts.findById(command.fromId()).orElseThrow();
        Account destination = accounts.findById(command.toId()).orElseThrow();

        // 4. Xử lý logic nghiệp vụ và thay đổi trạng thái
        source.debit(command.amount());
        destination.credit(command.amount());

        return new TransferReceipt(
                UUID.randomUUID(),
                command.commandId(),
                "COMPLETED"
        );
    }

    private void lock(long accountId) {
        accounts.findByIdForUpdate(accountId)
                .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
}
```

Khi ứng dụng yêu cầu cấp khóa:

- Tài khoản có ID nhỏ (ví dụ 101) luôn được ưu tiên lấy khóa.
- Nếu cả luồng T1 (101 sang 202) và luồng T2 (202 sang 101) chạy đồng thời, cả hai đều phải yêu cầu khóa tài khoản 101 trước.
- Không luồng nào lấy được khóa 202 nếu chưa nắm giữ 101. Do đó, đồ thị chờ (wait graph) không bao giờ bị khép kín, triệt tiêu hoàn toàn nguy cơ deadlock từ phía ứng dụng.

## 2. Thiết kế Cơ chế Thử lại Phục hồi (Bounded Retry Architecture)

Mặc dù trật tự chuẩn giải quyết tranh chấp cục bộ, cơ sở dữ liệu vẫn có khả năng phát sinh deadlock liên quan đến khóa ngoại (foreign key locks), hoạt động bảo trì, hoặc lỗi timeout. Để khắc phục, hệ thống cần có cơ chế thử lại tách biệt (retry coordinator), bao bọc phía ngoài ranh giới của `@Transactional`.

```java
package example.transfer;

import java.sql.SQLException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.NestedRuntimeException;
import org.springframework.dao.CannotAcquireLockException;
import org.springframework.stereotype.Service;

@Service
public class RetryingTransferCoordinator {

    private static final Logger log =
            LoggerFactory.getLogger(RetryingTransferCoordinator.class);

    private final OrderedTransferAttempt attempt;
    private final RetryProbe probe;

    public RetryingTransferCoordinator(
            OrderedTransferAttempt attempt,
            RetryProbe probe
    ) {
        this.attempt = attempt;
        this.probe = probe;
    }

    /**
     * Phương thức này KHÔNG sử dụng annotation @Transactional.
     * Đảm bảo mỗi lần gọi attempt.execute() sẽ khởi tạo một giao dịch mới.
     */
    public TransferReceipt transfer(TransferCommand command) {
        int maxAttempts = 3;
        for (int i = 1; i <= maxAttempts; i++) {
            try {
                return attempt.execute(command);
            } catch (CannotAcquireLockException e) {
                if (isDeadlock(e)) {
                    log.warn("Lỗi khóa phát hiện tại lượt {}/{} cho command {}", 
                            i, maxAttempts, command.commandId());
                    probe.recordDeadlockVictim(command.commandId(), i);
                    
                    if (i == maxAttempts) {
                        throw new TransferTemporarilyUnavailableException(
                                "Chuyển tiền gián đoạn sau " + maxAttempts + " lần thử", e);
                    }
                    
                    backoff(i);
                    // Vòng lặp tiếp tục, khởi tạo giao dịch mới
                } else {
                    // Nếu lỗi do lock timeout, vẫn ném ngoại lệ nếu cần thiết
                    throw e; 
                }
            }
        }
        throw new AssertionError("Không thể chạm tới đoạn code này");
    }

    private boolean isDeadlock(NestedRuntimeException e) {
        Throwable root = e.getRootCause();
        if (root instanceof SQLException sqlEx) {
            return "40P01".equals(sqlEx.getSQLState());
        }
        return false;
    }

    private void backoff(int attemptIndex) {
        try {
            long delay = 50L * attemptIndex;
            Thread.sleep(delay);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Tiến trình retry bị ngắt", e);
        }
    }
}
```

### Tại sao không nên retry bên trong phương thức `@Transactional`?

Khi PostgreSQL (thông qua lỗi deadlock) buộc hủy bỏ một giao dịch, trạng thái của kết nối JDBC chuyển sang chế độ thất bại (`in_failed_sql_transaction`). Bất kỳ truy vấn nào tiếp theo trong context đó sẽ bị cấm bằng lỗi `25P02`. Giao dịch cần phải được `ROLLBACK` và nhường chỗ cho một chu trình hoàn toàn mới. 

Việc bao bọc lệnh gọi `attempt.execute(command)` bên trong một phương thức bình thường (không có Transaction) giúp Spring AOP mở (begin) và đóng (commit/rollback) giao dịch một cách nguyên tử tại từng vòng lặp của quá trình thử lại.

## 3. Quản lý tác động phụ không thể hoàn tác (Un-rollbackable side effects)

Khi giao dịch cơ sở dữ liệu rollback, chỉ có dữ liệu trong PostgreSQL quay trở lại trạng thái cũ. Các hành động I/O từ xa (ví dụ: gọi API ngân hàng đối tác hoặc gửi Email) đã thực hiện không thể tự động thu hồi. 

**Quy tắc thiết kế:** Tuyệt đối không nhúng các lệnh gọi dịch vụ bên ngoài vào trong mã nguồn đánh dấu `@Transactional`.

```java
// THIẾT KẾ SAI - Rủi ro gửi email nhiều lần nếu giao dịch retry
@Transactional
public void transfer() {
    source.debit();
    destination.credit();
    emailService.sendSuccess(source.id()); // <-- Có thể gửi thông báo nhưng DB rollback
}

// THIẾT KẾ ĐÚNG - Outbox Pattern
@Transactional
public void transfer() {
    source.debit();
    destination.credit();
    outboxRepository.save(new EmailEvent(source.id())); // <-- Lưu cùng transaction
}
// EmailService sẽ quét (poll) bảng Outbox để thực thi việc gửi email một cách độc lập
```

## 4. Đặc tả luồng xử lý phục hồi chuẩn (Recovery Flow)

Mô phỏng trường hợp luồng T2 trở thành nạn nhân trong deadlock (Victim) và được thực hiện retry:

| Bước | Thực thi tại Luồng 1 (T1) | Thực thi tại Luồng 2 (T2) |
| ---: | --- | --- |
| 1 | `BEGIN` giao dịch MỚI (T1) | `BEGIN` giao dịch MỚI (T2) |
| 2 | Khóa tài khoản A (ID 101) | *Chờ do tài khoản A bị T1 khóa* |
| 3 | Khóa tài khoản B (ID 202) | |
| 4 | Trừ A 100, Cộng B 100 | |
| 5 | `COMMIT` (T1 hoàn tất) | |
| 6 | Trả về thông báo thành công | Nhận khóa tài khoản A |
| 7 | | *Lỗi xử lý (Giả định)* PostgreSQL thông báo Deadlock, T2 nhận `40P01` |
| 8 | | Kích hoạt `ROLLBACK` |
| 9 | | Sleep 50ms (Backoff) |
| 10 | | `BEGIN` giao dịch MỚI (Retry T2) |
| 11 | | Khóa A, Khóa B (Truy xuất số dư MỚI NHẤT từ T1) |
| 12 | | Trừ B 70, Cộng A 70 |
| 13 | | `COMMIT` (T2 hoàn tất) |

*Mô phỏng tại bước 7 mang tính minh họa nếu hệ thống có can thiệp ngoại vi; với trật tự chuẩn, lỗi deadlock sẽ không xảy ra.* Cơ chế Retry chỉ kích hoạt khi có các nguyên nhân hệ thống khác (ví dụ tranh chấp với một Job Batch). Toàn bộ số dư (balances) được nạp lại trong giao dịch sau khi Retry, ngăn ngừa hiện tượng mất mát dữ liệu do nạp ghi đè (lost update).

## 5. Các sai lầm cần lưu ý (Anti-patterns to avoid)

1. Phụ thuộc vào `synchronized`: Từ khóa `synchronized` của Java chỉ ngăn chặn tương tranh cục bộ trên JVM hiện tại, không có hiệu quả trên kiến trúc triển khai đa máy chủ (Scale-out) hoặc Kubernetes pod replicas.
2. Dùng ID nội bộ ngẫu nhiên: Sắp xếp theo một yếu tố không nhất quán (ví dụ UUID) khiến hệ quy chiếu thay đổi, phá vỡ trật tự khóa chuẩn.
3. Che giấu ngoại lệ: Cố tình bắt (catch) `Exception` nhưng không báo cáo, để mặc giao dịch tiếp diễn trong trạng thái hư hỏng.
4. Lạm dụng retry không giới hạn thời gian lùi (backoff): Không thiết lập `Thread.sleep` giữa các chu kỳ thử lại sẽ gây hiệu ứng "bão lỗi" (retry storm), khiến cơ sở dữ liệu ngưng trệ khi quá tải.

Giải pháp Trật tự khóa đồng nhất (Canonical lock order) và Phục hồi ngoài ranh giới giao dịch (Retry boundary abstraction) là mô hình xử lý triệt để được áp dụng rộng rãi trong thiết kế các hệ thống tài chính phân tán.
