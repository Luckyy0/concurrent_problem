# Broken implementation

## Code gây race condition

Đây là code một developer có thể viết khi muốn tái sử dụng sequence và dữ liệu
gần nhất giữa các lời gọi:

```java
package com.example.checkout;

import java.util.Objects;
import org.springframework.stereotype.Service;

@Service
public class ReceiptDraftService {

    private long nextSequence = 41;
    private String lastCustomerId;

    public ReceiptDraft createDraft(String customerId) {
        Objects.requireNonNull(customerId, "customerId");

        lastCustomerId = customerId;
        long sequence = ++nextSequence;

        return new ReceiptDraft(sequence, lastCustomerId);
    }
}
```

```java
package com.example.checkout;

import java.util.Objects;

public record ReceiptDraft(long sequence, String customerId) {

    public ReceiptDraft {
        if (sequence <= 0) {
            throw new IllegalArgumentException("sequence must be positive");
        }
        Objects.requireNonNull(customerId, "customerId");
    }
}
```

Controller dùng constructor injection nhưng mọi request vẫn đi vào cùng service
instance:

```java
package com.example.checkout;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/receipt-drafts")
public class ReceiptDraftController {

    private final ReceiptDraftService receiptDraftService;

    public ReceiptDraftController(ReceiptDraftService receiptDraftService) {
        this.receiptDraftService = receiptDraftService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ReceiptDraft create(@RequestBody CreateReceiptDraftRequest request) {
        return receiptDraftService.createDraft(request.customerId());
    }
}
```

```java
package com.example.checkout;

public record CreateReceiptDraftRequest(String customerId) {
}
```

## Vì sao code trông có vẻ hợp lý

- Unit test gọi tuần tự luôn thấy sequence tăng.
- Mỗi statement Java nhìn ngắn và không có collection phức tạp.
- Spring khởi tạo singleton an toàn, dễ bị hiểu nhầm là mọi access sau đó cũng
  thread-safe.
- Field không phải database state nên developer có thể không nghĩ đến
  transaction hay lock.

## Preconditions để lỗi xuất hiện

1. Bean dùng singleton scope mặc định.
2. Server có ít nhất hai request worker.
3. Hai lời gọi `createDraft` bị xen kẽ.
4. Không có cùng một monitor/lock bao quanh toàn bộ invariant.

`nextSequence` và `lastCustomerId` đều là shared mutable state. Không operation
nào tạo happens-before edge giữa hai request thread.

## Những “fix” chưa đủ

### Chỉ thêm volatile

```java
private volatile long nextSequence;
```

`volatile` hỗ trợ visibility nhưng `++nextSequence` vẫn là:

```text
read → add → write
```

Hai thread vẫn có thể cùng read `41` rồi cùng write `42`.

### Chỉ đổi counter sang AtomicLong

```java
private final AtomicLong nextSequence = new AtomicLong(41);
```

`incrementAndGet()` sửa counter nhưng `lastCustomerId` vẫn bị request khác ghi
đè. Thread-safe field riêng lẻ không bảo vệ invariant gồm nhiều field.

### Chỉ thêm @Transactional

```java
@Transactional
public ReceiptDraft createDraft(String customerId) {
    // ...
}
```

Spring transaction gắn với thread/database connection; nó không serialize
method calls và không lock Java heap field.

