# Các phản mẫu thiết kế (Anti-patterns) — Thử lại vô hạn và quản lý transaction sai lầm

## 1. Khai báo thực thể có quản lý phiên bản

Cấu trúc ban đầu của ví và lịch sử thao tác:

```java
@Entity
@Table(name = "reward_wallet")
public class RewardWallet {
    @Id
    private Long walletId;

    @Column(nullable = false)
    private long points;

    // Chốt chặn của cơ chế khóa lạc quan
    @Version
    private long version;

    protected RewardWallet() {
    }

    public void credit(long delta) {
        if (delta <= 0) {
            throw new IllegalArgumentException("Số điểm cộng thêm phải lớn hơn 0");
        }
        points = Math.addExact(points, delta);
    }
}
```

```sql
create table reward_credit (
    command_id uuid primary key,
    wallet_id bigint not null,
    points bigint not null check (points > 0)
);
```

---

## 2. Phản mẫu 1 — Tích hợp vòng lặp bên trong `@Transactional`

```java
@Transactional
public CreditResult credit(CreditCommand command) {
    for (;;) { // Vòng lặp vô hạn
        try {
            RewardWallet wallet = wallets.findById(command.walletId())
                    .orElseThrow();
            wallet.credit(command.points());
            credits.save(RewardCredit.from(command));
            wallets.flush();
            return CreditResult.applied(command.commandId());
        } catch (ObjectOptimisticLockingFailureException conflict) {
            // Nỗ lực vô ích: clear() không thể phục hồi transaction đã hỏng
            entityManager.clear();
        }
    }
}
```

**Lý do lỗi:**
Khi một ngoại lệ khóa lạc quan được phát sinh, transaction hiện tại sẽ bị Spring đánh dấu bắt buộc rollback. Phương thức `entityManager.clear()` chỉ có thể xóa các thực thể khỏi bộ đệm, nhưng không thể khôi phục lại trạng thái hợp lệ của transaction, cũng không tạo ra được trạng thái connection mới. 
Vòng lặp sẽ tiếp tục hoạt động trên một transaction đã hỏng, và trong lần xử lý kế tiếp, Spring sẽ ném ra lỗi liên quan tới transaction không thể phục hồi.

> **Khuyến cáo:** Không đưa vòng lặp xử lý lại vào bên trong một ngữ cảnh transaction. Để có các nỗ lực thử lại độc lập, ngữ cảnh transaction (`@Transactional`) phải được gọi **TỪ BÊN TRONG** một vòng lặp quản lý tổng thể.

---

## 3. Phản mẫu 2 — Khởi tạo transaction mới nhưng không có độ trễ tạm dừng

```java
public CreditResult credit(CreditCommand command) {
    while (true) {
        try {
            // Hàm creditOnce đã được tách với REQUIRES_NEW
            return attempts.creditOnce(command);
        } catch (ObjectOptimisticLockingFailureException conflict) {
            // BỎ QUA: Thiếu các cơ chế kiểm soát số lần lặp và thời gian trì hoãn.
        }
    }
}
```

**Lý do lỗi:**
Dù ranh giới transaction đã được thiết kế đúng (sử dụng `REQUIRES_NEW`), các yêu cầu thất bại sẽ ngay lập tức được gửi lại database. Điều này dẫn đến sự tái tạo xung đột tại cùng một thời điểm và tạo ra vòng lặp vô hạn. Nếu có vấn đề gây nghẽn, các thread này sẽ cày nát hệ thống connection và gây lãng phí toàn bộ tài nguyên.

---

## 4. Phản mẫu 3 — Sử dụng `@Retryable` không chọn lọc

```java
// Bắt mọi loại RuntimeException để thử lại
@Retryable(retryFor = RuntimeException.class)
public CreditResult credit(CreditCommand command) {
    return attempts.creditOnce(command);
}
```

**Lý do lỗi:**
Cơ chế thử lại được áp dụng sai cho toàn bộ lỗi, bao gồm cả lỗi xác thực dữ liệu, lỗi ánh xạ, hay lỗi gián đoạn hệ thống. Việc thử lại chỉ nên được thực hiện đối với các lỗi liên quan tới quản lý tài nguyên đồng thời như `ObjectOptimisticLockingFailureException`. Bỏ qua phân loại lỗi dẫn tới lãng phí tài nguyên và làm hệ thống phớt lờ các ngoại lệ nghiệp vụ nghiêm trọng.

---

## 5. Phản mẫu 4 — Tự động tạo mới định danh (Command ID) trong quá trình thử lại

```java
// Sinh mã ID mới thay vì giữ mã gốc
CreditCommand next = new CreditCommand(
        UUID.randomUUID(),
        original.walletId(),
        original.points()
);
```

**Lý do lỗi:**
Mã lệnh (`command ID`) được thiết kế để đảm bảo tính duy nhất và tính lũy đẳng. Nếu transaction đầu tiên commit thành công trên database nhưng kết nối phản hồi tới máy chủ ứng dụng bị gián đoạn, thao tác tái sinh Command ID sẽ vô hiệu hóa cơ chế chống trùng lặp, dẫn tới tình trạng cộng điểm hai lần.

---

## 6. Phản mẫu 5 — Kích hoạt độ trễ tạm dừng bên trong transaction

```java
@Transactional
public void creditOnce(...) {
    // Xử lý logic...
    // Kích hoạt tạm dừng thread trực tiếp
    retryBackoff.pause(...); // Hành vi giữ chặn connection
}
```

**Lý do lỗi:**
Khoảng thời gian tạm dừng chỉ được thực hiện SAU KHI transaction đã kết thúc (commit hoặc rollback). Nếu hệ thống gọi độ trễ bên trong transaction, thread sẽ chiếm giữ connection của database, nhanh chóng dẫn đến cạn kiệt connection pool.

---

## 7. Phản mẫu 6 — Tái sử dụng thực thể từ bộ đệm cũ

```java
// Truy xuất dữ liệu ví chỉ một lần duy nhất trước vòng lặp
RewardWallet stale = wallets.findById(id).orElseThrow();

for (...) {
    transactionTemplate.execute(status -> {
        stale.credit(delta);
        return wallets.save(stale); // Truyền thực thể cũ vào transaction mới
    });
}
```

**Lý do lỗi:**
Thực thể `stale` đã bị trói buộc với phiên bản và ngữ cảnh bộ đệm cũ. Transaction mới cần phải thực hiện nạp lại thực thể trực tiếp từ database để có thông hiện trạng mới nhất, thay vì cố gắng thao tác lại trên trạng thái không hợp lệ.

---

## 8. Phản mẫu 7 — Quản lý số lần thử nhưng bỏ qua thời gian tối đa

```java
for (int attempt = 1; attempt <= 10; attempt++) {
    backoff.pause(attempt);
}
```

**Lý do lỗi:**
Mặc dù 10 lần thử nghe có vẻ an toàn, nếu độ trễ ở database tăng hoặc thời gian chờ cộng dồn quá lâu, tổng thời gian xử lý sẽ vượt xa cấu hình timeout của yêu cầu API. Hệ thống phải đảm bảo theo dõi thời hạn của toàn bộ tiến trình thay vì chỉ đếm số vòng lặp thuần túy.

---

## 9. Phản mẫu 8 — Cố gắng thử lại khi ngoại lệ nghiệp vụ phát sinh

Khi cấu hình nghiệp vụ của tài khoản bị hạn chế hoặc không đáp ứng điều kiện, việc thử lại sẽ không thể thay đổi được trạng thái bị từ chối từ database. Thử lại ở đây là lãng phí tài nguyên máy tính; chỉ các xung đột tài nguyên nhất thời mới nên được hỗ trợ phục hồi.

---

## 10. Phản mẫu 9 — Thực hiện các thao tác ngoại vi trước khi commit transaction

```java
wallet.credit(points);
// Cảnh báo: Gọi API ngoại vi TRƯỚC KHI trạng thái database xác nhận
notificationClient.send(command.commandId());
wallets.flush();
```

**Lý do lỗi:**
Nếu lệnh `flush()` thất bại hoặc transaction bị rollback, hệ thống ngoại vi đã xử lý xong thông tin không chính xác. Các sự kiện ngoại vi bắt buộc phải được xử lý thông qua cơ chế hộp thư trung gian sau khi chắc chắn commit thành công.

---

## 11. Phản mẫu 10 — Phụ thuộc vào khóa cục bộ trong hệ thống phân tán

Việc sử dụng các cấu trúc đồng bộ hóa (Mutex/ReentrantLock) trong cấp độ JVM chỉ giới hạn được đụng độ ở một máy chủ ứng dụng duy nhất. Đối với kiến trúc đa phiên bản, phương pháp này không có khả năng bảo toàn tính nhất quán. Database mới là cấp độ đáng tin cậy để xử lý phân giải đồng thời.

---

## 12. Hướng dẫn tái hiện và phân tích lỗi qua thực nghiệm

- **Thiết lập:** Cố định phiên bản của ví.
- **Tương tranh:** Dùng cơ chế rào chắn để đồng bộ thời gian bắt đầu của nhiều thread truy xuất cùng lúc.
- **Tiến trình xử lý:** Kiểm soát thời điểm chạy lệnh đồng bộ hoặc tạo điều kiện cho một tiến trình ưu tiên.
- **Quan trắc:** Đừng chỉ kiểm tra kết quả dữ liệu đầu cuối. Cần thu thập các metric về lượng truy vấn và các transaction bị vứt bỏ.
- **Bảo mật trạng thái thử nghiệm:** Áp dụng cấu hình timeout cho toàn bộ Latch/Futures và triển khai Testcontainers với database PostgreSQL thực tế.
- **Thiết lập Annotation:** Đảm bảo không sử dụng `@Transactional` trên phương thức kiểm thử bao quanh các thread thử nghiệm.

---

## 13. Các biện pháp tùy tiện cần tránh sử dụng

- Lạm dụng việc gia tăng giới hạn số lần thử.
- Thiết lập thời gian trì hoãn cố định mà không tích hợp cơ chế độ lệch ngẫu nhiên.
- Áp dụng thử lại cho toàn bộ các tập hợp lớp ngoại lệ phổ biến như `DataAccessException`.
- Hy vọng làm mới lại transaction lỗi bằng cách gọi `entityManager.clear()`.
- Kích hoạt tính năng `@Transactional(REQUIRES_NEW)` qua một hàm gọi nội bộ trên cùng một lớp (làm mất hiệu lực proxy của Spring).
- Bỏ qua thao tác khởi tạo/truy xuất lại đối tượng.
- Bỏ qua bước kiểm tra xác thực transaction đã thành công trước đó trong luồng thử lại.
- Trả về mã thành công khi hệ thống trên thực tế đã đạt trạng thái cạn kiệt xử lý.
- Đo lường báo cáo chỉ dựa trên lưu lượng ảo thay vì sử dụng chỉ số thực tế về mức độ xung đột dữ liệu.
