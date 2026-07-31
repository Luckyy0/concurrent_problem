# Đống Code Khai Tử: Cầm Đèn Chạy Trước Ô Tô, Khóa Nguồn Trước Đích (Code lỗi — khóa tài khoản nguồn rồi tài khoản đích)

## 1. Cấu Trúc Khung Xương (Entity tối thiểu)

```java
package example.transfer;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.math.BigDecimal;

@Entity
@Table(name = "account")
public class Account {

    @Id
    private Long id;

    @Column(nullable = false, precision = 19, scale = 2)
    private BigDecimal balance;

    protected Account() {
    }

    public Long id() {
        return id;
    }

    public BigDecimal balance() {
        return balance;
    }

    public void debit(BigDecimal amount) {
        if (amount.signum() <= 0) {
            throw new IllegalArgumentException("amount must be positive");
        }
        if (balance.compareTo(amount) < 0) {
            throw new InsufficientFundsException(id, amount);
        }
        balance = balance.subtract(amount);
    }

    public void credit(BigDecimal amount) {
        balance = balance.add(amount);
    }
}
```

Ví dụ này Sếp cố tình bóp cái Đồ án (domain) lại cho bé xíu để em nhìn thấu cái Deadlock. Code Ngân Hàng chốt sổ thật (Production financial model) người ta còn múa cả Sổ cái (ledger), Dấu vết (audit), Khóa trùng (idempotency) và Đối soát cuối ngày (reconciliation); Ba cái đó Sếp đéo nhét vô đây làm gì cho rối rắm.

## 2. Thằng Kho Giữ Cửa (Repository lấy khóa bi quan - `pessimistic lock`)

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

Lệnh này xúi thằng đệ Hibernate nhổ bọt ra câu SQL y chang thế này:

```sql
select id, balance
from account
where id = ?
for update;
```

Mỗi lần thốt ra câu lệnh trên, Nó giật Lấy Cái Ổ Khóa Dòng (row-level lock) và Ôm Chặt Lấy Cho Tới Khi Hàm Giao Dịch Chốt Sổ Xong Xuôi (transaction end).

## 3. Dịch Vụ Mù Quáng: Trông Thì Hợp Lý Nhưng Đi Lùi (Service trông hợp lý nhưng tạo thứ tự ngược)

```java
package example.transfer;

import java.math.BigDecimal;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BrokenTransferService {

    private final AccountRepository accounts;

    public BrokenTransferService(AccountRepository accounts) {
        this.accounts = accounts;
    }

    @Transactional(isolation = Isolation.READ_COMMITTED)
    public void transfer(long fromId, long toId, BigDecimal amount) {
        if (fromId == toId) {
            throw new IllegalArgumentException("source equals destination");
        }

        // TỘI ÁC BẮT ĐẦU TỪ ĐÂY: Trình tự lấy khóa ăn theo hướng chuyển tiền
        Account source = lock(fromId);
        Account destination = lock(toId);

        source.debit(amount);
        destination.credit(amount);
        // Thằng Kiểm Tra Rác (Dirty checking) sẽ xả 2 câu UPDATE trước khi gõ Búa Chốt Sổ (commit).
    }

    private Account lock(long accountId) {
        return accounts.findByIdForUpdate(accountId)
                .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
}
```

Lệnh Chuyển A→B sẽ vồ khóa A trước. Lệnh Chuyển B→A lại tham lam vồ khóa B trước. Rõ ràng Mọi Đứa Giao Dịch Đều Đang Khư Khư Bảo Vệ Khúc Rút Tiền Của Nó, NHƯNG LẠI ĐÉO HỀ CÓ MỘT THỨ TỰ TÔN TI CHUNG (total order chung).

> **Nói ngắn gọn:** Tội của Mày không nằm ở chuyện xài thiếu Bùa `FOR UPDATE`; Cái Ngu Nằm Ở Chỗ Mày ĐI XIN MỘT MỚ Ổ KHÓA NHƯNG LẠI PHỤ THUỘC VÀO CHIỀU ĐI CỦA ĐÍCH HAY NGUỒN.

## 4. Bãi Chiến Trường Chéo Cánh (Thứ tự SQL xen kẽ gây lỗi)

```sql
-- Thằng T1 Lên Tiếng
begin;
select * from account where id = 101 for update; -- T1 túm được A

-- Thằng T2 Lên Tiếng
begin;
select * from account where id = 202 for update; -- T2 túm được B

-- Thằng T1 Lại Kêu Lên
select * from account where id = 202 for update; -- T1 đứng Há Mỏ chờ T2 thả B

-- Thằng T2 Kêu Lên Trả Đũa
select * from account where id = 101 for update; -- VÒNG LUẨN QUẨN BÙNG NỔ! Bắt Buộc Có Đứa Chết!
```

Đồng hồ đếm ngược hết giờ, 1 trong 2 đứa nếm mùi Thất Bại Ngọt Ngào:

```text
ERROR: deadlock detected
SQLSTATE: 40P01
```

Thằng DB PostgreSQL nó khoái bắn đứa nào nó bắn, có thể T1, có thể T2 (chọn victim). Mày Lập Trình Tuyệt Đối CẤM DỰA DẪM Vào Việc Thằng Connection Nào Yếu Đuối Hay Bị Bắn Lặp Đi Lặp Lại.

## 5. Cấp Cứu Tào Lao Ngay Tại Hiện Trường Đống Rác (Retry sai bên trong transaction cũ)

Bị DB bắn cho tơi tả ở Production (nhận mã `40P01`), Mấy Đứa Mới Học Nghề hay nhét đoạn này vào nè:

```java
package example.transfer;

import jakarta.persistence.EntityManager;
import java.math.BigDecimal;
import org.springframework.dao.CannotAcquireLockException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class BrokenRetryingTransferService {

    private final BrokenTransferSteps steps;
    private final EntityManager entityManager;

    public BrokenRetryingTransferService(
            BrokenTransferSteps steps,
            EntityManager entityManager
    ) {
        this.steps = steps;
        this.entityManager = entityManager;
    }

    @Transactional
    public void transfer(long fromId, long toId, BigDecimal amount) {
        for (int attempt = 1; attempt <= 3; attempt++) {
            try {
                steps.runInsideCurrentTransaction(fromId, toId, amount);
                return;
            } catch (CannotAcquireLockException deadlockOrTimeout) {
                entityManager.clear();
                // NGU XUẨN: Xịt rửa cái Persistence Context ĐÉO ĐẺ RA ĐƯỢC 1 cái Giao Dịch DB (database transaction) Mới Đâu!
            }
        }
        throw new TransferTemporarilyUnavailableException();
    }
}
```

Bởi vì một khi thằng PostgreSQL nhả rác `40P01` ra, CÁI GIAO DỊCH HIỆN TẠI COI NHƯ ĐÃ THÀNH XÁC ƯỚP (aborted state).
Bất kể mày đánh lệnh gì tiếp, nó cũng sẽ Vỗ Mặt Chửi `25P02` (`in_failed_sql_transaction`) cho tới khi Mày Tự Tay Gọi Xe Rác Dọn Đi (rollback). Cái chai thuốc tẩy `EntityManager.clear()` Của Thằng JPA chỉ bứt Entity Ra Thôi; NÓ KHÔNG BIẾT VÀ KHÔNG THỂ LÀM DB DỌN DẸP GIAO DỊCH (Rollback JDBC) CŨNG KHÔNG HỀ NHẢ MỚ Ổ KHÓA CŨ RA!

Chưa hết, Chữ `CannotAcquireLockException` Bọc Ở Ngoài Dày Cộp Bao Quanh Hàng Đống Thứ Không Lấy Được Khóa Khác Nữa. Muốn Viết Luật Retry Thật Sự Phải Lột Vỏ Ra (inspect SQLSTATE/cause), Chứ Cấm Bốc Hốt Rằng: Thấy Cái Tên Ngoại Lệ Đó Nghĩa Là Đang Bị Kẹt Xe!

## 6. Món Võ Mèo Cào Ở Một Mình (Khóa cục bộ không bảo vệ nhiều instance)

```java
private final Object transferMutex = new Object();

public void transfer(...) {
    synchronized (transferMutex) {
        transactionalWorker.transfer(...);
    }
}
```

Cái Bùa Này chỉ hù được mấy cái Luồng (threads) Đang Lanh Chanh Trong Đúng Cái 1 Cái Xó Máy Chủ Tụi Nó Đang Đứng Thôi (1 JVM). Thằng App-1 Và Mấy Thằng App-2 Giữ Cái Ổ Khóa Nội Bộ Chả Liên Quan Gì Đến Nhau Nhưng Vẫn Lao Vào Cấu Xé Chung 1 Ổ Khóa Dòng Của Thằng PostgreSQL, Kết Quả Banh Xác Như Thường! Nó Lại Còn Làm Ứ Đọng Tốc Độ Chạy (throughput) Trên Từng Máy Mà Chẳng Giải Quyết Được Chút Quyền Lợi Nào Về Sự Thống Nhất Chung (correctness boundary).

## 7. Setup Bẫy Để Xem Chúng Nó Cắn Nhau (Điều kiện để tái hiện)

1. Sổ cái Đã Ghi Chép Vào Sẵn 2 dòng Account mang 2 Số ID Khác Nhau.
2. Vác 2 Ống Nước Riêng Biệt (physical PostgreSQL connections) Bơm 2 Cục Giao Dịch Chạy Cùng Lúc.
3. Kẻ T1 Giành Xong A Trước Mới Đi Ngó B.
4. Kẻ T2 Tranh Xong B Trước Khi Liếc Ngó A.
5. Cả 2 Đứa Mở Giao Dịch Miệng Thổi Sáo Há Mỏ Đợi Cục Khóa Tiếp Theo.
6. Đồng Hồ Giám Thị Chỉnh Nhanh (`deadlock_timeout` ngắn) Cho Vừa Mắt Test Nhưng Khóa Bạc Đầu (`lock_timeout`) Đừng Có Nổ Giữa Chừng.
7. Vác DB Thật Chơi Thật Trân Bằng PostgreSQL; Thằng H2 Sinh Ra Chả Phải Để Làm Bằng Chứng Cho Cái Màn Bắt Lỗi Xịn Xò Này.

Kê Lệnh Rào Chắn `CountDownLatch` Dằn Mặt Sau Cục Khóa Thứ 1 Để Bắt Tụi Nó Chen Ngang (interleaving). Chứ Cứ Lùa 2 Lệnh Gửi Cùng Nhau Đi Rất Dễ Bị Xịt, Và Màn Kịch Lỗi Không Bao Giờ Thấy Trên Máy Chấm Tự Động (CI run).

## 8. Đừng Ráng Chữa Cháy Kiểu Phèn (Các cách sửa chưa đủ)

- Nới Lỏng Giờ Chờ Khóa Cửa (`lock_timeout`): Chờ Mòn Gối Cho Xong Nhưng Cục Tức Cắn Đuôi Nhau Vẫn Không Hết Nhé.
- Xài Bùa Thử Lại Bất Tử Cứ Thấy Lỗi Là Chạy Tới Bến: Nó Đẻ Ra Trận Mưa Súng Đạn Liên Hoàn Gây Sụp (retry storm) Qua Cả Hạn Chót Của Hệ Thống.
- Sợ Lỗi Nhai Nuốt Xong Báo Success Lá Cải: Kẻ Chết Thay Của Ta Đã Đem Vứt Chôn Rác Mất Xác (rollback) Mà Khách Hàng Ở Kia Lại Chén Kết Quả Báo Láo À?
- Sắp Xếp Luồng Của 1 Cửa Này Thôi Nè: Còn Cái Lũ Batch Chạy Đêm / Đối Soát Bên Cạnh Nó Cầm Ngược Khóa Tới Sáng Sớm Cho Tụi Em Nhận Rác Nữa Kìa.
- Trình Lười Đọc Số Nhảm `SELECT` Của Gái Đẹp Xong Mới Xin Kẹo `FOR UPDATE`: Không Ép Nhường Hàng Từ Sớm (reserve row) Xong Chạy Đi Cầm Trật Khóa Lại.
- Vặn Nút Giờ Giám Thị Khám Nhỏ Tới Dấu Chấm Đỏ Dưới Production: Đè Chết Cổ Bắt Giám Thị Bào CPU Nhưng Đéo Cắn Đuôi Nhau Mất Đâu.
- Bắt Giao Dịch Làm Đồ Tể / Alo Remote Ra Đời Gọi Notification Ngay Giữa Đoạn Xin Nhau Khóa Cửa Kìa: Hút Trọn Đời Đợi (lock lifetime) Lên Đỉnh Mây Mở Rộng Địa Hình Nổ Bom (blast radius).

## 9. Đổ Nền Gạch Dựng Sân (DDL và dữ liệu khởi tạo cho thí nghiệm)

```sql
create table account (
    id bigint primary key,
    balance numeric(19, 2) not null check (balance >= 0)
);

insert into account(id, balance)
values (101, 1000.00), (202, 1000.00);
```

Rào Lưới Nhện `balance >= 0` Chỉ Để Chống Chế Phòng Hờ Lớp Vỏ Hù Dọa Ở Case Này Thôi Nhé. Mơ Đi Nó Bằng Thế Nào Được Vài Con Logic Ràng Buộc Kế Toán Nhúng Kĩ Ở Lớp Application Hay Chục Bức Tường Sổ Cái Nhà Bank Thật Xây Nên Đâu!
