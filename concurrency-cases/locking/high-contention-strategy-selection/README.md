# Bài toán LOCK-005 — Lựa Chọn Chiến Lược Khi Hệ Thống Quá Tải Tương Tranh (Strategy Selection Under High Contention)

## 1. Tóm tắt vấn đề (Overview)

Hãy xem xét kịch bản: Sản phẩm `SKU-2024` là mặt hàng flash-sale với `500` đơn vị trong kho. Trong vòng 10 giây đầu tiên, hệ thống nhận `800` yêu cầu đặt mua đồng thời từ nhiều máy chủ ứng dụng. Mỗi yêu cầu mua `1` sản phẩm.

Nếu sử dụng lock lạc quan (`@Version`), hầu hết các yêu cầu sẽ xung đột version và buộc phải thử lại (retry). Với 800 yêu cầu cùng tranh chấp một bản ghi, tỷ lệ xung đột ở lần thử đầu tiên lên đến 90%. Mỗi lần thử lại lại gia tăng tải cho database, tạo thành bão thử lại (retry storm).

Nếu chuyển sang lock bi quan (`FOR UPDATE`), các yêu cầu sẽ xếp hàng chờ đợi lần lượt. Hàng đợi dài khiến mỗi yêu cầu phải chờ trung bình hàng trăm mili-giây. Kết nối database bị giữ trong suốt thời gian chờ, dẫn đến cạn kiệt connection pool.

Cả hai chiến lược đều bảo đảm tính đúng đắn (correctness), nhưng không chiến lược nào duy trì được thông lượng (throughput) chấp nhận được khi mức tương tranh vượt ngưỡng thiết kế. Bài toán này phân tích khung lựa chọn chiến lược dựa trên đặc điểm tương tranh cụ thể.

> **Ghi chú quan trọng:** Mục tiêu không phải tìm "chiến lược tốt nhất" cho mọi trường hợp, mà là hiểu rõ hành vi và giới hạn của từng chiến lược để chọn phương pháp phù hợp với mức tương tranh thực tế.

## 2. Các thực thể và Trạng thái chia sẻ (Actors and shared state)

| Thực thể | Vai trò |
| --- | --- |
| Bảng tồn kho (`inventory_item`) | Bản ghi sản phẩm `SKU-2024`, điểm tranh chấp chính (hot row) |
| Máy chủ ứng dụng (Instance 1–N) | Nhiều pod xử lý song song, mỗi pod có connection pool riêng |
| Connection pool | Mỗi instance có tối đa `20` kết nối, tổng cộng `60–100` kết nối |
| Bảng lịch sử (`order_reservation`) | Ghi nhận kết quả đặt hàng dựa trên mã lệnh (Command ID) |
| Hệ thống giám sát (Monitoring) | Theo dõi tỷ lệ thành công, thời gian chờ, tỷ lệ thử lại |

Điểm tranh chấp tập trung hoàn toàn tại một bản ghi duy nhất: sản phẩm `SKU-2024`. Đây là bài toán hot-row điển hình, nơi mọi yêu cầu đều cần cập nhật cùng một dòng dữ liệu trong cùng một khoảng thời gian.

## 3. Các quy tắc bất biến (Invariants)

```text
Số lượng hàng có sẵn (available_quantity) >= 0

Tổng số lượng đơn hàng RESERVED không vượt quá tổng kho ban đầu.

Mỗi mã lệnh (Command ID) chỉ được xử lý MỘT lần duy nhất.

Khi hàng hết, mọi yêu cầu sau phải nhận kết quả OUT_OF_STOCK, không được chấp nhận thêm.
```

Tất cả bốn chiến lược được phân tích trong bài toán này đều bảo vệ các quy tắc bất biến trên. Điểm khác biệt nằm ở hành vi khi mức tương tranh cao: thông lượng, độ trễ, tải thử lại, và khả năng chịu tải của connection pool.

## 4. Bốn chiến lược so sánh (Strategy overview)

| Chiến lược | Cơ chế | Hành vi khi xung đột |
| --- | --- | --- |
| Lock lạc quan (`@Version`) | Phát hiện xung đột tại thời điểm flush/commit | Phía thua phải thử lại toàn bộ transaction |
| Lock bi quan (`FOR UPDATE`) | Khóa bản ghi trước khi đọc | Phía thua xếp hàng chờ |
| Cập nhật nguyên tử (`Conditional UPDATE`) | Tích hợp kiểm tra vào mệnh đề `WHERE` | Phía thua nhận kết quả 0 dòng, không cần thử lại |
| Hàng đợi tuần tự (Serial queue) | Tuần tự hóa yêu cầu trước khi gửi xuống database | Kiểm soát tải tại tầng ứng dụng |

## 5. Thuật ngữ cần biết (Terminology)

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| Bản ghi nóng (`Hot row/Hot key`) | Một bản ghi bị nhiều transaction tranh chấp đồng thời trong thời gian ngắn. |
| Bão thử lại (`Retry storm`) | Hiện tượng các lần thử lại thất bại gây ra thêm nhiều lần thử lại, nhân bản tải hệ thống. |
| Xếp hàng lock (`Lock convoy`) | Nhiều transaction xếp hàng chờ đợi lock trên cùng một bản ghi, giữ kết nối trong suốt thời gian chờ. |
| Kiểm soát đầu vào (`Admission control`) | Cơ chế giới hạn số lượng yêu cầu được phép vào hệ thống tại một thời điểm. |
| Đánh giá lại điều kiện (`Predicate recheck`) | PostgreSQL tự động kiểm tra lại mệnh đề `WHERE` sau khi lấy được lock trên bản ghi đã bị cập nhật. |
| Thông lượng (`Throughput`) | Số lượng yêu cầu xử lý thành công trong một đơn vị thời gian. |
| Tải hệ thống (`System load`) | Tổng tài nguyên tiêu thụ bao gồm cả các lần thử lại thất bại. |
| Hàng đợi tuần tự (`Serial queue`) | Cơ chế tuần tự hóa các yêu cầu tại tầng ứng dụng trước khi gửi xuống database. |

## 6. Điều hướng tài liệu (Navigation)

- [Thiết kế lỗi với @Version và FOR UPDATE (broken-code.md)](broken-code.md)
- [Phân tích hành vi từng chiến lược dưới tải cao (analysis.md)](analysis.md)
- [Giải pháp và khung lựa chọn (solutions.md)](solutions.md)
- [Thực nghiệm so sánh chiến lược (experiments.md)](experiments.md)
- [Lock lạc quan @Version (LOCK-001)](../optimistic-version-conflict/README.md)
- [Thử lại khi tương tranh (LOCK-002)](../optimistic-retry-contention/README.md)
- [Lock bi quan FOR UPDATE (LOCK-003)](../pessimistic-write-for-update/README.md)
- [Cập nhật nguyên tử có điều kiện (LOCK-004)](../conditional-atomic-update/README.md)
- [Serializable retry (DB-009)](../../postgresql/serializable-retry/README.md)
- [Kiểm thử tương tranh](../../concepts/concurrency-testing.md)

## 7. Tác động tới hệ thống (Production impact)

### Khi sử dụng lock lạc quan trên bản ghi nóng
- Tỷ lệ `OptimisticLockException` tăng theo cấp số nhân khi tương tranh tăng.
- Mỗi lần thử lại tiêu thụ một kết nối database và một transaction, nhân bản tải lên 3–5 lần.
- Thời gian phản hồi trung bình tăng vọt do tích lũy backoff.
- Hệ thống bên ngoài nhận kết quả trễ, gây timeout ở tầng API Gateway.

### Khi sử dụng lock bi quan trên bản ghi nóng
- Hàng đợi chờ lock kéo dài, mỗi kết nối bị giữ suốt thời gian chờ.
- Connection pool cạn kiệt nhanh chóng khi hàng đợi vượt quá kích thước pool.
- Các yêu cầu không liên quan đến sản phẩm nóng cũng bị ảnh hưởng do hết kết nối.
- Lỗi `55P03` (lock timeout) hoặc `40P01` (deadlock) tăng đột biến.

### Khi thiếu kiểm soát đầu vào
- Database trở thành nút thắt cổ chai duy nhất cho toàn hệ thống.
- Tải flash-sale lan sang các chức năng khác không liên quan.
- Cascading failure: connection pool → request timeout → load balancer retry → tải tăng thêm.

## 8. Khuyến nghị áp dụng (Applicability)

Bài toán này phù hợp khi:
- Hệ thống có các bản ghi nhận lưu lượng tương tranh cao bất thường (flash-sale, hot event, viral product).
- Chiến lược lock hiện tại bảo vệ tính đúng đắn nhưng gây suy giảm hiệu năng nghiêm trọng.
- Kỹ sư cần quyết định giữa lock lạc quan, lock bi quan, atomic SQL, hoặc hàng đợi tuần tự.
- Mức tương tranh thay đổi theo thời gian: thấp trong ngày thường, cực cao trong sự kiện.

Bài toán KHÔNG đưa ra con số hiệu năng cụ thể (benchmark). Mọi quyết định phải dựa trên đo lường thực tế trên hạ tầng cụ thể.

## 9. Phạm vi tài liệu (Scope boundary)

Bài toán này cung cấp khung lựa chọn chiến lược định tính (qualitative selection framework). Các quyết định cụ thể cho từng nghiệp vụ (tồn kho, thanh toán, booking) thuộc phạm vi của các bài toán nghiệp vụ tương ứng:
- Bán vượt tồn kho: `ECOM-001`, `ECOM-002`.
- Rút tiền trùng lặp: `BANK-001`.
- Giữ chỗ sự kiện: `BOOK-001`.

Bài toán cũng không thảo luận các cơ chế phân tán như Redis lock hoặc Kafka partition; những chủ đề đó thuộc `REDIS-*` và `DIST-*`.
