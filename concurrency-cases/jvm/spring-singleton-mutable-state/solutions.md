# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp khuyến nghị: singleton không giữ trạng thái request

Cách ngon nhất, sạch sẽ nhất là làm cho service "mất trí nhớ" (`stateless`). Dữ liệu của request nào thì nằm yên trong tham số hoặc biến chạy nội bộ của request đó thôi. Việc sinh ra ID thì tách riêng cho một anh bạn dependency lo, anh bạn này phải cam kết là "chấp mọi thể loại đa luồng":

```java
package com.example.checkout;

import java.util.UUID;

@FunctionalInterface
public interface DraftIdGenerator {

    UUID nextId();
}
```

```java
package com.example.checkout;

import java.util.UUID;
import org.springframework.stereotype.Component;

@Component
public final class UuidDraftIdGenerator implements DraftIdGenerator {

    @Override
    public UUID nextId() {
        return UUID.randomUUID();
    }
}
```

```java
package com.example.checkout;

import java.util.Objects;
import org.springframework.stereotype.Service;

@Service
public final class ReceiptDraftService {

    private final DraftIdGenerator idGenerator;

    public ReceiptDraftService(DraftIdGenerator idGenerator) {
        this.idGenerator = idGenerator;
    }

    public ReceiptDraft createDraft(String customerId) {
        Objects.requireNonNull(customerId, "customerId");

        return new ReceiptDraft(
                idGenerator.nextId(),
                customerId
        );
    }
}
```

```java
package com.example.checkout;

import java.util.Objects;
import java.util.UUID;

public record ReceiptDraft(UUID id, String customerId) {

    public ReceiptDraft {
        Objects.requireNonNull(id, "id");
        Objects.requireNonNull(customerId, "customerId");
    }
}
```

## Tại sao giải pháp hoạt động

- Service của chúng ta giờ đây sạch bóng, không còn một cái biến chung nào để tranh nhau cả.
- `customerId` trở thành biến cục bộ. Việc của anh nào anh nấy làm, không ai quấy rầy ai.
- Bản nháp `ReceiptDraft` khi đã tạo ra là bất di bất dịch (immutable).
- Thằng `UUID.randomUUID()` đã được build sẵn để an toàn cho mọi mặt trận đa luồng. Cứ tự nhiên mà xài không lo đọc-sửa-ghi dẫm chân lên nhau.
- Không còn khái niệm luồng này phải chờ luồng kia. Không có tranh chấp thì khỏi lo nghẽn cổ chai.

Mình tiêu diệt tận gốc điểm tranh chấp, chứ không phải tìm cách nhét ổ khoá vào.

> **Nói ngắn gọn:** Ai có phần nấy. Dữ liệu request nào chạy theo request đó nên không có chuyện thằng này ghi đè thằng kia.

Nếu cái ID của bạn cần là duy nhất tuyệt đối theo chuẩn nghiệp vụ, bạn nên cắm thêm ràng buộc duy nhất (unique constraint) dưới database. Nếu nghiệp vụ bắt buộc phải dùng số đếm tăng dần, hãy nhường lại sân chơi cho database sequence:

```sql
CREATE SEQUENCE receipt_draft_seq;

SELECT nextval('receipt_draft_seq');
```

Sequence của DB nó tự "chơi" đa máy chủ dễ như ăn kẹo, có điều nếu có lỗi rớt mạng hay rollback thì coi chừng số thứ tự nó nhảy cóc.

## Phương án 1: AtomicLong cho sequence cục bộ

```java
@Service
public final class LocalReceiptDraftService {

    private final AtomicLong nextSequence = new AtomicLong(41);

    public ReceiptDraft createDraft(String customerId) {
        long sequence = nextSequence.incrementAndGet();
        return new ReceiptDraft(sequence, customerId);
    }
}
```

Cách này "chữa cháy" tạm ổn nếu bạn chỉ muốn lấy ID đếm trong phạm vi của 1 máy chủ đang chạy. Nhớ là `customerId` vẫn phải để làm biến cục bộ nha! Tuy nhiên, đừng bao giờ xài cách này cho các định danh quan trọng sống còn toàn hệ thống, bởi vì mỗi khi bạn khởi động lại ứng dụng là bộ đếm trên RAM cũng "về mo", và mỗi máy chủ nó lại có một cái `AtomicLong` riêng lẻ tẻ.

## Phương án 2: vùng bảo vệ bằng synchronized

```java
public synchronized ReceiptDraft createDraft(String customerId) {
    long sequence = ++nextSequence;
    return new ReceiptDraft(sequence, customerId);
}
```

Gắn `synchronized` vô thì hệ thống tự biến thành một cái "toilet 1 buồng", luồng thứ hai phải đứng ngoài chờ đến khi luồng thứ nhất đi ra thì mới được vào. Giải quyết gọn gàng cho 1 máy chủ, nhưng tốc độ sẽ rùa bò nếu như có quá nhiều người truy cập cùng lúc. Thêm nữa, máy A khoá thì mặc máy A, máy B nó vẫn cứ tung tăng cấp số trùng bình thường.

Nếu code bên trong vòng `synchronized` gọi qua các hàm có lock khác, cẩn thận kẻo dính "deadlock" (cả 2 cùng chờ nhau đến sáng). May mà trong ví dụ này chúng ta chỉ có 1 ổ khoá thôi nên cũng ít rủi ro.

## Những lựa chọn không khuyến nghị

- **Xài biến `volatile`:** giúp nhìn thấy dữ liệu mới nhanh, nhưng cái đoạn cộng dồn thì vẫn nát.
- **Xài `ThreadLocal`:** Dễ làm luồng chết ngộp hoặc sót data cũ trong thread pool. Muốn dùng phải nhớ dọn dẹp rất mất công. Cứ xài thẳng tham số (parameter) cho lành.
- **Service đổi sang scope request:** Tránh được đụng độ nhưng lại tự làm khổ mình bằng cách kéo theo mớ bùng nhùng của lifecycle object. Stateless singleton vừa nhanh vừa rõ.
- **Distributed lock (Khoá phân tán):** Như kiểu giết gà dùng dao mổ trâu. Đắt đỏ, cồng kềnh, trừ khi bạn không còn cách nào khác.

## So sánh các đánh đổi

| Giải pháp | Mức bảo đảm | Thông lượng và độ trễ | Tranh chấp và thử lại | Nguy cơ deadlock | Độ phức tạp | Mở rộng nhiều node |
| --- | --- | --- | --- | --- | --- | --- |
| Stateless + UUID | Tốt lắm; nếu cần thêm DB constraint cho chắc ăn | Siêu cao, không phải chờ đợi | Hết tranh chấp, khỏi cần thử lại | Không bao giờ | Thấp | Tuyệt vời |
| `AtomicLong` + dữ liệu cục bộ | Chỉ trong nội bộ 1 máy chủ | Cao | Chạy trâu bò quá thì có thể hơi khựng xíu | Không | Thấp | Đừng dại mà xài khi chạy nhiều node |
| `synchronized` | Bao ngon trong nội bộ 1 máy chủ | Chậm đi vì phải xếp hàng tuần tự | Đứng chờ mòn mỏi; không retry | Ít (nếu chỉ 1 ổ khoá) | Vừa vừa | Vô dụng với đa máy chủ |
| Database sequence | Ổn cho mọi node | Tốn thời gian gọi tới DB một chuyến | DB lo hết vụ xếp hàng rồi | Rất thấp | Vừa vừa | Tuyệt vời khi lấy DB làm mốc |
| Distributed lock | Chờ timeout này nọ mệt mỏi | Trễ do độ trễ của mạng | Chờ khoá lòi kèn, hay lỗi và phải thử lại | Hên xui do rủi ro nghẽn mạng | Siêu cao | Phức tạp hoá vấn đề rồi |

## Lưu ý khi áp dụng thực tế

- Tuyệt đối không bao giờ được lưu các thứ như `HttpServletRequest`, tài khoản khách hàng, hay mấy thứ linh tinh của 1 request vào field của `singleton service`.
- Nên xài parameter, các đối tượng bất biến (immutable), hoặc MDC của logging (nhưng nhớ dọn dẹp kỹ sau khi xài).
- Muốn làm chỉ số mét (metrics) thì lôi `Micrometer` hay hàng làm sẵn ra, đừng tự ngồi code bộ đếm.
- Cần biết cái ID của mình có vai trò gì: nội bộ hay xài chung, tạm bợ hay xài lâu dài, có cần đếm số hay không được mất số nào không?
- Cái khái niệm an toàn luồng và khái niệm `idempotent` (gọi lại chục lần vẫn thế) là 2 thứ khác nhau. Nếu cần vụ gọi lại thì phải làm hẳn một cơ chế kiểm tra và lưu vết riêng.
