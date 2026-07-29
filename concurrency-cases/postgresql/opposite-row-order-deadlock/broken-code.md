# Code lỗi — khóa tài khoản nguồn rồi tài khoản đích

## Entity tối thiểu

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

Ví dụ cố ý giữ domain nhỏ để nhìn rõ deadlock. Production financial model còn
cần ledger, audit, idempotency và reconciliation; đó không phải scope của case.

## Repository lấy khóa bi quan (`pessimistic lock`)

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

Hibernate phát SQL tương đương:

```sql
select id, balance
from account
where id = ?
for update;
```

Mỗi call acquire row-level lock và giữ lock tới transaction end.

## Service trông hợp lý nhưng tạo thứ tự ngược

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

        Account source = lock(fromId);      // order phụ thuộc hướng transfer
        Account destination = lock(toId);

        source.debit(amount);
        destination.credit(amount);
        // Dirty checking flushes both UPDATE statements before commit.
    }

    private Account lock(long accountId) {
        return accounts.findByIdForUpdate(accountId)
                .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
}
```

Một request A→B lấy row A trước; request B→A lấy row B trước. Mỗi transaction
đúng là đang bảo vệ state đã đọc, nhưng tập hợp locks không có total order chung.

> **Nói ngắn gọn:** lỗi không nằm ở việc thiếu `FOR UPDATE`; lỗi nằm ở thứ tự
> acquire nhiều `FOR UPDATE` locks phụ thuộc source/destination.

## Thứ tự SQL xen kẽ gây lỗi

```sql
-- T1
begin;
select * from account where id = 101 for update; -- giữ A

-- T2
begin;
select * from account where id = 202 for update; -- giữ B

-- T1
select * from account where id = 202 for update; -- chờ T2

-- T2
select * from account where id = 101 for update; -- cycle, một victim
```

Sau khoảng kiểm tra deadlock, một session nhận lỗi dạng:

```text
ERROR: deadlock detected
SQLSTATE: 40P01
```

PostgreSQL có thể chọn T1 hoặc T2 làm victim; application không được phụ thuộc
actor cụ thể sẽ thua.

## Retry sai bên trong transaction cũ

Đoạn sau thường được thêm sau khi production bắt đầu thấy `40P01`:

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
                // Sai: clear persistence context không tạo database transaction mới.
            }
        }
        throw new TransferTemporarilyUnavailableException();
    }
}
```

Khi PostgreSQL đã trả `40P01`, transaction hiện tại ở aborted state. Statement
tiếp theo thường nhận SQLSTATE `25P02` (`in_failed_sql_transaction`) cho tới khi
rollback. `EntityManager.clear()` chỉ detach entities; nó không rollback JDBC
transaction và không release locks.

Ngoài ra, `CannotAcquireLockException` có thể bao cả lock acquisition failures
khác. Retry policy phải inspect SQLSTATE/cause, không suy luận mọi exception cùng
class đều là deadlock.

## Khóa cục bộ không bảo vệ nhiều instance

```java
private final Object transferMutex = new Object();

public void transfer(...) {
    synchronized (transferMutex) {
        transactionalWorker.transfer(...);
    }
}
```

Cách này chỉ serialize threads trong một JVM. App-1 và App-2 có hai mutex khác
nhau nhưng cùng khóa các PostgreSQL rows, nên database deadlock vẫn xảy ra. Nó
còn làm throughput trong mỗi node giảm mà không tạo correctness boundary chung.

## Điều kiện để tái hiện

1. Hai account rows đã commit và có ID khác nhau.
2. Hai physical PostgreSQL connections chạy hai transactions đồng thời.
3. T1 hoàn tất lock A trước khi xin B.
4. T2 hoàn tất lock B trước khi xin A.
5. Cả hai giữ transaction mở tới khi thử acquire row thứ hai.
6. `deadlock_timeout` đủ ngắn cho test nhưng `lock_timeout` không được nổ trước
   detector.
7. Test dùng PostgreSQL thật; H2 không phải bằng chứng cho detector/SQLSTATE này.

`CountDownLatch`/barrier tại sau lock thứ nhất tạo đúng interleaving. Chỉ gửi hai
request “gần cùng lúc” không đảm bảo cycle xuất hiện ở mọi CI run.

## Các cách sửa chưa đủ

- Tăng `lock_timeout`: chỉ giới hạn wait; opposite order vẫn tồn tại.
- Retry vô hạn ngay khi thấy exception: có thể tạo retry storm và vượt deadline.
- Catch rồi trả success: victim đã rollback, nên caller nhận kết quả sai.
- Chỉ sort trong endpoint này: batch/reconciliation path vẫn có thể khóa ngược.
- Dùng plain `SELECT` trước `FOR UPDATE`: không reserve row và không sửa ordering.
- Giảm `deadlock_timeout` production xuống cực thấp: tăng detector overhead, không
  loại bỏ cycle.
- Gọi remote risk/notification service giữa hai locks: kéo dài lock lifetime và
  mở rộng blast radius.

## DDL và dữ liệu khởi tạo cho thí nghiệm

```sql
create table account (
    id bigint primary key,
    balance numeric(19, 2) not null check (balance >= 0)
);

insert into account(id, balance)
values (101, 1000.00), (202, 1000.00);
```

Constraint `balance >= 0` là defense-in-depth cho ví dụ, không thay thế
application validation hay một financial ledger hoàn chỉnh.
