# Concurrency Experiments

Thử nghiệm này sử dụng Testcontainers và Spring Boot `CountDownLatch` để chứng minh hiện tượng mất mát dữ liệu và sự hiệu quả của giải pháp Atomic Delta.

## Test Setup

```java
@SpringBootTest
@Testcontainers
class LedgerPostingServiceConcurrencyTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private LedgerPostingService ledgerPostingService;

    @Autowired
    private BalanceProjectionRepository balanceRepository;

    @Autowired
    private LedgerEntryRepository ledgerEntryRepository;

    private static final String ACCOUNT_ID = "3003";

    @BeforeEach
    void setup() {
        ledgerEntryRepository.deleteAll();
        balanceRepository.deleteAll();

        // Khởi tạo tài khoản với số dư 5M
        BalanceProjection balance = new BalanceProjection();
        balance.setAccountId(ACCOUNT_ID);
        balance.setProjectedBalance(new BigDecimal("5000000.0000"));
        balanceRepository.save(balance);
        
        ledgerPostingService.postTransaction(ACCOUNT_ID, new BigDecimal("5000000.0000"), "INIT_001");
    }

    @Test
    void testConcurrentLedgerPosting_AtomicDelta() throws InterruptedException {
        int threadCount = 100;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch readyLatch = new CountDownLatch(threadCount);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch doneLatch = new CountDownLatch(threadCount);

        for (int i = 0; i < threadCount; i++) {
            final int index = i;
            executor.submit(() -> {
                readyLatch.countDown();
                try {
                    startLatch.await(); // Đợi tất cả thread sẵn sàng
                    // Nạp 10,000 VND x 50 threads, Rút 5,000 VND x 50 threads
                    BigDecimal amount = (index % 2 == 0) ? new BigDecimal("10000.0000") : new BigDecimal("-5000.0000");
                    String refId = "TX_CONC_" + index;
                    ledgerPostingService.postTransaction(ACCOUNT_ID, amount, refId);
                } catch (Exception e) {
                    // Ignore exceptions for test simplicity (e.g., duplicate ref)
                } finally {
                    doneLatch.countDown();
                }
            });
        }

        readyLatch.await();
        startLatch.countDown(); // Kích hoạt đồng loạt
        doneLatch.await(); // Chờ hoàn tất

        // Kiểm tra Invariant
        BigDecimal expectedBalance = new BigDecimal("5000000.0000")
                .add(new BigDecimal("10000.0000").multiply(new BigDecimal(50)))
                .subtract(new BigDecimal("5000.0000").multiply(new BigDecimal(50))); // = 5M + 500K - 250K = 5,250,000

        BalanceProjection finalBalance = balanceRepository.findById(ACCOUNT_ID).orElseThrow();
        
        BigDecimal sumLedger = ledgerEntryRepository.findAll().stream()
                .map(LedgerEntry::getAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

        // Assertion
        assertThat(sumLedger).isEqualByComparingTo(expectedBalance);
        assertThat(finalBalance.getProjectedBalance()).isEqualByComparingTo(sumLedger);
        System.out.println("Final Projected Balance: " + finalBalance.getProjectedBalance());
        System.out.println("Ledger Sum: " + sumLedger);
        
        executor.shutdown();
    }
}
```

## Observability Notes
1. Khi chạy test với phiên bản `broken-code`, test sẽ fail lập tức vì `finalBalance.getProjectedBalance()` khác xa với `sumLedger`.
2. Mở query log PostgreSQL (bằng cách set `spring.jpa.show-sql=true`), với cách Atomic Delta, chúng ta sẽ thấy rất nhiều câu `UPDATE balance_projection SET projected_balance = projected_balance + ? ...`.
3. Số lượng query `INSERT INTO ledger_entry` luôn luôn là 101 bản ghi (1 INIT + 100 giao dịch), bất chấp `projection` có cập nhật thành công hay không.
