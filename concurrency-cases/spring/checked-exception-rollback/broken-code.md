# Broken checked-exception path

## Domain model tối thiểu

Payout và wallet là mutable projections; ledger entry là append-only audit record.
Các entity mappings thông thường được rút gọn để tập trung vào transaction; JPA
no-arg constructors và getters không liên quan được lược bỏ:

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

Trong production ledger nên dùng amount, currency, direction và immutable
references rõ ràng. Case dùng factory để tránh coi balance projection là financial
history duy nhất:

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

## Repository locking không sửa rollback rule

Repositories lock hai mutable aggregates để loại trừ concurrent mutation không
liên quan:

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

Row locks bảo vệ concurrent writes trong lúc transaction chạy. Chúng không quyết
định exception nào làm rollback.

## Checked business exception

```java
public final class BeneficiaryRejectedException extends Exception {
    public BeneficiaryRejectedException(UUID beneficiaryId) {
        super("Beneficiary is blocked: " + beneficiaryId);
    }
}
```

Policy chỉ đọc local registry đã có trong cùng database/cache policy snapshot; nó
không gọi remote service trong transaction:

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

## Broken service

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

Code không thiếu proxy: caller gọi public method trên Spring bean thật. Vấn đề là
`@Transactional` không khai báo rollback rule cho checked exception.

## Điều Spring thực hiện

Path rút gọn:

```text
proxy begins Tx
  -> method mutates managed entities
  -> method throws BeneficiaryRejectedException
  -> rollback rule returns false
  -> Hibernate flushes pending SQL
  -> transaction manager commits
  -> proxy rethrows BeneficiaryRejectedException
```

Nếu SQL đã flush sớm, row locks vẫn giữ tới commit. Nếu chưa flush, Hibernate gửi
SQL trong commit. Cả hai path đều có cùng durable outcome: changes committed.

> **Nói ngắn gọn:** exception thoát khỏi method không tự động đồng nghĩa transaction
> rollback; exception type và configured rollback rule mới quyết định.

## Preconditions tái hiện

- `prepare()` được gọi qua Spring proxy và tạo/join transaction `REQUIRED`.
- Beneficiary policy ném đúng checked exception đã khai báo.
- Exception thoát khỏi public method, không bị đổi thành unchecked exception.
- Không có `rollbackFor` hoặc programmatic rollback-only.
- Flush/commit không gặp một database error độc lập.
- Dispatcher chọn `PROCESSING` payout sau commit.

## Những cách “sửa” chưa đủ

- Thêm `throws BeneficiaryRejectedException` vào signature; Java declaration không
  cấu hình Spring rollback.
- Gọi `saveAndFlush()` trước policy; flush không phải rollback.
- Dùng row lock; lock chỉ serialize access và được release khi broken path commit.
- Catch exception, log rồi return; proxy càng không thấy failure để rollback.
- Mong PostgreSQL suy ra business rejection từ Java exception.
- Retry sau exception mà không kiểm tra committed state/idempotency key.
