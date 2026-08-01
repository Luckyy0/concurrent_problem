# Phân Tích Mã Nguồn Lỗi: Khóa tài nguyên theo thứ tự động (Broken code — dynamic lock ordering)

## 1. Cấu trúc thực thể (Entity definition)

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

Ví dụ này sử dụng một mô hình nghiệp vụ (domain model) tối giản để tập trung vào cơ chế gây ra Deadlock. Trong các hệ thống tài chính thực tế, mô hình này sẽ phức tạp hơn với sổ cái (ledger), lưu trữ lịch sử kiểm toán (audit trail), xử lý trùng lặp (idempotency) và nghiệp vụ đối soát.

## 2. Truy xuất dữ liệu với khóa bi quan (Pessimistic lock repository)

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

Sử dụng annotation `@Lock(LockModeType.PESSIMISTIC_WRITE)` sẽ tạo ra câu lệnh SQL như sau thông qua Hibernate:

```sql
select id, balance
from account
where id = ?
for update;
```

Lệnh `FOR UPDATE` thực hiện cấp khóa độc quyền cấp dòng (row-level lock) trên các bản ghi thỏa mãn điều kiện. Khóa này sẽ được nắm giữ cho đến khi giao dịch chứa nó kết thúc (commit hoặc rollback).

## 3. Dịch vụ gây lỗi: Khóa theo tham số truyền vào (Broken service implementation)

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

        // LỖI NGHIÊM TRỌNG: Trình tự khóa phụ thuộc vào tham số đầu vào
        Account source = lock(fromId);
        Account destination = lock(toId);

        source.debit(amount);
        destination.credit(amount);
        // Quá trình Dirty checking của Hibernate sẽ sinh các lệnh UPDATE trước khi commit
    }

    private Account lock(long accountId) {
        return accounts.findByIdForUpdate(accountId)
                .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
}
```

Với cấu trúc trên, lệnh chuyển từ A sang B sẽ khóa tài khoản A trước, trong khi lệnh chuyển từ B sang A sẽ khóa tài khoản B trước. Các giao dịch cố gắng bảo vệ dữ liệu nhưng lại bỏ qua yêu cầu thiết lập một trật tự chuẩn chung (canonical order). Điều này khiến hệ thống dễ bị rơi vào trạng thái bế tắc (deadlock) khi các luồng hoạt động đồng thời đan chéo nhau.

## 4. Xung đột luồng thao tác (Interleaved SQL execution)

Khi hai tiến trình tương tác ngược chiều diễn ra song song:

```sql
-- Giao dịch T1 (từ A -> B)
begin;
select * from account where id = 101 for update; -- T1 khóa A (ID: 101)

-- Giao dịch T2 (từ B -> A)
begin;
select * from account where id = 202 for update; -- T2 khóa B (ID: 202)

-- T1 tiếp tục:
select * from account where id = 202 for update; -- T1 phải chờ T2 giải phóng B

-- T2 tiếp tục:
select * from account where id = 101 for update; -- T2 phải chờ T1 giải phóng A -> DEADLOCK!
```

Kết quả là một trong hai giao dịch sẽ nhận được ngoại lệ từ PostgreSQL:

```text
ERROR: deadlock detected
SQLSTATE: 40P01
```

Cơ sở dữ liệu tự động chọn một nạn nhân để hủy bỏ (abort). Ứng dụng không thể dự đoán hoặc thao túng quá trình lựa chọn này.

## 5. Áp dụng Retry không hợp lệ (Invalid retry within the same transaction)

Lập trình viên đôi khi cố gắng bắt ngoại lệ và thử lại ngay trong khối giao dịch (transaction block) hiện tại:

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
                // LỖI KỸ THUẬT: Xóa persistence context không tạo ra giao dịch CSDL mới.
            }
        }
        throw new TransferTemporarilyUnavailableException();
    }
}
```

Khi PostgreSQL trả về ngoại lệ `40P01`, giao dịch cơ sở dữ liệu đã chuyển sang trạng thái `aborted`. Bất kỳ lệnh SQL nào tiếp theo trong giao dịch này sẽ bị từ chối với lỗi `25P02` (`in_failed_sql_transaction`) cho đến khi lệnh `ROLLBACK` được thực hiện. 

Sử dụng `EntityManager.clear()` chỉ xóa trạng thái của bộ nhớ đệm (Persistence Context), nhưng không tác động đến giao dịch JDBC (JDBC transaction) đang tồn tại và không nhả khóa (lock) cơ sở dữ liệu. Thêm vào đó, `CannotAcquireLockException` có thể bao gồm nhiều trường hợp lỗi lấy khóa khác (như lock timeout), yêu cầu hệ thống phải phân tích cụ thể mã `SQLSTATE` thay vì chỉ phụ thuộc vào tên class Exception.

## 6. Giải pháp khóa cục bộ (Local synchronization issue)

Sử dụng cơ chế khóa cục bộ của Java:

```java
private final Object transferMutex = new Object();

public void transfer(...) {
    synchronized (transferMutex) {
        transactionalWorker.transfer(...);
    }
}
```

Cơ chế này chỉ đồng bộ hóa các luồng trên cùng một máy ảo (JVM). Khi ứng dụng triển khai trên môi trường nhiều máy chủ (multi-instance), các JVM độc lập không chia sẻ `transferMutex` sẽ tiếp tục tranh chấp các bản ghi (rows) trong cơ sở dữ liệu và vẫn dẫn đến Deadlock. Đồng thời, cấu trúc khóa độc quyền (exclusive lock) này sẽ làm giảm đáng kể khả năng đáp ứng (throughput) của hệ thống.

## 7. Các điều kiện tái hiện (Reproduction criteria)

1. Có sẵn các bản ghi tài khoản để thực hiện giao dịch, mỗi bản ghi mang một ID định danh.
2. Hai giao dịch thực thi trên hai luồng và hai kết nối vật lý (physical connections) độc lập.
3. Giao dịch T1 hoàn thành việc khóa tài nguyên A.
4. Giao dịch T2 hoàn thành việc khóa tài nguyên B trước khi T1 bắt đầu yêu cầu khóa B.
5. Cấu hình tham số kiểm tra deadlock (`deadlock_timeout`) nhỏ hơn tham số chờ khóa (`lock_timeout`).
6. Thử nghiệm trên cơ sở dữ liệu PostgreSQL thực tế thay vì dùng cơ sở dữ liệu in-memory (như H2) để đảm bảo độ chính xác của cơ chế MVCC.

## 8. Các phương pháp khắc phục chưa toàn diện (Incomplete mitigations)

- **Tăng `lock_timeout`:** Tăng thời gian chờ có thể giải quyết các lỗi chậm do quá tải, nhưng không khắc phục được tình trạng chờ chéo của deadlock.
- **Retry không giới hạn:** Dẫn đến "retry storm", tiêu hao băng thông kết nối và tài nguyên tính toán của cơ sở dữ liệu.
- **Thực hiện một chiều (One-way process):** Loại bỏ việc tranh chấp từ một quy trình, nhưng deadlock vẫn có thể xảy ra do các công việc khác (như batch jobs hoặc đối soát).
- **Điều chỉnh cấu hình giám sát PostgreSQL:** Điều chỉnh `deadlock_timeout` để giảm tần suất kiểm tra có thể giúp giảm sử dụng CPU của PostgreSQL nhưng không ngăn chặn được deadlock.

## 9. Khởi tạo cấu trúc bảng thí nghiệm (DDL setup)

```sql
create table account (
    id bigint primary key,
    balance numeric(19, 2) not null check (balance >= 0)
);

insert into account(id, balance)
values (101, 1000.00), (202, 1000.00);
```

Ràng buộc (constraint) `balance >= 0` đảm bảo tính toàn vẹn cơ bản. Tuy nhiên, logic kiểm tra và trừ tiền thường phải được thực hiện ở tầng ứng dụng (application code) sau khi các bản ghi đã được khóa thành công để xử lý các yêu cầu nghiệp vụ phức tạp.
