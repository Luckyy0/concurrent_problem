# Mô phỏng kiểm thử (Experiments)

Chúng ta có thể dùng Testcontainers cùng JUnit và Spring Boot để tái hiện lại lỗi Lost Update và chứng minh tính đúng đắn của giải pháp.

## 1. Setup Testcontainers

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@SpringBootTest
@Testcontainers
public class AccountConcurrencyTest {

    @Container
    public static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private AccountService accountService;
    
    @Autowired
    private AccountRepository accountRepository;
}
```

## 2. Test Case Synchronized by CountDownLatch

Sử dụng `CountDownLatch` để buộc hai luồng cùng chạy vào thời điểm tranh chấp.

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;
import java.math.BigDecimal;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

@Test
public void testLostUpdate() throws InterruptedException {
    // 1. Prepare data
    Account account = new Account();
    account.setAccountNumber("2002");
    account.setBalance(new BigDecimal("5000000.0000"));
    accountRepository.save(account);
    Long accId = account.getId();

    // 2. Setup concurrency tools
    int threadCount = 2;
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(threadCount);
    ExecutorService executor = Executors.newFixedThreadPool(threadCount);

    // Thread 1: Credit +2,000,000
    executor.submit(() -> {
        try {
            startLatch.await(); // Wait for the signal
            accountService.updateBalance(accId, new BigDecimal("2000000.0000"));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            doneLatch.countDown();
        }
    });

    // Thread 2: Debit -500,000
    executor.submit(() -> {
        try {
            startLatch.await(); // Wait for the signal
            accountService.updateBalance(accId, new BigDecimal("-500000.0000"));
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            doneLatch.countDown();
        }
    });

    // 3. Fire threads and wait
    startLatch.countDown();
    doneLatch.await(10, TimeUnit.SECONDS);

    // 4. Assert invariant
    Account finalAccount = accountRepository.findById(accId).orElseThrow();
    
    // Expected: 5,000,000 + 2,000,000 - 500,000 = 6,500,000
    // Nếu dùng code lỗi (Broken code), assertion này sẽ FAIL (trả về 4,500,000 hoặc 7,000,000).
    // Nếu áp dụng Giải pháp (Solution 1), assertion này sẽ PASS.
    assertEquals(0, new BigDecimal("6500000.0000").compareTo(finalAccount.getBalance()));
}
```

## 3. Observability Notes
- Trong quá trình chạy test nếu áp dụng **Solution 2 (Optimistic Locking)**, bạn sẽ thấy `ObjectOptimisticLockingFailureException` được ném ra ở thread chạy chậm hơn. Bạn phải viết logic handle/retry ở lớp service (tạo một transaction mới thông qua `@Retryable`) để test thực sự pass.
- Với **Solution 1 (Atomic Delta)**, sẽ thấy trong log SQL của Hibernate câu lệnh dạng `UPDATE accounts SET balance = balance + ? WHERE id = ?`. Không có exception nào, thread 2 tự động chờ (lock wait) sau đó được xử lý đúng đắn bởi PostgreSQL MVCC, số dư luôn chuẩn xác.
