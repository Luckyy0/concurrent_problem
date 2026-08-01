# Bài toán LOCK-004 — Cập nhật An toàn với Điều kiện (Conditional atomic `UPDATE`)

## 1. Tóm tắt vấn đề (Overview)

Hãy xem xét kịch bản: Kho hàng có mặt hàng số `77` với số lượng tồn (available) là `5`. Có hai transaction đồng thời yêu cầu mua `4` đơn vị sản phẩm.
Nếu ứng dụng xử lý theo luồng Đọc-Kiểm tra-Ghi (Read-Check-Write) cơ bản: Đọc số lượng hiện tại là `5`, kiểm tra `5 > 4` (thỏa mãn), sau đó cập nhật số lượng còn `1`. Do không có cơ chế lock, cả hai transaction đều đọc được giá trị `5` ban đầu và cùng thực hiện cập nhật. Hậu quả là database ghi nhận số lượng tồn kho còn `1`, nhưng hệ thống đã chấp nhận cả 2 yêu cầu (tổng cộng `8` đơn vị). Đây là lỗi mất cập nhật (Lost update) dẫn đến việc bán hàng vượt quá số lượng tồn (Overselling).

Để khắc phục, quy tắc nghiệp vụ nên được tích hợp trực tiếp vào câu lệnh cập nhật thông qua cơ chế cập nhật nguyên tử có điều kiện (Conditional atomic update):

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE product_id = :productId
  AND :quantity > 0
  AND available_quantity >= :quantity;
```

Nếu số dòng bị ảnh hưởng (affected-row count) trả về là `1`, transaction đã đặt hàng thành công. Nếu trả về `0`, lệnh không thỏa mãn điều kiện và không có dữ liệu nào bị thay đổi; ứng dụng phải ghi nhận trạng thái từ chối (ví dụ: hết hàng).

> **Ghi chú quan trọng:** Không nên thực hiện tính toán số lượng tồn kho trên vùng nhớ ứng dụng (RAM) rồi ghi đè số tuyệt đối xuống database. Hãy thực thi việc trừ số lượng bằng một câu lệnh database nguyên tử (atomic operation) để đảm bảo tính nhất quán dữ liệu tại thời điểm ghi.

## 2. Các thực thể và Trạng thái chia sẻ (Actors and shared state)

| Thực thể | Trạng thái ban đầu |
| --- | --- |
| Bảng tồn kho (`inventory_item`) | Sản phẩm `77`, có sẵn (`available`) `5`, đã giữ (`reserved`) `0` |
| Lệnh A | Yêu cầu `4` sản phẩm, thực thi trên Máy chủ 1 |
| Lệnh B | Yêu cầu `4` sản phẩm, thực thi trên Máy chủ 2 |
| Bảng giữ chỗ (`inventory_reservation`) | Lưu lịch sử kết quả dựa trên mã lệnh (command ID) |
| Hộp thư gửi đi (`outbox_event`) | Chỉ được tạo ra khi đơn hàng đã commit thành công |

Điểm tranh chấp tập trung tại bản ghi của sản phẩm số `77`. Hai luồng xử lý xuất phát từ hai yêu cầu khác nhau nên cơ chế chống trùng lặp (Idempotency) không giải quyết được vấn đề này; phương pháp cập nhật kèm điều kiện (conditional mutation) là giải pháp cốt lõi để bảo vệ tính nhất quán của dữ liệu tồn kho.

## 3. Các quy tắc bất biến (Invariants)

Trong điều kiện không có transaction nhập/xuất kho nào khác tác động, hệ thống phải đảm bảo:

```text
Số lượng hàng có sẵn (available_quantity) >= 0

Số lượng hàng đã giữ chỗ (reserved_quantity) = Tổng số lượng của các lệnh có trạng thái RESERVED

Hàng có sẵn + Hàng đã giữ chỗ = Tổng hàng ban đầu trong kho

Mỗi mã lệnh (Command ID) chỉ được tạo ra MỘT kết quả duy nhất.

Phía gọi chỉ nhận được thông báo hoàn tất sau khi transaction đã thực sự commit.
```

Lưu ý: Mặc dù có thể thiết lập ràng buộc (Constraint) trong database như `CHECK (available_quantity >= 0)` như một lớp phòng thủ sâu (Defense in depth), điều này vẫn chưa đủ để ngăn chặn lỗi ghi đè mất dữ liệu (Lost update) nếu luồng logic ở tầng ứng dụng bị thiết kế sai.

## 4. Ranh giới transaction (Transaction boundaries)

Một quy trình xử lý `InventoryReservationTx.reserve()` cần tuân thủ các bước sau:

1. Đăng ký mã lệnh (Command ID) để đảm bảo tính duy nhất, hoặc đọc lại kết quả nếu mã lệnh đã được xử lý.
2. Thực thi câu lệnh SQL `UPDATE ... RETURNING` có kèm điều kiện.
3. Nếu trả về `0` dòng: Ghi nhận trạng thái `OUT_OF_STOCK` (hết hàng) và KHÔNG tạo bản ghi sự kiện (outbox event).
4. Nếu trả về `1` dòng: Ghi nhận trạng thái `RESERVED` kèm số lượng tồn kho còn lại, đồng thời tạo bản ghi sự kiện (outbox event).
5. Xác nhận (Commit) hoặc Hủy (Rollback) TOÀN BỘ các thao tác trên trong cùng một transaction.

Tuyệt đối KHÔNG thực hiện các tác vụ ngoại vi (remote I/O) như gọi API thanh toán, gửi thông điệp Kafka/RabbitMQ, hoặc logic tạm dừng (Thread.sleep) bên trong transaction này. Nếu quá trình xử lý thất bại, lệnh Rollback phải hoàn trả toàn bộ số lượng tồn kho như trước khi thực thi.

## 5. Cơ chế hoạt động của cập nhật đồng thời (Concurrent UPDATE behavior)

Giả định PostgreSQL đang hoạt động ở mức cô lập `READ COMMITTED` (mặc định):

```text
Transaction A (Tx-A) thực hiện UPDATE: Đánh giá 5 >= 4 → Hợp lệ → lock bản ghi (Row lock), cập nhật số lượng tồn còn 1.
Transaction B (Tx-B) thực hiện UPDATE: Yêu cầu lock trên cùng bản ghi → Bị chặn, chuyển sang trạng thái chờ (Wait).
Tx-A COMMIT: Giải phóng lock.
Tx-B: Database tự động đánh giá lại điều kiện (Predicate recheck) trên phiên bản dữ liệu mới nhất → 1 >= 4 (Không hợp lệ) → Trả về 0 dòng cập nhật.
```

Nếu Tx-A bị hủy (Rollback), Tx-B sẽ đánh giá lại trên giá trị gốc là `5`, điều kiện hợp lệ và cập nhật thành công (ảnh hưởng 1 dòng).
Sức mạnh của phương pháp này nằm ở việc tính toán số lượng được database thực thi trên **phiên bản dữ liệu mới nhất (current row version)** đã commit, thay vì dựa trên dữ liệu mà tầng ứng dụng đọc được ban đầu.

## 6. Hợp đồng kết quả (Outcome contract)

| Kết quả từ database | Trạng thái nghiệp vụ |
| --- | --- |
| Số dòng bị ảnh hưởng (Affected rows) là `1` | `RESERVED` (Thành công) |
| Số dòng bị ảnh hưởng là `0` (sản phẩm tồn tại) | `OUT_OF_STOCK` (Hết hàng) |
| Lỗi `55P03` (Lock timeout) | `BUSY` (Hệ thống quá tải), thực hiện Rollback |
| Lỗi `40P01` (deadlock) hoặc `40001` (Serialization failure) | Lỗi kỹ thuật; Hệ thống có thể thử lại (Retry) một cách an toàn |
| Lỗi ràng buộc (Constraint / Insert / Outbox) sau UPDATE | Thực hiện Rollback toàn bộ transaction |
| Mã lệnh trùng lặp khớp hoàn toàn (Duplicate fingerprint) | Trả về kết quả cũ, KHÔNG trừ hàng nhiều lần |
| Mã lệnh trùng lặp sai dữ liệu (Different fingerprint) | Lỗi `IDEMPOTENCY_MISMATCH` (Xung đột dữ liệu) |

Lưu ý: Kết quả "Số dòng bị ảnh hưởng = 0" có thể do nhiều nguyên nhân (như hết hàng hoặc sai ID). Tầng ứng dụng cần phân biệt rõ các trường hợp này để trả về mã lỗi nghiệp vụ phù hợp.

## 7. Các thuật ngữ kỹ thuật (Terminology)

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| Cập nhật kèm điều kiện (`Conditional mutation`) | Chỉ thực hiện thay đổi dữ liệu nếu điều kiện nghiệp vụ vẫn đúng tại thời điểm ghi. |
| Câu lệnh nguyên tử (`Atomic statement`) | Tích hợp kiểm tra logic (WHERE) và thay đổi dữ liệu (SET) vào một câu lệnh duy nhất. |
| Số dòng bị ảnh hưởng (`Affected-row count`) | Giá trị hệ thống trả về, biểu thị số bản ghi thực sự bị thay đổi. |
| Đánh giá lại điều kiện (`Predicate recheck`) | Cơ chế của PostgreSQL tự động kiểm tra lại mệnh đề `WHERE` sau khi lấy được lock. |
| Phiên bản dòng mới nhất (`Current row version`) | Dữ liệu mới nhất đã được commit mà câu lệnh tương tác trực tiếp. |
| Mệnh đề `RETURNING` | Lệnh trong PostgreSQL cho phép trả về trực tiếp giá trị dữ liệu ngay sau khi cập nhật. |
| Hành động rỗng (`No-op`) | Lệnh thực thi thành công nhưng không có bản ghi nào thỏa mãn điều kiện để thay đổi. |
| Cập nhật hàng loạt (`Bulk DML`) | Cập nhật trực tiếp bằng SQL, bỏ qua bộ đệm của ORM (như Hibernate/JPA). |
| Phòng thủ nhiều lớp (`Defense in depth`) | Sử dụng Constraint tại database để bảo vệ dữ liệu bên cạnh logic tại tầng ứng dụng. |

## 8. Điều hướng tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu lock Và SSI (analysis.md)](analysis.md)
- [Giải Pháp Cập Nhật An Toàn (solutions.md)](solutions.md)
- [Thực Nghiệm Với Testcontainers (experiments.md)](experiments.md)
- [Thao Tác Nguyên Tử (Atomic database operations)](../../concepts/atomic-database-operations.md)
- [Vòng Đời lock PostgreSQL (PostgreSQL locks)](../../concepts/postgresql-locks.md)
- [Kiểm Thử Tương Tranh (Concurrency testing)](../../concepts/concurrency-testing.md)
- [Lỗi Mất Cập Nhật (DB-001)](../../postgresql/lost-update-mvcc/README.md)
- [Vòng Đời lock Hàng Bảng (DB-007)](../../postgresql/row-table-lock-lifecycle/README.md)

## 9. Tác động tới hệ thống (Production Impact)

Nếu áp dụng giải pháp không đúng cách, hệ thống có thể đối mặt với:
- Bán vượt quá số lượng hàng tồn (Overselling).
- Sai lệch số liệu giữa bảng tồn kho và bảng lịch sử (Audit data mismatch).
- Bỏ qua kết quả số dòng bị ảnh hưởng, xem lệnh thực thi như luôn thành công và trả về thông tin sai cho người dùng.
- Sử dụng lệnh Bulk DML trực tiếp nhưng không làm sạch bộ đệm (Persistence context) của JPA, dẫn đến trạng thái dữ liệu cũ (Stale state) bị lưu đè trong quá trình flush.
- Không xử lý tính lũy đẳng (Idempotency), cho phép thực hiện lại một lệnh và dẫn đến trừ hàng nhiều lần.
- Đánh đồng lỗi hết thời gian chờ lock (Lock timeout) với lỗi hết hàng (OUT_OF_STOCK), làm ẩn giấu tình trạng quá tải của database.
- Phát ra thông điệp (Publish event) trước thời điểm commit. Khi transaction Rollback, hệ thống ngoại vi sẽ nhận các thông tin không chính xác.
- Sử dụng từ khóa đồng bộ cấp JVM (`synchronized`): Không có tác dụng trong kiến trúc hệ thống phân tán (Multi-instance architecture).

## 10. Khuyến nghị áp dụng (Applicability)

Phương pháp cập nhật nguyên tử có điều kiện (Conditional atomic UPDATE) là giải pháp tối ưu khi:
- Các thuộc tính dữ liệu và điều kiện kiểm tra nằm gọn trong một bản ghi duy nhất.
- Logic kiểm tra có thể được diễn đạt rõ ràng thông qua mệnh đề `WHERE` và `SET` của SQL.
- Transaction thất bại do không thỏa mãn điều kiện chỉ cần nhận kết quả 0 dòng (Affected rows = 0) thay vì phát sinh ngoại lệ phức tạp.
- Ứng dụng cần ưu tiên hiệu năng: Một câu lệnh ngắn gọn sẽ tối ưu hơn quy trình mở lock - tải dữ liệu - tính toán - cập nhật.

Nếu quy trình xử lý phức tạp, ảnh hưởng chéo đến nhiều bảng hoặc các bản ghi chưa tồn tại, hãy cân nhắc sử dụng lock bi quan `PESSIMISTIC_WRITE` (LOCK-003) hoặc cơ chế cô lập mức độ cao (`SERIALIZABLE`). Nếu cần xử lý đối tượng lớn có tần suất xung đột thấp, lock lạc quan `@Version` là một sự lựa chọn phù hợp.

## 11. Phạm vi tài liệu (Scope boundary)

Case study này tập trung phân tích phương pháp biến đổi logic kiểm tra thành câu lệnh UPDATE có điều kiện trên một bản ghi xác định.
- Để tìm hiểu về lỗi mất dữ liệu tổng quát, tham khảo bài `DB-001`.
- Để nắm rõ vòng đời cơ chế lock, tham khảo bài `DB-007`.
- Đối với chiến lược xử lý khi hệ thống bị quá tải tương tranh (High contention/Throttling), tham khảo bài `LOCK-005`.
