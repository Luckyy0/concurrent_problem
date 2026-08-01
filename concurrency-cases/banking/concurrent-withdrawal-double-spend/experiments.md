# Experiments

Đoạn code kiểm chứng sử dụng `Testcontainers` (PostgreSQL) và `CountDownLatch` để mô phỏng tương tác đồng thời. Test sẽ chứng minh giải pháp đã ngăn chặn được double spending.

## 1. Test Code (JUnit 5 + Spring Boot)

```java
package com.example.banking.concurrency;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.math.BigDecimal;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringBootTest
class WithdrawalConcurrencyTest {

    @Autowired
    private AccountService accountService;

    @Autowired
    private AccountRepository accountRepository;

    @BeforeEach
    void setUp() {
        accountRepository.deleteAll();
        Account acc = new Account();
        acc.setId(1001L);
        acc.setAvailableBalance(new BigDecimal("1000000"));
        accountRepository.save(acc);
    }

    @Test
    void testConcurrentWithdrawals_WithConditionalUpdate() throws InterruptedException {
        int threads = 2;
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        CountDownLatch latch = new CountDownLatch(1);
        CountDownLatch done = new CountDownLatch(threads);

        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger failCount = new AtomicInteger(0);

        for (int i = 0; i < threads; i++) {
            executor.submit(() -> {
                try {
                    // Đợi tất cả thread sẵn sàng tại vạch xuất phát
                    latch.await();
                    
                    // Thực thi rút tiền đồng thời.
                    // Sử dụng phương thức withdrawAtomic() hoặc withdrawPessimistic()
                    accountService.withdrawAtomic(1001L, new BigDecimal("800000"));
                    successCount.incrementAndGet();
                } catch (Exception e) {
                    failCount.incrementAndGet();
                } finally {
                    done.countDown();
                }
            });
        }

        // Bắt đầu đồng thời
        latch.countDown();
        // Đợi các threads chạy xong (timeout 5s)
        done.await(5, TimeUnit.SECONDS);
        
        executor.shutdown();

        // Verify Invariant
        Account account = accountRepository.findById(1001L).orElseThrow();
        
        System.out.println("Final Balance: " + account.getAvailableBalance());
        System.out.println("Success Tx: " + successCount.get());
        System.out.println("Failed Tx: " + failCount.get());

        // Assertions
        assertEquals(new BigDecimal("200000.0000"), account.getAvailableBalance());
        assertEquals(1, successCount.get(), "Chỉ 1 giao dịch được thành công");
        assertEquals(1, failCount.get(), "1 giao dịch phải bị từ chối");
    }
}
```

## 2. Observability & DB Logs
Để quan sát cách PostgreSQL xử lý lock, ta có thể bật cấu hình log in ra câu lệnh SQL:
Trong `application.properties`:
```properties
spring.jpa.show-sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### Nếu sử dụng giải pháp **Pessimistic Lock**:
Ta sẽ thấy log in ra hai câu lệnh:
`SELECT ... FROM accounts WHERE id = ? FOR UPDATE`
Thread thứ hai sẽ bị block và delay tại câu lệnh `SELECT FOR UPDATE` này, chứng tỏ nó đang chờ lock từ Database (nhờ cơ chế Row Exclusive Lock). Sau khi thread 1 thực hiện commit, câu lệnh log của thread 2 mới được tiếp tục in ra màn hình hoặc báo exception.

### Nếu sử dụng giải pháp **Conditional Update**:
Ta sẽ thấy hai lệnh UPDATE tương tự nhau được gọi liên tiếp. Nhưng trong đó chỉ một lệnh thực sự thay đổi dữ liệu (affected rows = 1), lệnh thứ hai sẽ bị block chờ, và sau khi tiếp tục thì không thay đổi bản ghi nào do không thoả mãn mệnh đề `WHERE` trên state mới nhất (affected rows = 0).
