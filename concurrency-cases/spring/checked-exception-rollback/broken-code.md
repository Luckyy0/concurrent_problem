# Đoạn mã nguồn bị lỗi theo đường dẫn checked-exception

## Mô hình nghiệp vụ tối giản (Domain model)

Giao dịch chi trả (Payout) và ví (wallet) được mô phỏng dưới dạng các đối tượng có thể thay đổi trạng thái; bản ghi sổ cái (ledger entry) đóng vai trò là một lưu vết kiểm toán chỉ được phép thêm mới (append-only). Các thiết lập ánh xạ thực thể (entity mappings) thông thường đã được rút gọn để tập trung thảo luận vào giao dịch; phần JPA no-arg constructors hay các hàm getters vốn không liên quan cũng đã được lược bỏ:

```java
public enum PayoutStatus {
    RECEIVED,
    PROCESSING,
    REJECTED,
    COMPLETED
}
```

```java
@Entity
@Table(name = "payout_request")
public class PayoutRequest {
    @Id
    private UUID id;

    private UUID walletId;
    private UUID beneficiaryId;
    private long amount;

    @Enumerated(EnumType.STRING)
    private PayoutStatus status;

    public void markProcessing() {
        if (status != PayoutStatus.RECEIVED) {
            throw new IllegalStateException("Payout is not RECEIVED");
        }
        status = PayoutStatus.PROCESSING;
    }
}
```

```java
@Entity
@Table(name = "wallet_account")
public class WalletAccount {
    @Id
    private UUID id;

    private long availableBalance;

    public void placeHold(long amount) {
        if (amount <= 0 || availableBalance < amount) {
            throw new IllegalStateException("Insufficient available balance");
        }
        availableBalance -= amount;
    }
}
```

Trong thực tế production, sổ cái nên có thêm thông tin số tiền (amount), loại tiền tệ (currency), hướng giao dịch (direction) và các tham chiếu không thể thay đổi (immutable references). Trong ví dụ này, việc sử dụng hàm tạo (factory) nhằm tránh nhầm tưởng rằng số dư khả dụng (balance projection) là thông tin lịch sử tài chính duy nhất:

```java
@Entity
@Table(
    name = "ledger_entry",
    uniqueConstraints = @UniqueConstraint(
        name = "uk_ledger_payout_type",
        columnNames = {"payout_id", "entry_type"}
    )
)
public class LedgerEntry {
    @Id
    private UUID id;

    private UUID payoutId;
    private UUID walletId;
    private long amount;
    private String entryType;

    public static LedgerEntry payoutHold(
        UUID payoutId,
        UUID walletId,
        long amount
    ) {
        return new LedgerEntry(
            UUID.randomUUID(),
            payoutId,
            walletId,
            amount,
            "PAYOUT_HOLD"
        );
    }
}
```

## Khóa kho lưu trữ (Repository locking) không thể sửa quy tắc rollback

Các kho dữ liệu (Repositories) sẽ khóa hai cấu trúc đối tượng dữ liệu thay đổi để loại trừ các tác động xen ngang (concurrent mutation) không mong muốn:

```java
public interface PayoutRepository
        extends JpaRepository<PayoutRequest, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select p from PayoutRequest p where p.id = :id")
    Optional<PayoutRequest> findByIdForUpdate(UUID id);
}
```

```java
public interface WalletRepository
        extends JpaRepository<WalletAccount, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select w from WalletAccount w where w.id = :id")
    Optional<WalletAccount> findByIdForUpdate(UUID id);
}
```

Khóa dòng (Row locks) bảo vệ hệ thống khỏi các thao tác ghi đồng thời trong quá trình giao dịch đang chạy. Tuy nhiên, chúng không hề có khả năng quyết định loại ngoại lệ nào sẽ kích hoạt quá trình rollback.

## Ngoại lệ nghiệp vụ có kiểm tra (Checked business exception)

```java
public final class BeneficiaryRejectedException extends Exception {
    public BeneficiaryRejectedException(UUID beneficiaryId) {
        super("Beneficiary is blocked: " + beneficiaryId);
    }
}
```

Chính sách nghiệp vụ (Policy) chỉ tiến hành đối chiếu với danh sách bị khóa trên bộ nhớ cục bộ, hoặc sử dụng bản sao thông qua chung bộ đệm cơ sở dữ liệu (cache policy snapshot); nó hoàn toàn không gọi sang các dịch vụ từ xa (remote service) làm kéo dài thời gian giao dịch:

```java
@Component
public class BeneficiaryPolicy {
    private final BlockedBeneficiaryRepository blockedBeneficiaries;

    public void verify(UUID beneficiaryId)
            throws BeneficiaryRejectedException {
        if (blockedBeneficiaries.existsById(beneficiaryId)) {
            throw new BeneficiaryRejectedException(beneficiaryId);
        }
    }
}
```

## Dịch vụ lỗi (Broken service)

```java
@Service
public class PayoutPreparationService {
    private final PayoutRepository payouts;
    private final WalletRepository wallets;
    private final LedgerEntryRepository ledger;
    private final BeneficiaryPolicy beneficiaryPolicy;

    public PayoutPreparationService(
        PayoutRepository payouts,
        WalletRepository wallets,
        LedgerEntryRepository ledger,
        BeneficiaryPolicy beneficiaryPolicy
    ) {
        this.payouts = payouts;
        this.wallets = wallets;
        this.ledger = ledger;
        this.beneficiaryPolicy = beneficiaryPolicy;
    }

    @Transactional
    public void prepare(UUID payoutId)
            throws BeneficiaryRejectedException {
        PayoutRequest payout = payouts.findByIdForUpdate(payoutId)
            .orElseThrow();
        WalletAccount wallet = wallets
            .findByIdForUpdate(payout.getWalletId())
            .orElseThrow();

        payout.markProcessing();
        wallet.placeHold(payout.getAmount());
        ledger.save(LedgerEntry.payoutHold(
            payout.getId(),
            wallet.getId(),
            payout.getAmount()
        ));

        beneficiaryPolicy.verify(payout.getBeneficiaryId());
    }
}
```

Đoạn code trên không hề bị lỗi gọi nội bộ mất proxy: phương thức gọi (caller) vẫn gọi hàm công khai trên hạt đậu (bean) Spring chính thức. Vấn đề nằm ở chỗ chú thích `@Transactional` đã bỏ quên khai báo quy tắc rollback (rollback rule) cho ngoại lệ có kiểm tra kia.

## Điều mà Spring thực sự sẽ làm

Đây là toàn bộ đường đi vắn tắt (Path rút gọn):

```text
proxy mở giao dịch (begins Tx)
  -> phương thức tiến hành thay đổi trên các thực thể được quản lý (mutates managed entities)
  -> phương thức ném ra BeneficiaryRejectedException
  -> quy tắc rollback (rollback rule) kiểm tra và trả về false (do là checked exception)
  -> Hibernate đẩy (flushes) các lệnh SQL đang chờ xuống cơ sở dữ liệu
  -> trình quản lý giao dịch ra lệnh commit (commits)
  -> proxy ném lại (rethrows) BeneficiaryRejectedException ra ngoài
```

Nếu lệnh SQL đã được chủ động flush sớm từ trước, các khóa dòng vẫn được giữ nguyên cho đến tận khi commit. Nếu chưa flush, Hibernate sẽ tự động gửi đi mọi lệnh SQL trong quá trình commit. Cả hai trường hợp này đều đi tới một cái kết chắc nịch (durable outcome): các thay đổi đã được commit thành công.

> **Nói ngắn gọn:** Ngoại lệ có thoát khỏi phương thức hay không thì cũng không đồng nghĩa với việc giao dịch đó tự động rollback; kiểu ngoại lệ (exception type) và quy tắc rollback đã được cấu hình (configured rollback rule) mới là thứ có quyền sinh sát.

## Các điều kiện tiền đề để tái hiện lỗi (Preconditions tái hiện)

- Hàm `prepare()` được gọi qua Spring proxy và tạo mới (hoặc tham gia) vào một giao dịch với chính sách lan truyền `REQUIRED`.
- Chính sách người thụ hưởng ném ra chính xác ngoại lệ có kiểm tra đã được khai báo.
- Ngoại lệ này phải lọt qua ranh giới hàm công khai, không bị biến tấu đổi thành ngoại lệ không kiểm tra (unchecked exception) trong quá trình xử lý.
- Code không có `rollbackFor` cũng như không có thao tác đánh dấu rollback-only trong hàm (programmatic rollback-only).
- Quá trình flush/commit không tình cờ vấp phải một lỗi độc lập khác của cơ sở dữ liệu.
- Tiến trình điều phối sẽ quét thấy trạng thái `PROCESSING` ngay sau khi giao dịch này commit.

## Những cách "sửa" chưa tới hoặc sai lầm hoàn toàn

- Cố thêm từ khóa `throws BeneficiaryRejectedException` vào chữ ký (signature) hàm; việc khai báo trong Java không có tác dụng gì trong việc cấu hình tính năng rollback của Spring.
- Gọi hàm `saveAndFlush()` trước lệnh kiểm tra chính sách; flush không hề đồng nghĩa với rollback.
- Dùng khóa dòng (row lock); khóa chỉ có tác dụng sắp xếp thứ tự ưu tiên xử lý đồng thời, và đằng nào khóa cũng sẽ được giải phóng khi cái tiến trình sai lầm kia chịu commit.
- Bắt (Catch) ngoại lệ, ghi lại log rồi kết thúc (return) nhẹ nhàng; proxy ở ngoài càng không hề nhận biết được lỗi nào xảy ra để mà rollback.
- Chờ mong một phép màu từ cơ sở dữ liệu PostgreSQL rằng nó sẽ ngầm tự hiểu một loại trừ chối nghiệp vụ (business rejection) từ một ngoại lệ Java.
- Cho gọi thử lại (Retry) sau khi gặp ngoại lệ mà không thèm kiểm tra xem trạng thái hay khóa từ chối lũy đẳng (idempotency key) đã được commit bền vững dưới cơ sở dữ liệu hay chưa.
