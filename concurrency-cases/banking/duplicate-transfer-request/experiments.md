# Kiểm thử & Tái tạo lỗi

Phần này hướng dẫn cách viết Integration Test sử dụng Testcontainers (PostgreSQL) và `CountDownLatch` để chứng minh sự tồn tại của lỗi và xác nhận giải pháp đã khắc phục hoàn toàn.

## Test Setup

```java
@SpringBootTest
@Testcontainers
public class TransferServiceConcurrencyTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TransferService transferService;

    @Autowired
    private AccountRepository accountRepository;

    @Autowired
    private TransferRequestRepository requestRepository;

    @BeforeEach
    void setUp() {
        requestRepository.deleteAll();
        accountRepository.deleteAll();

        // Khởi tạo tài khoản
        Account acc1 = new Account("1001", new BigDecimal("1000000.0000"));
        Account acc2 = new Account("2002", new BigDecimal("0.0000"));
        accountRepository.saveAll(List.of(acc1, acc2));
    }

    @Test
    void testDuplicateTransferRequest_Concurrent() throws InterruptedException {
        String idempotencyKey = "retry-test-123";
        BigDecimal amount = new BigDecimal("500000.0000");

        int threadCount = 2;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch latch = new CountDownLatch(1);
        CountDownLatch doneLatch = new CountDownLatch(threadCount);
        
        List<String> responses = new CopyOnWriteArrayList<>();

        // Giả lập 2 client gửi request cùng lúc
        for (int i = 0; i < threadCount; i++) {
            executor.submit(() -> {
                try {
                    latch.await(); // Đợi cho đến khi tất cả thread sẵn sàng
                    TransferResponse response = transferService.executeTransfer(
                        idempotencyKey, "1001", "2002", amount
                    );
                    responses.add(response.getStatus());
                } catch (Exception e) {
                    responses.add("ERROR: " + e.getClass().getSimpleName());
                } finally {
                    doneLatch.countDown();
                }
            });
        }

        // Bắn súng lệnh để các thread chạy đồng thời
        latch.countDown();
        doneLatch.await(5, TimeUnit.SECONDS);

        // --- Kiểm tra Invariants (Bất biến) ---
        
        // 1. Chỉ 1 request được xử lý SUCCESS, các request khác báo ALREADY_PROCESSED hoặc ERROR
        long successCount = responses.stream().filter(r -> r.equals("SUCCESS")).count();
        assertEquals(1, successCount, "Chỉ được phép 1 request thành công");

        // 2. Số dư của người gửi phải trừ đúng 1 lần
        Account fromAcc = accountRepository.findById("1001").orElseThrow();
        assertEquals(0, new BigDecimal("500000.0000").compareTo(fromAcc.getBalance()), 
            "Số dư phải còn 500,000 VND");

        // 3. Số dư của người nhận phải cộng đúng 1 lần
        Account toAcc = accountRepository.findById("2002").orElseThrow();
        assertEquals(0, new BigDecimal("500000.0000").compareTo(toAcc.getBalance()), 
            "Số dư người nhận phải là 500,000 VND");

        // 4. Chỉ có 1 bản ghi lưu trong transfer_request
        long requestCount = requestRepository.count();
        assertEquals(1, requestCount, "Chỉ có 1 bản ghi transfer request");
        
        executor.shutdown();
    }
}
```

## Observability Notes
Khi chạy test với phiên bản `broken-code`, test sẽ fail tại assertion `assertEquals(1, successCount)` vì cả hai thread đều trả về `SUCCESS`. Đồng thời, số dư tài khoản `1001` sẽ bằng `0` thay vì `500000`, và có 2 record được insert vào database với cùng một `idempotency_key`.

Khi áp dụng giải pháp `Unique Constraint` và `saveAndFlush`, Log sẽ ghi nhận một lỗi `DataIntegrityViolationException` bị chặn lại và test sẽ pass do một trong hai thread nhận về kết quả `ALREADY_PROCESSED`. Điều này thỏa mãn yêu cầu `Exactly-Once Processing`.
