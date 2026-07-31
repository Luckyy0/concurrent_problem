# Giải pháp giao dịch ngắn và áp lực ngược (Short transaction and backpressure solutions)

## Mục tiêu thiết kế

```text
Tiếp nhận lượng công việc gọi từ xa vừa đủ (admit bounded remote work)
  -> lấy nhanh một bản ảnh DB (short DB snapshot), nhả ngay kết nối (connection released)
  -> đưa ra quyết định dựa trên hệ thống từ xa, nằm ngoài giao dịch (remote decision outside transaction)
  -> mở giao dịch DB cực kỳ chóng vánh (short DB transaction)
  -> khóa / nạp lại / thẩm định lại tính hợp lệ (lock/reload/revalidate)
  -> chốt commit, đánh dấu trạng thái bị lỗi thời / không làm gì cả (stale/no-op), hoặc cho phép lặp lại quy trình nếu nằm trong giới hạn cho phép (bounded retry decision)
```

> **Nói ngắn gọn:** Đừng bao giờ quàng cái lồng giao dịch (transaction) ôm trọn vào một phạm vi có độ trễ phập phù (latency domain) mà cơ sở dữ liệu hoàn toàn không có khả năng kiểm soát.

## Giải pháp 1 — Lấy bản ảnh, gọi hệ thống từ xa, thẩm định lại (Snapshot, remote call, revalidate)

### Cỗ máy đọc bản ảnh bất biến (Immutable snapshot reader)

```java
public record PaymentRiskSnapshot(
    UUID paymentId,
    UUID customerId,
    long amount,
    long version,
    PaymentStatus status
) {
    RiskSubject toRiskSubject() {
        return new RiskSubject(
            paymentId,
            customerId,
            amount,
            version
        );
    }
}
```

```java
@Service
public class PaymentSnapshotReader {
    private final PaymentOrderRepository payments;

    public PaymentSnapshotReader(PaymentOrderRepository payments) {
        this.payments = payments;
    }

    @Transactional(readOnly = true)
    public PaymentRiskSnapshot read(UUID paymentId) {
        PaymentOrder payment = payments.findById(paymentId)
            .orElseThrow();
        return payment.riskSnapshot();
    }
}
```

Phương thức (Method) trả về ngay đối tượng dữ liệu DTO trước thời điểm giao dịch dập cầu dao kết thúc (transaction end); tuyệt đối không tuồn lậu một thực thể đang bị buộc dây (managed entity) hay cấu trúc kéo lười (lazy association) mang ra cho những xử lý bên ngoài (remote phase) tùy ý dùng.

### Trình điều phối không thuộc phạm trù giao dịch (Coordinator không transactional)

```java
@Service
public class PaymentRiskCoordinator {
    private final PaymentSnapshotReader snapshots;
    private final BoundedRiskGateway riskGateway;
    private final PaymentDecisionWriter decisions;

    public PaymentRiskCoordinator(
        PaymentSnapshotReader snapshots,
        BoundedRiskGateway riskGateway,
        PaymentDecisionWriter decisions
    ) {
        this.snapshots = snapshots;
        this.riskGateway = riskGateway;
        this.decisions = decisions;
    }

    public ApprovalResult assessAndApprove(
        UUID paymentId,
        OperationDeadline deadline
    ) {
        PaymentRiskSnapshot snapshot = snapshots.read(paymentId);

        if (snapshot.status() != PaymentStatus.RISK_PENDING) {
            return ApprovalResult.alreadyResolved(
                paymentId,
                snapshot.status()
            );
        }

        RiskDecision decision = riskGateway.assess(
            snapshot.toRiskSubject(),
            deadline
        );

        return decisions.apply(snapshot, decision, deadline);
    }
}
```

Tuyệt đối cấm kỵ việc đắp thêm chú giải `@Transactional` lơ lửng trên đầu trình điều phối (coordinator). Phi vụ đọc bản ảnh (Snapshot transaction) bắt buộc phải hạ màn kết liễu gọn gàng trước khi cái máy `riskGateway.assess()` kịp ho he gào thét.

### Giao dịch chốt dữ liệu siêu ngắn (Short commit transaction)

```java
@Service
public class PaymentDecisionWriter {
    private final PaymentOrderRepository payments;

    public PaymentDecisionWriter(PaymentOrderRepository payments) {
        this.payments = payments;
    }

    @Transactional
    public ApprovalResult apply(
        PaymentRiskSnapshot assessed,
        RiskDecision decision,
        OperationDeadline deadline
    ) {
        deadline.requireCommitBudget();

        PaymentOrder current = payments
            .findByIdForUpdate(assessed.paymentId())
            .orElseThrow();

        if (!current.matchesAssessedSubject(assessed)) {
            return ApprovalResult.staleDecision(
                assessed.paymentId()
            );
        }

        if (decision.isRejected()) {
            current.reject(decision.reasonCode());
            return ApprovalResult.rejected(current.getId());
        }

        current.approve(decision);
        return ApprovalResult.approved(current.getId());
    }
}
```

Lệnh gác đền `matchesAssessedSubject()` hạch hỏi tra khảo gắt gao kiểm tra kĩ càng phiên bản (version), trạng thái (status), lượng tiền (amount) và cả mã khách hàng/nguồn đối tượng đã thẩm định (customer/decision subject). Bộ khóa (Lock) chỉ được đụng tới và trói giam cẩn thận ở thời khắc sinh tử chớp nhoáng: tải lại, tra soát, cập nhật, rồi chốt sổ (reload/check/update/commit).

### Tại sao rào chắn tính đúng đắn lại được bảo vệ kiên cố?

- Quãng thời gian ngâm mình chờ gọi mạng xa xôi (Remote wait) hoàn toàn không lấy đi bất kì sinh mạng kết nối JDBC (JDBC connection) nào.
- Những cú bẻ lái biến động dữ dội đan chéo (Concurrent change) chỉ làm hệ thống nháy mắt cười nhẹ báo cáo vênh dữ liệu (snapshot mismatch), chứ chẳng thể nào đè bẹp xé toạc được dữ liệu gốc (không bị overwrite).
- Đám loi nhoi giành giật chực hờ nuốt chửng cùng một dòng dữ liệu (Same-order contenders) chỉ phải xếp hàng nén chặt trong một khoảng không gian nhỏ bé lúc ấn chốt sổ (short commit phase).
- Kẻ chậm chân bại trận (Loser) cúp đuôi ẵm về cái mã lỗi mốc meo `staleDecision` hoặc đành phải thừa nhận sự đã rồi (stored terminal outcome), cấm không được manh động đắp lại cái báo cáo đánh giá cũ rích (không apply decision cũ).
- PostgreSQL vững vàng kiêu hãnh khẳng định vị thế nắm giữ lằn ranh quyết định số phận tối cao (authoritative state boundary).

Trình điều phối (Coordinator) có dư xăng để gọi ngược về khâu đọc bản ảnh/đánh giá lại (take snapshot/assess lại) nếu phát giác ra lỗi quyết định thối rữa mốc meo (decision stale) miễn là quỹ thời hạn tổng (overall deadline) còn rủng rỉnh túi tiền, nhưng cuộc lội ngược dòng nỗ lực (retry) lại phải được trói chặt kiềm chế có quy cũ (bounded) cũng như cuộc hội thoại hệ thống từ xa phải bảo chứng được bản chất vô can chỉ đọc lặp lại y nguyên (read-only/idempotent) tùy hỉ theo tính chất bối cảnh.

## Giải pháp 2 — Lệnh áp dụng với điều kiện hợp nhất một cục (Atomic conditional apply)

Nếu quy trình chốt (apply decision) chỉ là cuộc nhấc chân chuyển dời nhẹ nhàng của trạng thái (state transition đơn giản), chộp lấy thủ pháp tính đếm hàng biến động (affected-row count):

```java
public interface PaymentOrderRepository
        extends JpaRepository<PaymentOrder, UUID> {

    @Modifying
    @Query(
        value = """
            update payment_order
               set status = :newStatus,
                   version = version + 1,
                   risk_reason = :reason
             where id = :paymentId
               and status = 'RISK_PENDING'
               and version = :assessedVersion
               and amount = :assessedAmount
               and customer_id = :assessedCustomer
            """,
        nativeQuery = true
    )
    int applyDecision(
        UUID paymentId,
        long assessedVersion,
        long assessedAmount,
        UUID assessedCustomer,
        String newStatus,
        String reason
    );
}
```

```java
@Transactional
public ApprovalResult apply(
    PaymentRiskSnapshot assessed,
    RiskDecision decision
) {
    int changed = payments.applyDecision(
        assessed.paymentId(),
        assessed.version(),
        assessed.amount(),
        assessed.customerId(),
        decision.isApproved() ? "APPROVED" : "REJECTED",
        decision.reasonCode()
    );

    return changed == 1
        ? ApprovalResult.applied(assessed.paymentId())
        : ApprovalResult.staleDecision(assessed.paymentId());
}
```

Phép màu quyền uy lệnh UPDATE sẽ tóm vội ngay cùm khóa dòng (row lock) chỉ ở duy nhất mốc ngàm thi hành chớp nhoáng (statement/commit window). PostgreSQL mẫn cán dò lại biểu thức (re-evaluate predicate) sau cơn đợi chờ ròng rã (after waiting); hễ số dòng bị thay đổi sụt về `0` (affected rows 0) thì quy thẳng ngay sang lỗi rác mốc (stale) hoặc hư vô (no-op). Kế sách này chém tiệt đường qua lại rườm rà (giảm round trips) nhưng bù lại khâu vạch lộ trình trạng thái dọn dẹp kiểm toán (domain transition/audit logic) vẫn phải đảm đương bảo vệ tròn vẹn nghĩa vụ mười mươi.

## Giải pháp 3 — Vách ngăn kìm hãm gọi mạng (Bounded remote bulkhead) chặn trước giao dịch DB

Cấu trúc hình hài của một tấm vách ngăn phân khu (local bulkhead) đặc trưng:

```java
@Component
public class BoundedRiskGateway {
    private final Semaphore permits;
    private final RiskClient client;

    public BoundedRiskGateway(
        RiskClient client,
        RiskCapacityProperties capacity
    ) {
        this.client = client;
        this.permits = new Semaphore(
            capacity.maxConcurrentCalls(),
            true
        );
    }

    public RiskDecision assess(
        RiskSubject subject,
        OperationDeadline deadline
    ) {
        boolean acquired;
        try {
            acquired = permits.tryAcquire(
                deadline.remoteAdmissionNanos(),
                TimeUnit.NANOSECONDS
            );
        } catch (InterruptedException interrupted) {
            Thread.currentThread().interrupt();
            throw new RiskDependencyInterruptedException(interrupted);
        }

        if (!acquired) {
            throw new RiskBulkheadFullException();
        }

        try {
            return client.assess(
                subject,
                deadline.remoteTimeout()
            );
        } finally {
            permits.release();
        }
    }
}
```

Khâu xếp hàng kìm hãm chờ qua ải phân khu (Bulkhead wait) được cấp một sổ thông hành có báo tử thời gian (timeout) và bị đá văng ra rìa khỏi lãnh thổ giao dịch (nằm ngoài transaction). Định mức trần cho mỗi mảnh đất máy chủ (Per-instance limit) bắt buộc phải qua bước cân đong đo đếm tổng hòa cùng tổng số lính tráng máy chủ hiện diện (số instances) và khả năng nạp tải dốc của cỗ máy gọi từ xa (remote capacity). Một thư viện tấm chèn vách ngăn/cầu chì sập mạng (library bulkhead/circuit breaker) dư khả năng thay ngôi thế mạng đoạn mã hiện thân (implementation), nhưng nguyên lý cắm rễ/hợp đồng giao kèo tải trọng (placement/capacity contract) thì vẫn bất khả xâm phạm.

Khối máy bộ thực thi (Executor) chuyên biệt tiếp đón dòng lời gọi mạng (remote calls) cũng không được thả lỏng mà phải giăng dây thép gai bó hẹp khuôn khổ chầu chực (bounded queue) cùng một cơ chế thẳng thừng đá đích (explicit rejection policy). Đừng hòng tính chuyện chơi trò hàng đợi không đáy (unbounded queue) hòng tráo đổi từ thảm cảnh chết đói hồ chứa sang sự phình to không phanh của bộ nhớ/độ trễ (đổi pool starvation thành memory/latency growth).

## Giải pháp 4 — Cỗ máy trạng thái bất đồng bộ siêu kiên cố (Durable asynchronous state machine)

Rất đắc dụng khi lời thì thầm gọi mạng cứ dai dẳng nhức nhối (remote latency lớn), buộc lòng phải có chiếc khiên lá bùa lội ngược dòng lặp lại (durable retry) hoặc lời khẩn cầu giao dịch đồng bộ không đáng để giam cầm chết gí một mạch luồng xương máu (synchronous request không nên giữ thread).

Phát súng giao dịch 1 (Tx-1):

```sql
update payment_order
set status = 'RISK_REQUESTED',
    version = version + 1
where id = :id
  and status = 'RISK_PENDING';

insert into outbox_event(
    id, aggregate_id, event_type, payload, created_at
)
values (
    :eventId, :paymentId, 'RISK_ASSESSMENT_REQUESTED',
    :payload, now()
);
```

Luồng thợ thuyền làm mướn (Worker):

```text
xí giành lấy miếng mồi từ hộp thư đi (claim outbox) -> chốt sổ đóng dấu giành mồi (commit claim)
hú gọi hệ thống mạng xa tít tắp mà chẳng mảy may bén mảng chạm vào vỏ bọc giao dịch DB (call risk API outside DB transaction)
mở bung phát súng giao dịch 2 (open Tx-2) -> áp dụng hờ hững có điều kiện bằng mã định danh/mã phiên bản/mã gửi dội (conditional apply by payment/version/request ID) -> chốt sổ đóng hòm (commit)
```

Bảng ngã rẽ phán quyết định tội (Decision table):

| Tình cảnh thực tại (Current state) | Trùng khít đối tượng yêu cầu/cấp phiên bản chưa? (Matching request/version?) | Bản án thi hành tay thợ (Worker action) |
| --- | --- | --- |
| `RISK_REQUESTED` | Xin thưa Có | Phệt luôn lệnh ấn định (Apply decision) |
| Gõ mõ tới đích kết thúc (Terminal) | Bất chấp hệ luỵ (Bất kỳ) | Đóng dấu lặp lại/Quay xe (Replay/no-op) |
| Lệch dòng đời cấp số/lệch pha trạng thái (Version/state changed) | Khóc thét Không | Dập mác quyết định ôi thiu (Mark decision stale) |
| Phụ thuộc rớt mạng chập chờn (Dependency transient failure) | Không có khái niệm (N/A) | Nương tựa kiên cường quay vòng tái kích có hạn (Durable bounded retry) |

Khối tuần hoàn công việc (Workflow) đòi hỏi cao độ cái tính không đụng hàng (uniqueness) của hộp thư đi outbox, một gã thợ ăn hàng có tài nhai lại vạn lần chẳng xê dịch nếp nhăn (idempotent consumer), bộ máy tời kéo cẩu vực lại những xác chết tắc nghẽn (stuck-state recovery) và tầm nhìn siêu việt thấu quang minh (observability). Bù lại, hệ thống nhàn nhã thanh thoát chẳng còng lưng vác nợ ôm cái của nợ mồi chài (request/DB resources) lặn lội xuyên qua đầm lầy độ trễ mạng dài đằng đẵng (remote latency).

## Giải pháp 5 — Ngân sách thời gian chốt chặn và Bức tường lửa cảnh vệ DB (Timeout budget và database guardrails)

Pháp thuật Timeouts tạo khiên giam cầm rủi ro thảm họa (contain failure):

```text
thời hạn phán quyết sinh tử của yêu cầu tổng thể (overall request deadline)
  ├─ mốc lọt khe điểm xét duyệt mạng + kinh phí bao gọi mạng (remote admission + remote call budget)
  ├─ kinh phí chen lấn xô đẩy tranh mượn hồ (pool acquisition budget)
  ├─ kinh phí sập bẫy chờ khóa/thực thi công đường án kiện (lock/statement budget)
  └─ lằn ranh đỏ lùi quân cuốn gói biên độ an toàn/kết quả phản hồi (rollback/response margin)
```

Giao dịch tí hon cực ngắn lách luật của PostgreSQL (PostgreSQL short transaction) đôi khi vận dụng tài mọn hàng rào địa phương (local guardrails):

```sql
set local lock_timeout = '250ms';
set local statement_timeout = '500ms';
```

Mấy con số hiện diện chỉ đắp đổi che mắt minh họa làm phép màu trong thử nghiệm (minh họa test/configuration). Con số xương máu vắt ra từ thực chiến (Production values) khăng khăng phải được chế ra chiết xuất (derive) từ chỗ đếm lùi thời hạn còn chót (remaining deadline) và luôn phải được bảo vệ cài cắm bởi bộ máy quy chuẩn uy quyền (trusted configuration), cực lực phản đối vụ nhồi thẳng thừng nguyên xi cặn bã nhét từ phễu nhập liệu người dùng (raw user input) xả xuống SQL.

Chiếc đồng hồ ngắt mạch của giao dịch Spring (Spring transaction timeout) sắm vai như bức tường phụ thứ yếu (thêm một guardrail) bảo lãnh cho nội bộ khu lõi dữ liệu, chứ chẳng mang theo tài hèn sức mọn đòi thay thế gạt phăng đi chốt chặn kết nối của hệ gọi mạng (remote client timeout). Chốt khóa ngắt chờ `connectionTimeout` tại đáy hồ cũng lèm bèm chỉ thắt cổ (giới hạn) đám dân đen cầu mượn ngóng trông (pending borrower wait), chẳng phải sinh ra để cắt tiết (không phải query/transaction timeout) câu truy vấn bạo ngược.

Chẳng may đứt phựt gục ngã (Sau timeout/error), phải dập tắt thu hồi giao dịch khẩn cấp (rollback transaction) trước khi màng tới chuyện hồi sinh tái khám (retry). Nghiêm cấm hâm nóng tái đấu (Không retry) hòng tranh giật ngắt khóa/mượn hồ (acquisition/lock timeout) trong nháy mắt (ngay lập tức) nếu biết tòng tọc hồ chứa/cơ sở dữ liệu đang phì nộn bức tử (vẫn saturated).

## Giải pháp 6 — Phanh hãm ngược ngay tại đầu ngõ phễu hứng (Backpressure ở ingress)

Đến lúc mà lũ tải trọng công việc rủi ro ngụp lặn lơ lửng (in-flight risk work) ăn mòn chạm nóc ngân sách định mức (đạt budget):

- hất văng sập mặt ngay từ đầu vòng gửi xe (reject sớm) với câu báo lỗi ngập lụt quá tải nhã nhặn (overload response phù hợp);
- tống giam vào rọ nhốt hàng đợi siết cổ (queue bounded) đi kèm bộ óc tính sổ xét duyện canh thời hạn chót (deadline-aware admission);
- trút vứt bỏ (shed) đám việc ruồi muỗi phận hèn (low-priority work);
- khóa chặt nọng tay đám van lạy kêu gào năn nỉ cạy cửa lại (stop server-side retries);
- dập cầu dao phá tan mạch (open circuit) một khi bè lũ lính đánh thuê đâm ra bệnh hoạn yếu ớt (dependency unhealthy);
- bơm gửi kèm thẻ bùa `Retry-After` (propagate `Retry-After`) duy chỉ khi bộ óc luật lệ máy khách năn nỉ đủ tư cách an toàn rành rọt (client retry policy an toàn).

Bộ sàng lọc khắt khe (Admission) xông pha giáp lá cà trước khi giao dịch DB (DB transaction) manh nha trỗi dậy. Phanh hãm ngược (Backpressure) đã hô biến sự mòn mỏi bất tận không đáy héo hon chờ tài nguyên (unbounded resource wait) xoay vần thành kết cục đanh thép sòng phẳng và tường tận hiển thị (bounded, observable outcome).

## Đóng khung cỡ hồ (Pool sizing)

Kích thước hồ (Pool size) đơn thuần là món đồ chơi kiểm soát thể tích dung lượng (capacity control), chẳng phải lưỡi gươm diệt gốc rễ tật nguyền (không phải root fix). Cần rà soát (Xem xét):

- lượng tiền dư dả ngân quỹ móc khóa kết nối (database safe connection budget);
- số lượng dàn trải máy chủ quân lính/lá chắn dự bị tử nạn/bơm rút lính tự động (instance count/failover/autoscaling);
- mật độ lấn sân cọ xát của bầy giao dịch ruồi muỗi thông thường (normal short transaction concurrency);
- khoảnh sân dự trữ dành riêng cho ban bệ quản lý điều tiết/cải cách di dời (reserved operational/migration capacity);
- sức thở CPU/mặt đường truyền I/O gánh câu lệnh truy vấn (query CPU/I/O), chứ đừng có chăm chăm nhòm ngó mỗi số lượng luồng réo gọi (not only request threads);
- bức phác họa phân phối độ lì lợm nhây bám thời lượng kết nối (connection usage duration distribution);
- chặng đường ngắc ngoải đợi mượn/chực chờ (pool pending/acquisition latency).

Tổng phình to của cái hồ đã khai báo (Tổng configured pool capacity) không được phép âm thầm ăn trộm lấn át vọt qua ngân sách kết nối của ngài PostgreSQL (PostgreSQL budget). Khảo nghiệm tra xét (Benchmark) phải nhè đúng đám lưu lượng nặng đô tiêu biểu (workload đại diện); đập tan ảo mộng đi tìm hòn đá tảng "CPU × một hằng số cố định" (universal “CPU × constant”) mà áp dụng xằng bậy cho mọi mặt trận cơ sở dữ liệu.

## Nước cờ đi tắt tối ưu lặp lại trên cùng đơn hàng (Same-order duplicate optimization)

Vừa giật vội được mộc bản khóa dòng (Sau khi acquire lock), nghía coi dòm ngó xem có đụng trúng bảng trạng thái chung kết đóng hòm (terminal state) nào không trước khi dấn thân gọi mạng phương xa (trước remote work). Lợi thế từ việc chẻ ranh giới (Với split boundary), gã thợ coi sóc bản ảnh (snapshot reader) dư sức nhổ toẹt ngay kết quả chung cuộc đã đóng kho (stored terminal result ngay). Thậm chí nấp đằng sau lúc hạ bút chốt (Commit phase), cũng cần tái soát (recheck) để trừ khử mầm loạn đấu đá ganh đua (race).

Chìa khóa chống trùng/dấu tích lưu vết quyết định (Idempotency key/command result) mở đường cho tên đưa thư lại (duplicate request) tự tin sao y bản chính dội lại tin báo (replay) thay vì hì hục dắt trâu gõ mõ đánh giá lại rủi ro (đánh giá risk lại). Nó tuyệt nhiên chưa thể chiếm lĩnh ngai vàng đập phá thế chân cho cơ chế phòng hộ kìm kẹp tường thành vách ngăn/vỏ hồ (bulkhead/pool protection) bảo vệ trước hàng ngũ những yêu cầu xa lạ, riêng rẽ (distinct requests).

## Hành vi đứt gánh (Failure behavior)

| Mối họa ngầm (Failure) | Giao dịch còn há mồm mở toác hớ hênh lúc hóng hớt chờ đợi? (Transaction đang mở khi wait?) | Quả báo kết cục (Outcome) |
| --- | --- | --- |
| Đâm sầm vách ngăn kiệt sức (Bulkhead full) | Thề Không | Hất văng, không thèm ngó ngàng bôi bẩn chọc ngoáy DB (Fail-fast/no DB mutation) |
| Lỗi ngắt chờ xa xứ (Remote timeout) | Kêu Không | Dẹp tiệm bước hạ màn (No commit phase) |
| Dòng đời lật mặt giữa chừng lúc đàm phán xa xứ (State changed during remote) | Loay hoay chui vô khe hẹp đoạn cập nhật (Chỉ short apply Tx) | Đồ ôi thiu hỏng bét/đứng im (Stale/no-op) |
| Chờ mốc meo khóa chặn ở khâu thi hành (Lock timeout in apply) | Dính chấu Có, tí xíu (Có, short) | Đập gẫy cuốn gói (Rollback); quy hoạch khống chế dập lại (bounded policy) |
| Trắng tay xin không nổi bát nước hồ (Pool acquisition timeout) | Đã vớ được miếng kết nối nào đâu (Chưa có connection) | Nổ tung quá tải/thất thủ (Overload/fail) |
| Chàng thợ giao đi giao lại (Worker redelivery) | Cấm lách vào giao dịch gọi xa (Không qua remote Tx) | Ấp ủ đắp lại/hát lại y nguyên không mảy may xê dịch (Idempotent apply/replay) |

## Bàn cân đo đong đếm đánh đổi được mất (Trade-off comparison)

| Cán cân lựa chọn (Lựa chọn) | Mức độ bám rễ dính dớp kết nối/Khóa dòng (Connection/lock duration) | Mô hình kéo ghì độ trễ (Latency model) | Chỉ số rối rắm (Complexity) | Bến đỗ lý tưởng (Best fit) |
| --- | --- | --- | --- | --- |
| Lôi hệ gọi xa vào lồng giao dịch (Remote inside Tx) | Dai dẳng y như dây thun chờ mạng (Bằng remote latency) | Ép buộc đồn dập (Synchronous) | Hời hợt ảo tung chảo (Thấp bề mặt) | Phá sản, tẩy chay (Không khuyến nghị) |
| Dập bản ảnh + xét duyệt nắn gân lại (Snapshot + revalidate) | Lóe sáng chớp nhoáng vòng lặp DB (Short DB phases) | Ép buộc đồn dập (Synchronous) | Tàm tạm sương sương (Vừa) | Sứ mệnh hóng hớt chỉ đọc nhòm ngó (Read-only remote decision) |
| Phạt đòn ập vào cập nhật đi kèm lệnh cấm (Atomic conditional apply) | Nhanh như chớp (Rất ngắn) | Ép buộc đồn dập (Synchronous) | Tàm tạm sương sương (Vừa) | Pha bẻ lái dời trạng thái nhẹ hều (Simple state transition) |
| Trạm trung chuyển máy trạng thái bất đồng bộ đúc bền (Async state machine/outbox) | Lóe sáng chớp nhoáng vòng lặp DB (Short DB phases) | Kiểu gì cũng tới (Eventual) | Cao chót vót (Cao) | Đánh đu bám váy phụ thuộc lê lết/ẩm ương mờ mịt (Long/unreliable dependency) |
| Chỉ nhăm nhe khoét to bể bơi hồ (Bigger pool only) | Không nhúc nhích (Không đổi) | Sống nay chết mai đợi thời (Failure bị dời) | Lùn tịt (Thấp) | Tinh chỉnh vuốt ve công lực sau khi lấp vá lỗ hổng ranh giới (Capacity tune sau boundary fix) |

## Kim chỉ nam phán xử cho vấn nạn này (Recommendation cho case này)

1. Giương cao ngọn cờ Giải pháp 1 (Solution 1) cùng với hòn đá tảng bất khả xâm phạm `PaymentRiskSnapshot` (immutable `PaymentRiskSnapshot`).
2. Trấn thủ cổng vách ngăn/giới hạn gọi mòn mỏi (bulkhead/remote timeout) án ngữ chễm chệ ngay lối vào trước khu lăng tẩm giao dịch hạ bút (trước commit transaction).
3. Tra tấn thẩm tra gắt gao (Revalidate) lại vòng đời/vóc dáng/danh tính (version/status/subject) ẩn mình dưới lớp kén khóa dòng chớp nhoáng (short row lock) hay đòn cập nhật gài bẫy (conditional UPDATE).
4. Phun trả kết quả hư hao ôi thiu/án binh bất động (Return stale/no-op) cho đàng hoàng sáng sủa; những cuộc đấu trí nài nỉ định đoạt lại trong vành đai khống chế (bounded re-assessment) chỉ được đặc cách một khi quỹ hầu bao thời hạn vẫn sung túc (khi deadline cho phép).
5. Mang giải pháp hộp thư đi gánh vác bất đồng bộ (async outbox) ra dùng nếu như điều khoản hợp đồng độ mượt mà từ xa (remote SLA) hắt hủi không thèm khớp với nhịp đập đồng thuận liền tay (synchronous request).
6. Uốn nắn nới chỉnh mức hồ/giới hạn ngắt (Tune pool/timeouts) chỉ sau khi mà cơn bệnh phù nề của tuổi thọ giao dịch (transaction duration) đã bị kìm hãm bóp ngắn không thương tiếc.

## Cẩm nang chuẩn bị ứng chiến hệ thống (Production checklist)

### Ranh giới (Boundary)

- [ ] Nghiêm cấm hóng hớt gọi mạng/đợi luồng (Không remote/executor wait) chen chân mọc rễ trong lồng giao dịch.
- [ ] Chối phăng tuyệt giao hắt hủi thực thể giam lỏng (Managed entity) ra khỏi ranh giới nhòm ngó dị đồng bộ/ngoài hành tinh (không đi qua async/remote phase).
- [ ] Trạm hạ bút chốt hạ giao dịch (Commit transaction) bắt buộc xốc nách lôi cổ ra đo đạc tra khảo lại vóc dáng hiện tại (reload/revalidate current state).
- [ ] Ổ khóa bảo bối dòng dữ liệu (Row lock) chỉ chực chờ hé sáng duy nhất trong phút chốc làm phép cải biến (tồn tại trong short mutation phase).
- [ ] Phép hủy diệt đoạt mạng (Cancellation) lúc nào cũng dắt mũi lôi tuột hệ thống về chốn dọn dẹp quy mô gom hẹp (bounded cleanup).

### Hầu bao dung tích và giới hạn lùi (Capacity và timeout)

- [ ] Lô cốt vách ngăn ngoài luồng/hàng đợi siết cổ (Remote bulkhead/queue bounded) phục binh đi trước rào chấn hỏa lực bắn DB (trước DB work).
- [ ] Quỹ thời gian tổng tài (Overall deadline) rót ngân lượng chẻ đều cho từng cứ điểm mặt trận (phân bổ cho mọi phase).
- [ ] Bầy đàn ngắt báo hại hồ/khóa/thực thi/gọi mạng (Pool/lock/statement/remote timeouts) gộp chung lại cấm tiệt vượt mặt ngân khố tổng (không cộng vượt deadline).
- [ ] Bề thế dạ dày hồ mượn (Pool capacity) đếm cộng dồn bao hàm đủ trọn vẹn bè lũ cứ điểm (tính theo toàn bộ instances).
- [ ] Hạt giống hồi sinh nài nỉ (Retries) ủ sẵn chiêu lùi bước giãn cơ (có backoff) và chớ hề bơm thêm nọc độc thổi phồng núi gánh (không khuếch đại overload).

### Huyết mạch dữ liệu và guồng máy vận hành (Data và vận hành)

- [ ] Bản án đã thiu mốc gỉ rét (Stale decision) cấm cửa không được lưu danh hạ bút (không được apply).
- [ ] Đám trùng lặp quấy rối (Duplicate request) hất ngược bật dội ra (replay) dựa vào phù hiệu lệnh/kết quả chống mục nát (durable command/result) khi cấp thiết.
- [ ] Guồng quay vòng bất đồng bộ (Async workflow) ốp ngay chiêu thức hộp thư đi/phục hồi bất hoại thần công (outbox/idempotent recovery).
- [ ] Biểu đồ đo lường vạch hồ (Pool metrics) móc xích kết đôi (correlate) rành rọt với chỉ số rề rà của vòng gọi mạng/khóa kẹp/sống thọ giao dịch (remote/lock/transaction duration).
- [ ] Miếng cơm manh áo của giới quản trị/phu phen di cư dữ liệu (Admin/migration capacity) yên vị nằm ngoan ngoãn trong quỹ bao che của mẹ PostgreSQL (nằm trong PostgreSQL budget).
