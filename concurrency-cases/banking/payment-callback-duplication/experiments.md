# Thử nghiệm (Experiments)

Chúng ta có thể kiểm chứng tình trạng lỗi và giải pháp bảo vệ tính lũy đẳng cũng như cập nhật trạng thái hợp lệ bằng bài kiểm tra tích hợp với Testcontainers.

## 1. Thiết lập Test Base

Sử dụng JUnit 5, Spring Boot Test và Testcontainers cho PostgreSQL.

```java
@SpringBootTest
@Testcontainers
class PaymentCallbackConcurrencyTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private PaymentCallbackService callbackService;

    @Autowired
    private PaymentRepository paymentRepository;

    @Autowired
    private PaymentEventRepository eventRepository;

    private UUID paymentId;

    @BeforeEach
    void setup() {
        eventRepository.deleteAll();
        paymentRepository.deleteAll();

        Payment payment = new Payment();
        payment.setId(UUID.randomUUID());
        payment.setPaymentCode("PAY-777");
        payment.setStatus("PENDING");
        payment.setAmount(new BigDecimal("1000.00"));
        paymentRepository.save(payment);
        
        paymentId = payment.getId();
    }
}
```

## 2. Kịch bản kiểm tra: Xử lý Concurrent Duplicate Callback

Kịch bản này mô phỏng PSP gửi 3 callback y hệt nhau cùng lúc. 

```java
@Test
void testDuplicateCallbacks_ShouldProcessOnlyOnce() throws InterruptedException {
    int threadCount = 3;
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch doneLatch = new CountDownLatch(threadCount);
    
    // Chuẩn bị 3 request hoàn toàn giống nhau (cùng provider_event_id)
    CallbackRequest request = new CallbackRequest(paymentId, "evt_01", "AUTHORIZED");

    for (int i = 0; i < threadCount; i++) {
        executorService.submit(() -> {
            try {
                startLatch.await(); // Đợi để chạy đồng thời
                callbackService.processCallback(request);
            } catch (Exception e) {
                // Ignore exception do Optimistic Locking hoặc Unique Constraint gây ra ở thread thua cuộc
            } finally {
                doneLatch.countDown();
            }
        });
    }

    startLatch.countDown(); // Kích hoạt tất cả threads
    doneLatch.await(5, TimeUnit.SECONDS);

    // Kiểm tra Bất biến (Invariant Assertions)
    Payment finalPayment = paymentRepository.findById(paymentId).orElseThrow();
    assertEquals("AUTHORIZED", finalPayment.getStatus(), "Trạng thái phải là AUTHORIZED");
    
    // Chỉ duy nhất 1 event được lưu thành công nhờ Unique Constraint
    long eventCount = eventRepository.count();
    assertEquals(1, eventCount, "Chỉ được lưu 1 event trong database");
}
```

## 3. Kịch bản kiểm tra: Xử lý Out-of-Order Callback

Kịch bản mô phỏng mạng bị trễ, request `CAPTURED` được hệ thống xử lý nhanh chóng thành công, sau đó request `AUTHORIZED` (thuộc chu trình bị trễ) mới tới.

```java
@Test
void testOutOfOrderCallbacks_ShouldPreventRegression() {
    // 1. Nhận được và xử lý CAPTURED (đến trước dù logic là đến sau)
    CallbackRequest reqCaptured = new CallbackRequest(paymentId, "evt_02", "CAPTURED");
    callbackService.processCallback(reqCaptured);
    
    // 2. Nhận được AUTHORIZED (đến sau do mạng trễ/retry)
    CallbackRequest reqAuthorized = new CallbackRequest(paymentId, "evt_01", "AUTHORIZED");
    callbackService.processCallback(reqAuthorized);
    
    // Kiểm tra Bất biến
    Payment finalPayment = paymentRepository.findById(paymentId).orElseThrow();
    
    // Status không được lùi về AUTHORIZED
    assertEquals("CAPTURED", finalPayment.getStatus(), "Status không được phép lùi (Regression)");
    
    // Cả 2 events đều được lưu log lại (vì là các sự kiện khác nhau có id khác nhau)
    long eventCount = eventRepository.count();
    assertEquals(2, eventCount, "Cả hai events phải được lưu vào log");
}
```

## 4. Ghi chú quan sát (Observability Notes)

- Nếu chạy mã cũ chưa được fix (từ `broken-code.md`), test duplicate callback sẽ sinh ra 3 bản ghi `payment_events` giống hệt nhau, và số lượng ghi log cập nhật payment sẽ chạy nhiều lần (gây lãng phí hiệu năng hoặc gọi API fulfillment nhiều lần). Test Out-of-Order sẽ thất bại do trạng thái cuối cùng rơi về `AUTHORIZED`.
- Nếu có cấu hình SLF4J logger, khi bắt được `DataIntegrityViolationException`, bạn sẽ thấy các dòng log `Duplicate callback ignored: evt_01`. Điều này giúp bộ phận vận hành an tâm rằng hệ thống đang chặn trùng lặp thành công.
- Lưu ý việc sử dụng `saveAndFlush(event)` bên trong `processCallback`: việc flush là bắt buộc để Hibernate gửi ngay `INSERT` xuống DB và văng lỗi Unique Constraint. Nếu không `flush`, Hibernate có thể đẩy lỗi ra ngoài hàm, khiến `@Transactional` rollback mà chúng ta không thể try-catch và khôi phục (swallow exception) để phản hồi HTTP 200 OK cho Provider.
