# Experiments (Kiểm thử đồng thời)

Sử dụng Testcontainers (PostgreSQL) và `CountDownLatch` để mô phỏng chính xác race condition trên môi trường kiểm thử.

## Cấu trúc Test Class

```java
@SpringBootTest
@Testcontainers
public class SettlementReversalRaceTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private AuthorizationService service; // Giả sử tiêm service tương ứng

    @Autowired
    private AccountRepository accountRepository;
    
    @Autowired
    private AuthorizationRepository authorizationRepository;

    @BeforeEach
    public void setup() {
        authorizationRepository.deleteAll();
        accountRepository.deleteAll();

        Account acc = new Account();
        acc.setId("ACC-001");
        acc.setLedgerBalance(new BigDecimal("5000000"));
        acc.setHoldBalance(new BigDecimal("1000000"));
        accountRepository.save(acc);

        Authorization auth = new Authorization();
        auth.setId("AUTH-555");
        auth.setAccountId("ACC-001");
        auth.setAmount(new BigDecimal("1000000"));
        auth.setStatus("AUTHORIZED");
        auth.setExpiresAt(LocalDateTime.now().minusMinutes(5)); // Cố tình cho hết hạn
        authorizationRepository.save(auth);
    }

    @Test
    public void testConcurrentCaptureReverseExpire() throws InterruptedException {
        int threads = 3;
        ExecutorService executor = Executors.newFixedThreadPool(threads);
        CountDownLatch readyLatch = new CountDownLatch(threads);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch doneLatch = new CountDownLatch(threads);

        // Task 1: Capture
        executor.submit(() -> {
            readyLatch.countDown();
            try {
                startLatch.await();
                service.capture("AUTH-555"); // hoặc sử dụng try-catch tùy cách thiết kế lỗi
            } catch (Exception e) {
                System.out.println("Capture failed: " + e.getMessage());
            } finally {
                doneLatch.countDown();
            }
        });

        // Task 2: Reverse
        executor.submit(() -> {
            readyLatch.countDown();
            try {
                startLatch.await();
                service.reverse("AUTH-555");
            } catch (Exception e) {
                System.out.println("Reverse failed: " + e.getMessage());
            } finally {
                doneLatch.countDown();
            }
        });

        // Task 3: Expire
        executor.submit(() -> {
            readyLatch.countDown();
            try {
                startLatch.await();
                service.expire("AUTH-555");
            } catch (Exception e) {
                System.out.println("Expire failed: " + e.getMessage());
            } finally {
                doneLatch.countDown();
            }
        });

        readyLatch.await(); // Đợi cả 3 thread sẵn sàng
        startLatch.countDown(); // Kích hoạt chạy đồng thời!
        doneLatch.await(5, TimeUnit.SECONDS); // Đợi tất cả xong

        // Assertions (Kiểm tra Invariant)
        Authorization finalAuth = authorizationRepository.findById("AUTH-555").orElseThrow();
        Account finalAcc = accountRepository.findById("ACC-001").orElseThrow();

        // 1. Trạng thái không thể là AUTHORIZED nữa, phải là 1 trong 3 trạng thái kết thúc
        Assertions.assertTrue(List.of("CAPTURED", "REVERSED", "EXPIRED").contains(finalAuth.getStatus()));

        // 2. Toàn vẹn số dư (Hold balance phải luôn về 0)
        Assertions.assertEquals(0, finalAcc.getHoldBalance().compareTo(BigDecimal.ZERO), 
            "Hold balance must be exactly 0, not negative or positive");

        // 3. Ledger Equation
        if ("CAPTURED".equals(finalAuth.getStatus())) {
            Assertions.assertEquals(0, finalAcc.getLedgerBalance().compareTo(new BigDecimal("4000000")), 
                "Ledger must be deducted if captured");
        } else {
            Assertions.assertEquals(0, finalAcc.getLedgerBalance().compareTo(new BigDecimal("5000000")), 
                "Ledger must remain unchanged if reversed or expired");
        }
        
        executor.shutdown();
    }
}
```

## Observability Notes (Ghi chú về khả năng quan sát)

Khi chạy test này trên mã **Broken Code** (không lock):
- Test sẽ `FAIL` tại Assertion số 2: `Hold balance must be exactly 0`. Thực tế log in ra `-2000000` hoặc `-1000000` tùy vào thứ tự các thread commit.
- State cuối cùng của `Authorization` sẽ ngẫu nhiên, không phản ánh chính xác nghiệp vụ nào đã thực thi logic trừ tiền.

Khi chạy test này trên mã áp dụng **Solution 3 (Conditional Update)**:
- Test sẽ `PASS`. 
- Trong console sẽ hiển thị exception do ứng dụng chủ động ném ra (`State transition failed...`) của ít nhất 2 luồng đến muộn. Chỉ 1 luồng duy nhất cập nhật thành công và hoàn tất giao dịch tài chính.
- `hold_balance` luôn là `0`.
