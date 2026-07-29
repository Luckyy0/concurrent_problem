# Explicit failure contracts and fixes

## Chọn contract trước khi chọn annotation

Có hai business contracts hợp lệ nhưng khác nhau:

```text
Contract A — exceptional rejection:
  throw BeneficiaryRejectedException
  => rollback toàn bộ preparation unit

Contract B — recorded rejection:
  return Rejected
  => commit payout = REJECTED, không hold, không executable state
```

Broken implementation vô tình tạo contract thứ ba: throw `Rejected` nhưng commit
`PROCESSING`. Không annotation nào thay thế việc chọn rõ một trong hai contracts.

> **Nói ngắn gọn:** exception và durable state phải kể cùng một câu chuyện.

## Solution 1 — Explicit `rollbackFor` tại public boundary

Nếu checked exception nghĩa là unit thất bại hoàn toàn:

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

    @Transactional(
        rollbackFor = BeneficiaryRejectedException.class
    )
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

Exception phải thoát khỏi method và call phải đi qua Spring proxy. Khi interceptor
match type:

1. physical transaction được marked rollback-only/rolled back;
2. Hibernate changes, kể cả SQL đã flush, không commit;
3. PostgreSQL release row locks ở rollback;
4. dispatcher không thấy `PROCESSING`/hold;
5. proxy rethrow checked exception sau rollback.

### Conflict và competing request

Nếu retry/concurrent request chạm cùng locked rows, nó có thể block tới khi Tx-A
rollback. Sau đó nó reload committed `RECEIVED`/balance `1000`; nó không quan sát
intermediate state. Rejected attempt là loser và fails; dispatcher query observes
a no-op vì không có executable state.

### Đặt rule ở đâu?

Đặt trên public use-case boundary thật sự tạo transaction. Nếu project có nhiều
checked failure types, dùng một common base type có semantics rõ:

```java
public abstract class RollbackBusinessException extends Exception {
    protected RollbackBusinessException(String message) {
        super(message);
    }
}
```

```java
@Transactional(rollbackFor = RollbackBusinessException.class)
public void prepare(UUID payoutId)
        throws BeneficiaryRejectedException {
    // ...
}
```

Không đưa mọi `Exception` vào rule chỉ vì tiện nếu codebase còn checked exceptions
biểu thị recoverable/non-transactional outcome. Inventory failure contracts trước.

## Solution 2 — Dùng unchecked domain exception

Nếu exception luôn biểu thị abort và team convention coi domain failure là
unchecked:

```java
public final class BeneficiaryRejectedException
        extends RuntimeException {
    public BeneficiaryRejectedException(UUID beneficiaryId) {
        super("Beneficiary is blocked: " + beneficiaryId);
    }
}
```

Default Spring rule sẽ rollback. Cách này giảm annotation lặp lại nhưng thay đổi
Java API contract: compiler không buộc caller handle exception.

Phù hợp khi:

- exception là unrecoverable trong current use case;
- global exception handler ánh xạ nó sang response đúng;
- team có convention nhất quán;
- tests khóa transaction outcome.

Không đổi inheritance chỉ để “lừa framework” nếu business semantics thật sự là
expected result.

## Solution 3 — Commit một rejected domain result có chủ đích

Khi cần lưu rejection để audit/support, validation diễn ra trước executable
mutation và method return một sealed result:

```java
public sealed interface PreparationResult
        permits PreparationResult.Ready, PreparationResult.Rejected {

    record Ready(UUID payoutId) implements PreparationResult {}

    record Rejected(
        UUID payoutId,
        String reasonCode
    ) implements PreparationResult {}
}
```

Policy trả decision thay vì checked exception:

```java
public sealed interface BeneficiaryDecision
        permits BeneficiaryDecision.Allowed,
                BeneficiaryDecision.Blocked {

    record Allowed() implements BeneficiaryDecision {}
    record Blocked(String reasonCode) implements BeneficiaryDecision {}
}
```

```java
@Transactional
public PreparationResult prepare(UUID payoutId) {
    PayoutRequest payout = payouts.findByIdForUpdate(payoutId)
        .orElseThrow();

    BeneficiaryDecision decision =
        beneficiaryPolicy.evaluate(payout.getBeneficiaryId());

    if (decision instanceof BeneficiaryDecision.Blocked blocked) {
        payout.markRejected(blocked.reasonCode());
        return new PreparationResult.Rejected(
            payout.getId(),
            blocked.reasonCode()
        );
    }

    WalletAccount wallet = wallets
        .findByIdForUpdate(payout.getWalletId())
        .orElseThrow();

    wallet.placeHold(payout.getAmount());
    ledger.save(LedgerEntry.payoutHold(
        payout.getId(),
        wallet.getId(),
        payout.getAmount()
    ));
    payout.markReadyForDispatch();
    outbox.save(PayoutOutbox.ready(payout));

    return new PreparationResult.Ready(payout.getId());
}
```

Dispatcher không chọn một generic intermediate state. Nó consumes committed outbox
event hoặc chỉ claim `READY_FOR_DISPATCH`.

SQL constraints/indexes hỗ trợ invariant:

```sql
create unique index uk_ledger_payout_hold
    on ledger_entry (payout_id, entry_type)
    where entry_type = 'PAYOUT_HOLD';

create unique index uk_payout_ready_outbox
    on payout_outbox (payout_id, event_type)
    where event_type = 'PAYOUT_READY';
```

Rejected branch commit `REJECTED` để audit nhưng không tạo hold/outbox. Return value
nói đúng durable outcome; caller không được diễn giải nó là rollback.

### Trade-off

Domain state machine và result type rõ hơn nhưng code nhiều hơn. Nếu unexpected
checked infrastructure failure xảy ra sau mutations, boundary vẫn cần rollback
rule hoặc exception translation đúng.

## Solution 4 — Programmatic boundary khi API phải return result

`TransactionTemplate` hữu ích khi application API không được ném checked exception:

```java
@Service
public class PayoutPreparationFacade {
    private final TransactionTemplate transactions;
    private final PayoutPreparationWork work;

    public PayoutPreparationFacade(
        PlatformTransactionManager transactionManager,
        PayoutPreparationWork work
    ) {
        this.transactions = new TransactionTemplate(transactionManager);
        this.work = work;
    }

    public PreparationResult prepare(UUID payoutId) {
        return transactions.execute(status -> {
            try {
                work.prepare(payoutId);
                return new PreparationResult.Ready(payoutId);
            } catch (BeneficiaryRejectedException rejected) {
                status.setRollbackOnly();
                return new PreparationResult.Rejected(
                    payoutId,
                    "BENEFICIARY_BLOCKED"
                );
            }
        });
    }
}
```

`work.prepare()` ở đây là plain work chạy bên trong template transaction, không tự
mở một confusing nested boundary. `setRollbackOnly()` phải xảy ra trước callback
return.

Trade-off là imperative transaction API và result `Rejected` đi kèm database
rollback, nên caller phải hiểu rejection không được audit trong cùng transaction.
Nếu cần durable rejection audit, ghi nó trong transaction mới sau rollback và
chấp nhận independent commit contract.

## Solution 5 — Reusable meta-annotation

Một codebase có convention ổn định có thể centralize rule:

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Transactional(rollbackFor = RollbackBusinessException.class)
public @interface BusinessTransactional {
}
```

```java
@BusinessTransactional
public void prepare(UUID payoutId)
        throws BeneficiaryRejectedException {
    // ...
}
```

Tên annotation phải nói rõ semantics. Đừng tạo nhiều meta-annotations gần giống
nhau mà reviewer không biết effective propagation/isolation/rollback rule.

## Catch ở outer scope và `UnexpectedRollbackException`

Nếu inner boundary dùng `rollbackFor` và join outer transaction, nó có thể mark
physical transaction rollback-only. Outer code không nên swallow rồi giả định vẫn
commit được:

```java
@Transactional
public BatchResult runBatch(UUID payoutId) {
    try {
        payoutPreparation.prepare(payoutId);
        return BatchResult.ready();
    } catch (BeneficiaryRejectedException rejected) {
        return BatchResult.rejected();
    }
}
```

Outer return bình thường nhưng commit có thể ném `UnexpectedRollbackException`.
Correct options:

- để rejection thoát qua outer boundary;
- chuyển validation trước mutation và return committed `Rejected`;
- tách từng item thành transaction độc lập nếu batch contract cho phép;
- dùng programmatic boundary với result rõ ràng.

Không catch chỉ để “giữ transaction sống” sau khi nó đã rollback-only.

## Không dùng `REQUIRES_NEW` như rollback fix mặc định

Đặt work vào `REQUIRES_NEW` làm rollback rule có hiệu lực cho inner physical
transaction nếu configured đúng, nhưng nó thay atomicity:

- suspend outer transaction;
- dùng thêm connection;
- inner outcome độc lập outer;
- outer rollback không đảo inner commit;
- lock ordering có thể phức tạp hơn.

Chỉ dùng khi independent unit là requirement, không dùng để che checked-exception
classification mơ hồ.

## So sánh lựa chọn

| Lựa chọn | Durable rejection | API outcome | Correctness risk | Operational trade-off |
| --- | --- | --- | --- | --- |
| `rollbackFor` | Không trong cùng Tx | Checked exception | Thấp khi boundary rõ | Atomic, đơn giản |
| Unchecked domain exception | Không trong cùng Tx | Runtime exception | Convention có thể bị lạm dụng | Ít annotation |
| Return committed `Rejected` | Có | Domain result | State machine phải chặt | Audit rõ, thêm code |
| `TransactionTemplate` + rollback-only | Không trong cùng Tx | Domain result | Dễ quên set flag | Boundary explicit |
| `REQUIRES_NEW` audit | Có độc lập | Exception/result | Partial outcome có chủ đích | Thêm connection/complexity |

## Recommendation cho case này

Với invariant ban đầu, dùng Solution 1:

1. giữ public proxied boundary;
2. thêm type-safe `rollbackFor`;
3. để exception thoát qua proxy;
4. không dispatch từ intermediate `PROCESSING`;
5. test committed database state sau exception bằng transaction mới;
6. thêm unique keys/idempotency riêng cho duplicate delivery;
7. chuyển sang Solution 3 nếu product yêu cầu lưu rejection.

## Production checklist

### Failure contract

- [ ] Mỗi checked exception được phân loại commit hay rollback.
- [ ] Method outcome khớp durable state.
- [ ] Catch/wrap không làm mất rollback signal.
- [ ] Outer scope biết inner có thể mark rollback-only.
- [ ] Expected rejection dùng exception hoặc result có chủ đích.

### Transaction

- [ ] Rule nằm trên public boundary gọi qua proxy.
- [ ] Flush không bị nhầm với commit.
- [ ] Remote I/O nằm ngoài blocking transaction khi có thể.
- [ ] `REQUIRES_NEW` có independent-outcome requirement.
- [ ] Retry tạo transaction mới và reload committed state.

### Database và dispatcher

- [ ] Dispatcher chỉ claim terminal-ready state hoặc outbox event.
- [ ] Ledger append-only và có business uniqueness key.
- [ ] Rejected path không tạo financial hold.
- [ ] Multi-instance tests assert committed visibility.
- [ ] Reconciliation phát hiện rejected/executable mismatch.
