# Thiết lập ranh giới đúng và các đòn fail-fast (Correct boundary and fail-fast options)

## Mục tiêu thiết kế

Một kế sách vẹn toàn (giải pháp đúng) phải chiếu rọi tỏ tường (làm rõ) bốn mấu chốt:

1. đúng cánh cổng công khai (public entry) nào mới có quyền năng đẻ ra (tạo) cái giao dịch vật lý (physical transaction);
2. mức cô lập thực dụng (effective isolation) nào được phái đi cản địa bảo vệ toàn vẹn khối nghiệp vụ sinh tử (business unit);
3. những lời réo gọi vọng từ hốc kẹt (inner call) kia được quyền đu bám (join) hay ngạo mạn vỗ ngực xưng tên lập bang phái riêng (tạo independent transaction);
4. thảm cảnh lệch pha (mismatch) sẽ bị hất gáo nước lạnh tước mạng tức tưởi (fail-fast) hay lại giả điếc lấp liếm vuốt ve (được chấp nhận có chủ đích).

> **Nói ngắn gọn:** quăng ngay cái lệnh rào cản (isolation) vào đúng ngay cái ổ sinh ấp trứng (nơi transaction bắt đầu), rồi xắn tay áo kiểm tra độ chịu đòn thật (kiểm thử setting thật) và nhặt xác kết cục trần trụi (outcome thật).

## Giải pháp 1 (Solution 1) — Cắm mốc cô lập ngạo nghễ ở ranh giới tình huống chóp bu (outer use-case boundary)

Đây là lối thoát hiểm chân ái (lựa chọn mặc định) một khi toàn thể bản tấu chương (report) phải dựa dẫm bấu víu vào một thước phim đóng băng (stable snapshot):

```java
@Service
public class ReportFacade {
    private final SnapshotQueryService snapshotQueries;

    public ReportFacade(SnapshotQueryService snapshotQueries) {
        this.snapshotQueries = snapshotQueries;
    }

    @Transactional(
        isolation = Isolation.REPEATABLE_READ,
        readOnly = true
    )
    public PriceReport generate(long productId) {
        return snapshotQueries.readTwice(productId);
    }
}
```

Bọn nô bộc nội bộ (Inner service) thậm chí đâm ra thanh nhàn chẳng thèm đeo bùa (không annotated) miễn chúng chỉ bị sai phái nội trong cái chiến dịch này (trong use case này):

```java
@Service
public class SnapshotQueryService {
    private final PriceRepository prices;
    private final SnapshotProbe probe;

    public PriceReport readTwice(long productId) {
        long first = prices.getPrice(productId);
        probe.afterFirstRead();
        long second = prices.getPrice(productId);
        return new PriceReport(first, second);
    }
}
```

Bằng không, lũ oắt con (inner method) hãy mạnh dạn đeo hàm `MANDATORY` hòng giương oai bóc phốt (document) bắt trói tên đầu sỏ (caller) phải è cổ ra mở giao dịch:

```java
@Transactional(propagation = Propagation.MANDATORY, readOnly = true)
public PriceReport readTwice(long productId) {
    // ...
}
```

Nhãn dán `MANDATORY` chỉ giỏi thói rình mò (kiểm tra) dò dẫm hơi hám giao dịch tồn tại chứ cấm có tài cán đoái hoài đóng dấu ấn (không tự xác nhận) vào cái mức cô lập. Chốt lại, chóp bu vòng ngoài (outer boundary) cộng hưởng bãi thử nghiệm máu (integration test) vẫn muôn đời là tòa án phán quyết tối cao (source of truth).

### Thiên thời địa lợi (Khi phù hợp)

- Tờ tấu chương/Chiến dịch (Một report/use case) đòi sống đòi chết cần đúng một tấm ảnh đóng băng (cùng snapshot) từ phôi thai đến lúc nhắm mắt (từ đầu đến cuối).
- Lưỡi gươm Chốt hạ/Hủy diệt (Commit/rollback) phải vung lên chặt đứt bao vây trọn gói mọi ngóc ngách của chiến dịch (toàn bộ operation).
- Kẻ phát lệnh (Caller) tuyệt nhiên không được lén lút múa may ngầm định (ngầm định) cái ranh giới cô lập.

### Khắc cốt ghi tâm (Lưu ý)

Phương thức công đài (Public method) bắt buộc phải trườn qua lăng kính mờ ảo của Spring proxy. Tuyệt nhiên cấm múa rìu qua mắt thợ (Không gọi) trò `this.generate(...)`, tuyệt giao thói giấu giếm cắm bùa (không đặt annotation) trên đầu mấy thằng lâu la núp lùm (private helper), và chớ dại đi đục khoét khởi tạo dịch vụ bừa bãi bằng phép `new`.

## Giải pháp 2 (Solution 2) — Đúc bản ảnh cô thế bơ vơ bằng `REQUIRES_NEW`

Chỉ lôi ra múa (Dùng) khi đường cày bản ảnh (snapshot query) thực sự là một hòn đảo biệt lập (independent unit):

```java
@Service
public class SnapshotQueryService {

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        isolation = Isolation.REPEATABLE_READ,
        readOnly = true
    )
    public PriceReport readTwice(long productId) {
        // vớt lẹ hai mẻ lưới trong cùng một thước phim độc thủ (two reads in one independent snapshot)
    }
}
```

Lời thỉnh cầu (Call) phải lạng lách chui tọt qua một cái ngõ bean proxy khác bọt. Gặp lúc giao dịch đại ca (outer transaction) chình ình cản lối, Spring xách cổ treo lơ lửng nó lên (suspend), nã đạn xé vỏ bọc ra một kết nối/giao dịch vật lý mới tinh tươm, xong xuôi dọn dẹp (sau khi inner hoàn tất) thì mới thả gã đại ca xuống cho chạy tiếp (resume).

Chỉ bốc lá bài (cách) này khi năm ngón tay (điều sau) úp mở đều điểm huyệt chí tử (đúng):

- Đóng hòm/hủy diệt một cõi (independent commit/rollback) là cái thiết quân luật (requirement);
- Bề tôi (inner) không có nhu cầu chui rúc hóng hớt cái đống hổ lốn chưa chốt (uncommitted work) của đại ca (outer);
- Vũng chứa nước (connection pool) vẫn còn khe hở nhét vừa (headroom) đám nheo nhóc sinh sau đẻ muộn đan chéo (nested concurrent calls);
- Đám lính lác giam cầm (lock ordering) đã được săm soi mổ xẻ rạch ròi;
- Kịch bản gã đại ca hộc máu quay đầu (outer rollback sau đó) cũng chả màng phá bĩnh bôi xóa chiến tích của tiểu nhân (không cần đảo kết quả inner).

Giả như cái tấu chương (report) chỉ ăn hại mỗi việc xem suông (chỉ đọc), cái "ngai vàng biệt lập" (commit độc lập) đỡ ngứa mắt dọn dẹp mấy đống nát nửa vời (partial write) nhưng hỡi ôi nó vẫn ngang nhiên thay máu bản đồ ảnh tĩnh (thay snapshot scope) cùng nguồn sống (resource usage).

## Giải pháp 3 (Solution 3) — Khua chiêng múa trống dùng `TransactionTemplate` đúc ranh giới nhảy múa (động)

Ngay lúc cái chốt cô lập (isolation) gió chiều nào theo chiều ấy bấu víu (phụ thuộc) vào hệ loại nghiệp vụ (operation type) hoặc những cung đường lượn lờ (flow) chống đối (không phù hợp) kiểu khoác áo proxy annotation, hãy tự tay tạc ranh giới bằng mã lệnh sắt thép (đặt boundary programmatically):

```java
@Service
public class SnapshotReportService {
    private final TransactionTemplate snapshots;
    private final SnapshotQueryService queries;

    public SnapshotReportService(
        PlatformTransactionManager transactionManager,
        SnapshotQueryService queries
    ) {
        this.snapshots = new TransactionTemplate(transactionManager);
        this.snapshots.setPropagationBehavior(
            TransactionDefinition.PROPAGATION_REQUIRES_NEW
        );
        this.snapshots.setIsolationLevel(
            TransactionDefinition.ISOLATION_REPEATABLE_READ
        );
        this.snapshots.setReadOnly(true);
        this.queries = queries;
    }

    public PriceReport generate(long productId) {
        return snapshots.execute(status -> queries.readTwice(productId));
    }
}
```

Nhược bằng không khao khát cái ngai vàng độc cô (independent unit), cứ réo gọi (gọi) cái template ngay lúc thâm tâm đinh ninh chưa hề có mống giao dịch nào ngự trị và tự đút túi xài `PROPAGATION_REQUIRED`. Cấm tiệt trò nặn nhồi lồng ghép template (dùng template lồng nhau) mà không có lấy một mẩu giấy (không document) khai báo ngọn ngành luân lý đình chỉ/sáp nhập (suspend/join semantics).

Nước cờ sáng (Ưu điểm) là cái nôi khai sinh (creation point) trần trụi sờ sờ vạch lá tìm sâu ngay giữa đống mã (code). Đòn chí tử (Nhược điểm) lại là mớ lề luật giao dịch (transaction policy) bị quậy tung mù mịt vào cái mâm điều binh khiển tướng (orchestration) buộc lòng phải đào bới soi mói (test kỹ hơn) giữa muôn trùng ngã rẽ trốn thoát (nhiều exit path).

## Giải pháp 4 (Solution 4) — Cởi trói mắt thần vạch trần (validation) đám giao dịch ma cũ (existing transaction)

Đòn thất sát (Fail-fast) soi rọi phanh phui những lời khai báo bề tôi (inner declaration) ngu xuẩn lộn xộn (không tương thích):

```java
@Configuration
class TransactionConfiguration {

    @Bean
    JpaTransactionManager transactionManager(EntityManagerFactory emf) {
        JpaTransactionManager manager = new JpaTransactionManager(emf);
        manager.setValidateExistingTransaction(true);
        return manager;
    }
}
```

Giây phút tên đại ca (outer) `READ_COMMITTED` ngạo nghễ réo tên thằng tiểu bối (inner REQUIRED) `REPEATABLE_READ`, kẻ canh gác (manager) lập tức gạt phăng phũ phàng (từ chối join) thay vì hèn nhát câm nín hùa theo (silently chạy) nhúng chàm ở cái xó xỉnh mục nát (weaker level).

Trong lòng chảo Spring Boot khói lửa (project thực tế), phải khư khư ôm ấp bảo vệ (cần giữ nguyên) lũ con cưng chỉnh sửa (customizations) đang hì hục đắp nặn cho quan ngự y giao dịch (transaction manager như DataSource, JPA dialect, rollback policy, observability). Lỡ Boot đã trót nặn ra tay sai (manager) rồi, tài tình nhất là hóa phép tô vẽ lại đệ tử hiện hữu (customize bean hiện hữu) thay cho thói đẻ thêm một thằng thứ hai (khai báo một bean thứ hai) nhọc xác. Bám gắt gao (Xác nhận) chỉ chừa đúng một vị quản gia độc tôn (manager) hòng cung phụng cho cái lệnh `@Transactional`.

### Chiếc kính lúp (Validation) không có tuổi lấn sân bản thiết kế ranh giới (boundary design)

Đòn kiểm chứng (Validation):

- bới bèo ra bọ (phát hiện conflict);
- bất tài vô dụng chả biết nâng cấp (không nâng isolation);
- hết thuốc chữa mù màu trong việc vỗ ngực cam đoan đám mã vô danh (không annotated) đang mải miết phi ngựa đúng làn (đang chạy trong đúng level);
- chọc ngoáy phá nát bấy nhầy bới móc ra ti tỉ thứ (có thể làm lộ) chắp vá lỗi lầm (mismatch) thâm căn cố đế hồi xuất binh (rollout).

Phải cởi trói sớm (Nên bật sớm), xua quân dập lửa (chạy) bằng binh đoàn tích hợp (integration suite), kiểm kê soi xét hạch toán mọi lùm cây bụi cỏ (nested scopes) rồi sau cuối mới được hất ra giang hồ (rollout production) nếu hệ thống xương mục (legacy) còn giăng đầy những ngõ ngách nối chui rúc (implicit joins).

## Giải pháp 5 (Solution 5) — Gõ đầu cấu hình mặc định bể chứa (datasource default) hóa thân pháp lệnh toàn cục (global policy)

Hoàn toàn thừa sức ốp thẳng trần mặc định (đặt default) PostgreSQL/session trong trường hợp hầu như (phần lớn) đám giao dịch cùng bâu xúm chầu chực xin chung một ân huệ (cùng cần một level). Ngặt một nỗi, mặc định (default) cốt lõi là thứ luật làng (deployment policy), không được đôn vinh làm thần chú độc tôn (cách duy nhất) hòng bôi trát diễn đạt cái xương sống cốt tử (business-critical invariant) của vương quốc.

Rút kiếm ra xài (Nếu dùng), thì phải thề (cần):

- tạc tượng khắc chữ đóng khuôn rập khuôn (cấu hình giống nhau) đồng quy tận mọi môi trường (environments);
- đập bàn dằn mặt vũng chứa (pool) bắt nó gột rửa (reset) lại tấm áo (session state);
- đem `Isolation.DEFAULT` ra đòn đọa đày băm vằm (test) thu về cho bằng được chiến lợi phẩm như ý (effective value mong muốn);
- dán mắt soi mói gắt gao độ trơn tru tải trọng (throughput), cảnh xô xát đổ máu (contention) và những cái chết do đứt đoạn chuỗi hàng (serialization failures);
- vẫn phải trần truồng thẳng thừng (vẫn explicit) xắn tay áo ở ngay cửa ải (boundary) mang trọng bệnh ứ đọng riêng (yêu cầu đặc biệt).

Cuộc cách mạng lật đổ dời non lấp biển (Đổi toàn hệ thống) giật `READ COMMITTED` bay lên cõi cao vợi hơn (cao hơn) dư sức kích nổ những cuộc từ chối hộc máu/cắn răng thử lại (abort/retry) và bóp nghẹt nhốt giam tấm bản đồ lâu dài (giữ snapshot lâu hơn). Cấm tiệt thi hành trảm quyết vội vã như một món thuốc vá víu tạm bợ (hot fix) mà khinh thường khâu đo lường thực chiến (thiếu benchmark).

## Dẫu lên cõi thượng thừa nhưng cô lập vẫn trơ trọi bất tài (Khi isolation chưa đủ)

Bức tường băng (Stable snapshot) che chắn cản gió độc đứt gãy luồng đọc (read consistency), nhưng cái luật thép dính dáng sinh tử máu me (invariant có writes) có khi thèm khát (có thể cần):

- thanh kiếm độc tôn/truy quét/đá văng (unique/check/exclusion constraint);
- nhát chém cập nhật bọc thép nguyên khối (atomic conditional update);
- ngai vàng ảo tưởng niềm tin (optimistic version);
- bùa cấm `SELECT ... FOR UPDATE`;
- bóp cổ khóa mõm quyền sở hữu riêng tư/thế lực ứng dụng (advisory/application ownership phù hợp);
- con ngáo ộp `SERIALIZABLE` cặp kè khâu dội bom lại bị xiết chặt (bounded retry).

Đơn cử, đôi pháo thủ (hai transaction) chập chững liếc dọc "vẫn trống chỗ" (đọc “còn chỗ”) đoạn hung hăng nã đạn (insert) đâm ra xéo nát cái giới hạn thùng chứa (capacity) dù chúng đã trát phấn an bài nhắm chung một kẽ nứt (mỗi transaction đọc một snapshot ổn định). Hái lựa phương kế (mechanism) bám sát lề luật (invariant) và nhát đâm chém nhau (write conflict), chớ đừng mê muội nhắm mắt ngây ngô ước ao mỗi cái "đọc trơn tru" (đọc nhất quán).

## Lên bàn cân đo đong đếm (So sánh lựa chọn)

| Cán cân lựa chọn (Lựa chọn) | Khối gạch ranh giới (Physical boundary) | Bức tường cô lập còn thở? (Isolation có hiệu lực?) | Trả giá nát nhà (Trade-off chính) |
| --- | --- | --- | --- |
| Cô lập chễm chệ vương miện ngoại vi (Isolation trên outer entry) | Độc đinh một phát (Một transaction) thầu nguyên mâm use case | Gật đầu Có | Bờ cõi hiển hiện (Boundary rõ); giao dịch cắm rễ dằng dặc (transaction có thể dài) |
| Lính hầu vòng trong (Inner REQUIRED) | Ăn theo chui lủi (Join outer) | Chỉ le lói hên xui khi thằng lớn cho phép (outer tương thích) | Bấp bênh thả trôi theo ngõ vào (Dễ phụ thuộc call path) |
| Sát thủ vòng trong (Inner REQUIRES_NEW) | Độc lập giang sơn (Transaction độc lập) | Gật đầu Có | Thòng lọng treo (Suspend), đút lót thêm cửa (connection), án mạng ắt một trong hai (independent outcome) |
| Khay khuôn đúc bùa (`TransactionTemplate`) | Trần trụi vạch vôi từng viên (Explicit theo block) | Gật đầu Có (mỗi khi viên kia mọc rễ giao dịch) | Đổ đống phế liệu điều binh (Nhiều orchestration code) |
| Kính lúp soi dĩ vãng (Validate existing) | Vô dụng chả dựng được nhà mới (Không tạo boundary mới) | Điếc chả biết nâng cấp (Không upgrade); xử trảm lệch pha (fail mismatch) | Phơi bày đống giẻ rách lậu (Có thể làm lộ lỗi legacy) |
| Ngài mặc định bệ xí (Datasource default) | Mở trói cho bá tánh DEFAULT (Mọi DEFAULT transaction) | Dật dờ tùy thổ nhưỡng (Có theo environment) | Bãi mìn rải thảm ác liệt (Blast radius lớn), giao kèo đứt quai (contract kém cục bộ) |

## Kim chỉ nam phán xử cho vấn nạn này (Recommendation cho case này)

1. Tống cổ cái bùa `REPEATABLE_READ` trèo lên ngự trị (lên) đầu đài `ReportFacade.generate()`.
2. Kẹp chặt (Giữ) nhốt cả đôi SELECT quấn quít (trong) chung nôi giao dịch vòng ngoài (outer transaction).
3. Đày đọa tên chóp bu truy vấn (query service) xuống hàng lính hầu (helper) hoặc ban cho cái trát `MANDATORY`.
4. Kích hoạt còi báo động (Bật existing-transaction validation) sau một hồi sục sạo rà mìn (inventory) mấy ngõ cụt (nested calls).
5. Moi gan móc ruột đo thực tài cô lập (Query effective isolation) trong đấu trường lửa PostgreSQL (integration test).
6. Đóng đinh (Assert) kết quả tờ sớ (report) mười mươi là `100/100` phơi mình dưới những cú nhào nặn (interleaving) đã trói chặt.
7. Đừng có ho he ban phát đặc ân làm lại (Chỉ thêm retry) trừ khi vòng kim cô (level) lọt bẫy có quyền sát nghiệp (abort) đàng hoàng (hợp lệ) và cái phép kia trầy vi tróc vẩy chục lần vẫn trơ trơ (idempotent).

## Cẩm nang chuẩn bị ứng chiến hệ thống (Production checklist)

### Ranh giới (Boundary)

- [ ] Cửa ngõ (Public entry) bị ép phải đút lót trườn qua (gọi qua) Spring proxy.
- [ ] Lá chắn (Isolation) chễm chệ chôn chân ở ngay cái nôi nặn ra (physical transaction creation point).
- [ ] Bức phả hệ lan truyền (Propagation) của muôn vàn ổ chuột vòng trong (inner scope) bị lột trần đưa lên giấy (document).
- [ ] Ám tiễn `REQUIRES_NEW` chỉ ném vào mặt kẻ thực sự đáng xưng vương độc lập (independent unit).
- [ ] Chối phăng tuyệt giao hắt hủi mấy cái vòi tự sướng (self-invocation) hay cái bùa chui rúc giấu giếm (private annotation).

### Kho báu Database (Database)

- [ ] Lõi sức mạnh cô lập của quỷ PostgreSQL (PostgreSQL effective isolation) được đóng dấu giáp lai (xác nhận).
- [ ] Mặc định của bè lũ trạm nối/hơi thở (Datasource/session default) rập khuôn sao y (giống nhau) trên mọi mặt trận (environments).
- [ ] Mốc sống (Transaction duration) và mấy mảnh băng phong vĩnh cửu (long-lived snapshots) bị đè ngửa ra theo dõi gắt gao.
- [ ] Án phạt SQLSTATE `40001` trang bị thêm bùa hồi sinh thắt trói (retry policy bounded) nếu đường cùng (cần).
- [ ] Gông cùm/khóa mõm (Constraints/locking) được nạp đạn viện trợ bảo kê cái huyết mạch tuôn trào (write invariant).

### Thử lửa và Vận hành (Test và vận hành)

- [ ] Bài test liều mạng nhai nuốt PostgreSQL Testcontainers thứ thiệt, khạc nhổ tẩy chay mớ đồ giả (không dùng) H2 cho chuyện thâm sâu MVCC (semantics).
- [ ] Mớ hỗn độn cài răng lược (Interleaving) nhờ cậy đống then chốt/tường rào (latch/barrier) mang bom hẹn giờ (timeout), nói không với trò trì hoãn thời gian rùa bò (time-based delay).
- [ ] Giáng đòn kiểm chứng (Assert) cả mớ dây nhợ (setting) thông số lẫn kết quả túi tiền (business outcome).
- [ ] Đoạn mã kiểm thử (Test method) tuyệt nhiên không chơi trò ném lựu đạn bọc kín (bọc transaction) đâm lén (ngoài ý muốn).
- [ ] Hồ chứa (Pool) đo lường cắt may (sizing) độ giãn nở xô đẩy (concurrency) nhét cho vừa họng `REQUIRES_NEW`.
- [ ] Quyển sổ nhật ký (Log) rỗng tuếch không cất giấu (không chứa) tà vẹt (dữ liệu) yếu hầu mỏng manh (nhạy cảm).
