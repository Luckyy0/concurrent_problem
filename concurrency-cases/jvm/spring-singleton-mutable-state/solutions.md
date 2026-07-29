# Giải pháp, code đã sửa và các đánh đổi

## Giải pháp khuyến nghị: singleton không giữ trạng thái request

Dữ liệu của request chỉ tồn tại trong parameter hoặc local variable. Việc tạo ID
được tách thành một dependency có contract an toàn cho nhiều luồng:

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
  hiện chuỗi đọc–sửa–ghi trên state của service.
- Không còn actor thua phải chờ, thất bại hoặc thử lại vì không còn xung đột trên
  state dùng chung.

Giải pháp loại bỏ điểm tranh chấp thay vì chỉ đặt khóa bao quanh nó.

> **Nói ngắn gọn:** mỗi request chỉ làm việc với dữ liệu của chính nó nên request
> khác không còn state để ghi đè.

Nếu ID là durable business identity đòi hỏi uniqueness tuyệt đối, UUID tại
application layer vẫn phải đi cùng database unique constraint. Nếu cần global
monotonic sequence, dùng database sequence:

```sql
CREATE SEQUENCE receipt_draft_seq;

SELECT nextval('receipt_draft_seq');
```

Database sequence xử lý concurrency giữa nhiều application instance, nhưng
không cam kết gap-free sequence khi transaction rollback.

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

Cách này phù hợp khi contract chỉ yêu cầu sequence duy nhất trong vòng đời một
JVM. `customerId` vẫn phải là local variable. Không dùng local counter cho định
danh bền vững hoặc dùng chung toàn hệ thống, vì restart làm mất state và mỗi
node có một `AtomicLong` riêng.

## Phương án 2: vùng bảo vệ bằng synchronized

```java
public synchronized ReceiptDraft createDraft(String customerId) {
    long sequence = ++nextSequence;
    return new ReceiptDraft(sequence, customerId);
}
```

Cùng một monitor bảo đảm tại một thời điểm chỉ một luồng vào method và tạo quan
hệ xảy ra-trước giữa hai lần gọi. T2 phải chờ đến khi T1 thoát method; application
không cần retry. Cách này đúng trong một JVM nhưng buộc mọi request chạy lần
lượt, làm tăng độ trễ khi mức tranh chấp cao và không bảo vệ App B.

Nếu lock bao nhiều resource hoặc method gọi code có lock khác, deadlock risk có
thể xuất hiện. Case hiện tại chỉ có một monitor nên risk thấp.

## Những lựa chọn không khuyến nghị

- **`volatile` counter:** visibility đúng nhưng compound increment vẫn sai.
- **`ThreadLocal`:** dễ giữ dữ liệu trên pooled thread, yêu cầu cleanup và che
  giấu data flow; parameter đã đơn giản hơn.
- **request-scoped service:** tránh chia sẻ field nhưng tăng lifecycle
  complexity mà không cần thiết; stateless singleton rõ hơn.
- **distributed lock:** quá phức tạp cho việc có thể loại bỏ shared state hoặc
  dùng authoritative ID generator.

## So sánh các đánh đổi

| Giải pháp | Mức bảo đảm | Thông lượng và độ trễ | Tranh chấp và retry | Nguy cơ deadlock | Độ phức tạp vận hành | Mở rộng nhiều node |
| --- | --- | --- | --- | --- | --- | --- |
| Stateless + UUID | Cách ly request tốt; thêm DB constraint nếu cần uniqueness tuyệt đối | Cao, không có hàng chờ tại service | Không còn state dùng chung nên không cần retry do conflict | Không | Thấp | Tốt |
| `AtomicLong` + dữ liệu cục bộ | Chỉ bảo đảm trong một JVM và một vòng đời process | Cao | CAS có thể tranh chấp khi rất nóng; thường không cần application retry | Không | Thấp | Không phù hợp cho global identity |
| `synchronized` | Đúng trong một JVM nếu mọi access dùng cùng monitor | Độ trễ tăng vì các request chạy lần lượt | Luồng thứ hai phải chờ; không retry | Thấp với một lock, tăng khi có nhiều lock | Thấp/trung bình | Không bảo vệ nhiều node |
| Database sequence | Cấp sequence dùng chung giữa các node | Thêm một DB round trip | Database quản lý tranh chấp; thường không cần application retry | Rất thấp cho một sequence call | Trung bình | Tốt khi database là ranh giới dùng chung |
| Distributed lock | Phụ thuộc TTL, ownership và fencing | Thêm network latency | Có chờ khóa, lỗi và retry | Có rủi ro từ protocol và failure mode | Cao | Quá phức tạp cho case này |

## Lưu ý khi áp dụng thực tế

- Không lưu `HttpServletRequest`, authentication principal, customer data hoặc
  correlation ID trong singleton field.
- Dùng parameter, immutable command hoặc logging MDC có lifecycle/cleanup đúng.
- Metrics dùng Micrometer/thread-safe instrument thay vì counter tự viết.
- Xác định rõ contract của ID: cục bộ hay toàn hệ thống, tạm thời hay bền vững,
  có cần tăng dần hoặc không được có khoảng trống hay không.
- Khả năng xử lý lặp an toàn (`idempotency`) khi client retry là một quy tắc khác
  với thread safety của service. Nếu nghiệp vụ yêu cầu, cần idempotency key và
  durable record riêng.
