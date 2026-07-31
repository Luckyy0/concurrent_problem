# Giải Pháp Cứu Mạng: Xếp Hàng Theo Chuẩn Và Gọi Giao Dịch Mới Để Chạy Lại (Giải pháp canonical ordering và fresh-transaction retry)

## 1. Mục Tiêu Thiết Kế (Mục tiêu thiết kế)

Giải pháp chính yếu của Sếp gồm hai lớp bảo vệ độc lập như cái khiên và áo giáp:

```text
Phòng Bệnh (prevention): Mọi giao dịch đụng chạm nhiều ví tiền đều BẮT BUỘC PHẢI KHÓA theo đúng 1 cái Trật Tự Chuẩn Mực (canonical order).
Chữa Bệnh (recovery): Nếu xui xẻo tột độ vẫn ăn Đạn Kẹt Xe 40P01 (từ chỗ khác nã tới), THÌ PHẢI VỨT BỎ (rollback) VÀ MỞ MỘT GIAO DỊCH MỚI TINH để làm lại trong một số lần giới hạn (bounded retry).
```

Cái Thuốc Xếp Hàng giúp trị tận gốc căn bệnh kẹt vòng tròn của hai ví tiền đã biết. Còn Thuốc Chạy Lại thì đi hốt rác cho những vụ đụng độ mồ côi từ mấy luồng code khác chọc vào, và nó giúp trả về một cái kết rõ ràng cho khách khi đã hết cứu được (vượt giới hạn).

## 2. Giải Pháp Số 1 — Khóa Tài Khoản Bằng Cột Mốc Bất Di Bất Dịch (Giải pháp 1 — Khóa account theo stable ID)

### Lệnh Truyền Tới Tấp (Command đầu vào)

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
            throw new IllegalArgumentException("Đưa cái lệnh đéo có ID thế?");
        }
        if (fromAccountId == toAccountId) {
            throw new IllegalArgumentException("Chuyển từ túi trái sang túi phải à?");
        }
        if (amount == null || amount.signum() <= 0) {
            throw new IllegalArgumentException("Tiền phải là số dương nha mày!");
        }
    }
}
```

Cái thẻ `commandId` kia sinh ra để dọn đường cho việc Lấy Dấu Vết (audit) hoặc Chống Gọi Hai Lần (idempotency) của ngành Ngân Hàng. Ở ví dụ hẹp này Sếp không xài nó để vỗ ngực xưng tên "Chạy 1 Lần Duy Nhất" đâu nhé.

### Thằng Thủ Kho Nhặt Ổ Khóa (Repository)

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

### Một Chuyến Xe Hàng Đóng Gói Riêng (Một attempt trong một transaction riêng)

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
        // TẤT CẢ QUỲ XUỐNG: SẮP XẾP TÔN TI TRẬT TỰ ĐÂY NÀY!
        long firstId = Math.min(
                command.fromAccountId(),
                command.toAccountId()
        );
        long secondId = Math.max(
                command.fromAccountId(),
                command.toAccountId()
        );

        // THÁNH CHỈ TRUYỀN RA: ĐỨA NHỎ CHỊU KHÓA TRƯỚC, LỚN KHÓA SAU
        Account first = lock(firstId);
        Account second = lock(secondId);

        // Giờ mới ngó xem đứa nào bị trừ, đứa nào được cộng
        Account source = command.fromAccountId() == first.id()
                ? first
                : second;
        Account destination = command.toAccountId() == first.id()
                ? first
                : second;

        source.debit(command.amount());
        destination.credit(command.amount());
        accounts.flush(); // Bắt Thằng Hibernate Nhả Nước Bọt Xuống DB Ngay Đi

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

Cái Bùa Mở Rộng `REQUIRES_NEW` cực kỳ có giá trị ở đây, vì Thằng `OrderedTransferAttempt` này đóng vai trò LÀ ĐẠI CA CAO NHẤT cho một cuốc xe đụng độ, được thét lôi ra từ một Thằng Sếp Tổng Không Thích Đụng Tay Vào Giao Dịch (non-transactional coordinator). Tuyệt đối cấm bế nguyên mâm này bỏ vô 1 Cục Giao Dịch Trùm Sổ Khác Ở Ngoài Lớn Hơn (outer business transaction): Bởi vì Tự Ý Chốt Sổ Ngang hông sẽ đánh nát Cái Thống Nhất Chung (atomicity) của ông lớn bên ngoài; Đã thế lúc Mày Xin Đình Chỉ Giao Dịch Đang Chạy Để Xin Cái Mới Là Mày Ngốn Thêm Của Công Ty 1 Cái Ống Nước Bơm Nữa Rồi Đấy (connection pool).

Lệnh Bắt Cửa 2 Lần `lock()` Kia BẰNG TAY LÀ DO SẾP CỐ TÌNH CÀI ĐẶT. Nghe Lời Lũ Dạy Đời Gộp 2 Thằng Lại Bằng Hàm `WHERE id IN (...)` SẼ LÀM DB ĐÉO BIẾT MÓC THẰNG NÀO TRƯỚC VÀ KIỂM TOÁN VIÊN VÀO BẮT NHỐT MÀY VÌ ĐÉO GIẢI TRÌNH ĐƯỢC THỨ TỰ. Đoạn Code Viết Trắng Ra Dễ Đọc Thế Mới Là Bảo Vệ Chén Cơm Của Đội Dev!

### Tại sao Lưới An Toàn Không Bị Xé Rách? (Vì sao invariant được bảo vệ?)

Cả Cuốc Chuyển A→B Lẫn B→A Đều Mở Tròng Mắt Xin Thằng Cửa `101` (Bé) TRƯỚC:

```text
Thằng T1 Giựt 101 ── Giựt Tiếp 202 ── Sửa Dữ Liệu ── Gõ Búa Chốt Sổ
Thằng T2 Há Mỏ Đợi 101 ───────────────────── Lượm 101 Đang Rơi ── Giựt 202 ── Chốt Sổ Theo Mông T1
```

T2 Há mỏ đợi `101` Chứ Nó ĐÉO CÓ GÌ TRONG TAY để làm T1 Khát Nước (chưa giữ `202`), Nên Chiều Ngược Mũi Tên Kẹt Xe Bị Chặt Đứt. Đủ Đồ Chơi Rồi Mới Đem Lên Bàn Xử, Thằng PostgreSQL Thấy Ngon Thì Ghi Xuống Sổ Kép, Xấu Thì Nó Hất Xuống Biển Hết Nguyên Mâm Mới Đẹp (rollback).

> **Nói ngắn gọn:** Chiều Chuyển Tiền LÀ NGHIỆP VỤ (business) nên Ai Chuyển Ai Nhận Theo Lệnh Đó; Nhưng ĐI XIN Ổ KHÓA CỬA Thì Dựa Theo SỐ CHỨNG MINH THƯ (stable account ID). CẤM LẤY CHỨC DANH NGHIỆP VỤ RA LÀM THỨ TỰ XẾP HÀNG XIN KHÓA!

## 3. Giải Pháp Số 2 — Vác Chổi Dọn Rác Từ Vòng Ngoài (Giải pháp 2 — Retry bên ngoài transaction)

Thuốc Xếp Hàng Chuẩn là Vắc-xin Mũi 1, nhưng Code Đấu Trường Máu (Production) Bắt Buộc Vẫn Phải Thủ Sẵn Băng Gạc Đỡ Lỗi `40P01`. Ta Lấy Thằng Điều Đào Chạy Lại (Spring Retry coordinator) Cắt Lìa Ra Khỏi Thằng Lính Làm Thuê Cầm Giao Dịch (transactional worker) Thành 2 Cục Beans Riêng Cho Cửa Nhà Dễ Quan Sát Kẻ Xâm Nhập (advisor boundary).

### Bác Sĩ Chuẩn Đoán Lỗi Lòi Phèo 40P01 (Phân loại SQLSTATE)

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
            // Lột cái vỏ ra mà coi, có đúng là Lỗi Do Database (SQLException) không!
            if (current instanceof SQLException sqlException
                    && containsSqlState(sqlException, "40P01")) {
                return true;
            }
        }
        return false; // Éo phải kẹt xe, mày tự đi mà khóc
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

### Cái Xác Đi Dạo (Exception dành riêng cho retry policy)

```java
package example.transfer;

public final class DeadlockVictimException extends RuntimeException {

    public DeadlockVictimException(Throwable cause) {
        super("Sếp ơi, DB nó Bắn Chết Chùm Con Vì Kẹt Xe Rồi", cause);
    }
}
```

Đút cái Giấy Báo Khám Bệnh này vào Trong Lính Làm Thuê:

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
            // Kêu Toáng Lên: Sếp Ơi Nó Bắn Em!
            throw new DeadlockVictimException(failure);
        }
        throw failure;
    }
}
```

Nội Hàm `executeLocked` Kia Nó Bọc Nguyên Y Xì Bức Tranh Số 1 Ở Trên Đấy Nhé. Lỗi Mới Chế Này Phóng Xuyên Tường Xuyên Thủ Thủng Cả Chốt Kiểm Giao Dịch (transaction interceptor); Vậy Là Xe Rác Nó Dọn Hết Rác Giao Dịch Cũ Banh Ta Lông RỒI NÓ MỚI BAY TỚI TAY THẰNG SẾP ĐIỀU ĐÀO ĐƯỢC!

### Thằng Điều Phối Khỏa Thân Không Mặc Áo Giao Dịch (Retry coordinator không có transaction)

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
                    random = true // Ngủ gật xí rồi chạy lại cho khỏi đạp lên nhau
            )
    )
    public TransferReceipt transfer(TransferCommand command) {
        // Sai Culi Đâm Đầu Chạy
        return attempts.execute(command);
    }

    @Recover
    TransferReceipt exhausted(
            DeadlockVictimException failure,
            TransferCommand command
    ) {
        // Culi Đâm Đầu Bể Sọ Tới 3 Lần Thôi Kêu Xe Cấp Cứu Trả Lời Cho Khách 
        throw new TransferTemporarilyUnavailableException(
                command.commandId(),
                failure
        );
    }
}
```

Gắn cái mác `@EnableRetry` Của Ông Nội Spring Lên App Thì Nó Mới Chạy Nhé Cưng. Đồng Hồ Báo Thức Của Khách (Production deadline) Bắt Buộc Phải Bọc Chọn Cả Mớ Thuốc 3 Lần Uống Kia Nhé (bao trùm cả attempts, backoff và response cleanup); Lệch Cái `maxAttempts` Chưa Đủ Để Hù Thằng Đếm Ngược Lỗi Đâu.

Những Thứ Tuyệt Đối Không Cho Bọn Nó Làm Lại Lần 2 (Không retry):

- Trương Ruồi Không Có Tiền hoặc Mù Đéo Thấy Ví Ở Đâu (insufficient funds / not found);
- Lệnh Sai Bắt Tại Trận (validation failure);
- Hết Giờ Xàm Xí Nào Đó Mà Đéo Phân Loại (arbitrary timeout);
- Không Biết DB Chốt Sổ Chưa Nữa Xíu Làm Lại Có Kêu Cái Thẻ Idempotency Ra Không Đây (ambiguous commit);
- Lỡ Tay Nhắn Tin Cáo Phó Ra Nước Ngoài Rồi Kêu Cái Loa Giật Lại Bằng Nhau Đâu (non-idempotent side effect).

## 4. Chơi Khung Đếm Ngược Ở Mức Vạch Trắng Của DB (Timeout trong phạm vi transaction)

Gắn Khóa Đồng Hồ Bo Riêng Một Con Hẻm Xin Đồ Chứ Không Ép Chết Cả Cái Nhà Nhé:

```sql
select set_config('lock_timeout', '1500ms', true);
select set_config('statement_timeout', '3000ms', true);
```

Cục Trắng `true` Ở Đây Nghĩa Là: Chỉ Tính Trong Đời Cái Giao Dịch Khốn Nạn Này Thôi Đó, Xong Việc Reset! Ba Mớ Con Số Này Chỉ Là Đồ Nháp Cho Đỡ Xấu Test; Khênh Lên Production Phải Đo Nhịp Tim Đo Máu Mủ Trên Từng Sợi SLO Của Ứng Dụng Mới Điền Vô Nhé, Xài Ngu Cấm Đổ Thừa.

Báo Khởi Mỏi Chờ Dài Cổ `lock_timeout` NÓ SẼ ÓI RA MÃ MỒ HÔI TRỘM `55P03`, CHỨ ĐÉO PHẢI CỤC ĐẠN `40P01` KẸT XE ĐÂU NHA. Giới hạn Hết Giờ (Timeout) Giúp Cầm Máu Chứ Không Đóng Đóng Vai Bác Sĩ Cắt Trĩ Bứng Gốc Bệnh Kẹt Vòng Tròn! (containment, không phải proof loại bỏ circular wait).

## 5. Rải Quân Theo Đội Hình Chuẩn Thế Nào Cho Không Chết (Triển khai canonical order an toàn)

Quân Luật Ban Ra Rồi Là Mọi Đứa Đều Phải Răm Rắp Chấp Hành:

1. Chĩa Súng Gọi Đầu Thằng Nào Dám Đụng Vào Bảng `account`: Chạy Chống Quá Tải API, Thằng Quét Rác Đêm (batch), Thằng Hẹn Giờ (scheduler), Mấy Tên Đối Soát (reconciliation), Tool Cho Admin Quậy Và Kể Cả Cái Lũ Xài Giấu Lệnh DB (stored procedure).
2. Trịch Thượng Khai Sinh Ra Đúng 1 Cái Chìa Khóa (stable unique key) Và Bộ Máy Sắp Xếp Nhất Quán (comparator).
3. Tuyệt Giao Rạch Ròi Nhiệm Vụ Code Kinh Doanh Ra Khỏi Cuộc Tranh Vị Trí Khóa Cửa.
4. Chơi Chạm Chán Hơn 3 Thằng ID Cùng Lúc Thì Phải Sàng Lọc Lọc Lại Rác Bị Trùng (sort distinct IDs) RỒI Mới Xin Từng Ổ Khóa; Đứa Nào Nhét Rác Lặp (duplicate) Hoặc Đâm Đầu Vô Vách Của Chính Nó (self-transfer) Thì Đạp Ra Ngoài Lồng Bàn Hết.
5. Tay Chạm Tới Nhiều Cái Tháp Khác Nhau Thì Ráp Bậc Thang Quy Tắc Global Từ Đầu Bảng: Tầng `customer → account → transfer`.
6. Phóng Phiên Bản Lên (Deploy) Đéo Có Chuyện Thằng Xài Luật Mới Nắm Cổ Đập Thằng Đang Chạy Lệnh Cũ.
7. Vẫn Dữ Ống Hút Nhòm Bảng Số `40P01` VÀ Khung Thuốc Cứu Sinh Retry Đi Nhé Vì DB Hay Ăn Dơ Chui Xin Thêm Lắm Trò Mình Éo Biết Đâu (database acquire thêm resources).

TUYỆT ĐỐI NGHIÊM CẤM TÍNH XẾP HÀNG THEO HÀM BĂM RÁC CỦA MÁY (hash), CHỮ VIẾT LOẰNG NGOẰNG ĐỊA PHƯƠNG (locale-sensitive) HAY CON SỐ MA DỄ THAY ĐỔI THEO HIỂN THỊ CỦA BỌN UX/UI ĐÉO QUAN TRỌNG NHÉ (mutable display number)!

## 6. Sổ Nam Tào Bắt Hồn Tình Huống Trớ Trêu (Hành vi khi lỗi)

| Giờ Tử Thần Bắt | Tình Trạng Ở Phòng Chẩn Đoán DB | Lệnh Ra Quyết Định Tại Trại Hòm Của Java Code |
| --- | --- | --- |
| Sửa lưng Trước Cổng Giao Dịch | Không có cục Khóa nào được đẻ ra | Táng Bạt Tai Lỗi Dữ Liệu Input, Chết Đi Mà Làm Lại Từ Đầu Xa Xôi (không retry DB) |
| Đợi Quá Hạn Phơi Rốn `lock_timeout` | Báo Bể Kế Hoạch 1 Lệnh; Chết Chùm Giao Dịch Luôn (rollback) | Chụp Thuốc Đắp Mù Do Thiếu Oxi Chứ Đéo Kêu Đánh Nhau (Temporary contention/deadline) |
| Lãnh Đạn Mảnh Khờ Bị Kẹt Trắng `40P01` | Sập Hầm Attempt Xoá Xác Lỗi, Quẳng Hết Bộ Khóa Luôn | Bơm Máy Phục Hồi Hút Sạch Không Bệnh Đội Lên Khung Cứu Thương Fresh Mới Tinh (Nếu Đủ Điệu Kiện) |
| Dành Chiến Thắng Vinh Quang Ở Đích | Hai Tiền Ví Được Niêm Phong Keo Chết Gắn Bó Chung Một Khúc Đóng Sổ | In Giấy Biên Nhận Quá Chuẩn Chỉ Sau Tiếng Tiếng Chuông Cuối (commit) |
| Mòn Cổ Chai Chết Vạch Cứu Thương Retry Cụt Kịch | Bãi Rác Cũ Đã Dọn Mất Tiêu Đâu Ai Mà Lăn Ra Đó Hồi Nào | Quăng Cho Biển Cấm Lối Phục Hồi Unavailable Báo Treo Xin Lần Khác |
| Khách Đi Rút Cáp Ổ Cắm Phút Cuối (Lúc Nã Commit) | Hồi hộp lắm DB Báo Sát Nút Hay Hụt Chẳng Rõ Đâu (ambiguous) | Vác Cuốn Sổ Chữa Bùa Hai Lần (Idempotency) Hay Bảng Dò Status Tìm Khác Gấp Ráp Chứ Sao! |

## 7. Móc Lốp Gọi Cứu Tinh Ngoài Tường (External side effects)

Thắt Cổ Treo Chim Tuyệt Đối KHÔNG ĐƯỢC Gọi Bà Thím Điện Báo Chăm Sóc Khách Hay Check Điểm Lừa Đảo Chéo HTTP RA NGOÀI ĐƯỜNG TRONG KHI ĐANG NGẬM HAI CỤC KHÓA VÀ NẰM LIỆT TRONG GIAO DỊCH!!!
Thích thì xài trò rải thính báo Hỷ (event sau transfer) Kiểu Này Nè:

- Kê Hàng Chui Vô Cục Sổ Giỏ Đựng Tin Nhắn Sắp Đẩy (outbox row) Vùi Nó Vô Trong 1 Lệnh Commit DB Chung Thôi;
- Thằng Shipper Phóng Theo Giao Nó (publisher gửi sau commit);
- Đứa Húp Tin (consumer) Thì Cầm Cục Móc Nhai Gạn Lọc Đỡ Trùng Bài Kêu Lên Lại Bằng Thẻ Ký Danh Môn (event ID);
- Nạp Thẻ Xin Cửa Retry Làm Lại Thì Xài Bùa Ký Danh Môn (idempotency record) Chuẩn Form Mà Vào Đánh Cho Thẳng Lưng Lên.

Thùng Outbox Giải Cứu Dính Sốt Đồng Tử Điển DB-Với-Cục-Ghi-Tin (atomicity) Nó Tự Mò Ép Vô, Còn Khung Rửa Háng Xác Đáng Sạch Bo Đã Nhận Đứt Đuôi Việc Giao Tới Hết Mầm Không Nhé (delivery semantic) Cút Ra Khỏi Đây Để Mấy Lão Đại Design Banking Đụng Trận Khác Chở Nó Qua Bờ Phía Bên Kia.

## 8. Chế Tạo Bùa Phép Theo Kiểu Gà Mờ Mất Kèo Số 1 (Phương án 1 — Một aggregate/coarse lock)

Thay Bùa Bằng Thuốc Giả Khóa Cục Gỗ Đại Diện (Đường Sắp Tới Nhóm Bơm Tiền Customer):

```sql
select id
from account_partition_guard
where id = :guard_id
for update;
```

Gỡ Rối Được Một Đống Sơ Đồ Xếp Khóa Trúc Trắc NHƯNG Trả Giá Bằng Việc Lùa Mọi Người Ra Xếp 1 Hàng Ngang Ép Chết Đồng Tiền Của Đội Lính Dù Bọn Nó Đi Hai Xe Độc Lập Chẳng Dẫm Chân Ai Mảy May! Cửa Khóa Cục Súc Này Nó Thọt Tốc Độ Chạy (throughput) Chạm Đáy, Cục Nóng Thần Sầu Gây Delay Dài Cổ Bóp Chết Lực (latency/pool pressure). Khuyên Nhủ Đừng Đụng Tới Trừ Lúc Cửa Hàng Éo Ai Tới Và Em Thích Đỡ Nhức Não Để Khỏe Cái Đời Thì Lấy! (Chỉ hợp lý khi contention thấp và correctness/simplicity quan trọng hơn parallelism).

## 9. Chế Tạo Bùa Mèo Hoang Cạo Mũi Số 2 (Phương án 2 — Serialize bằng queue/owner)

Bơm Cổng Phân Chia Đồn Bốt (Partition commands) Phủ Lên Đám Chủ Sở Hữu (account ownership) Lùa Giảm Sự Tranh Khách Đau Đầu Xuống Ế Và Giải Toả Lưu Lượng Khó Ở (backpressure). NHƯNG ĐÁ ĐƯỜNG Chéo Cánh Giữa Hai Đứa Khác Tỉnh Lại Hộc Máu Đẻ Lắm Trò Của Các Cổng Ngoại Giao Đi Với Nhau Cục Mịch Nát Cả Xác (cross-partition).
Xếp Hàng Nhạc Khúc Chờ Qua Sông Queue Làm Đéo Có Khung Cửa Sổ Bắn Cắt Hợp Nhất Atomic Đỉnh Cao Mà Tự Dưng Nuốt Thêm Tròng Bơm Trễ Sống Chết Đau Tim (delivery), Sập Trôi Trôi Sông (crash), Kẹt Vòng Đợi Quái Gở (ordering), Thêm Mớ Đồ Chơi Kép Vui Vẻ Nữa Đau (idempotency complexity). Bỏ Đi Mà Đừng Cố Vá Vào Nếu Mà Chỉ Cần Trị Cái Nếp Chữa Khóa Bình Thường Quất Sạch Ngon Cành Ngay Ở Trực Tiếp!

## 10. Chế Tạo Bùa Tâm Linh Mù Quáng Số 3 (Phương án 3 — Optimistic locking)

Nhét Chữ `@Version` Tránh Đụng Ánh Mắt Xỏ Xiên Của `FOR UPDATE` Nằm Trốn Ảo Diệu Bên Đọc Số, NHƯNG Lọt Lưới Lúc Update Phang 2 Cục Sổ Vào Mồm Liền Một Lúc Ở Cuối Bãi Bơi Thằng DB Thì Có Mà Ăn Cá:

- Kẹt Lác Ở Lỗ Cống Trút Thải Xuống Nền Xi Măng (block ở flush);
- Tát Lửa Tát Tai Nhau Nhanh Qua Giới Hạn Cởi Áo Thay (conflict version) Mới Nhào Dzô Xin Nhau Xin Xin Chết Cái Tiền Máy (retry);
- Nướng Chả Khét Kẹt Lòng Chặt Giữa Khung Kéo Áo Trái Ngược Mâm Bên Quán KIA Tranh Nhau Móc Nhau Nhé Khéo Gãy Dăng (deadlock nếu SQL updates/other locks xuất theo opposite order).

Thằng Múa Đuôi Hibernate Nó Ban Khóa Nhét Mồm `hibernate.order_updates=true` Tưởng Rằng Hay Tránh Bớt Nát Xác Cái Bể Kẹt Trí Tưởng Cục Cứ Thổi Hết NHƯNG Kèo Phá Nhà Giảm Thì Có Chứ Đéo Có Ai Giải Toả Căn Chức Đồng Tâm Đánh Rác Khẩn Mưu Hay Vác Dao Trọng Lệnh Đè Tiết (business revalidation). Lấy Ổ Optimistic Dán Xài Nơi Vắng Vẻ Người Qua Lại Ít Và Chi Phí Bơm Xin (retry) Khá Ưa Ưa; Còn Đồ Máu Lửa Hot Giành Nhanh Bật Khách Hàng (hot multi-row transfer) Thường Xuyên Nằm Không Mất Xác Kèm Gọn Gàng!

## 11. Chế Tạo Bùa Khống Chế Tứ Trụ Số 4 (`SERIALIZABLE`)

Xếp Hàng Phép `SERIALIZABLE` Trùm Bảo Vệ Vực Nước Xoáy Anomaly Bằng Khiên SSI Oai Phong Khét Mù NHƯNG Nó Rũ Áo Éo Thề Sẽ Hứa Giao Dịch Chắc Cố Được Chui Lot Sang Lọt Khung Kẽ Cuối, Và NÓ ĐÉO ĐÀN ÁP CÁI TÍNH CỐ CHẤP KẸT XE TRONG DÒNG VÍ KHÓA (ordinary lock deadlock)!!!
Lính Ứng Dụng Code Tự Gánh Mà Vuốt Cục Đạn Đuổi Cổ Giết Ngay `40001` Lẫn Cái Lỗi Bùa Khờ `40P01` Lấy Lồng Đạp Đuổi Sinh Ra Vươn Đời Gọi Thuốc Uống Nháp Thằng Lính Cứu Trợ Retry Làm Lại Mới Toanh Tươi. Bò Vào File `DB-009` Để Ăn Bật Nhé.

## 12. Chế Tạo Bùa Lá Bài Chơi Ảo Số 5 (Phương án 5 — Advisory lock)

Chìa Khóa Tư Vấn Chặn Miệng Ngoài Trời Chỉ Vui Cửa Vui Nhà NẾU ĐỨA NÀO CŨNG Chơi Sạch Giữ Mồm Chọn Khóa (key), Khoanh Vùng Sạch (scope) Và Đọc Kỹ Nhả Phao (unlock semantics) Thật Tâm Huyết! DB Không Hút Óc Đi Khóa Giùm Hàng Cửa Sổ Kệ Bạn Khóc Bất Lực Đâu Bỏ Mặc Rào Rỗng Lệnh Ràng Kẻ Khác Xơi Vào Trái Phe Bạn Vô Biên Rất Tức Lắm! Sửa Mâm Gắn Dính Ràng 2 Thằng Hai Phía Áo Khóa Theo Cạnh Xương Cột Mốc `canonical row locking` Rẻ Thúi Dễ Cầm Đũa Bật Rõ Còi Mà Lại Ép Cái Lão Đạp Hàng Lệnh Tự Dưng Bay Tay Nhanh Gắn Buộc Nhau Ăn Thề Vào Hơn Lắm Con Sói Cú Trắng KIA (mutation tự tham gia)!

## 13. Phân Trần Ai Mất Ai Được Trên Sàn Nhảy (So sánh trade-off)

| Thuốc Trị | Sự Đỉnh Đạt Chuẩn Sạch Sẽ (Correctness) | Sức Quạt Tải Tốc Độ Bốc Khói Bắn Xé (Throughput/latency) | Nhu Cầu Uống Xin Xin Cấp Cứu (Retry) | Số Liệt Lỗi Kẹt Chết Bóp Đuôi (Deadlock risk) | Sự Khó Ưa Dễ Bị Quát Khi Trông Xếp Hàng Chạy Test Vui Nhà (Vận hành) | Lướt Êm Sàn Song Song Xóa Án JVM Node Phân Thân Không Bờ (Multi-instance) |
| --- | --- | --- | --- | --- | --- | --- |
| Lính Tuân Theo Hàng Chuẩn (Canonical row order) | Bao Cứng Luôn Với Đồ Nghề Kê Sổ Trong Protocol Kịp | Song Song Dàn Đội Hình Nếu Hai Hội Chẳng Dẫm Chân Ai Mảy May; Chỉ Xếp Mỏ Chờ Lũ Hot Row Bu Đánh | Bắt Buộc Vẫn Cần Áo Cứu Sinh Khi Bí Quá Giữ Thân (bounded fallback) | Mất Máu Bức Gãy Xoay Chuẩn Cycle Nát Banh Bãi Cái Kiểu Dễ Xưa Đã Biết | Kính Lúp Dò Đường Sạch Làng Soi Rọi Sạch Từng Thằng Sọc Mã (audit path) | Dư Sức Chuẩn Do Cái DB PostgreSQL Bọc Đuôi Ẩn Ràng Chuẩn! |
| Trạm Gác Ổ Thần Mù (Coarse guard row) | Dễ Hút Áp Chỉnh Minh Thấy Ngay Tại Ngay Đập Thẳng | Xếp Một Đống Vô Tư Xé Nát Xe Chạy Khi Quán Đông Dài Cổ Nát Tiền Phải Cần (latency cao khi hot) | Ít Gãy Kẹt Đuôi Vẫn Yêu Tiên Xin Lại Khi Nằm Kéo (failure handling) | Gần Tiêu Biến Nếu Chọn Đúng Kéo Nhau Chung Chỉ Lôi Vô Cùng Cửa (guard order thống nhất) | Lác Mắt Bật Màn Hỏng Lửa Ánh Còi Theo Dõi Hot-key Liên Miên Trục Khảo | Vẫn Còn Nóng Ấm Sống Bền Bỉ Ngon Lành Chịu Trận! |
| Hàng Chờ Lùa Chó Xé Kênh (Queue/owner) | Phóng Mắt Chút Bức Thiết Dò Ngó Cổng Ngoại Lệ Quăng Xác Qua Quần Chữ Giao Mồm Tác Động Giao Tụ Đống | Chịu Xếp Bậc Sửa Giãn Thở Trễ Máy Rớt Nhanh Độ Delay Đếm Ngược Ngửa Tay Xin (backpressure/queue latency) | Chói Lỗ Bức Nhau Nhả Ép Tiêu Ngược (Redelivery/idempotency) | Quăng Bom Thay Nửa Khu Sang Nguy Cơ Vỡ Cắt Đứt Giáp Cánh Biên Đời (cross-owner risk) | Lên Đỉnh Khó Nhằn Đau Ruột Nặng Gánh Gồng Theo Mỏi Mắt Cột Nhịp Tiêu! | Sống Lai Rai Vẫn Có Mặt Sáng Thằng Cùng Vác Dấu Ngang Dọc Sống Lâu Đen |
| Phù Hiệu Hy Vọng Ngáo Mơ Ảo (`@Version`) | Soi Chớp Viết Phao Sai Bịp Rán Nhầm Stale Writes | Lướt Nhanh Lúc Trống Trơn Sân Nắng Bóng Bông Tình Báo (conflict hiếm) | Lò Nung Cục Nhau Máu Amplification Thở Hắc Đói Cơm Áo Bóp Rát Nát Mày Trôi Nước Ở Ngưỡng Lôi Ngang (hot) | TRÒ LỪA: Ép Đéo Hết Cái Tật Kẹt Đuôi Chéo Cánh!!! | Khá Chấp Nhận Qua Bờ Nằm Yên Vung Khúc Lát Giây! | Xài Nháp Đua Qua Khúc Giữa Được Dạo Lướt Bóng Ẩn Được Xài Vui Cùng Lúc! |
| Cái Lưới Tuyệt Sát Kén Giờ Oái ăm (`SERIALIZABLE`) | Đỉnh Tôn Bá Bền Che Anomalies | Ngắt Đuôi Đứt Kênh Ăn Cáo Cấp Abort Lúc Trống Phang Bãi Tới Tấp Đụng Cựa (contention) | Khóc Kêu Uống Mép Nhai Sinh `40001` Gấp Khẩn Bắt Buộc Đeo Vào Ngực Tức | Bùa Méo Che Nổi Dấu Thần Rác Trượt Bão Ẩn Lấp Vết Hằn Chó Kéo Cái 40P01 Máu Nhồi Đuôi Kẽ | Rát Đầu Đâm Đớn Hơn Lũ Quần Tụ Kia Rên Trụy Trục Căng Xé Ráng Thức Đêm | Quất Tuốt Ở Vùng Biển Kia Cánh DB Gánh Phụ Ế! Đỡ |
| Bùa Nhảm Cục Bộ Ngang Phè Nhà Quê (JVM-local mutex) | Gắn Ràng Ruột Đúng Nhõn Khung Giao Đứa Lũ Đời Cái JVM Mà Thôi Nhá | Tụt Dép Từng Tràng Lên Tiếng Xếp Cục Lại! | ÉO Giải Quyết ĐƯỢC Khung Trục Khủng Khiếp Database Họa Khéo Này Ơi! | Họa Sập Quần Luân Đỉnh Nối Lại Bữa Bò Bức Tử DB Ở Nửa Quần Node Đêm Vách Kín Nhất Xíu Vây Ảo! | Quá Dễ Lấp Yếu Sinh Kém Thiểu Năng Ở Góc Tường Mờ Nhạt Chữ Đời Này Lát Đất Sai Vạch Cả (Sai scope) | XỊT BỎ BÊ LÚ LẪN TÉ MÁU!!! (KHÔNG!) |

## 14. Dò Bảng Check Hàng Chốt Hạ Trước Khi Bung Lụa Live Bán Chác (Checklist trước production)

### Cú Đánh Vạch Đồ Khóa Đè Chết (Thứ tự khóa)

- [ ] Lấy Giấy Ghi Mực Nét Rạch Cứng Mã Hiệu Chuẩn Rõ Xếp Bài So Đo Chọn Bộ Nhất Quán Không Quẹo Đường Được Báo Cáo.
- [ ] Soi Bất Cứ Con Nhện Đụng DB Nào Kéo Tài Khoản Đi Phải Luôn Ép Theo Chỉ Tiêu Đứa Nhỏ Nhường Lớn Đi Đúng Tôn Ti Theo Trật Tự Trấn.
- [ ] Nhớ Lên Cái Bậc Cấp Tầng Các Loại Lớn Đè Nhỏ Tình Báo Ổ Đạn Rõ Ràng Nghiêm Lệnh Phả Hệ Chống Quay Xe Dốc.
- [ ] Chỉ Được Dở Trò Phán Xử Tiền Của Phép Cộng Trừ (mutation) Và Check Hàng Input Sau Đúng Lúc Tóm Thành Công 2 Cái Tờ Lệnh Rút Gươm Khóa Hai Bên Lỗ Tay Chặt Nịch! Cầm Hơi Ngại Cắt Tiền Đi.

### Sổ Bàn Giao Dịch Dọn Rác & Hồi Sinh Xin Kẹo DB Xót Bụng Khẩn Vớt (Transaction và retry)

- [ ] Kẻ Đứng Trình Đầu Báo Gọi Chạy Cấp Cứu Mới Nhớ Bóc Lột Cởi Áo Khoác Giao Dịch Ra Chứ Lôi Ngồi Gánh Là Toạc Quần! (không outer transaction)
- [ ] Quất Proxy Chuyển Ánh Bật Giao Dịch Cắt Ngang Làm Ổ Chứa Tươi Mới Bo Chặt Sự Băng Giá Rác Đổ Vũng Lỗ Khung Giao Tích Persistence Mới Toe.
- [ ] Rà Nét Chữ Mực Xác Đáng Nhất Bắt Đeo Lỗi Nổ Xe Bắn Sạch Tới Cái Đuôi Mầm Lạ Mới Chịu Nhả Cục Chén Chạy Tiếp Còn Không Văng Dép Giữa Đường Lệnh Phóng (`40P01` / Đã Bấm Đèn Duyệt Kéo).
- [ ] Ghi Chú Rành Mạch Nắm Đuôi Vết Phân Số Khung Chặn Giới Cực Đỉnh Cho Gọi Cứu (cap), Chịu Bắt Sóng Hấp Nhịp Ru Ngủ Đo Giật Lùi Sốc Kép Random Chậm Rớt (jitter/backoff) Lẫn Chốt Toàn Sân Đóng Kệ Tổng Thể (deadline).
- [ ] Hết Xí Quách Thì Khạc Văng Rõ Câu Chữ Cấm Đoán (domain/API outcome rõ) Thay Vì Cười Giả Trân Mờ Tịt Đợi Quẳng Xác Ứ Lên Chờ Đâu Nữa Trống Lốc Ai Trông Chó.
- [ ] Tuyệt Tuyệt Hết Ngáp Được Vọc Entity Cũ Sình Mùi Bắt Cố Ở Sổ Cũ Vác Sang Giữ Ép Vắt Óc Lên Gánh Cuộc Đời Nhá (Không reuse entity từ failed attempt).

### Mắt Đảo Cảnh Chó Cắn Cáo Quật Thùng Chống Vỡ Tường Lò Đào (Vận hành)

- [ ] Vặn Nút Ép Trọng Đồng Hồ Xả Giờ Bắn Tiễn `lock_timeout` và `statement_timeout` Đè Khít Cho Chết Chùm Chảy Thụt Lút Cái Ngưỡng Chịu Khách Đập (latency budget) Vừa Ý Vãi Chó Á.
- [ ] Kê Ống Dòm Đo Nịp Trực Soi Bảng Huyết Tử Trì `pg_stat_database.deadlocks`, Bộ Đầu Súng Sát Thủng Trực Tiếp Của DB SQLSTATE Và Quét Dải Thang Máy Kêu Retry Liên Hồi Tần Tần Kêu Rên Cuốn Chảy Giọt Nhỏ Nhanh Nữa Đo (metrics).
- [ ] Giấy In Ghi Log Gỡ DB So Sổ (deadlock log) Bắt Được Gáy Kẻ Phạm Cắn Bịt Nhau Đúng Lịch Nhưng PHẢI CHE Kín Thân Tên Dữ Liệu Ví Đồ Lấp Phân Khách Dấu Gắt Rành Gọn Gàng! (không lộ dữ liệu nhạy cảm).
- [ ] Quăng Hết Mấy Cái Cú Điện Kéo Mỏi Đời Lôi API Nhảm Cục Kéo Gọi Ngoại Đất Ra Dọn Sạch Trắng Hàng Ghế Trống Phía Trước Thằng Múa Khóa Nắm Kẹp Sổ Đang Nhả Lấp Khung Nặng Nề DB (Remote I/O).
- [ ] Giới Thể Gian Manh Sóng Lỗi Lởn Vởn Chưa Rõ Nghìn Mặt Hết Ngay Hay Éo Vào Báo Chưa Tới Cuối Đường Bóp Tiền (ambiguous commit) PHẢI Nhét Cái Khiên Giám Thị Mép Rào Ngăn Đạp Đôi Vô Tình (idempotency/status strategy) Mò Đồ Đích Thật Dòm Đường Sạch Đẹp Ngủ Quên Cũng Ngon Ơ.
- [ ] Kéo Test Hồi Quy Lọc Sót Mực Đạp Mìn Bằng DB Thật Xịn Trấu Vỏ Lưỡi Bò Testcontainers Postgres Ra Trấn Phép Khai Hoang, H2 Ếch Nhựa Biến Đi Biến Xéo Không Đủ Trình So Cựa Chờ Tới Sáng Sớm Cửa!!!
