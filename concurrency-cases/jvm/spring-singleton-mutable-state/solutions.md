# Solutions, fixed code và trade-offs

## Giải pháp khuyến nghị: stateless singleton

Request data chỉ tồn tại trong parameter/local variable. ID generation được
tách thành dependency có contract thread-safe:

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

## Vì sao hoạt động

- Service không còn mutable field thay đổi theo request.
- `customerId` là local reference của invocation; request khác không thể thay
  thế nó.
- `ReceiptDraft` immutable và được tạo hoàn chỉnh trước khi trả về.
- `UUID.randomUUID()` hỗ trợ concurrent invocation; mỗi lời gọi không phụ
  thuộc read-modify-write trên state của service.
- Không có loser phải block, fail hoặc retry vì không còn conflict trên shared
  application state.

Đây là removal of contention thay vì chỉ bao contention bằng lock.

Nếu ID là durable business identity đòi hỏi uniqueness tuyệt đối, UUID tại
application layer vẫn phải đi cùng database unique constraint. Nếu cần global
monotonic sequence, dùng database sequence:

```sql
CREATE SEQUENCE receipt_draft_seq;

SELECT nextval('receipt_draft_seq');
```

Database sequence xử lý concurrency giữa nhiều application instance, nhưng
không cam kết gap-free sequence khi transaction rollback.

## Alternative 1: AtomicLong cho local sequence

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

Điều này hoạt động nếu contract chỉ yêu cầu unique sequence trong vòng đời một
JVM. `customerId` phải vẫn là local variable. Không dùng nó cho durable/global
identity vì restart reset state và mỗi node có `AtomicLong` riêng.

## Alternative 2: synchronized critical section

```java
public synchronized ReceiptDraft createDraft(String customerId) {
    long sequence = ++nextSequence;
    return new ReceiptDraft(sequence, customerId);
}
```

Cùng monitor tạo mutual exclusion và happens-before. T2 block cho tới khi T1
thoát method; không retry. Cách này đúng trong một JVM nhưng serialize mọi
request, tăng queueing latency dưới contention và không bảo vệ App B.

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

## So sánh trade-offs

| Solution | Correctness | Throughput/latency | Contention/retry | Deadlock risk | Operational complexity | Horizontal scalability |
| --- | --- | --- | --- | --- | --- | --- |
| Stateless + UUID | Mạnh cho request isolation; DB constraint nếu cần absolute uniqueness | Cao, không queue trên service | Không shared contention; không retry do conflict | Không | Thấp | Tốt |
| `AtomicLong` + local data | Mạnh chỉ trong một JVM/lifetime | Cao | CAS contention khi rất nóng; thường không application retry | Không | Thấp | Kém cho global identity |
| `synchronized` | Mạnh trong một JVM nếu mọi access cùng monitor | Latency tăng do serialization | Thread thứ hai block; không retry | Thấp với một lock, tăng nếu nhiều lock | Thấp/trung bình | Không bảo vệ nhiều node |
| Database sequence | Global sequence allocation giữa node | Thêm DB round trip | Database quản lý contention; application không retry thông thường | Rất thấp cho sequence call | Trung bình | Tốt, DB là shared boundary |
| Distributed lock | Phụ thuộc TTL/ownership/fencing | Thêm network latency | Có lock wait/failure/retry | Có protocol/failure risks | Cao | Phức tạp, không cần cho case này |

## Production considerations

- Không lưu `HttpServletRequest`, authentication principal, customer data hoặc
  correlation ID trong singleton field.
- Dùng parameter, immutable command hoặc logging MDC có lifecycle/cleanup đúng.
- Metrics dùng Micrometer/thread-safe instrument thay vì counter tự viết.
- Xác định rõ ID contract: local, global, durable, monotonic hay gap-free.
- Idempotency của client retry là invariant khác với thread safety của service;
  cần idempotency key/durable record nếu business yêu cầu.

