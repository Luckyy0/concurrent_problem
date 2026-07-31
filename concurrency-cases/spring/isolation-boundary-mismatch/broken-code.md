# Đặt annotation sai bét nhè (Broken annotation placement)

## Vòng trong REQUIRED cũng chẳng làm nên trò trống nâng cấp (Inner REQUIRED không nâng isolation)

```java
@Service
public class ReportFacade {
    private final SnapshotQueryService snapshotQueries;

    @Transactional
    public PriceReport generate(long productId) {
        return snapshotQueries.readTwice(productId);
    }
}
```

```java
@Service
public class SnapshotQueryService {
    private final PriceRepository prices;
    private final SnapshotProbe probe;

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public PriceReport readTwice(long productId) {
        long first = prices.getPrice(productId);
        probe.afterFirstRead();
        long second = prices.getPrice(productId);
        return new PriceReport(first, second);
    }
}
```

Cuộc hội thoại (Call) ung dung đi xuyên qua cửa proxy vòng trong (inner proxy), thế nhưng cái bộ quy tắc lan truyền (propagation) mặc định mang tên `REQUIRED` đã dính chặt nó vào giao dịch vật lý ở vòng ngoài (outer physical transaction). Khổ nỗi, vòng ngoài lại được khai sinh bằng thẻ `Isolation.DEFAULT`; mà mặc định của DataSource/PostgreSQL thì muôn đời là `READ COMMITTED`. Bùa chú dán ở vòng trong (Inner annotation) hoàn toàn vô dụng trong việc tái khởi động (restart) hay nâng cấp (upgrade) giao dịch.

> **Nói ngắn gọn:** màn đánh chặn (proxy interception) quả là có diễn ra thật, thế nhưng lính gác (interceptor) lại phát hiện ra giao dịch đã sờ sờ tồn tại thế nên đành kéo nhau vào (join) hùa theo đúng mấy cái luật lệ (attributes) đã được ấn định (quyết định) từ khuya rồi.

## Quái chiêu tự gọi mình (Self-invocation variant)

```java
public PriceReport generate(long productId) {
    return this.readWithStableSnapshot(productId);
}

@Transactional(isolation = Isolation.REPEATABLE_READ)
public PriceReport readWithStableSnapshot(long productId) { ... }
```

Nếu cánh cửa (entry) không được phủ bùa giao dịch (transactional) và tiếng vọng tự thân (self-call) lại vô tình lách qua mắt (bỏ qua) proxy, thì cái mộng ước xây dựng một giao dịch dịch vụ vĩ đại (intended service transaction) coi như tan tành mây khói; mấy cái lệnh Repository bên trong (repository calls) đành phải tủi nhục chạy núp bóng trong từng giao dịch riêng lẻ con con (giao dịch riêng). Đây chính xác là quái thai SPR-001 hòa huyết cùng với dị tật lệch pha cô lập (isolation mismatch).

## Niềm tin mù quáng vào mặc định nguồn dữ liệu (Datasource default assumption)

Cái mác `@Transactional(isolation = Isolation.DEFAULT)` chẳng thề thốt đảm bảo cho bạn một mức cố định (fixed level) lẫy lừng di động đâu. Nó bám váy dựa dẫm (dùng) vào cái mặc định (default) của bộ máy quản lý giao dịch/nguồn dữ liệu/cơ sở dữ liệu. Môi trường (Environment) chỉ cần hắt hơi xổ mũi tráo cái `default_transaction_isolation` là đủ đảo lộn (đổi behavior) ván cờ nếu cấu hình (config) không dứt khoát rạch ròi (explicit).

## Những liều thuốc chữa trị nửa vời (chưa đủ)

- Dán nhãn (Đặt) isolation chót vót (cao) trên những thằng đệ tự kín đáo (private/self-invoked helper).
- Cắm bùa (Annotate) REQUIRED ở vòng trong (inner) rồi mơ tưởng hão huyền rằng nó sẽ đạp đổ (override) cả vòng ngoài.
- Ẵm cờ `readOnly=true` rồi nhắm mắt ảo tưởng đó là bức tường bản ảnh tĩnh (stable snapshot).
- Cầu viện đám `saveAndFlush`/`EntityManager.clear`; xin thưa nó cũng chả rụng được cọng lông làm đổi rời mức cô lập vật lý (không đổi physical isolation).
- Hộc tốc bẻ lái (Đổi sang) sang ngõ `REQUIRES_NEW` chỉ với hi vọng vớt vát thể diện cho cái annotation nó sống lay lắt (có hiệu lực) mà đui mù (không xem) bỏ mặc chuyện hậu họa đóng chốt chia ly (independent commit), nhòi thêm đường truyền thứ hai (second connection) và cảnh huynh đệ tương tàn khóa lẫn nhau (outer lock interaction).
- Đo đếm phô trương (Chỉ assert) uy thế cái annotation bằng ngón nghề phản chiếu (reflection); trong khi mù tịt (không đo) cái mức cô lập thực tiễn (effective transaction).
- Hùng hồn Chọn `SERIALIZABLE` nhưng lóng ngóng (không xử lý) vứt xó khâu nhồi nắn thử lại cho mấy ca đứt gánh đồng bộ (serialization retry).

## Tiền đề tái sinh thảm họa (Điều kiện tái hiện)

Giao dịch vật lý vòng ngoài (Outer physical transaction) đã cắm cọc tọa lạc ở cái mức yếu sinh lý (weaker isolation) từ đời tám hoảnh; màn canh gác rà soát đặc tính ngỗ nghịch (validation incompatible attributes) thì bị dập cầu dao tắt ngúm (tắt - mà đây lại là thói quen mặc định của nhiều kẻ); thằng ghi (writer) xông xênh nhảy xổ vào đóng chốt (commit) trúng ngay cái khe hẹp giữa hai câu lệnh (statements); trong khi cái thuật toán đầu não của gã đọc (reader logic) lại thèm khát kháo nhau về một bức ảnh tĩnh bất động (stable snapshot).
