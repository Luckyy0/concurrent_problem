# Kịch bản kiểm thử (Experiments)

Chúng ta sử dụng Spring Boot Test, Testcontainers (PostgreSQL) và `CountDownLatch` để tạo ra điều kiện race condition kiểm chứng cả hai phiên bản code: lỗi và sau khi sửa chữa.

## Cấu trúc Test Class

```java
@SpringBootTest
@Testcontainers
class AuthorizationServiceConcurrencyTest {

    @Autowired
    private AuthorizationService authorizationService;

    @Autowired
    private AccountRepository accountRepository;

    @Autowired
    private AuthorizationHoldRepository holdRepository;

    @Test
    void testConcurrentAuthorization_Broken() throws InterruptedException {
        // 1. Setup dữ liệu
        Account acc = new Account();
        acc.setId(4004L);
        acc.setAccountNo("1234567890");
        acc.setPostedBalance(new BigDecimal("2000000.00"));
        accountRepository.save(acc);

        // 2. Chuẩn bị threads
        int threadCount = 2;
        ExecutorService executor = Executors.newFixedThreadPool(threadCount);
        CountDownLatch startLatch = new CountDownLatch(1);
        CountDownLatch endLatch = new CountDownLatch(threadCount);

        // Task A: 1,500,000
        executor.submit(() -> {
            try {
                startLatch.await();
                authorizationService.authorizeTransaction(4004L, new BigDecimal("1500000.00"));
            } catch (Exception e) {
                System.out.println("Tx A Error: " + e.getMessage());
            } finally {
                endLatch.countDown();
            }
        });

        // Task B: 1,200,000
        executor.submit(() -> {
            try {
                startLatch.await();
                authorizationService.authorizeTransaction(4004L, new BigDecimal("1200000.00"));
            } catch (Exception e) {
                System.out.println("Tx B Error: " + e.getMessage());
            } finally {
                endLatch.countDown();
            }
        });

        // 3. Kick-off concurrency
        startLatch.countDown();
        endLatch.await();

        // 4. Kiểm tra
        List<AuthorizationHold> holds = holdRepository.findAll();
        BigDecimal totalHeld = holds.stream()
            .map(AuthorizationHold::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        // Ở bản code bị lỗi, tổng hold = 2,700,000
        // Assertion này sẽ PASS nếu code bị lỗi thực sự sinh ra thấu chi.
        Assertions.assertEquals(2, holds.size(), "Both holds were created");
        Assertions.assertEquals(new BigDecimal("2700000.00"), totalHeld);
    }
}
```

## Quan sát kết quả (Observability Notes)

- **Khi chạy với đoạn code lỗi**: Bài test trên sẽ pass hoàn toàn (nghĩa là 2 giao dịch đều thành công), cho thấy tổng số tiền giữ (`totalHeld`) bằng 2,700,000, vi phạm quy tắc `totalHeld <= posted_balance` (2,000,000). Điều này tái hiện thành công sự cố.
- **Sau khi áp dụng Cách 1 (Pessimistic Lock)**:
  - Một trong hai thread (vd Tx B) sẽ bị chặn lại ở truy vấn `SELECT ... FOR UPDATE` khi log SQL của Hibernate được in ra.
  - Khi Tx A hoàn thành, Tx B tiếp tục, nhưng nó đọc được có 1,500,000 đã được hold. Nó cố gắng tính `available_balance = 2000000 - 1500000 = 500000`.
  - Lúc này `500000 < 1200000`, vì thế Tx B ném ra `InsufficientFundsException`.
  - Kết quả: Bài test sẽ fail vì tổng số holds không còn là 2,700,000 mà là 1,500,000, và chỉ có 1 hold được lưu thành công. (Khi đó ta đổi Assertion của test để verify tính đúng đắn).

### Debugging Logs (Cấu hình)
Thêm cấu hình sau vào `application-test.yml` để theo dõi transaction và các locks:
```yaml
logging:
  level:
    org.springframework.orm.jpa.JpaTransactionManager: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```
Bạn sẽ quan sát được thứ tự lock, block thread và các exception liên quan bị bắn ra ở console.
