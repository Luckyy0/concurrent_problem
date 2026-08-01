# Cách triển khai bị lỗi

## Đoạn code gây điều kiện tranh chấp

Hãy xem thử đoạn code dưới đây. Đây là một sai lầm rất phổ biến: dev nhà ta thường có thói quen khai báo biến chung (field) trong service để đếm số sequence và lưu tên khách hàng cho tiện tái sử dụng.

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

Phía Controller thì xài dependency injection quen thuộc. Dù bạn gọi nó bao nhiêu lần, thì các request cũng chạy thẳng vào đúng duy nhất một instance của `ReceiptDraftService`.

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

- Khi bạn chạy unit test (chạy từng luồng một), số sequence luôn tăng đều đặn, nhìn rất nuột.
- Code Java viết quá ngắn gọn, bạn nhìn dòng `++nextSequence` cứ tưởng nó là một cục nguyên khối, chứ ai ngờ nó tách ra nhiều bước ngầm bên dưới.
- Spring quảng cáo là khởi tạo singleton rất an toàn. Nhiều người hiểu lầm "an toàn" có nghĩa là sau đó gọi hàm thoải mái không lo lỗi luồng.
- Vì không đụng tới database, dev thường tặc lưỡi bỏ qua vụ phân tích vùng tranh chấp dữ liệu hay cơ chế khoá (lock).

## Điều kiện để lỗi xuất hiện

Để cái bug này "bật ngửa" ra, bạn chỉ cần đủ 4 yếu tố:

1. Spring bean đang xài scope mặc định (singleton).
2. Server đang bật ít nhất 2 luồng để chạy request song song.
3. Hai request xui rủi gọi hàm `createDraft` rơi đúng vào cùng một phần nghìn giây (xen kẽ nhau).
4. Không có bất kỳ ổ khoá (lock) nào gom toàn bộ logic này lại.

Lúc này, `nextSequence` và `lastCustomerId` trở thành các bãi chiến trường để các luồng giành nhau đọc/ghi. Không có trật tự nào cả, mạnh ai nấy chạy!

> **Nói ngắn gọn:** Hai request cùng lúc lôi hai biến chung ra đọc và sửa. Không có khoá chặn lại, nên luật nghiệp vụ bị phá vỡ hoàn toàn.

## Các cách sửa tưởng đúng nhưng chưa đủ

Nhiều khi thấy lỗi, chúng ta vội vàng quăng ngay vài keyword vào với hy vọng sẽ hết. Nhưng coi chừng:

### Chỉ thêm volatile

```java
private volatile long nextSequence;
```

Keyword `volatile` chỉ giúp các luồng "nhìn thấy" dữ liệu mới nhất ngay lập tức. Nhưng ngặt nỗi, biểu thức `++nextSequence` thật ra là 3 bước rời rạc:

```text
read → add → write
```

Hai luồng hoàn toàn có thể cùng đọc số `41`, sau đó cùng cộng thêm 1, rồi hớn hở cùng lưu số `42` vào. Vậy là xong phim, mất một cập nhật!

### Chỉ đổi counter sang AtomicLong

```java
private final AtomicLong nextSequence = new AtomicLong(41);
```

Okay, thay bằng `AtomicLong` với hàm `incrementAndGet()` thì số đếm an toàn rồi đó. Nhưng bạn quên mất ông nội `lastCustomerId` à? Nó vẫn bị request khác ghi đè lên cái rẹt. Làm một biến an toàn không có nghĩa là toàn bộ quy trình đều an toàn đâu.

### Chỉ thêm @Transactional

```java
@Transactional
public ReceiptDraft createDraft(String customerId) {
    // ...
}
```

Đừng thần thánh hoá `@Transactional`! Nó chỉ tạo một ranh giới bảo vệ cho database connection, gắn liền với luồng hiện tại. Nó không hề có phép thuật nào để khoá lại cái biến nằm trên bộ nhớ RAM (Java heap) của bạn đâu nhé. Hai luồng vẫn sẽ dẫm chân lên nhau như thường.
