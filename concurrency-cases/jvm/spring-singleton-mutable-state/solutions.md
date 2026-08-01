# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp khuyến nghị: singleton không giữ trạng thái request

Dữ liệu của request chỉ tồn tại trong tham số hoặc biến cục bộ. Việc tạo ID
được tách thành một dependency có hợp đồng (contract) an toàn cho nhiều luồng:

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

- Service không còn field thay đổi theo từng request.
- `customerId` là biến cục bộ của một lời gọi; request khác không thể thay thế
  giá trị này.
- `ReceiptDraft` không thay đổi sau khi được tạo và được hoàn thiện trước khi trả
  về.
- `UUID.randomUUID()` hỗ trợ nhiều lời gọi đồng thời; mỗi lời gọi không thực
  hiện chuỗi đọc–sửa–ghi trên trạng thái của service.
- Không còn luồng bị chậm phải chờ, thất bại hoặc thử lại vì không còn xung đột trên
  trạng thái dùng chung.

Giải pháp loại bỏ điểm tranh chấp thay vì chỉ đặt khóa bao quanh nó.

> **Nói ngắn gọn:** mỗi request chỉ làm việc với dữ liệu của chính nó nên request
> khác không còn trạng thái để ghi đè.

Nếu ID là định danh nghiệp vụ bền vững đòi hỏi tính duy nhất tuyệt đối, UUID tại
tầng ứng dụng vẫn phải đi cùng ràng buộc duy nhất của cơ sở dữ liệu. Nếu cần
chuỗi tăng dần toàn cục, dùng database sequence:

```sql
CREATE SEQUENCE receipt_draft_seq;

SELECT nextval('receipt_draft_seq');
```

Database sequence xử lý đồng thời giữa nhiều application instance, nhưng
không cam kết chuỗi không có khoảng trống khi transaction rollback.

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

Cách này phù hợp khi yêu cầu chỉ cần sequence duy nhất trong vòng đời một
JVM. `customerId` vẫn phải là biến cục bộ. Không dùng bộ đếm cục bộ cho định
danh bền vững hoặc dùng chung toàn hệ thống, vì khởi động lại làm mất trạng thái và mỗi
node có một `AtomicLong` riêng.

## Phương án 2: vùng bảo vệ bằng synchronized

```java
public synchronized ReceiptDraft createDraft(String customerId) {
    long sequence = ++nextSequence;
    return new ReceiptDraft(sequence, customerId);
}
```

Cùng một monitor bảo đảm tại một thời điểm chỉ một luồng vào phương thức và tạo quan
hệ xảy ra-trước giữa hai lần gọi. T2 phải chờ đến khi T1 thoát phương thức; phía gọi
không cần thử lại (retry). Cách này đúng trong một JVM nhưng buộc mọi request chạy lần
lượt, làm tăng độ trễ khi mức tranh chấp cao và không bảo vệ App B.

Nếu lock bao nhiều resource hoặc phương thức gọi code có lock khác, nguy cơ deadlock
có thể xuất hiện. Case hiện tại chỉ có một monitor nên rủi ro thấp.

## Những lựa chọn không khuyến nghị

- **`volatile` counter:** tính hiển thị đúng nhưng phép cộng gộp vẫn sai.
- **`ThreadLocal`:** dễ giữ dữ liệu trên luồng trong pool, yêu cầu dọn dẹp và che
  giấu luồng dữ liệu; dùng tham số đã đơn giản hơn.
- **service có phạm vi theo request:** tránh chia sẻ field nhưng tăng độ phức tạp
  của vòng đời mà không cần thiết; stateless singleton rõ hơn.
- **khóa phân tán (distributed lock):** quá phức tạp cho việc có thể loại bỏ trạng thái
  dùng chung hoặc dùng bộ tạo ID chính.

## So sánh các đánh đổi

| Giải pháp | Mức bảo đảm | Thông lượng và độ trễ | Tranh chấp và thử lại | Nguy cơ deadlock | Độ phức tạp vận hành | Mở rộng nhiều node |
| --- | --- | --- | --- | --- | --- | --- |
| Stateless + UUID | Cách ly request tốt; thêm DB constraint nếu cần uniqueness tuyệt đối | Cao, không có hàng chờ tại service | Không còn trạng thái dùng chung nên không cần thử lại do xung đột | Không | Thấp | Tốt |
| `AtomicLong` + dữ liệu cục bộ | Chỉ bảo đảm trong một JVM và một vòng đời process | Cao | CAS có thể tranh chấp khi rất nóng; thường không cần phía gọi thử lại | Không | Thấp | Không phù hợp cho định danh toàn cục |
| `synchronized` | Đúng trong một JVM nếu mọi access dùng cùng monitor | Độ trễ tăng vì các request chạy lần lượt | Luồng thứ hai phải chờ; không thử lại | Thấp với một lock, tăng khi có nhiều lock | Thấp/trung bình | Không bảo vệ nhiều node |
| Database sequence | Cấp sequence dùng chung giữa các node | Thêm một vòng kết nối DB | Database quản lý tranh chấp; thường không cần phía gọi thử lại | Rất thấp cho một lời gọi sequence | Trung bình | Tốt khi database là ranh giới dùng chung |
| Distributed lock | Phụ thuộc TTL, ownership và fencing | Thêm độ trễ mạng | Có chờ khóa, lỗi và thử lại | Có rủi ro từ protocol và failure mode | Cao | Quá phức tạp cho case này |

## Lưu ý khi áp dụng thực tế

- Không lưu `HttpServletRequest`, principal xác thực, dữ liệu khách hàng hoặc
  định danh tương quan trong field của singleton.
- Dùng tham số, đối tượng command bất biến hoặc logging MDC có vòng đời/dọn dẹp đúng.
- Metrics dùng Micrometer/công cụ an toàn luồng thay vì bộ đếm tự viết.
- Xác định rõ hợp đồng của ID: cục bộ hay toàn hệ thống, tạm thời hay bền vững,
  có cần tăng dần hoặc không được có khoảng trống hay không.
- Khả năng xử lý lặp an toàn (`idempotency`) khi phía gọi thử lại là một quy tắc khác
  với an toàn luồng của service. Nếu nghiệp vụ yêu cầu, cần khóa idempotency và
  bản ghi bền vững riêng.
