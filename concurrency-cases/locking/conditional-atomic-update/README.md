# Bài toán LOCK-004 — Cập nhật an toàn với điều kiện (Conditional atomic `UPDATE`)

## Tóm tắt câu chuyện

Tưởng tượng kho hàng của bạn có mặt hàng số `77` đang còn đúng `5` chiếc. Có hai khách hàng cùng lúc bấm nút thanh toán, mỗi người đều muốn mua `4` chiếc.
Nếu ứng dụng của bạn làm theo kiểu ngây thơ: Đọc kho lên thấy còn `5`, kiểm tra thấy `5 > 4` là thỏa mãn, rồi ghi đè con số `1` (5 trừ 4) xuống Database... thì cả hai luồng sẽ cùng thấy hợp lệ! Kết cục là Database bị ghi đè chỉ còn `1`, nhưng bạn đã nhận đặt cọc của cả 2 khách (tổng cộng `8` chiếc) trong khi kho chỉ có `5`. Chào mừng bạn đến với thảm họa thất thoát hàng!

Khi quy tắc nghiệp vụ có thể gói gọn lại trong một điều kiện, hãy gửi thẳng "ý định" của bạn xuống Database để nó tự xử lý:

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE product_id = :productId
  AND :quantity > 0
  AND available_quantity >= :quantity;
```

Nếu số dòng bị ảnh hưởng (affected-row count) trả về là `1`, nghĩa là bạn đã đặt hàng thành công. Nếu trả về `0`, nghĩa là câu lệnh không thỏa mãn điều kiện và chẳng có gì thay đổi; lúc này ứng dụng KHÔNG ĐƯỢC PHÉP báo thành công hay lưu dữ liệu đặt hàng.

> **Nói ngắn gọn:** Đừng bao giờ hỏi Database "Còn hàng không?" rồi đem về RAM tính toán xong mới ghi lại. Hãy ra lệnh trực tiếp: "Chỉ trừ hàng giùm tôi nếu ngay tại giây phút ghi, số lượng vẫn còn đủ".

## Các diễn viên và Trạng thái tranh chấp

| Thành phần | Trạng thái ban đầu |
| --- | --- |
| Bảng tồn kho (`inventory_item`) | Sản phẩm `77`, đang có sẵn (available) `5`, đã giữ chỗ (reserved) `0` |
| Lệnh A | Đơn `A`, số lượng mua `4`, chạy trên Máy chủ 1 |
| Lệnh B | Đơn `B`, số lượng mua `4`, chạy trên Máy chủ 2 |
| Bảng giữ chỗ (`inventory_reservation`) | Bảng lưu lịch sử kết quả dựa trên mã lệnh (command ID) |
| Hộp thư gửi đi (`outbox_event`) | Chỉ được tạo ra khi đơn hàng đã chốt (commit) thành công |

Điểm tranh giành đẫm máu nhất chính là dòng dữ liệu của sản phẩm số 77. Hai luồng đến từ hai nơi khác nhau mang mã khác nhau, nên cơ chế chống trùng lặp (idempotency) không giúp ích gì ở đây; chính mẹo "cập nhật kèm điều kiện" (conditional mutation) mới là tấm khiên bảo vệ kho hàng.

## Những quy tắc bất di bất dịch (Invariant)

Khi không có lệnh nhập/xuất kho nào xen ngang, ta phải đảm bảo:

```text
Số lượng hàng có sẵn (available_quantity) >= 0

Số lượng hàng đã giữ chỗ (reserved_quantity) 
  = Tổng số lượng của các đơn có trạng thái RESERVED (đã giữ)

Hàng có sẵn + Hàng đã giữ chỗ = Tổng hàng ban đầu trong kho

Mỗi mã lệnh (command ID) chỉ được phép tạo ra ĐÚNG MỘT kết quả.

Hệ thống gọi (Caller) chỉ nhận được thông báo RESERVED sau khi giao dịch đã thực sự chốt (commit).
```

Lưu ý: Bạn có thể cài thêm cờ kiểm tra dưới Database `CHECK (available_quantity >= 0)` để phòng hờ (defense in depth), nhưng chừng đó là chưa đủ để phát hiện lỗi ghi đè mất dữ liệu (lost update).

## Ranh giới Giao dịch (Transaction)

Một lần thử chạy hàm `InventoryReservationTx.reserve()` sẽ trải qua các bước:

1. Đăng ký cái mã lệnh (command ID) hoặc đọc lại kết quả nếu lệnh này từng chạy rồi.
2. Bắn câu SQL `UPDATE ... RETURNING` có kèm điều kiện.
3. Nếu trả về `0` dòng: Lưu trạng thái `OUT_OF_STOCK` (hết hàng) và KHÔNG tạo thư gửi đi (outbox).
4. Nếu trả về `1` dòng: Lưu trạng thái `RESERVED` cùng số lượng còn lại và tạo thư gửi đi.
5. Chốt (commit) hoặc Hủy (rollback) TẤT CẢ các bước trên cùng lúc.

Tuyệt đối KHÔNG nhét các việc như: gọi API thanh toán, gửi tin nhắn RabbitMQ/Kafka, hoặc vòng lặp chờ đợi (retry wait) vào trong Giao dịch này. Nếu quá trình UPDATE sau đó bị lỗi, lệnh Rollback phải hoàn trả lại toàn bộ số đếm kho như cũ.

## Tại sao nhiều luồng cùng `UPDATE` mà vẫn không sai?

Giả sử PostgreSQL đang ở mức cách ly `READ COMMITTED` (mặc định):

```text
Giao dịch A (Tx-A) UPDATE: Kiểm tra 5 >= 4 → Đúng → Dòng này bị khóa, số lượng còn 1
Giao dịch B (Tx-B) UPDATE: Đòi đánh vào đúng dòng đó → Bị chặn lại, phải đứng chờ
Tx-A COMMIT (Chốt sổ): Chìa khóa được nhả ra
Tx-B: Tự động đánh giá lại điều kiện WHERE trên dữ liệu thực tế mới nhất → 1 >= 4 là Sai → Báo cập nhật 0 dòng
```

Nếu Tx-A xui xẻo bị Rollback, Tx-B sẽ đánh giá lại trên con số `5` gốc và nó sẽ giành chiến thắng (cập nhật 1 dòng).
Sức mạnh ở đây là phép cộng/trừ được Database tự làm trên **phiên bản dữ liệu mới nhất dưới ổ cứng**, chứ ứng dụng không hề gửi một con số cụ thể tính từ đồ "thiu" trên RAM.

## Bảng quy ước kết quả (Outcome contract)

| Database trả về | Kết quả nghiệp vụ |
| --- | --- |
| Sửa được 1 dòng (affected rows `1`) | Báo `RESERVED` (Giữ chỗ thành công) |
| Sửa 0 dòng (và chắc chắn dòng đó có tồn tại) | Báo `OUT_OF_STOCK` (Hết hàng) |
| Báo lỗi `55P03` do hết giờ chờ khóa | Báo `BUSY` (Hệ thống bận), sau đó Rollback |
| Báo lỗi `40P01` hoặc `40001` | Lỗi kỹ thuật; Có thể tự động thử lại (retry) nếu an toàn |
| Bị văng lỗi Constraint / Insert / Outbox sau khi UPDATE | Phải Rollback lại toàn bộ quá trình |
| Trùng mã lệnh y chang (Duplicate fingerprint) | Trả về kết quả cũ, KHÔNG bị trừ hàng 2 lần |
| Trùng mã lệnh nhưng dữ liệu sai lệch (Different fingerprint)| Báo `IDEMPOTENCY_MISMATCH` (Lỗi xung đột dữ liệu) |

Lưu ý: Kết quả "Sửa 0 dòng" đôi khi bị hiểu nhầm là do "sản phẩm không tồn tại". Code của bạn phải đủ thông minh để phân biệt 2 trường hợp này, đừng có ngồi đoán mò từ con số `0`.

## Các thuật ngữ kỹ thuật cần thuộc lòng

| Thuật ngữ | Ý nghĩa trong bài toán này |
| --- | --- |
| Cập nhật kèm điều kiện (`conditional mutation`) | Chỉ sửa dữ liệu nếu như điều kiện nghiệp vụ vẫn còn đúng. |
| Câu lệnh `UPDATE` nguyên tử (`atomic UPDATE`) | Gom việc kiểm tra (WHERE) và việc sửa dữ liệu (SET) vào chung 1 câu lệnh duy nhất. |
| Số dòng bị ảnh hưởng (`affected-row count`) | Con số trả về báo hiệu có bao nhiêu dòng thực sự đã được sửa. |
| Đánh giá lại điều kiện (`predicate recheck`) | Sự thông minh của PostgreSQL: Nó tự động xét lại cụm `WHERE` ngay khi đối thủ vừa nhả khóa ra. |
| Phiên bản dòng mới nhất (`current row version`) | Dữ liệu tươi mới nhất đang có dưới ổ cứng mà câu lệnh được phép đụng vào. |
| Mệnh đề `RETURNING` | Một chiêu của PostgreSQL giúp trả về kết quả ngay sau khi Update thành công. |
| Không làm gì cả (`no-op`) | Lệnh chạy thành công nhưng chả có dòng nào thỏa mãn để thay đổi. |
| Cập nhật hàng loạt (`bulk DML`) | Cập nhật trực tiếp bằng SQL, lách qua mặt bộ đệm của JPA/Hibernate. |
| Phòng thủ nhiều lớp (`defense in depth`) | Dùng rào chắn (Constraint) của DB để bảo hiểm thêm, nhưng nó không thay thế được logic code. |

## Điều hướng tài liệu

- [Code read–check–write bị hỏng](broken-code.md)
- [Timeline, row lock và predicate recheck](analysis.md)
- [Spring/JPA/JDBC solution và trade-offs](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [Atomic database operations](../../concepts/atomic-database-operations.md)
- [PostgreSQL locks và lock lifetime](../../concepts/postgresql-locks.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)
- [DB-001 — Lost update dưới MVCC](../../postgresql/lost-update-mvcc/README.md)
- [DB-007 — Row/table lock lifecycle](../../postgresql/row-table-lock-lifecycle/README.md)

## Hậu quả thảm khốc trên Production nếu dùng sai

- Chấp nhận giữ chỗ nhiều hơn số lượng hàng tồn (bán âm kho).
- Báo cáo dữ liệu (Projection) và bảng lịch sử (Audit) vênh nhau.
- Viết lệnh UPDATE nhưng phớt lờ số dòng trả về (affected rows `0`), cứ ngỡ là cập nhật thành công và trả về Success cho khách!
- Gọi Bulk DML trực tiếp nhưng lại vướng bộ đệm (stale managed entity) của JPA, để rồi lúc sau Hibernate gọi flush và ghi đè làm mất kết quả atomic vừa chạy.
- Cho phép "Thử lại" (Retry) cùng một mã lệnh nhưng lại quên lưu lịch sử chống trùng (durable claim), dẫn đến bị trừ hàng nhiều lần.
- Cố tình bắt lỗi hết hạn chờ khóa (lock timeout) rồi báo luôn là "Hết hàng" (OUT_OF_STOCK) để che giấu chuyện Server đang quá tải.
- Gửi tin nhắn Kafka/RabbitMQ trước khi chốt sổ (commit), cuối cùng giao dịch lại bị Rollback, đẩy xuống hệ thống dưới một tin nhắn sai sự thật!
- Dùng từ khóa `synchronized` của Java: Code chạy mượt trên máy bạn, nhưng khi đưa lên nhiều máy chủ (scale-out) thì toang toàn tập.

## Lời khuyên: Khi nào nên áp dụng tuyệt chiêu này?

SQL Cập nhật Nguyên tử cực kỳ phù hợp khi:
- Dữ liệu bị trừ và điều kiện kiểm tra nằm chung trên đúng 1 dòng (dễ khoanh vùng).
- Bạn có thể diễn đạt logic ràng buộc thành cụm `WHERE` và `SET` của SQL.
- Kẻ chậm chân (loser) có thể ngậm ngùi nhận kết quả 0 dòng (affected-row `0`).
- Ứng dụng không cần phải móc cả đống bảng (load aggregate graph) lên RAM để quyết định.
- Bạn cần tốc độ: 1 câu lệnh ngắn gọn luôn nhanh hơn việc Mở khóa -> Tải lên -> Tính toán -> Ghi lại.

Tuy nhiên, nếu quy trình duyệt đơn của bạn phức tạp phải check qua hàng chục bước, hãy xem xét dùng Khóa bi quan `PESSIMISTIC_WRITE` (Bài LOCK-003). Nếu ít khi đụng độ nhưng đối tượng rất to, hãy dùng Khóa lạc quan `@Version`. Và nếu quy tắc dính đến những dòng dữ liệu chưa tồn tại (missing rows), 1 câu lệnh UPDATE là không đủ.

## Phạm vi tài liệu

Case study này chỉ tập trung vào việc biến điều kiện nghiệp vụ thành câu lệnh UPDATE có điều kiện trên một dòng đã biết.
Về lỗi mất dữ liệu chung chung, hãy xem `DB-001`. Về cơ chế hoạt động của khóa, xem `DB-007`. Về chiến thuật sinh tồn khi bị ngập lụt truy cập liên tục, xem bài `LOCK-005`.
