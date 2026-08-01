# Experiments

## Test Setup
Chúng ta sử dụng `Testcontainers` (PostgreSQL) và `CountDownLatch` để mô phỏng chính xác race condition khi hai thread cùng request gửi thanh toán với chung 1 khóa idempotency.

```java
@SpringBootTest
@Testcontainers
class IdempotentPaymentServiceTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private IdempotentPaymentService paymentService;

    @Autowired
    private PaymentRecordRepository paymentRepository;

    @Autowired
    private IdempotencyKeyRepository keyRepository;

    @Test
    void testConcurrentPaymentCreationWithSameIdempotencyKey() throws InterruptedException {
        String idempotencyKey = "test-key-" + UUID.randomUUID();
        BigDecimal amount = new BigDecimal("100.00");
        String currency = "USD";

        int threadCount = 2;
        ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch endLatch = new CountDownLatch(threadCount);

        // Lưu kết quả của từng thread
        ResponseEntity<String>[] results = new ResponseEntity[threadCount];

        for (int i = 0; i < threadCount; i++) {
            final int index = i;
            executorService.submit(() -> {
                try {
                    startLatch.await(); // Đợi để cùng bắt đầu đồng thời
                    results[index] = paymentService.processPayment(idempotencyKey, amount, currency);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    endLatch.countDown();
                }
            });
        }

        startLatch.countDown(); // Kích hoạt tất cả thread cùng chạy
        endLatch.await();       // Chờ tất cả thread xử lý xong
        executorService.shutdown();

        // 1. Kiểm tra bất biến tầng Database (Chỉ có 1 payment record được tạo)
        long paymentCount = paymentRepository.countByIdempotencyKey(idempotencyKey);
        assertEquals(1, paymentCount, "Should only create exactly ONE payment record");

        // 2. Kiểm tra Response
        ResponseEntity<String> r1 = results[0];
        ResponseEntity<String> r2 = results[1];

        boolean hasSuccess = (r1.getStatusCodeValue() == 200) || (r2.getStatusCodeValue() == 200);
        boolean hasConflict = (r1.getStatusCodeValue() == 409) || (r2.getStatusCodeValue() == 409);

        // Một request phải thành công, một request phải báo conflict (in_progress)
        assertTrue(hasSuccess, "One request must succeed");
        assertTrue(hasConflict, "The other request must fail with 409 Conflict due to concurrent processing");
        
        // 3. Kiểm tra trạng thái Key
        IdempotencyKey key = keyRepository.findById(idempotencyKey).orElseThrow();
        assertEquals("COMPLETED", key.getStatus());
    }
}
```

## Observability Notes
Khi chạy test hoặc monitor trên production, ta sẽ quan sát thấy:
- **Exceptions**: Trong logs sẽ xuất hiện 1 `DataIntegrityViolationException` (tương ứng với PSQLException: duplicate key value violates unique constraint). Spring Boot thường log lỗi này ở mức WARN hoặc DEBUG tùy cấu hình. 
- Mặc dù có exception bắn ra, đây là *hành vi mong muốn* của luồng điều khiển, ứng dụng được thiết kế để catch và xử lý nó mượt mà.
- **Latency**: Nếu theo dõi API latency, request chiến thắng claim khóa sẽ có latency bằng thời gian gọi đối tác ngoại. Request thất bại (conflict) sẽ trả về cực nhanh (vài ms).
- **Metric khuyên dùng**: Gắn prometheus counter đếm số lượng "Idempotency Conflict Rejects" để đo lường tỷ lệ retry ảo trên hệ thống.
