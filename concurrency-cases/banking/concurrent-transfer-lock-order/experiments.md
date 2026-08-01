# BANK-003: Thực nghiệm (Experiments)

## Chuẩn bị Testcontainers
Cấu hình PostgreSQL qua `Testcontainers` để tạo môi trường cơ sở dữ liệu thật, giúp tái hiện chính xác hành vi `lock` và `deadlock`.

```java
@SpringBootTest
@Testcontainers
class TransferServiceTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "update");
    }

    @Autowired
    private TransferService transferService;

    @Autowired
    private AccountRepository accountRepository;

    // ...
}
```

## Kịch bản kiểm thử (Test Case): Gây ra Deadlock hoặc chạy mượt mà

Chúng ta sử dụng `CountDownLatch` để đồng bộ hóa 2 luồng sao cho chúng cùng truy xuất dữ liệu trong cùng một thời điểm.

```java
@Test
void concurrentTransfer_ShouldNotDeadlock() throws InterruptedException {
    // 1. Khởi tạo dữ liệu
    Account a = new Account();
    a.setBalance(new BigDecimal("2000000"));
    a = accountRepository.save(a); // ID = 1

    Account b = new Account();
    b.setBalance(new BigDecimal("2000000"));
    b = accountRepository.save(b); // ID = 2

    Long accountAId = a.getId();
    Long accountBId = b.getId();

    // 2. Chuẩn bị đồng bộ hóa
    CountDownLatch startLatch = new CountDownLatch(1);
    CountDownLatch endLatch = new CountDownLatch(2);
    
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger failCount = new AtomicInteger(0);

    // 3. Thread 1: Chuyển từ A sang B (1,000,000)
    Thread t1 = new Thread(() -> {
        try {
            startLatch.await(); // Đợi hiệu lệnh bắt đầu
            transferService.transfer(accountAId, accountBId, new BigDecimal("1000000"));
            successCount.incrementAndGet();
        } catch (Exception e) {
            System.err.println("Thread 1 failed: " + e.getMessage());
            failCount.incrementAndGet();
        } finally {
            endLatch.countDown();
        }
    });

    // 4. Thread 2: Chuyển từ B sang A (500,000)
    Thread t2 = new Thread(() -> {
        try {
            startLatch.await(); // Đợi hiệu lệnh bắt đầu cùng lúc
            transferService.transfer(accountBId, accountAId, new BigDecimal("500000"));
            successCount.incrementAndGet();
        } catch (Exception e) {
            System.err.println("Thread 2 failed: " + e.getMessage());
            failCount.incrementAndGet();
        } finally {
            endLatch.countDown();
        }
    });

    t1.start();
    t2.start();

    // Phát hiệu lệnh bắt đầu
    startLatch.countDown();

    // Đợi 2 thread hoàn thành (có timeout để tránh treo test vô hạn)
    boolean completed = endLatch.await(5, TimeUnit.SECONDS);
    assertTrue(completed, "Các thread bị kẹt vô hạn!");

    // 5. Kiểm tra kết quả (Assertions)
    
    // Nếu dùng code đã fix, không giao dịch nào được phép lỗi vì deadlock
    assertEquals(2, successCount.get(), "Phải có 2 giao dịch thành công");
    assertEquals(0, failCount.get(), "Không có giao dịch nào bị Deadlock");

    // Xác minh bất biến số dư (Conservation of money)
    Account finalA = accountRepository.findById(accountAId).orElseThrow();
    Account finalB = accountRepository.findById(accountBId).orElseThrow();

    // A: 2M - 1M + 0.5M = 1.5M
    // B: 2M + 1M - 0.5M = 2.5M
    assertEquals(0, new BigDecimal("1500000").compareTo(finalA.getBalance()));
    assertEquals(0, new BigDecimal("2500000").compareTo(finalB.getBalance()));
}
```

## Ghi chú về khả năng quan sát (Observability Notes)
- Khi test chạy với đoạn code **chưa được sửa (Broken Code)**, trong log console (được cấu hình với `logging.level.org.hibernate.SQL=DEBUG`), ta sẽ thấy 2 lệnh `SELECT ... FOR UPDATE` đan xen nhau. 
- Sau khoảng 1 giây (hoặc cấu hình `deadlock_timeout` của PG), Console sẽ in ra exception `org.springframework.dao.CannotAcquireLockException` (do Hibernate throw). `failCount` sẽ đếm lên 1.
- Khi test chạy với đoạn code **đã sửa (Solutions)**, thứ tự log SQL sẽ cho thấy luồng nào đến sau sẽ phải chờ (blocking) thay vì bị exception. Sau đó khi luồng trước hoàn tất (commit) và giải phóng lock, luồng sau mới tiếp tục lấy `ResultSet` và hoàn thành update. Thời gian chạy tổng cộng sẽ xấp xỉ bằng tổng thời gian của 2 `transaction` tuần tự.
