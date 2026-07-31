# Phân tích Chuyên sâu: Cập nhật Cột Version và Xung Đột Khóa Lạc Quan

## 1. Bối cảnh Khởi động

Chúng ta có Sản phẩm `42`: giá đang là `100`, phiên bản `7`. Hai bạn nhân viên A và B có 2 luồng Giao dịch độc lập nhau, đang chạy chế độ `READ COMMITTED` (đọc dữ liệu đã chốt sổ).

## 2. Dòng Thời gian (Timeline) khi có `@Version`

| Bước | Nhóm của Editor A | Nhóm của Editor B |
| ---: | --- | --- |
| 1 | Tải lên: `100 / v7` | |
| 2 | | Tải lên: `100 / v7` |
| 3 | Sửa giá thành `90` | Sửa giá thành `80` |
| 4 | Bắn lệnh: `UPDATE ... version=7` → Sửa được `1` dòng | |
| 5 | Chốt sổ (commit): Dữ liệu giờ là `90 / v8` | |
| 6 | | Bắn lệnh: `UPDATE ... version=7` → Sửa được `0` dòng |
| 7 | | DB văng lỗi Optimistic, Giao dịch bị Hủy (rollback) |

Kết quả cuối cùng dưới DB: `90 / v8`. Người thua cuộc (Editor B) sẽ không được báo thành công giả, cũng không có chuyện lưu được một nửa dữ liệu (partial write).

> **Nói ngắn gọn:** Cái "expected version" (version mà mình kỳ vọng) đã biến quy luật cùi bắp "ai ghi sau kẻ đó thắng" thành một quy tắc sắt đá "so-sánh-rồi-mới-đặt": Chỉ có anh nào đang cầm đúng cái phiên bản hiện tại mới được phép cập nhật.

## 3. Bản Hợp Đồng SQL (SQL contract)

```sql
update product_offer
set price = :newPrice,
    title = :title,
    version = :nextVersion       -- Tăng lên 1
where offer_id = :offerId
  and version = :expectedVersion; -- Phải đúng Version cũ
```

Số dòng bị tác động (Affected rows) trả về từ Database:

- Trả về `1`: Chúc mừng, version khớp, dòng đã được tạo và version đã tăng.
- Trả về `0`: Chia buồn, dòng dữ liệu đã bị ai đó xóa mất, HOẶC cái version đã bị ai đó đổi. Lúc này Hibernate sẽ kết luận trạng thái dữ liệu trên tay bạn đã "ôi thiu" (stale).

Lập trình viên không nên dò chữ trong message lỗi để xử lý. Hãy bắt cái tín hiệu chuẩn `OptimisticLockException` của Jakarta Persistence; Spring thì thường bọc nó lại thành `ObjectOptimisticLockingFailureException`.

## 4. MVCC của PostgreSQL và Khóa Dòng (row lock)

Cả A và B khi gọi lệnh `SELECT` chay thì đều nhìn thấy chung dòng `v7`; Khóa `@Version` **KHÔNG HỀ khóa dữ liệu lúc bạn đọc**.
Khi có lệnh UPDATE bay xuống DB:

- A giật được khóa dòng (row lock), sửa thành `v8` và chốt sổ (commit).
- Giả sử B bay xuống lúc A đang sửa nhưng chưa kịp chốt sổ: B sẽ bị bắt Đứng Chờ cái khóa dòng đó (chờ row lock).
- Sau khi A chốt sổ xong xuôi mở khóa, lệnh của B mới được chạy, nhưng ngặt nỗi câu lệnh của B lại đòi `version=7` - thứ mà giờ đây không còn tồn tại trên dòng đó nữa.
- Kết quả B nhận về `0` dòng thay đổi, không hề ghi đè bậy bạ.

Như vậy, "Khóa Lạc Quan" vẫn có thể dính phải những pha Đứng Chờ Khóa Cứng (lock wait) chớp nhoáng ở khâu Lưu dữ liệu. Chữ "Lạc quan" ở đây ý nói là mình "phát hiện xung đột lúc lưu" thay vì phải bắt mọi người "xếp hàng chờ khóa từ lúc đọc", chứ không có nghĩa là lúc Lưu không xài Khóa (row lock) của PostgreSQL đâu nha!

## 5. Hibernate rà soát thay đổi (dirty checking) và Xả lệnh (flush)

Bạn gán (Set) giá trị trong Java thì chỉ là đổi Object trên RAM. Câu lệnh SQL thực sự chỉ được nã xuống DB khi:

- Bạn tự tay gọi hàm `EntityManager.flush()` hoặc `JpaRepository.flush()`.
- Chế độ tự xả (auto-flush) chạy trước khi nó phải thực hiện một lệnh Query liên quan.
- Tới lúc chốt Giao dịch (transaction commit).

Thời điểm Exception văng ra phụ thuộc vào việc xả (flush) lúc nào. Nếu bạn không tự tay gọi `flush`, Code trong hàm của bạn có khi vẫn chạy trơn tru đến phút cuối, rồi Proxy đứng bên ngoài lúc làm nhiệm vụ `commit` mới đạp trúng quả mìn Exception. Do đó, Ranh giới bên ngoài (API) phải luôn chuẩn bị tinh thần đón Lỗi.

## 6. Số phận của Giao dịch sau khi Xung Đột

Một khi `OptimisticLockException` văng ra, toàn bộ Giao Dịch hiện tại sẽ bị cắm cờ CHẾT (rollback-only). Cái Object (Entity) nằm trên RAM của kẻ thua cuộc (B) dù đang cầm giá trị `80`/`version=7` đi nữa thì cũng chỉ là thứ vô giá trị, không phải đồ thật dưới DB.

**Tuyệt đối KHÔNG ĐƯỢC:**

- Bắt (catch) lỗi xong ỉm đi rồi báo với Màn hình là Lưu Thành công.
- Gọi hàm `clear()` bùa chú hòng cứu vãn và đòi code tiếp trong cái Giao Dịch đã chết.
- Đem Object ôi thiu này đi lưu lại (`merge`) mà không thèm mở Giao Dịch mới hoặc kiểm tra lại Version.

Rollback là đóng sập cửa. Nếu Business cho phép Thử Lại, bạn BẮT BUỘC phải mở một Giao Dịch mới tinh, tải (reload) dữ liệu thật nhất từ DB lên lại. (Đọc thêm ở `LOCK-002`).

## 7. Version của Client và Version của Database giải quyết "Hai Cửa Sổ Hở"

Có tận hai cái "cửa sổ" rò rỉ dữ liệu ôi thiu:

1. **Cửa sổ Đứt kết nối (Disconnected window):** Khách B mở form tải v7, nhưng treo máy đi cafe; Trong lúc đó A vô sửa thành v8. B uống cafe xong bấm nút Lưu.
2. **Cửa sổ Giao Dịch (Transaction window):** Backend của B vừa bốc được v8 từ DB lên; nhưng đúng tích tắc trước khi B xả (flush) UPDATE xuống DB, một ông C (luồng khác) đã nhanh tay chốt thành v9.

Nếu bạn so sánh `command.expectedVersion` (Client truyền lên) với Entity Version trên RAM, bạn sẽ bịt được **Cửa sổ số 1**.
Cái mệnh đề `version` mà Hibernate chèn vào SQL sẽ bọc lót cho bạn cái **Cửa sổ số 2**.
Nếu bạn BỎ QUA 1 trong 2 cái này, Code của bạn kiểu gì cũng có ngày bị ghi đè dữ liệu.

Ở tầng HTTP API, người ta hay xài trò kẹp Version vào Header:

```text
(Lúc tải) GET /offers/42 → ETag: "7"
(Lúc lưu) PUT /offers/42 kèm Header If-Match: "7"
```

Nhớ bọc cái Response lỗi chuẩn chỉnh thành `412 Precondition Failed` hoặc `409 Conflict`. Đừng có đem nguyên cái Object hay mấy trường nhạy cảm xả ra cho Client xem lúc bắt lỗi.

## 8. Gắn Object ngoài ranh giới (`merge`) - Cẩn thận cái bẫy

Dù Hibernate có trách nhiệm soi Version khi bạn đem 1 Object lạc trôi bên ngoài gán lại (merge), thì thời điểm nó soi vẫn có thể lùi tới tận lúc flush/commit. Hàm `merge()` nhả ra cho bạn 1 BẢN COPY đã được theo dõi, còn cái Object truyền vào thì vẫn lang thang ngoài rìa.

Cách an toàn nhất cho lính mới là: Tự tay Map (chuyển đổi) DTO của Client sang cái Entity Mới Vừa Tải (current managed entity) và So Sánh Version bằng tay. Làm vậy vừa rõ ràng, vừa tránh được mấy cái lỗi cascade ngớ ngẩn hoặc client cố tình update dư trường (over-posting). Dù chọn cách nào, **TUYỆT ĐỐI KHÔNG BỎ CÁI VERSION CỦA CLIENT**.

## 9. Ranh giới của một Khối (Aggregate boundary)

Cột Version sinh ra là để bảo vệ cái Bảng Gốc (aggregate root). Nếu logic của bạn trải dài nhiều bảng:

- Version của từng dòng lẻ tẻ không thể tự nó ngăn chặn được lỗi Khớp Sai Tổng (write skew).
- Bạn sửa các Collection con bên trong (List, Set) có làm tăng Version thằng Cha hay không là do bạn cấu hình (mapping).
- Nếu bạn tự lấy tay viết SQL `UPDATE` hàng loạt (Bulk JPQL/Native), Hibernate sẽ mù màu không tăng Version cho bạn đâu. Nhớ tự tay cộng Version vào!

Làm mấy trò này thì phải Test thật kỹ mọi ngóc ngách!

## 10. Thế tại sao không tự code Thử Lại (auto-retry) cho khỏe?

Cái lệnh "Sửa thành giá 80" của ông B được gõ trong tâm thế ổng ĐANG NHÌN THẤY cái giá hiện tại (của Version 7). Giả sử A đã chốt thành `90 / v8`, mà Backend tự động nạp (reload) rồi âm thầm đè số `80` vào, thì khác nào Code đang lén lút thay mặt ông B phá hoại cái `90` của A mà B chẳng hề hay biết (chưa kịp Review)!

Người thua cuộc phải nhận lỗi Conflict, thấy được Dữ Liệu Hiện Tại (v8) để tự quyết định hợp nhất (merge) hay gõ lại. Nếu bạn update những thứ vô thưởng vô phạt mang tính Tích Lũy (như cộng điểm, đếm lượt xem), thì mới tự động Retry (Đọc kỹ `LOCK-002`).

## 11. Các Trạng thái Tai nạn (Commit, rollback, timeout, crash)

- Kẻ Thắng Cuộc (Winner) chốt sổ: Giá/Version đồng thời hiện hình dưới DB.
- Kẻ Thua Cuộc (Loser) đụng độ: Toàn bộ Giao dịch bị cuốn trôi; DB nhả mọi Khóa (lock).
- Quá giờ chờ Khóa (Lock/statement timeout): Hoàn toàn khác với Đụng độ (Optimistic conflict), đừng bắt nhầm Exception.
- Máy chủ sập TRƯỚC KHI chốt sổ: PostgreSQL lo dọn dẹp sạch sẽ. Mọi thứ xôi hỏng bỏng không.
- Máy chủ sập SAU KHI chốt sổ nhưng CHƯA BÁO CHO CLIENT: Trạng thái không rõ ràng. Cần phải có Mã Lệnh (Command ID/Audit) để tra cứu xem lệnh đó đã vô DB chưa.
- Hai người đua nhau XÓA: Ông đi sau sẽ vướng `affected rows 0`, nhưng thay vì chửi là Lỗi Đụng Độ thì phải tùy logic Business mà biến nó thành Lỗi Không Tìm Thấy (Not Found).

## 12. Chạy nhiều máy chủ (Multi-instance)

Vì cái đuôi `version = ?` được xét thẳng dưới PostgreSQL nên dù bạn chạy App-1 hay App-2 thì đều được bảo vệ như nhau. Đừng có cố nhét từ khóa `synchronized` của Java vào hàm, nó vừa vô dụng (không đỡ được server bên cạnh) vừa làm giảm hiệu năng.

Hãy bắt TẤT CẢ các bên tuân thủ Luật Version. Mấy ông viết SQL chay mà "lười" không chèn đoạn `version=version+1` vào lệnh UPDATE sẽ phá nát cơ chế phát hiện Lỗi của toàn hệ thống. Hãy siết quyền và Review kỹ các đoạn code tự viết SQL (migrations/batch jobs).

## 13. Bảng Phân hạng Vũ khí (So với các cơ chế khác)

| Vũ khí / Cơ chế | Hình phạt cho Kẻ thua cuộc | Đất dụng võ |
| --- | --- | --- |
| **`@Version` (Khóa Lạc Quan)** | DB trả `0` dòng sửa, Văng lỗi, Rollback | Dùng cho Các màn hình Sửa Thông tin (Aggregate edit), ít khi tranh giành nhau |
| Câu lệnh SQL có Điều Kiện | Báo lỗi hoặc không tùy Business đặt luật | Đếm số lượng / Trừ Tồn Kho |
| Khóa Bi Quan `FOR UPDATE` | Bị Block đứng chờ hoặc Quá Giờ (timeout) | Cần bắt mọi người xếp hàng tuần tự trước khi được cập nhật |
| Isolation `SERIALIZABLE` | Ném lỗi `40001`, bắt Code phải thử lại toàn bộ | Tính toán nhiều bảng phức tạp |

## 14. Bảng Phân Tội theo Lớp (Layer)

| Tầng (Layer) | Trách nhiệm |
| --- | --- |
| Màn hình / API | Không truyền Lên cái Version mà User đang xem. Làm mất "Dữ liệu kỳ vọng". |
| Spring | Tạo Ranh giới Giao dịch ảo và Chế biến Exception. |
| Hibernate / JPA | Theo dõi thay đổi, Sinh câu SQL có đuôi `version`, Kiểm tra số dòng sửa được. |
| DB PostgreSQL | Cấp MVCC, Cấp Khóa dòng (Row Lock) và Trả về số Dòng đã tác động. |

## 15. Kính lúp Quan Sát (Observability)

Khi đưa code lên Môi trường thật (Production), phải giám sát:

- Đếm số lần Xung đột Khóa Lạc Quan theo từng Màn hình/Bảng.
- Ghi Log Đẹp (Structured) ra cái `expected version` và `current version` khi có lỗi.
- Đếm xem trả về `409 Conflict` nhiều không và User có tải lại Form hay không.
- Giao dịch sống được bao lâu, Gọi flush ở xó xỉnh nào và Tổng số câu SQL.
- Nếu thấy Đụng Độ tự nhiên tăng phi mã sau một đợt Deploy hoặc chạy Bulk Job, khoan đổ lỗi cho mạng, coi chừng mấy ông viết SQL dạo Bypass cái luật Version rồi đó!
