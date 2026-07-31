# Các cam kết lỗi rõ ràng và cách khắc phục (Explicit failure contracts and fixes)

## Chọn cam kết (contract) trước khi chọn annotation

Chúng ta có hai cam kết nghiệp vụ (business contracts) hoàn toàn hợp lệ nhưng ý nghĩa khác nhau:

```text
Contract A — Từ chối thông qua ngoại lệ (exceptional rejection):
  ném ra BeneficiaryRejectedException
  => rollback toàn bộ công việc khởi tạo (preparation unit)

Contract B — Từ chối được ghi nhận (recorded rejection):
  trả về trạng thái Rejected
  => commit giao dịch payout với trạng thái = REJECTED, không giữ tiền (hold), không tạo trạng thái thực thi (executable state)
```

Kiến trúc bị lỗi (Broken implementation) đã vô tình tạo ra một cam kết thứ ba đầy mâu thuẫn: ném ra ngoại lệ `Rejected` nhưng lại đi commit trạng thái `PROCESSING`. Chẳng có cái annotation thần thánh nào có thể thay bạn chọn rõ ràng một trong hai hợp đồng trên.

> **Nói ngắn gọn:** Ngoại lệ ném ra và trạng thái dữ liệu lưu trữ phải kể chung một câu chuyện thống nhất.

## Giải pháp 1 — Dùng `rollbackFor` tường minh ở ranh giới giao tiếp (public boundary)

Nếu bạn xác định rằng ngoại lệ có kiểm tra (checked exception) đồng nghĩa với việc toàn bộ khối công việc phải thất bại:

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

Ngoại lệ bắt buộc phải được đẩy ra khỏi phương thức và đi qua proxy của Spring. Khi trình chặn (interceptor) nhận diện đúng kiểu hình:

1. Giao dịch vật lý (physical transaction) sẽ bị đánh dấu cờ rollback-only hoặc lập tức tiến hành rollback;
2. Những thay đổi của Hibernate, kể cả các câu lệnh SQL đã kịp flush, cũng sẽ không được commit;
3. PostgreSQL tiến hành nhả các khóa dòng (row locks) khi quá trình rollback diễn ra;
4. Tiến trình điều phối sẽ không bao giờ thấy trạng thái `PROCESSING` hay số tiền bị kẹt (hold);
5. Proxy sẽ tái đẩy lại (rethrow) ngoại lệ có kiểm tra đó ra sau khi đã rollback êm thấm.

### Xung đột (Conflict) và yêu cầu giành giật (competing request)

Nếu một yêu cầu thử lại (retry) hay yêu cầu đồng thời chạm vào đúng những dòng dữ liệu đang bị khóa, nó có thể bị chặn đứng chờ cho tới khi giao dịch Tx-A bị rollback. Ngay sau đó, nó sẽ nạp lại (reload) trạng thái an toàn là `RECEIVED` cùng số dư `1000`; nó sẽ không bị nhìn lén vào trạng thái lưng chừng (intermediate state). Nỗ lực gọi xử lý bị từ chối kia được coi là thất bại; và bất kỳ thao tác truy vấn nào từ tiến trình điều phối cũng vô thưởng vô phạt (no-op) vì chẳng có trạng thái thực thi nào tồn tại.

### Vậy nên đặt quy tắc ở đâu?

Bạn nên đặt nó ngay trên ranh giới Use-case (public use-case boundary) thực sự tạo ra giao dịch. Nếu dự án của bạn có quá nhiều kiểu lỗi nghiệp vụ có kiểm tra (checked failure types), hãy đúc kết ra một class lỗi gốc (common base type) có ý nghĩa rõ ràng:

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

Nhưng tuyệt đối không được gộp chung mọi `Exception` vào quy tắc chỉ vì cảm thấy tiện, đặc biệt nếu trong codebase vẫn còn tồn tại các lỗi có kiểm tra nhưng mang mục đích thông báo (recoverable/non-transactional outcome). Bạn cần rà soát lại (Inventory) các thỏa thuận xử lý lỗi trước tiên.

## Giải pháp 2 — Xài ngoại lệ nghiệp vụ không kiểm tra (unchecked domain exception)

Nếu ngoại lệ luôn ám chỉ sự đổ vỡ (abort) và cả team quy ước coi các lỗi nghiệp vụ là ngoại lệ không kiểm tra:

```java
public final class BeneficiaryRejectedException
        extends RuntimeException {
    public BeneficiaryRejectedException(UUID beneficiaryId) {
        super("Beneficiary is blocked: " + beneficiaryId);
    }
}
```

Thì quy tắc mặc định của Spring sẽ làm tốt việc rollback. Phương pháp này giảm thiểu số lần phải điền tham số lặp lại trên annotation nhưng lại thay đổi sâu sắc bản chất (Java API contract): trình biên dịch sẽ không còn bắt ép lập trình viên phải nhớ và xử lý lỗi đó nữa.

Điều này phù hợp khi:

- Ngoại lệ đó là không thể cứu vãn (unrecoverable) trong trường hợp hiện tại;
- Trình xử lý ngoại lệ toàn cục (global exception handler) dư sức ánh xạ lỗi thành thông báo (response) chính xác;
- Cả team có nguyên tắc nhất quán;
- Các bài kiểm thử đã siết chặt lại kết quả chốt hạ (khóa transaction outcome).

Tuy nhiên, đừng vì muốn "qua mặt framework" mà đổi tính thừa kế của ngoại lệ, trong khi ý nghĩa nghiệp vụ (business semantics) thực chất lại là mong đợi một kết quả trả về.

## Giải pháp 3 — Chủ động commit một kết quả nghiệp vụ bị từ chối có chủ đích

Khi hệ thống cần phải lưu lại kết quả từ chối nhằm phục vụ tra soát/kiểm toán (audit/support), toàn bộ khối logic kiểm tra hợp lệ (validation) phải diễn ra trước khi có bất kỳ tác động biến đổi dữ liệu thực thi nào, và hàm sẽ trả về một cấu trúc kết quả được đóng gói (sealed result):

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

Chính sách nghiệp vụ bây giờ trả về quyết định (decision) thay vì ném ra lỗi ngoại lệ checked:

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

Tiến trình điều phối không còn chọn đọc một trạng thái nửa vời chung chung nào cả. Nó chỉ thực hiện dựa trên các thông báo outbox đã commit (committed outbox event) hoặc đơn giản là chỉ lấy các luồng công việc dán nhãn `READY_FOR_DISPATCH`.

Việc vận dụng các khóa/ràng buộc (SQL constraints/indexes) sẽ củng cố rào chắn an toàn (invariant):

```sql
create unique index uk_ledger_payout_hold
    on ledger_entry (payout_id, entry_type)
    where entry_type = 'PAYOUT_HOLD';

create unique index uk_payout_ready_outbox
    on payout_outbox (payout_id, event_type)
    where event_type = 'PAYOUT_READY';
```

Nhánh bị từ chối chủ động đưa vào commit trạng thái `REJECTED` nhằm đánh dấu kiểm toán mà chẳng hề làm phát sinh các giao dịch giữ tiền hay đụng chạm (hold/outbox). Kiểu dữ liệu trả về nói rõ lên kết cục cuối cùng dưới database; người gọi (caller) cũng không mảy may dịch nhầm ý rằng nó đã bị rollback.

### Đánh đổi (Trade-off)

Tuy cỗ máy trạng thái (Domain state machine) và loại kết quả tường minh dễ dò hỏi hơn nhưng bù lại sẽ phải viết nhiều mã nguồn hơn. Giả sử sau mọi rào cản thay đổi lại có một lỗi hạ tầng (infrastructure failure) nào đó phát sinh bất thình lình, ranh giới đó vẫn phải nhờ tới sự giúp đỡ của quy tắc rollback hay cách phiên dịch (exception translation) thật mượt mà.

## Giải pháp 4 — Chặn giao dịch (Programmatic boundary) khi hàm bắt buộc phải trả về dữ liệu

Mẫu thiết kế `TransactionTemplate` sẽ tỏ rõ sức mạnh nếu API không cho phép được đẩy lỗi có kiểm tra:

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

Ở đây `work.prepare()` đơn thuần là một đoạn mã chạy bọc gọn trong một khung xử lý giao dịch chuẩn, không tự ý khơi mào một ranh giới (nested boundary) lòng vòng khó hiểu nào hết. Lưu ý, cờ `setRollbackOnly()` bắt buộc phải được kích hoạt trước khi gửi dữ liệu báo cáo về hàm gọi lại (callback return).

Điểm cần đánh đổi (Trade-off) là giao diện lập trình giao dịch theo kiểu bắt buộc (imperative transaction API) cộng thêm kết quả thông báo `Rejected` đi đôi với cú rollback chốt hạ; vậy nên caller tự thân phải hiểu rằng sự từ chối không hề được lưu vết vào cơ sở dữ liệu (audit) trong cùng đợt giao dịch đó. Còn nếu nghiệp vụ đòi hỏi một đoạn vết kiểm toán bền vững, buộc phải ghi sổ riêng vào một giao dịch sinh sau đẻ muộn, đồng nghĩa với việc chấp nhận rủi ro hai đợt chốt giao dịch là (independent commit contract).

## Giải pháp 5 — Tái sử dụng siêu chú giải (Reusable meta-annotation)

Trong một hệ thống mã nguồn nhất quán, tập trung quy định lại thành siêu chú giải (meta-annotation):

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

Việc đặt tên annotation cần mô tả bật lên được ranh giới (semantics) cần truyền đạt. Tránh đẻ thêm một rổ các loại meta-annotations có tên na ná nhau, khiến cả người duyệt mã nguồn (reviewer) cũng ú ớ chẳng rõ đâu là luật lệ cô lập/lan truyền (effective propagation/isolation/rollback rule) đang được sử dụng.

## Bắt lỗi vòng ngoài và cú sốc `UnexpectedRollbackException`

Nếu giới hạn bên trong (inner boundary) áp dụng thẻ `rollbackFor` đồng thời tham gia hòa mạng với vòng ngoài (outer transaction), nó thừa quyền năng đánh cờ "xin chỉ được rollback" vào ngay chính giao dịch vật lý đang chạy. Lớp phòng thủ ngoài không nên nhai nuốt (swallow) rồi mặc định cho rằng vòng chặn ngoài sẽ qua ải:

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

Khối mã ngoài cứ tưởng hoàn tất suôn sẻ nhưng lệnh commit sẽ thình lình ném ra một quả `UnexpectedRollbackException`.
Cách đúng đắn (Correct options) là:

- Cho phép cái quyết định từ chối thoát vượt ra khỏi ranh giới phòng vệ ngoài cùng;
- Dời khâu xác nhận (validation) sang đằng trước thao tác thay đổi trạng thái, đồng thời thông báo kết quả trả về `Rejected` thẳng tưng đã commit;
- Xe nhỏ từng mục một biến nó thành một lệnh truy soát riêng lẻ, nếu lề luật gộp nhóm khối lượng (batch contract) chấp nhận;
- Dùng cấu hình khoanh vùng thủ công với kết quả chốt (programmatic boundary với result rõ ràng).

Tuyệt đối cấm hành động chụp giữ lỗi cốt chỉ để "neo cho mạch nối giao dịch" không bị đứt đoạn sau khi mà lệnh dập rollback-only đã được đóng mộc phán quyết.

## Đừng dùng `REQUIRES_NEW` như liều thuốc tiên sửa lỗi rollback (rollback fix)

Đem ném một mớ việc vào rọ `REQUIRES_NEW` sẽ khiến cờ rollback phát huy uy lực cực mạnh đối với cái vòng giao dịch vật lý thu nhỏ, tuy nhiên nó phá hoại nguyên lý tính toàn vẹn (atomicity):

- Đóng băng (suspend) toàn bộ mạch giao dịch mẹ;
- Ăn thêm tài nguyên kết nối;
- Lỗi của thằng con không kéo theo mạch chết (outer) của người mẹ;
- Giao dịch mẹ bị sập không lay động, đảo ngược nổi (inner commit) kết quả đã lỡ chốt của đứa con;
- Giao thức chốt khóa (lock ordering) có nguy cơ trở nên rắc rối hơn.

Tóm lại, chỉ đưa thẻ bài này ra áp dụng khi công việc được yêu cầu hoàn thành 100% độc lập, chứ quyết không lấy nó hòng khỏa lấp cái sai về quy ước từ chối ngoại lệ có kiểm tra mập mờ (checked-exception classification mơ hồ).

## Bảng so sánh lựa chọn sửa đổi

| Lựa chọn | Sự từ chối có tính bền vững (Durable rejection) | Kết quả API | Rủi ro cho tính đúng đắn (Correctness risk) | Điểm đánh đổi thực tiễn (Operational trade-off) |
| --- | --- | --- | --- | --- |
| `rollbackFor` | Không lưu ở cùng Tx | Ngoại lệ Checked exception | Thấp, khi ranh giới đã phân rõ | Cấu trúc khối (Atomic), tinh giản |
| Đưa hẳn vào Unchecked domain exception | Không lưu ở cùng Tx | Ngoại lệ Runtime exception | Rất dễ bị "quen tay lạm dụng" luật định (Convention có thể bị lạm dụng) | Gọn nhẹ đỡ cần đánh thêm nhãn (annotation) |
| Trả kết quả `Rejected` rồi đẩy dữ liệu luôn | Chắc chắn có | Gói thông báo kết quả Domain result | Bắt buộc phải duy trì cỗ máy trạng thái (State machine) thật sự kín kẽ | Rành rọt trên mặt kiểm toán, tốn công code thêm |
| Khóa hãm bằng `TransactionTemplate` + dập cờ rollback-only | Không lưu ở cùng Tx | Gói thông báo kết quả Domain result | Sơ ý quên không kéo cờ `setRollbackOnly` là hỏng bét | Ranh giới phân mốc vô cùng rõ ràng (Boundary explicit) |
| Kiểm soát chéo dạng `REQUIRES_NEW` audit | Có nhưng chốt riêng | Báo cả Exception/gói Result | Dính sẹo "vết bẩn cục bộ" (Partial outcome) nằm lỳ trên kế hoạch | Cắn tài nguyên mạng, gia tăng tính phức tạp |

## Gợi ý trực tiếp cho tình huống (Recommendation cho case này)

Cứ bám chặt theo rào cản tính đúng đắn gốc (invariant ban đầu), dùng Giải pháp 1 (Solution 1):

1. Giữ lấy ranh giới công cộng được ủy thác qua cơ chế uốn nắn (public proxied boundary);
2. Cắm thêm lá bùa an toàn kiểu `rollbackFor`;
3. Bật nắp xả cho luồng khói sự cố được thoát thẳng vút bay qua đường hầm proxy;
4. Khóa tay khóa chân không cho phép dispatch mầm bệnh đi khi cái bọc kia còn nửa nạc nửa mỡ `PROCESSING`;
5. Thử nghiệm test khui nắp kho dữ liệu đã dập lệnh (committed database state) dẫu cho cơn bão lỗi (exception) ùa qua bằng một vòng giao dịch thăm dò mới;
6. Thêm cọc bảo vệ khóa định danh duy nhất (unique keys) hay giải pháp tái thẩm định một chiều (idempotency) ngõ hầu trị đám gửi lén bản nháp lần thứ 2 (duplicate delivery);
7. Chuyển hướng sang quy trình của Giải pháp 3 (Solution 3) nếu bộ phận thiết kế hệ thống kiên quyết muốn lưu một tờ hóa đơn lưu vết quá trình từ chối kia.

## Cẩm nang chuẩn bị ứng chiến hệ thống (Production checklist)

### Ranh giới đảm bảo sự cố (Failure contract)

- [ ] Mỗi một chiếc ngoại lệ check lỗi có kiểm tra đều được đôn lên đóng dấu chỉ định tường tận khi nào được commit hay lúc nao phải xả sạch sành sanh (rollback).
- [ ] Thông điệp kết quả làm việc (Method outcome) bắt tay vừa khít với bằng chứng ghi dưới cơ sở dữ liệu trạng thái (durable state).
- [ ] Quá trình vây bắt/ngụy trang (Catch/wrap) phải cam kết mười mươi không được làm nhòe hay để quên đánh rơi tín hiệu rollback còi báo hiệu khẩn (rollback signal).
- [ ] Vòng ôm trọn bên ngoài thấm nhuần việc nội khu lõi bên trong dư dả năng lực châm mồi nổ lệnh cờ hiệu đánh bay (rollback-only).
- [ ] Những sự vụ dội ngược từ chối lường trước (Expected rejection) đều rập khuôn dùng kết quả định đoạt rõ nghĩa (result có chủ đích) thay vì quăng quật ném bừa bom Exception.

### Khâu tiến hành khóa chặt dữ liệu (Transaction)

- [ ] Lệ làng phải gắn rành rọt ngay vành đai chung ngõ ngoài trước lúc xin được giao phó (public boundary) đi theo cơ chế bảo hiểm từ proxy.
- [ ] Thao tác mồi xả SQL tạm (Flush) không tài nào bị "hớ" đánh tráo khái niệm nhầm mác như lệnh quyết chốt cứng (commit).
- [ ] Liên lạc gọi cứu viện, chuyển phát thông tin gọi ra nước xa (Remote I/O) nên khéo léo để nằm bên ngoài khỏi vòng rào giam lỏng đợi lệnh (blocking transaction) nếu có thể xê dịch sắp xếp được.
- [ ] Quy trình cắm sổ kiểu mới tinh khôi (`REQUIRES_NEW`) có kèm giấy thỏa ước tự nhận án kết quả tự thân tách biệt mười mươi (independent-outcome requirement).
- [ ] Bấm nút chạy vớt gọi thao tác lại (Retry) thiết lập rành mạch một vòng luân chuyển mộc bản (transaction mới) rồi chịu khó khuân tải lại thông tin đang neo hiện trạng dữ liệu mới mẻ đang tồn đọng.

### Nút thắt với kho dữ liệu và tiến trình trung chuyển (Database và dispatcher)

- [ ] Khâu xuất xưởng điều quân đi (Dispatcher) chỉ chộp đúng nhịp lệnh với các khối dữ liệu nằm ở ga cuối đã có sổ đỏ (terminal-ready state) hay rập mộc dán vé thùng hàng outbox (outbox event).
- [ ] Dữ liệu bổ sung ghi dưới dạng tiếp nối (Ledger append-only) kèm theo chiếc khóa riêng chỉ định định danh ngặt nghèo (business uniqueness key).
- [ ] Cung đường đã bị từ chối khước từ thì cấm được cọc chéo hay khóa xiết dòng chảy của đồng tiền tài chính (financial hold).
- [ ] Trạm thu phát hệ đa cực (Multi-instance tests) bắt buộc phải kiểm duyệt xác thực tính khả dụng bóc trần sự thật hiển thị của bộ dạng đã niêm phong xong xuôi (committed visibility).
- [ ] Cơ chế bù trừ dọn sổ tính giá (Reconciliation) phải tài giỏi tìm vạch mặt những trường hợp khống, từ chối (rejected) lại còn đi chơi trò cờ bạc được phép hoạt động khả thi trót lọt (executable mismatch).
