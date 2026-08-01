# Mã nguồn làm sai số dư điểm

## 1. Lược đồ chưa đủ bảo vệ

```sql
CREATE TABLE loyalty_account (
    customer_id UUID PRIMARY KEY,
    points_balance BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);

CREATE TABLE loyalty_ledger_entry (
    entry_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES loyalty_account(customer_id),
    command_id UUID NOT NULL,
    entry_type VARCHAR(20) NOT NULL,
    points_delta BIGINT NOT NULL,
    order_id UUID,
    created_at TIMESTAMPTZ NOT NULL
);
```

Lược đồ thiếu:

- `CHECK (points_balance >= 0)`;
- ràng buộc duy nhất cho `(customer_id, command_id)`;
- lớp bảo vệ không cho sửa hoặc xóa sổ cái;
- một bản ghi lệnh để phát lại kết quả và kiểm tra dấu vân tay.

Ngay cả khi thêm `CHECK`, mã đọc–kiểm tra–ghi sau vẫn có thể tạo hai bút toán âm
trong khi hai giao dịch cùng ghi đè một số dư dương.

## 2. Thực thể số dư

```java
@Entity
@Table(name = "loyalty_account")
public class LoyaltyAccount {

    @Id
    @Column(name = "customer_id", nullable = false)
    private UUID customerId;

    @Column(name = "points_balance", nullable = false)
    private long pointsBalance;

    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    protected LoyaltyAccount() {
    }

    public boolean canSpend(long points) {
        return pointsBalance >= points;
    }

    public void spend(long points, Instant now) {
        pointsBalance -= points;
        updatedAt = now;
    }

    public void earn(long points, Instant now) {
        pointsBalance += points;
        updatedAt = now;
    }
}
```

Thực thể không có `@Version`. Phép cộng/trừ chạy trên giá trị đã tải vào bộ nhớ
Java, rồi Hibernate ghi một giá trị tuyệt đối.

## 3. Kho dữ liệu

```java
public interface LoyaltyAccountRepository
    extends JpaRepository<LoyaltyAccount, UUID> {
}

public interface LoyaltyLedgerRepository
    extends JpaRepository<LoyaltyLedgerEntry, UUID> {

    boolean existsByCustomerIdAndCommandId(
        UUID customerId,
        UUID commandId
    );
}
```

`existsBy...` không chiếm quyền trên mã lệnh chưa tồn tại. Hai lần gửi đồng thời
cùng một mã vẫn có thể cùng nhận `false`.

## 4. Dịch vụ tiêu điểm bị lỗi

```java
@Service
public class BrokenLoyaltyService {

    private final LoyaltyAccountRepository accounts;
    private final LoyaltyLedgerRepository ledger;
    private final Clock clock;

    public BrokenLoyaltyService(
        LoyaltyAccountRepository accounts,
        LoyaltyLedgerRepository ledger,
        Clock clock
    ) {
        this.accounts = accounts;
        this.ledger = ledger;
        this.clock = clock;
    }

    @Transactional
    public SpendPointsResult spend(SpendPointsCommand command) {
        if (ledger.existsByCustomerIdAndCommandId(
            command.customerId(),
            command.commandId()
        )) {
            throw new DuplicatePointCommandException();
        }

        LoyaltyAccount account = accounts.findById(command.customerId())
            .orElseThrow(LoyaltyAccountNotFoundException::new);

        if (!account.canSpend(command.points())) {
            throw new InsufficientPointsException();
        }

        Instant now = clock.instant();
        account.spend(command.points(), now);

        LoyaltyLedgerEntry entry = LoyaltyLedgerEntry.redeem(
            UUID.randomUUID(),
            command.customerId(),
            command.commandId(),
            command.orderId(),
            -command.points(),
            now
        );
        ledger.save(entry);

        return new SpendPointsResult(entry.id(), account.pointsBalance());
    }
}
```

Mã nguồn có giao dịch, lịch sử, bước kiểm tra số dư và bước chống lặp. Tuy nhiên,
ba thao tác có thẩm quyền đều bị tách:

```text
exists → đọc số dư → kiểm tra → ghi số dư tuyệt đối → chèn bút toán
```

`save()` có thể chưa gửi `INSERT` ngay. Hibernate thường đẩy SQL khi `flush()`
hoặc `COMMIT`, nên lỗi duy nhất có thể xuất hiện muộn hơn vị trí mã Java dự kiến.

## 5. Dịch vụ cộng điểm cũng bị mất cập nhật

```java
@Transactional
public EarnPointsResult earn(EarnPointsCommand command) {
    LoyaltyAccount account = accounts.findById(command.customerId())
        .orElseThrow(LoyaltyAccountNotFoundException::new);

    Instant now = clock.instant();
    account.earn(command.points(), now);

    LoyaltyLedgerEntry entry = LoyaltyLedgerEntry.earn(
        UUID.randomUUID(),
        command.customerId(),
        command.commandId(),
        command.orderId(),
        command.points(),
        now
    );
    ledger.save(entry);

    return new EarnPointsResult(entry.id(), account.pointsBalance());
}
```

Một giao dịch cộng và một giao dịch trừ cùng đọc số dư cũ có thể ghi đè nhau.
Sổ cái chứa cả hai bút toán nhưng bảng số dư chỉ phản ánh bút toán ghi sau.

## 6. Điều kiện tái hiện tiêu hai lần

```text
points_balance = 1.000
CMD-A tiêu 800 cho ORDER-A
CMD-B tiêu 800 cho ORDER-B
```

Hai mã lệnh và hai đơn khác nhau. Đây không phải cùng một yêu cầu bị gửi lại; cả
hai phải tranh cùng số dư.

Đặt rào chắn sau khi hai giao dịch đã đọc `1.000` và vượt qua `canSpend`, nhưng
trước khi chúng thay đổi thực thể.

## 7. Dòng thời gian tiêu hai lần

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | `BEGIN` | `BEGIN` |
| 2 | Không thấy `CMD-A` | Không thấy `CMD-B` |
| 3 | Đọc số dư `1.000` | Đọc số dư `1.000` |
| 4 | `1.000 >= 800`, chấp nhận | `1.000 >= 800`, chấp nhận |
| 5 | Tính số dư Java `200` | Tính số dư Java `200` |
| 6 | Chèn bút toán `-800` của A | Chèn bút toán `-800` của B |
| 7 | Ghi số dư tuyệt đối `200`; `COMMIT` | Chờ khóa dòng |
| 8 | | Ghi số dư tuyệt đối `200`; `COMMIT` |

Trạng thái cuối:

```text
bảng số dư                       =  200
tổng sổ cái gồm bút toán mở sổ  = -600
hai đơn đều nhận giảm giá bằng điểm
```

PostgreSQL đã tuần tự hóa hai câu `UPDATE`, nhưng câu sau vẫn ghi `200` vì giá
trị được tính trước trên ảnh chụp cũ. Khóa dòng tại thời điểm ghi không thể sửa
một quyết định đã được đưa ra trước đó.

## 8. Dòng thời gian cộng và trừ ghi đè nhau

Trạng thái đầu `1.000`, A cộng `500`, B trừ `300`:

| Bước | Giao dịch A | Giao dịch B |
| --- | --- | --- |
| 1 | Đọc `1.000`, tính `1.500` | |
| 2 | | Đọc `1.000`, tính `700` |
| 3 | Chèn bút toán `+500` | Chèn bút toán `-300` |
| 4 | Ghi `1.500` | Chờ rồi ghi `700` |

Số dư đúng là `1.200`. Kết quả `700` làm mất lần cộng; nếu thứ tự ghi đảo lại,
kết quả `1.500` làm mất lần trừ. Cả hai bút toán vẫn tồn tại nên đối soát phát
hiện chênh lệch.

## 9. Vì sao các cách sửa sau chưa đủ

### Chỉ thêm `@Transactional`

Mỗi lần gọi đã có giao dịch. Chú thích này không biến hai yêu cầu thành một vùng
loại trừ và không thay đổi câu `UPDATE` tuyệt đối do Hibernate sinh ra.

### Chỉ thêm `CHECK (points_balance >= 0)`

Hai giao dịch đều ghi `200`, nên ràng buộc vẫn đúng. Nó không biết sổ cái vừa có
hai bút toán tổng cộng `-1.600`.

### Tính `SUM` sổ cái trước khi tiêu

Hai giao dịch vẫn có thể cùng tính tổng `1.000` trước khi bên nào chèn bút toán.
Kết quả tổng không tự giữ một phần số dư và còn làm đường xử lý phụ thuộc vào số
lượng lịch sử ngày càng lớn.

### Chỉ dùng `synchronized`

Khóa chỉ bảo vệ một JVM. Hai máy chủ, tác vụ nền hoặc công cụ quản trị không
chia sẻ khóa đó.

### Chỉ thêm `@Version`

Khóa lạc quan có thể phát hiện một lần ghi đè, nhưng bên thua phải chạy lại toàn
bộ giao dịch với trạng thái mới. Đây là phương án hợp lệ khi tranh chấp hiếm,
không thay thế sổ cái, tính duy nhất của mã lệnh hay chính sách thử lại có giới
hạn.

### Chèn bút toán ở giao dịch riêng

Nếu sổ cái chốt trước rồi cập nhật số dư thất bại, lịch sử có bút toán chưa phản
ánh trong số dư. Nếu số dư chốt trước rồi chèn lịch sử thất bại, hệ thống mất dấu
vết. Hai bước phải cùng giao dịch khi dùng chung PostgreSQL.

### Sửa hoặc xóa bút toán để hoàn điểm

Lịch sử không còn thể hiện điều gì đã xảy ra. Hoàn điểm phải dùng một bút toán bù
mới và một mã lệnh duy nhất.

### Tạo mã mới cho lần thử lại

Mã mới biến cùng một ý định thành một lần tiêu khác. Sau lỗi mạng, phía gọi phải
dùng lại `command_id` ban đầu và tra cứu kết quả đã lưu.

## 10. Dấu hiệu trên hệ thống thật

- Số dư tài khoản khác tổng `points_delta` trong sổ cái.
- Hai đơn dùng tổng điểm lớn hơn số dư trước đó.
- Một bút toán cộng/trừ tồn tại nhưng số dư không phản ánh nó.
- Nhiều bút toán có cùng mã lệnh hoặc cùng tham chiếu nghiệp vụ.
- Khách hàng báo điểm “quay lại” hoặc biến mất sau hai thao tác gần nhau.
- Lỗi chỉ tăng khi có nhiều máy chủ hoặc đợt mua sắm cao điểm.
- Tác vụ sửa số dư trực tiếp không có bút toán điều chỉnh tương ứng.
- Giao dịch chờ lâu trên một tài khoản điểm hoạt động mạnh.

Log và metric không nên chứa mã khách hàng, mã lệnh hoặc chi tiết đơn hàng thô
làm nhãn có số lượng giá trị không giới hạn.
