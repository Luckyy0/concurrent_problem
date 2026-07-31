# Các cách code lỗi bẫy Junior — Thử lại vô hạn hoặc kẹt cứng trong giao dịch cũ

## Khai báo Entity có Đánh dấu Phiên bản (Versioned)

Đầu tiên, hãy nhìn cách chúng ta khai báo chiếc Ví và Lịch sử cộng điểm:

```java
@Entity
@Table(name = "reward_wallet")
public class RewardWallet {
    @Id
    private Long walletId;

    @Column(nullable = false)
    private long points;

    // Đây là "rào chắn" Khóa Lạc quan
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

## Lỗi 1 — Đặt vòng lặp CÙNG CHỖ với `@Transactional`

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
            // Lầm tưởng là xóa rác bộ đệm thì sẽ cứu vãn được
            entityManager.clear();
        }
    }
}
```

**Tại sao sai?**
Khi xảy ra lỗi đụng độ Khóa Lạc quan, Giao dịch (Transaction) hiện tại LẬP TỨC BỊ ĐÓNG MỘC tử hình (rollback-only). Bạn gọi `clear()` cũng không thể cứu nó, và càng không thể tạo ra một "bức ảnh" (snapshot) mới của Database. 
Vòng lặp này cứ chạy vô hạn, không có điểm dừng, và ở lần lặp tiếp theo nó sẽ văng ra mấy lỗi khó đỡ như `UnexpectedRollbackException`.

> **Nhớ kỹ:** Nếu bạn nhét vòng lặp VÀO TRONG Transaction, thì cái Transaction đó sẽ nuốt luôn vòng lặp. **Muốn thử lại (fresh attempts) sạch sẽ, thì Transaction phải ĐƯỢC GỌI TỪ BÊN TRONG vòng lặp.**

---

## Lỗi 2 — Đã bẻ được Giao dịch mới nhưng lại... thử lại tức thì

```java
public CreditResult credit(CreditCommand command) {
    while (true) {
        try {
            // attempts.creditOnce đã được tách ra và có REQUIRES_NEW
            return attempts.creditOnce(command);
        } catch (ObjectOptimisticLockingFailureException conflict) {
            // TRỐNG LỐC: Không giới hạn lần thử, không đếm giờ, không thèm chờ đợi!
        }
    }
}
```

**Tại sao sai?**
Tuy ranh giới Giao dịch đã sạch, nhưng bạn lại cho đám rớt hạng (losers) HÚC ĐẦU VÀO NGAY LẬP TỨC. Chúng sẽ lại đâm nhau sứt đầu mẻ trán cùng một tích tắc (lockstep conflicts). 
Chưa kể người dùng bấm Hủy (cancel) rồi mà cái vòng lặp này vẫn không biết điểm dừng, cày nát hồ bơi kết nối (pool) và database của bạn.

---

## Lỗi 3 — Xài Annotation `@Retryable` vô tội vạ

```java
// Bắt cào bằng MỌI LỖI RuntimeException
@Retryable(retryFor = RuntimeException.class)
public CreditResult credit(CreditCommand command) {
    return attempts.creditOnce(command);
}
```

**Tại sao sai?**
Lỗi logic validation, lỗi map sai object, lỗi đứt cáp quang, hay người dùng hủy ngang... tất cả đều bị bạn mang ra "thử lại". 
Bạn đang lãng phí tài nguyên mù quáng vì không chịu phân loại lỗi rạch ròi. Chỉ nên thử lại khi bị đụng độ Khóa (OptimisticLockingFailure).

---

## Lỗi 4 — Tự bóp dái: Mỗi lần thử lại đẻ ra một ID mới

```java
// Đẻ ID mới tinh cho mỗi lần thử lặp
CreditCommand next = new CreditCommand(
        UUID.randomUUID(),
        original.walletId(),
        original.points()
);
```

**Tại sao sai?**
Lỡ như lần thử đầu tiên chốt sổ (commit) THÀNH CÔNG, nhưng mạng bị rớt khiến máy chủ không nhận được Response. Bạn đem tạo ID mới và gửi lại... Chúc mừng, khách hàng được cộng điểm 2 lần! 
Mã Lệnh (`command ID`) sinh ra là để định danh nghiệp vụ (chống trùng), tuyệt đối không dùng nó làm mã đếm lần thử.

---

## Lỗi 5 — Đứng ngủ (Backoff) luôn TRONG GIAO DỊCH

```java
@Transactional
public void creditOnce(...) {
    // Làm việc...
    // Xong gọi ngủ đông luôn ở đây
    retryBackoff.pause(...); // Ôm luôn connection mà đi ngủ!
}
```

**Tại sao sai?**
Ngủ (Delay) chỉ được phép chạy SAU KHI đã kết thúc hoặc hủy (rollback) Giao dịch. Bạn ôm Transaction mà ngủ thì cả hệ thống xếp hàng chửi bạn vì cạn kiệt Connection pool.

---

## Lỗi 6 — Tái sử dụng cái Entity đã ôi thiu

```java
// Tải ví lên 1 lần duy nhất ở ngoài
RewardWallet stale = wallets.findById(id).orElseThrow();

for (...) {
    transactionTemplate.execute(status -> {
        stale.credit(delta);
        return wallets.save(stale); // Truyền đồ cũ vào Giao dịch mới
    });
}
```

**Tại sao sai?**
Cái Ví (Entity) này đã bị dính với bộ đệm (persistence context) cũ kỹ từ thuở nào, mang theo version cũ rích. Giao dịch mới cần phải bốc đồ mới (load lại), chứ không được ném đồ ôi thiu qua lại giữa các vòng lặp.

---

## Lỗi 7 — Chặn số lần thử, nhưng lại QUÊN chặn thời gian tối đa

```java
for (int attempt = 1; attempt <= 10; attempt++) {
    backoff.pause(attempt);
}
```

**Tại sao sai?**
10 lần thử nghe có vẻ an toàn. NHƯNG lỡ mỗi lần kết nối DB bị nghẽn mất 5 giây, cộng thêm thời gian ngủ... tổng cộng mất mẹ 1 phút! Client của bạn chắc chắn đã ngắt kết nối vì time-out (SLO) từ đời nào rồi. 
Thời gian tối đa (Deadline) bắt buộc phải đo cho TOÀN BỘ quá trình (lần thử + chờ đợi + ngủ + dọn rác).

---

## Lỗi 8 — Cố chấp thử lại với lỗi "Không được phép" (Business rejection)

Nếu Ví bị khóa (suspended), hay luật quy định không được phép cộng điểm nữa. Việc bạn tải lại Ví (reload) cũng đâu làm nó mở khóa ra? Thử lại chỉ là đốt tiền của Server. Chỉ nên ưu ái cho những lỗi do va chạm (transient), chứ đừng thương xót mọi kẻ thua cuộc.

---

## Lỗi 9 — Kéo API bên thứ ba (External call) vào nằm chung xuồng

```java
wallet.credit(points);
// Trót dại gửi email cho khách TRƯỚC KHI DB được lưu!
notificationClient.send(command.commandId());
wallets.flush();
```

**Tại sao sai?**
Hàm `flush()` thất bại -> DB Hủy lệnh (rollback). Tiền CHƯA ĐƯỢC CỘNG, nhưng Email báo "Anh đã nhận được 100 củ" đã gửi đi thành công! 
Tuyệt đối không gọi API ngoài khi chưa chốt sổ (commit). Hãy dùng hộp thư (outbox) nằm trong Giao dịch thành công và giải quyết ở 1 job khác.

---

## Lỗi 10 — Dùng khóa trên RAM (Local serialization) để lừa dối bản thân

Dùng mấy cái Khóa (Mutex/Lock) của Java thì có vẻ chặn được đụng độ. Nhưng đó chỉ là MỘT MÁY (pod). 
Lên Production chạy 5 máy thì App-2 vẫn đè bẹp App-1 như thường. Nó chỉ lừa được bạn khi code trên máy cá nhân, chứ không thay thế được sức mạnh phân xử Version của Database PostgreSQL.

---

## Hướng dẫn tái hiện đống lộn xộn này

- Khởi tạo 1 ví với version cố định.
- Cho N luồng cùng chạy, dùng rào chắn (barrier) ép tụi nó Tải dữ liệu CÙNG 1 ĐỒNG HỒ.
- Ép `flush` đồng loạt hoặc cho đứa thắng chạy qua trước.
- Không chỉ soi kết quả cuối, PHẢI ĐO xem máy chủ tốn bao nhiêu truy vấn (queries) / giao dịch (transactions) bị vứt bỏ.
- Mọi biến chốt chờ (Future/latch) đều phải có Timeout, và nhớ xài PostgreSQL xịn qua Testcontainers.
- Phải xóa `@Transactional` ở hàm bao bên ngoài cục Test.

---

## Các cách "chữa cháy" ngô nghê ĐỪNG BAO GIỜ LÀM

- Chỉ lo tăng số MaxAttempts lên 100 lần.
- Cài thời gian ngủ cứng ngắc không có xáo trộn ngẫu nhiên (jitter).
- Bắt lỗi chung chung kiểu `DataAccessException` rồi đem đi thử lại sạch sành sanh.
- Gọi `entityManager.clear()` hòng tẩy trắng tội lỗi trong cùng Giao dịch.
- Nhắn hàm lính đánh thuê `@Transactional(REQUIRES_NEW)` bằng lệnh gọi nội bộ (self-invocation) (Spring không bọc proxy đâu nhé).
- Thử lại mà lười không thèm móc object lên lại từ DB.
- Lười chặn các lệnh đã báo Chốt sổ (commit) trước đó.
- Lệnh bị văng lỗi Kiệt sức mà vẫn báo `Status 200 Success` về cho App Client.
- Chém gió các con số hiệu năng (throughput) thay vì vác metric đo đụng độ thật (conflict metrics) ra để báo cáo.
