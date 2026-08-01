# Experiments: Concurrent Settlement and Reconciliation Workers

## 1. Test Setup

Chúng ta sẽ sử dụng Testcontainers (PostgreSQL) và `CountDownLatch` để giả lập 3 worker chạy đồng thời tranh nhau lấy việc (claim work).

### Prerequisites
- Spring Boot Starter Data JPA
- Testcontainers PostgreSQL
- JUnit 5

## 2. Test Execution Code

```java
@SpringBootTest
@Testcontainers
class ReconciliationWorkerConcurrencyTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private SecureReconciliationService reconciliationService;

    @Autowired
    private SettlementLineRepository settlementRepo;

    @Autowired
    private AccountBalanceRepository accountRepo;

    @Autowired
    private LedgerEntryRepository ledgerRepo;

    @BeforeEach
    void setUp() {
        ledgerRepo.deleteAll();
        settlementRepo.deleteAll();
        accountRepo.deleteAll();

        accountRepo.save(new AccountBalance("SYSTEM_ACCOUNT", BigDecimal.ZERO));

        // Insert 100 pending settlement lines
        for (int i = 1; i <= 100; i++) {
            SettlementLine line = new SettlementLine();
            line.setId("S-" + i);
            line.setPartnerId("PARTNER_A");
            line.setAmount(new BigDecimal("100.00")); // Each discrepancy adds 100
            line.setStatus("PENDING");
            settlementRepo.save(line);
        }
    }

    @Test
    void testConcurrentWorkersClaimingAndReconciling() throws InterruptedException {
        int numberOfWorkers = 3;
        ExecutorService executor = Executors.newFixedThreadPool(numberOfWorkers);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch completionLatch = new CountDownLatch(numberOfWorkers);

        // Kích hoạt 3 worker chạy liên tục cho đến khi hết việc
        for (int i = 0; i < numberOfWorkers; i++) {
            executor.submit(() -> {
                try {
                    startLatch.await(); // Đợi để cùng chạy
                    // Worker loop
                    boolean hasMoreWork = true;
                    while (hasMoreWork) {
                        try {
                            // Hàm này sẽ gọi processPendingSettlementsSecurely
                            reconciliationService.processPendingSettlementsSecurely();
                            
                            // Check xem còn việc không (giả lập)
                            long remaining = settlementRepo.countByStatus("PENDING");
                            if (remaining == 0) {
                                hasMoreWork = false;
                            }
                        } catch (Exception e) {
                            // Bỏ qua lỗi vòng lặp
                        }
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    completionLatch.countDown();
                }
            });
        }

        // Bắt đầu!
        startLatch.countDown();
        completionLatch.await(30, TimeUnit.SECONDS);
        executor.shutdown();

        // 3. Invariant Assertions

        // 1. All 100 lines should be processed
        long pendingCount = settlementRepo.countByStatus("PENDING");
        assertEquals(0, pendingCount, "No pending lines should remain");

        // 2. Exactly 100 ledger entries created (idempotency check)
        long ledgerCount = ledgerRepo.count();
        assertEquals(100, ledgerCount, "Exactly 100 ledger entries should be created");

        // 3. Balance must be exactly 100 * 100 = 10,000
        AccountBalance systemAccount = accountRepo.findById("SYSTEM_ACCOUNT").orElseThrow();
        assertEquals(0, new BigDecimal("10000.00").compareTo(systemAccount.getBalance()),
                "System balance must be exactly 10,000");
    }
}
```

## 3. Observability Notes
Khi chạy test này trên console hoặc xem log của Hibernate:
- Bạn sẽ thấy câu lệnh SQL `SELECT ... FOR UPDATE SKIP LOCKED` được in ra.
- Dù có 3 luồng tranh nhau, PostgreSQL sẽ âm thầm trả về tập con (subset) kết quả cho mỗi luồng mà không throw `LockAcquisitionException`, giúp hiệu năng đạt tối đa.
- Không có lỗi `DataIntegrityViolationException` nào được throw (vì các worker không hề xử lý trùng dòng). 

Nếu muốn quan sát lỗi *Double Adjustment*, bạn có thể tạo một bài Test tương tự nhưng gọi tới class chứa `ANTI-PATTERN` từ file `broken-code.md`. Khi đó:
- `ledgerCount` có thể lên tới 150-200.
- `systemAccount.getBalance()` sẽ lên tới 15,000 - 20,000 (Sai lệch nghiêm trọng).
