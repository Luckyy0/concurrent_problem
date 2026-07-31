# Khóa lạc quan (Optimistic Locking) và Xung đột phiên bản

## Mục tiêu

Tài liệu này giải thích cách Java (cụ thể là JPA/Hibernate) sử dụng thẻ (annotation) `@Version` để phát hiện lỗi khi có nhiều luồng cùng lúc sửa một dữ liệu. Nó cũng hướng dẫn cách nhận biết lỗi xung đột và khi nào thì bạn được phép "thử lại" (retry) một cách an toàn. 

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Khóa lạc quan (`optimistic locking`) | Cơ chế rất "lạc quan": Cứ cho mọi người cùng đọc song song thoải mái, không khóa ai lại cả. Chỉ đến lúc ghi (lưu xuống) mới đi kiểm tra xem có bị ai đó sửa mất rồi không. |
| Cột phiên bản (`version column`) | Một con số tăng dần (ví dụ từ 1, 2, 3...) sau mỗi lần cập nhật thành công dữ liệu. |
| Phiên bản kỳ vọng (`expected version`) | Con số phiên bản mà bạn lấy được lúc đọc dữ liệu lên, dùng để kiểm tra lại lúc lưu xuống. |
| Dữ liệu thiu (`stale entity`) | Dữ liệu trên bộ nhớ của bạn đã cũ, vì dưới Database có ai đó vừa lưu một phiên bản mới hơn mất rồi. |
| Số dòng ảnh hưởng (`affected-row count`) | Kết quả trả về của lệnh SQL. Nếu trả về `0` nghĩa là phiên bản kỳ vọng không khớp (có lỗi xung đột). |
| Xung đột lạc quan (`optimistic conflict`) | Xảy ra khi một luồng khác đã lưu đè dữ liệu mới, trong lúc luồng hiện tại đang mải mê tính toán trên phiên bản cũ. |
| Khuếch đại thử lại (`retry amplification`) | Hiện tượng thảm họa: Nhiều luồng cùng bị lỗi, cùng đồng loạt thử lại, rồi lại cùng đạp lên nhau gây lỗi tiếp, vắt kiệt CPU. |
| Phạm vi thử lại (`retry boundary`) | Xác định rõ phải hoàn tác (rollback) từ đâu, tạo lại kết nối từ đâu và tải lại dữ liệu thế nào trước khi chạy lại. |

## Cơ chế biến hóa thành câu lệnh SQL

Khi bạn khai báo trong Java:
```java
@Version
private long version;
```

Khi gọi lệnh lưu, Hibernate sẽ tự động chế ra một câu lệnh UPDATE có kẹp thêm điều kiện `version`:
```sql
UPDATE inventory_item
SET available = :newAvailable,
    version = :nextVersion
WHERE sku = :sku
  AND version = :expectedVersion;
```

Nếu câu lệnh trả về số dòng ảnh hưởng (`affected-row count`) là `1`, bạn là người chiến thắng, dữ liệu được lưu và `version` tăng lên. 
Nếu trả về `0`, Hibernate biết ngay dữ liệu của bạn là đồ "thiu" (stale state) và lập tức ném ra lỗi `OptimisticLockException` (hoặc Spring sẽ ném ra `ObjectOptimisticLockingFailureException`).

> **Nói ngắn gọn:** Từ khóa `@Version` KHÔNG hề khóa (block) ai không cho đọc dữ liệu. Nó chỉ làm nhiệm vụ chặn đứng những kẻ mang dữ liệu cũ "thiu" đến đòi ghi đè lên dữ liệu mới nhất dưới Database.

## Bộ đệm (Persistence context) sẽ ra sao sau khi xung đột?

Hãy cẩn thận! Sau khi lệnh lưu (flush) bị lỗi văng ra exception, bộ đệm (`EntityManager`) của Hibernate đã bị "vấy bẩn". Giao dịch (Transaction) hiện tại gần như đã bị đánh dấu là hỏng (`rollback-only`), và các dữ liệu trên bộ nhớ của bạn đang mang những giá trị sai lệch.
Việc gọi lệnh `clear()` chỉ có tác dụng cắt đứt liên kết dữ liệu, chứ KHÔNG hề cứu vãn được cái giao dịch đã hỏng đó.

Để "thử lại" (retry) chuẩn xác, bạn bắt buộc phải làm theo các bước:
1. Để cho exception văng thẳng ra ngoài phạm vi của giao dịch hiện tại.
2. Hoàn tác (rollback) và đóng hoàn toàn giao dịch cùng bộ đệm cũ đó lại.
3. Cho luồng ngủ một chút (bounded backoff/jitter) để nhường đường nếu cần.
4. Mở một **giao dịch hoàn toàn mới** (physical transaction mới).
5. Đọc lại (reload) dữ liệu mới nhất từ Database và tính toán lại các điều kiện nghiệp vụ.
6. Lưu lại (commit), nếu vẫn lỗi thì tiếp tục phân loại xung đột.

## Không phải cứ lỗi là nhắm mắt "Thử lại" (Retry)

Bạn chỉ nên thử lại khi thuật toán tính toán của bạn chạy lại trên dữ liệu mới vẫn ra kết quả đúng (deterministic) và lần thử hỏng trước đó chưa kịp gửi dữ liệu ra ngoài (chưa có external side effect). 

Tuyệt đối KHÔNG thử lại một cách mù quáng đối với:
- Lỗi nghiệp vụ (ví dụ: kho không đủ hàng nên bị từ chối).
- Yêu cầu gửi trùng lặp mà không có cơ chế chặn trùng (idempotency).
- Bạn đã lỡ gọi API ra ngoài (ví dụ gửi Email) mà không rõ API đó đã chạy thành công hay chưa.
- Yêu cầu đã quá thời gian cho phép (deadline).
- Dữ liệu đang bị tranh chấp quá mức (hot-key) sinh ra bão thử lại (retry storm).

Người thua cuộc trong xung đột hoàn toàn có thể chọn cách báo lỗi thẳng ra cho người dùng (hoặc trả về kết quả rỗng) thay vì ngoan cố thử lại, nếu nghiệp vụ cho phép.

## Hiệu năng (Throughput) và Tranh chấp (Contention)

Khóa lạc quan cực kỳ tuyệt vời khi tỷ lệ các luồng giẫm chân lên nhau (conflict) là rất hiếm. 
Tuy nhiên, nếu bạn dùng nó cho các "điểm nóng" (hot key) - ví dụ như số lượng vé của một đêm nhạc BlackPink - nhiều người sẽ cùng hì hục tính toán, nhưng lúc lưu xuống chỉ có 1 người thắng. Những người còn lại phải tính toán lại (retry) từ đầu, làm tăng vọt lượng CPU sử dụng và lượng câu query bắn xuống Database.

Trong các trường hợp "nóng" đó, hãy chọn các phương án khác:
- Dùng lệnh SQL cập nhật có điều kiện thẳng (atomic conditional SQL).
- Khóa bi quan (pessimistic locking - `SELECT FOR UPDATE`).
- Phân mảnh dữ liệu theo key (partitioning).
- Đưa vào hàng đợi xử lý tuần tự (serial queue).
Hãy chọn phương án tùy thuộc vào nghiệp vụ, sự phân bổ điểm nóng và khả năng xử lý thử lại an toàn.

## Lời khuyên khi Kiểm thử (Testing)

Để test chức năng này, bạn phải ép 2 giao dịch cùng đọc lên chung một phiên bản (version). Cho giao dịch 1 lưu trước (commit), rồi mới cho giao dịch 2 lưu (flush). Hãy kiểm tra (assert) rằng:
- Lệnh UPDATE sinh ra thực sự có mang điều kiện `version` ở đuôi (WHERE).
- Người thua (giao dịch 2) chắc chắn nhận được lỗi xung đột (optimistic conflict).
- Giao dịch bị lỗi đã được hoàn tác (rollback) sạch sẽ.
- Lần thử lại (retry) sau đó đang dùng một giao dịch và bộ đệm hoàn toàn mới.
- Dữ liệu đã được tải lại (reload).
- Trạng thái cuối cùng và con số `version` phải đúng.

**Cảnh báo:** Đừng chỉ kiểm tra xem code có bắt (catch) đúng exception hay không! Một vòng lặp thử lại rất dễ bắt đúng lỗi nhưng lại ngoan cố tiếp tục chạy bên trong cái giao dịch đã hỏng (doomed transaction) đó.

## Liên kết tài liệu tham khảo

- [SPR-006 — Retry transaction boundary](../spring/retry-transaction-boundary/README.md)
- [LOCK-001 — Optimistic locking with @Version](../locking/optimistic-version-conflict/README.md)
- [LOCK-002 — Bounded optimistic retry](../locking/optimistic-retry-contention/README.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Deadlock và retry an toàn](deadlocks-and-retries.md)
- [Concurrency testing](concurrency-testing.md)
