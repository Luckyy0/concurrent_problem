# Phân tích Chuyên sâu — Cơ Chế Tích Hợp Kiểm Tra (Predicate) và Ghi Dữ Liệu (Mutation)

## 1. Trạng thái Ban đầu

Giả định cấu trúc dữ liệu trong database:
```text
Sản phẩm số 77:
  Tổng trong kho (on_hand_quantity)   = 5
  Đang có sẵn (available_quantity)    = 5
  Đã giữ chỗ (reserved_quantity)      = 0
  Phiên bản (revision)                = 10

Danh sách đơn đã giữ chỗ (inventory_reservation) = Rỗng
```

Lệnh A và Lệnh B đại diện cho hai yêu cầu hợp lệ, mỗi yêu cầu cần mua `4` sản phẩm. Các luồng này được xử lý độc lập trên hai máy chủ ứng dụng (application instances) với kết nối và transaction riêng biệt.

## 2. Kịch bản Sai lầm: Đọc-Kiểm tra-Ghi (Read-Check-Write)

| Bước | Transaction A (Máy chủ 1) | Transaction B (Máy chủ 2) | Hoạt động của PostgreSQL |
| --- | --- | --- | --- |
| 1 | `SELECT` bình thường → Đọc giá trị 5/0 | | Cung cấp Snapshot cho transaction A |
| 2 | | `SELECT` bình thường → Đọc giá trị 5/0 | Cung cấp Snapshot cho transaction B |
| 3 | Ứng dụng đánh giá: `5 >= 4` → Hợp lệ | Ứng dụng đánh giá: `5 >= 4` → Hợp lệ | Database không nắm bắt logic ứng dụng |
| 4 | Ứng dụng tính toán: Tồn 1 / Giữ 4 | Ứng dụng tính toán: Tồn 1 / Giữ 4 | |
| 5 | Insert lịch sử đơn A | Insert lịch sử đơn B | Khác mã lệnh nên thực thi thành công |
| 6 | `UPDATE` giá trị thành 1/4 | | Cấp lock bản ghi (Row lock) cho A |
| 7 | Xác nhận transaction (Commit) | `UPDATE` giá trị thành 1/4 (Đang chờ) | B chờ lock của A giải phóng |
| 8 | | Xác nhận transaction (Commit) | Câu lệnh của B không có điều kiện `WHERE`, ghi đè thành công (Affected rows = 1) |

Hệ quả: Trạng thái bảng tồn kho vẫn hợp lệ ở mặt toán học (1 >= 0), nhưng thực tế đã chấp nhận 2 đơn đặt hàng, tổng cộng 8 sản phẩm. Đây là lỗi ghi đè mất dữ liệu (Lost update), dẫn đến trạng thái bán hàng vượt quá số lượng tồn (Overselling).

## 3. Đối chiếu: Mong Đợi và Thực Tế

| Tiêu chí | Mong đợi (Expected) | Thực tế lỗi (Broken State) |
| --- | --- | --- |
| Số lượng đơn được duyệt | 1 đơn | 2 đơn |
| Tổng số lượng trong lịch sử | 4 đơn vị | 8 đơn vị (Bán âm kho) |
| Giá trị `reserved_quantity` | 4 đơn vị | 4 đơn vị |
| Giá trị `available_quantity` | 1 đơn vị | 1 đơn vị |
| Khớp sổ (Reconciliation) | Khớp hoàn toàn | Lệch 4 đơn vị |
| Trạng thái của đơn đến sau | `OUT_OF_STOCK` | `RESERVED` |

## 4. Giải Pháp: Cập Nhật Nguyên Tử (Atomic Statement)

Gộp logic nghiệp vụ vào một câu lệnh SQL duy nhất:

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity,
    revision = revision + 1
WHERE product_id = :productId
  AND :quantity > 0
  AND available_quantity >= :quantity
RETURNING available_quantity, reserved_quantity, revision;
```

Mệnh đề `WHERE` đóng vai trò là lớp bảo vệ (Predicate guard); mệnh đề `SET` thực thi việc sửa đổi (Mutation). PostgreSQL đảm bảo toàn bộ lệnh này được xử lý nguyên tử (Atomically) dựa trên phiên bản dữ liệu mới nhất (Current row version).

## 5. Kịch bản Thành công: Cập nhật Kèm điều kiện (Conditional UPDATE)

| Bước | Transaction A | Transaction B | Hoạt động của PostgreSQL |
| --- | --- | --- | --- |
| 1 | `UPDATE` mua 4 đơn vị | | Đánh giá `5 >= 4` → Hợp lệ, giá trị tạm thời là 1/4 |
| 2 | A duy trì lock (Chưa commit) | `UPDATE` mua 4 đơn vị | B yêu cầu lock, chuyển sang trạng thái chờ |
| 3 | Xác nhận transaction (Commit) | Đang chờ | A giải phóng lock bản ghi |
| 4 | | B tự động đánh giá lại (Recheck) | Cấp dữ liệu mới nhất: `available` = 1 |
| 5 | | Trả về 0 dòng bị ảnh hưởng | Đánh giá `1 >= 4` → Sai điều kiện |
| 6 | | Lưu lịch sử `OUT_OF_STOCK`, Commit | Lệnh B không cập nhật bảng tồn kho |

> **Ghi chú quan trọng:** Luồng B thất bại không phải do logic ứng dụng đọc lại dữ liệu, mà do hệ quản trị database (PostgreSQL) tự động đánh giá lại mệnh đề điều kiện (Predicate recheck) dựa trên trạng thái đã commit của luồng A.

## 6. Mức Cô lập `READ COMMITTED` và Đánh giá Lại (Predicate Recheck)

Trong mức cô lập `READ COMMITTED`, PostgreSQL tìm bản ghi dựa trên Snapshot. Khi bản ghi đang bị lock, transaction sẽ chờ (Wait) cho đến khi transaction đang giữ lock kết thúc (Commit hoặc Rollback).

### Khi transaction giữ lock thành công (Holder Commit)
PostgreSQL sẽ lấy phiên bản dữ liệu vừa được cập nhật và đánh giá lại mệnh đề `WHERE` (Predicate recheck).
- Nếu điều kiện vẫn đúng, luồng B tiến hành cập nhật.
- Nếu điều kiện sai, luồng B bỏ qua cập nhật và trả về số dòng bị ảnh hưởng là `0` (Affected rows = 0).

### Khi transaction giữ lock hủy bỏ (Holder Rollback)
Luồng B đánh giá trên phiên bản dữ liệu ban đầu, điều kiện đúng, và hoàn tất cập nhật thành công (Affected rows = 1).

Lưu ý: Cơ chế đánh giá lại chỉ hoạt động tin cậy với các điều kiện đơn giản trên cùng một dòng. Không áp dụng cho các truy vấn phức tạp chứa `JOIN` hoặc truy vấn chéo nhiều bảng.

## 7. Hành vi Cấp lock Bản ghi (Row Lock)

Lệnh `UPDATE` tự động cấp phát lock độc quyền trên bản ghi (Row-level lock). Những luồng đồng thời sẽ phải đối mặt với các tình huống:
- Chờ đợi và sau đó tự động đánh giá lại (Recheck).
- Vượt quá thời gian chờ (lock timeout), hệ thống phát sinh lỗi `55P03`.
- Rơi vào trạng thái chờ chéo (deadlock), hệ thống phát sinh lỗi `40P01`.

Lệnh `UPDATE` duy trì lock cho đến khi toàn bộ transaction kết thúc (Commit/Rollback). Do đó, cấm tích hợp các tác vụ ngoại vi (như gọi API, độ trễ mạng) sau lệnh `UPDATE` để tránh tắc nghẽn toàn hệ thống (Connection pool exhaustion).

## 8. Giao thức Dựa Trên Số Dòng (Affected-row Protocol)

Số dòng bị ảnh hưởng (Affected-row count) là phản hồi kỹ thuật để định hướng logic ứng dụng:

```text
1 dòng  → Transaction thỏa mãn điều kiện và cập nhật thành công.
0 dòng  → Lệnh không thỏa mãn điều kiện, không có thay đổi (Nên trả về trạng thái nghiệp vụ: Từ chối/Hết hàng).
>1 dòng → Cảnh báo lỗi logic, câu lệnh ảnh hưởng nhiều bản ghi hơn dự kiến.
```

Số lượng `0` không phát sinh ngoại lệ (Exception). Ứng dụng phải tự kiểm tra biến đếm này để điều hướng quy trình tạo đơn hàng một cách chính xác.

## 9. Mệnh Đề `RETURNING` để Tối Ưu

Việc sử dụng `UPDATE ... RETURNING` mang lại trạng thái dữ liệu chính xác nhất ngay sau khi cập nhật:
- Trả về 1 dòng dữ liệu → Cập nhật thành công và cung cấp dữ liệu tồn kho hiện tại.
- Trả về kết quả rỗng → Không có dữ liệu bị thay đổi.
Việc tách rời thao tác thành 2 câu lệnh (`UPDATE` rồi `SELECT`) có nguy cơ cao đọc nhầm dữ liệu từ một transaction khác vừa thực hiện cập nhật thành công (Read-after-write concurrency issue).

## 10. Phân loại Trường hợp Trả về 0 Dòng (Zero Rows Mapped)

Kết quả `0` dòng có thể phát sinh do:
- Bản ghi sản phẩm không tồn tại.
- Tham số truyền vào không hợp lệ (ví dụ: mua 0 sản phẩm).
- Bản ghi bị vô hiệu hóa (Disabled/Locked).
- Sản phẩm không đủ số lượng tồn kho (Hết hàng).

Đối với các quy trình phức tạp, cần kiểm tra kỹ đầu vào (Input validation) và tách biệt logic tìm kiếm bản ghi để phân định chính xác nguyên nhân dẫn đến kết quả 0 dòng, từ đó phản hồi mã lỗi (error code) phù hợp.

## 11. Xử Lý Bộ Đệm của Hibernate (Persistence Context Caching)

Khi thực thi SQL nguyên tử (Bulk DML) qua database, bộ đệm của JPA/Hibernate (Persistence context) sẽ không được tự động cập nhật.
- Đối tượng (Entity) trên vùng nhớ vẫn giữ giá trị cũ (Stale state).
- Trả đối tượng này về tầng API sẽ cung cấp dữ liệu không chính xác.
- Nếu bộ đệm bị buộc đẩy (Flush), trạng thái cũ có thể vô tình ghi đè lại dữ liệu vừa cập nhật bằng SQL, làm phá vỡ cơ chế nguyên tử.

**Các bước phòng thủ:**
1. Tránh truy xuất (Load) đối tượng bằng JPA nếu chỉ cần thực thi cập nhật SQL.
2. Nếu bắt buộc tải, đảm bảo thực hiện `flush` mọi thay đổi nháp trước, chạy lệnh SQL, sau đó làm sạch bộ đệm bằng `clear()`.
3. Khi cần tái sử dụng đối tượng, yêu cầu Hibernate tải lại từ database thông qua phương thức `refresh()`.
Sử dụng cờ `clearAutomatically=true` cần thận trọng, vì nó có thể loại bỏ toàn bộ các thay đổi chưa được ghi của các thực thể khác trong cùng transaction.

## 12. Thiết Kế Chuỗi Transaction (Transaction Composition)

Một transaction tiêu chuẩn bao gồm:
```text
BẮT ĐẦU TRANSACTION
  Lưu mã định danh chống trùng (Command ID).
  Thực thi cập nhật có điều kiện (Conditional UPDATE).
  Ghi nhận kết quả lịch sử (Thành công/Từ chối).
  Lưu thông điệp gửi ra ngoại vi (Outbox event) NẾU thành công.
CHỐT TRANSACTION (COMMIT)
```
Nếu khâu tạo Outbox event gặp lỗi, toàn bộ transaction phải được Rollback nhằm đảm bảo số tồn kho được phục hồi và duy trì tính nguyên vẹn dữ liệu (Data consistency). Không sử dụng các khối `try-catch` lén lút (Swallowing exceptions) để tiếp tục duy trì trạng thái transaction lơ lửng.

## 13. Tính Lũy Đẳng và Chống Trùng Lặp (Idempotency)

Phương pháp nguyên tử ngăn lỗi bán lố nhưng KHÔNG bảo vệ hệ thống khỏi việc khách hàng lặp lại lệnh gọi (Duplicate request), dẫn đến trừ hàng nhiều lần. Cần tích hợp khóa `command_id`:

```sql
INSERT INTO inventory_reservation (...)
VALUES (...)
ON CONFLICT (command_id) DO NOTHING
RETURNING command_id;
```

Cơ chế chặn theo khóa (Unique constraint) đảm bảo tính duy nhất của lệnh gọi. Lệnh kiểm tra lũy đẳng phải thuộc cùng một transaction vật lý với lệnh cập nhật tồn kho.

## 14. Kịch Bản Hoàn Tác và Lỗi Phản Hồi (Recovery Timeline)

| Tình huống Lỗi | Trạng thái Database | Phương pháp Phục hồi (Recovery) |
| --- | --- | --- |
| Lỗi mạng trước khi `UPDATE` | Không thay đổi | Khách hàng thực hiện lại (Fresh retry) |
| Lỗi sau `UPDATE`, trước khi Commit | Tự động Rollback | Khách hàng thực hiện lại lệnh bằng Command ID cũ |
| Commit THÀNH CÔNG, lỗi mất mạng | Đã chốt số lượng | Khách hàng phát lại Command ID, hệ thống trả về kết quả đã lưu trữ |
| Câu lệnh `UPDATE` tác động 0 dòng | Ghi nhận kết quả lịch sử (Nếu có logic) | Trả về thông báo Hết hàng (`OUT_OF_STOCK`) |
| Lỗi chờ lock / deadlock | Tự động Rollback | Tự động thử lại transaction mới theo thuật toán Backoff |

Tầng ứng dụng không được tự suy diễn việc "Mất kết nối mạng" đồng nghĩa với "Xử lý thất bại". Bắt buộc dựa vào Mã định danh duy nhất (Stable Command ID) để đọc lại trạng thái (Replay result).

## 15. So Sánh Với Các Mức Cô Lập (Isolation Levels) Khác

Trong mức cô lập `REPEATABLE READ` hoặc `SERIALIZABLE`, thay vì trả về kết quả 0 dòng, PostgreSQL có thể phát sinh ngoại lệ lỗi tuần tự hóa (`40001`). Ứng dụng phải được thiết kế để bắt lỗi này và thực hiện vòng lặp thử lại toàn bộ transaction (Whole-transaction retry). Không nên nâng cấp mức cô lập nếu logic có thể giải quyết dứt điểm bằng một mệnh đề `WHERE` cơ bản.

## 16. Giới Hạn Của Ràng Buộc (Constraint Boundaries)

```sql
CHECK (available_quantity >= 0)
CHECK (available_quantity + reserved_quantity = on_hand_quantity)
```
Các ràng buộc (Constraints) ở mức dòng (Row-level) bảo vệ định dạng toán học hợp lệ, nhưng không ngăn chặn được việc bảng `inventory_item` và bảng `inventory_reservation` bị mất đồng bộ chéo (Cross-table discrepancy). Do đó, ứng dụng vẫn cần các tiến trình hậu kiểm (Reconciliation jobs) để bảo vệ toàn vẹn logic tổng thể.

## 17. Hành Vi Trong Kiến Trúc Đa Máy Chủ (Multi-instance Behavior)

Database đảm nhận vai trò bộ điều phối nhất quán thông qua cơ chế lock và đánh giá lại (Recheck), giúp đảm bảo an toàn cho môi trường phân tán (Scale-out). Tuy nhiên, mức độ đồng thời cao trên một bản ghi duy nhất (Hot-row contention) có thể gây ra hiện tượng xếp hàng dài, tăng tỷ lệ lỗi quá thời gian chờ (lock timeout), yêu cầu hệ thống phải được tinh chỉnh hợp lý về kích thước Connection pool và giới hạn chờ.

## 18. Phân Tích Lỗi Theo Tầng (Root Cause by Layer)

| Tầng Xử Lý | Vấn đề Phổ Biến |
| --- | --- |
| Ứng dụng (Application) | Tách rời logic "Kiểm tra" và "Ghi", tính toán dựa trên dữ liệu cũ trong RAM. |
| Spring | Lầm tưởng `@Transactional` có khả năng gộp các câu lệnh SQL tự động thành mức nguyên tử. |
| Hibernate (JPA) | Không làm sạch bộ đệm sau khi cập nhật bằng SQL, gây ra lỗi ghi đè trạng thái. |
| PostgreSQL | Kỹ sư không hiểu rõ cơ chế Predicate recheck chỉ được kích hoạt bằng mệnh đề `WHERE`. |

## 19. Các Yêu Cầu Giám Sát (Observability)

Thiết lập đo lường trên môi trường thực tế:
- Tần suất các lệnh `UPDATE` trả về thành công (1 dòng) và từ chối (0 dòng).
- Thời lượng duy trì lock (Lock wait duration) và độ trễ transaction.
- Tần suất phát sinh mã lỗi `55P03` (lock timeout) và `40001` (Serialization failure).
- Tần suất phát hiện các yêu cầu trùng lặp (Duplicate request).
- Mức độ chênh lệch khi chạy các lệnh đối soát (Reconciliation alerts).

## 20. Giới Hạn Của Phương Pháp (Scope Boundaries)

Kỹ thuật cập nhật nguyên tử có điều kiện phù hợp khi các điều kiện kiểm tra chỉ phụ thuộc vào trạng thái của duy nhất một bản ghi. Khi nghiệp vụ đòi hỏi kiểm tra logic xuyên suốt nhiều dòng sản phẩm, hoặc phụ thuộc vào việc kiểm tra bản ghi chưa tồn tại, phương pháp này không đáp ứng đủ. Trong trường hợp đó, hệ thống nên sử dụng lock bi quan (Pessimistic Locking) hoặc cơ chế hàng đợi (Message queue) để xử lý tuần tự.
