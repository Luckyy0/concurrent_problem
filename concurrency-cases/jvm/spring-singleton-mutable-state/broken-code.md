# Cách triển khai bị lỗi

## Đoạn code gây điều kiện tranh chấp

Đoạn code dưới đây mô phỏng một cách triển khai dễ gặp: lập trình viên giữ sequence
và dữ liệu khách hàng gần nhất trong field để dùng lại giữa các lời gọi.

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

Controller dùng constructor injection nhưng mọi request vẫn đi vào cùng một service
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

## Vì sao đoạn code trông có vẻ hợp lý

- Kiểm thử đơn vị gọi tuần tự luôn thấy sequence tăng.
- Mỗi câu lệnh Java đều ngắn nên khó nhận ra chúng tạo thành một chuỗi nhiều bước.
- Spring khởi tạo singleton an toàn, nhưng điều này dễ bị hiểu nhầm thành mọi lần
  truy cập sau đó cũng an toàn cho nhiều luồng.
- Trạng thái không nằm trong database nên lập trình viên có thể bỏ qua việc phân tích vùng
  tranh chấp và cơ chế khóa.

## Điều kiện để lỗi xuất hiện

1. Bean dùng singleton scope mặc định.
2. Server có ít nhất hai luồng xử lý request.
3. Hai lời gọi `createDraft` bị xen kẽ.
4. Không có cùng một monitor/lock bao quanh toàn bộ logic nghiệp vụ (invariant).

`nextSequence` và `lastCustomerId` đều là trạng thái dùng chung có thể thay đổi.
Không có cơ chế đồng bộ nào tạo quan hệ xảy ra-trước giữa hai luồng xử lý
request.

> **Nói ngắn gọn:** cả hai request đang đọc và ghi cùng hai field mà không có
> khóa hoặc thao tác nguyên tử bảo vệ toàn bộ quy tắc.

## Các cách sửa tưởng đúng nhưng chưa đủ

### Chỉ thêm volatile

```java
private volatile long nextSequence;
```

`volatile` giúp một luồng nhìn thấy giá trị mà luồng khác đã ghi, nhưng
`++nextSequence` vẫn gồm ba bước:

```text
read → add → write
```

Hai luồng vẫn có thể cùng đọc `41` rồi cùng ghi `42`.

### Chỉ đổi counter sang AtomicLong

```java
private final AtomicLong nextSequence = new AtomicLong(41);
```

`incrementAndGet()` sửa lỗi của bộ đếm, nhưng `lastCustomerId` vẫn có thể bị
request khác ghi đè. Một field an toàn cho nhiều luồng không tự bảo vệ quy tắc
bao gồm nhiều field.

### Chỉ thêm @Transactional

```java
@Transactional
public ReceiptDraft createDraft(String customerId) {
    // ...
}
```

Spring transaction gắn với một luồng và database connection. Nó không buộc các
lời gọi phương thức phải chạy lần lượt và không khóa field nằm trong Java heap.
