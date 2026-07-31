# Cập nhật dữ liệu an toàn với điều kiện trực tiếp trong SQL (Conditional Atomic Update)

## Tại sao cần dùng cách này?

Thay vì bạn phải `SELECT` dữ liệu lên code Java (ví dụ: lấy số lượng hàng tồn kho), kiểm tra điều kiện (còn đủ hàng không), rồi mới gọi lệnh `UPDATE` để lưu lại... thì bạn có thể gộp tất cả việc kiểm tra và cập nhật này vào trong duy nhất một câu lệnh SQL. 

Cách làm này rất hiệu quả khi bạn chỉ cần kiểm tra dữ liệu của chính cái dòng bạn đang muốn cập nhật. Nó giúp tránh được lỗi khi có nhiều luồng (thread) hoặc nhiều yêu cầu (request) cùng sửa một dòng dữ liệu cùng lúc.

## Các từ khóa quan trọng cần nhớ

| Từ khóa | Giải thích |
| --- | --- |
| Cập nhật có điều kiện (conditional `UPDATE`) | Chỉ cập nhật dữ liệu nếu thỏa mãn điều kiện ở mệnh đề `WHERE` của câu SQL. |
| Số dòng bị ảnh hưởng (affected-row count) | Số lượng dòng dữ liệu thực tế đã được cập nhật thành công sau khi chạy câu lệnh. |
| Kiểm tra lại điều kiện (predicate recheck) | Nếu cơ sở dữ liệu (như PostgreSQL) phải chờ một luồng khác chạy xong mới đến lượt nó, nó sẽ kiểm tra lại điều kiện `WHERE` một lần nữa để đảm bảo dữ liệu mới nhất vẫn thỏa mãn điều kiện. |
| Trả lại dữ liệu sau cập nhật (`RETURNING`) | Cơ sở dữ liệu sẽ trả về ngay dữ liệu mới nhất vừa được cập nhật xong (thay vì bạn phải gọi thêm một câu `SELECT` nữa để lấy). |

## Cách viết code cơ bản

Ví dụ bạn muốn trừ đi số lượng hàng trong kho (`quantity`):

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
WHERE product_id = :productId
  AND :quantity > 0
  AND available_quantity >= :quantity; -- Điều kiện quan trọng: số lượng tồn phải lớn hơn hoặc bằng số lượng cần trừ
```

**Cách đọc kết quả:**
- **Số dòng bị ảnh hưởng = 1**: Câu lệnh đã thành công, hàng đã được trừ.
- **Số dòng bị ảnh hưởng = 0**: Không có dòng nào bị thay đổi. Điều này có nghĩa là điều kiện `WHERE` không thỏa mãn (ví dụ: hết hàng) hoặc sản phẩm không tồn tại.

**Lưu ý cực kỳ quan trọng:** Code của bạn BẮT BUỘC phải kiểm tra số lượng dòng bị ảnh hưởng này. Nếu bạn bỏ qua nó, dữ liệu có thể không được cập nhật thành công nhưng hệ thống của bạn vẫn báo là thành công, dẫn đến lỗi thất thoát dữ liệu.

## Hãy cộng/trừ trực tiếp trong SQL, đừng tính toán ở Java rồi mới lưu

**Tuyệt đối tránh cách làm này:**
```sql
-- Lấy dữ liệu cũ ở Java, tính toán ra con số cuối cùng rồi gửi xuống SQL. Rất dễ bị sai lệch khi chạy đồng thời!
UPDATE inventory_item
SET available_quantity = :valueCalculatedFromOldRead 
WHERE product_id = :productId;
```

**Nên làm theo cách này:**
```sql
-- Bảo SQL tự lấy giá trị hiện tại trừ đi số lượng cần thiết
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity
WHERE product_id = :productId
  AND available_quantity >= :quantity;
```
Khi bạn làm cách này, cơ sở dữ liệu sẽ khóa dòng dữ liệu lại, tự tính toán trừ đi và lưu lại. Không có khoảng thời gian hở nào để các yêu cầu khác chen ngang vào làm sai lệch số liệu.

## Cơ sở dữ liệu xử lý việc nhiều luồng cùng cập nhật như thế nào?

Giả sử có 2 yêu cầu (gọi là A và B) cùng muốn cập nhật một dòng dữ liệu tại cùng một thời điểm:
1. Yêu cầu A đến trước, thấy điều kiện đúng nên khóa dòng đó lại và bắt đầu cập nhật.
2. Yêu cầu B đến sau, cũng muốn sửa dòng đó nên phải đứng chờ yêu cầu A làm xong.
3. Nếu A cập nhật thành công (commit), thì B sẽ nhận được dữ liệu mới nhất do A vừa sửa, sau đó B sẽ tự động kiểm tra lại điều kiện `WHERE` (xem có còn đủ hàng không).
4. Nếu A bị lỗi và hủy bỏ (rollback), thì B sẽ lấy dữ liệu ban đầu chưa bị A sửa để kiểm tra.
5. Cuối cùng, B có thể cập nhật thành công hoặc không làm gì cả, tùy thuộc vào điều kiện `WHERE` có còn đúng hay không.

## Dùng `RETURNING` để lấy kết quả ngay lập tức

```sql
UPDATE inventory_item
SET available_quantity = available_quantity - :quantity,
    revision = revision + 1
WHERE product_id = :productId
  AND available_quantity >= :quantity
RETURNING available_quantity, revision; -- Lấy luôn giá trị sau khi cập nhật
```
Nhờ `RETURNING`, bạn biết chắc chắn cập nhật thành công hay không (có dòng trả về là thành công, không có là thất bại) và có luôn dữ liệu mới nhất mà không cần viết thêm lệnh `SELECT`.
*(Lưu ý: Nếu dùng Spring Data JPA với `@Modifying`, nó chỉ trả về số lượng dòng bị ảnh hưởng. Nếu muốn hứng dữ liệu từ `RETURNING`, bạn nên dùng thư viện JDBC hoặc jOOQ sẽ dễ hơn).*

## Xử lý khi kết quả không như ý muốn (Cập nhật 0 dòng)

Khi bạn cập nhật theo 1 ID cụ thể, kết quả trả về chỉ có thể là 1 (thành công) hoặc 0 (không có gì thay đổi). Nếu kết quả trả về lớn hơn 1, tức là code của bạn đang bị lỗi logic.

Nếu kết quả là `0`, trong Java bạn phải xác định được tại sao lại bằng 0:
- Do trong kho không còn đủ hàng?
- Do ID sản phẩm không tồn tại?
- Do dữ liệu đầu vào bị sai?

Đừng gom tất cả các trường hợp báo `0` thành một lỗi chung chung, hãy phân biệt rõ để báo lỗi chính xác cho người dùng.

## Gói gọn mọi thứ vào một giao dịch (Transaction)

Thường thì khi trừ tiền hoặc trừ kho, bạn còn phải làm nhiều việc khác nữa (như lưu lịch sử giao dịch). Tất cả các thao tác ghi vào cơ sở dữ liệu này PHẢI được đặt trong cùng một giao dịch (dùng `@Transactional` trong Spring Boot). 
Nếu một bước phía sau bị lỗi, toàn bộ giao dịch sẽ bị hoàn tác (rollback), dữ liệu kho sẽ được trả lại như cũ. Tuy nhiên, việc gọi ra các hệ thống bên ngoài (gọi API khác) không được tính vào giao dịch này.

## Các lưu ý khi dùng Spring Data JPA

Nếu bạn viết hàm cập nhật trực tiếp như sau:
```java
@Modifying
@Query(value = """
        UPDATE inventory_item
        SET available_quantity = available_quantity - :quantity
        WHERE product_id = :productId
          AND available_quantity >= :quantity
        """, nativeQuery = true)
int reserveIfEnough(long productId, int quantity);
```
Bạn cần nhớ rằng: Lệnh này cập nhật thẳng xuống cơ sở dữ liệu, bỏ qua bộ nhớ đệm (cache/persistence context) của JPA.
Do đó, nếu bạn đang lấy đối tượng (entity) đó ra trước đó, dữ liệu của entity trong code Java lúc này sẽ là dữ liệu cũ, không khớp với cơ sở dữ liệu nữa.

**Cách giải quyết:** 
- Đừng tải entity đó lên trước khi gọi lệnh cập nhật này trong cùng một giao dịch.
- Nếu phải dùng, hãy xóa sạch bộ nhớ đệm sau khi cập nhật bằng cấu hình `clearAutomatically = true` trong `@Modifying`, buộc JPA phải đọc lại dữ liệu mới nếu cần thiết.

## Chốt giao dịch, hủy bỏ và thử lại (Retry)

| Kết quả thực thi | Ý nghĩa thực tế |
| --- | --- |
| Sửa được 1 dòng, rồi `commit` | Cập nhật đã được lưu chắc chắn vào Database. |
| Sửa 0 dòng, rồi `commit` | Không có gì thay đổi; thường dùng để lưu lại lịch sử báo lỗi (từ chối). |
| Sửa được 1 dòng, rồi bị lỗi `rollback` | Dữ liệu được trả lại y như cũ trước khi cập nhật. |
| Lỗi SQL `55P03` | Không chờ được để khóa (lock) dữ liệu; giao dịch bắt buộc phải hủy (`rollback`). |
| Lỗi SQL `40P01` | Lỗi bế tắc (Deadlock); Database tự chọn giao dịch này để hủy giải cứu hệ thống. |
| Lỗi SQL `40001` | Lỗi tuần tự hóa (Serialization failure), thường xảy ra ở các mức cách ly cao. |

- Chỉ tự động thử lại (retry) khi gặp lỗi kỹ thuật như các mã lỗi SQL ở trên. 
- Không được tự động thử lại nếu lỗi do logic nghiệp vụ (ví dụ: lỗi hết tiền, hết hàng) vì nó sẽ mãi mãi lỗi.
- Mỗi lần thử lại phải mở một giao dịch (`Transaction`) hoàn toàn mới, và nhớ đặt giới hạn số lần thử.

## Cập nhật nhiều dòng cùng lúc

Nếu câu lệnh `UPDATE` của bạn ảnh hưởng tới nhiều dòng dữ liệu, hãy cẩn thận vì nó phức tạp hơn:
- Dễ sinh ra bế tắc (deadlock) do Database tự động khóa từng dòng một theo thứ tự ngẫu nhiên.
- Số lượng dòng bị ảnh hưởng không còn chắc chắn là `0` hay `1` nữa. Bạn phải kiểm tra cẩn thận xem số dòng cập nhật thành công có đúng với số lượng bạn mong muốn không.
- Nếu bạn cần tất cả các dòng phải thành công 100%, hãy lấy số dòng bị ảnh hưởng so sánh trực tiếp với tổng số dòng bạn dự định cập nhật.

## Dòng dữ liệu không tồn tại và các điều kiện phức tạp

Câu `UPDATE` không thể tự tạo mới dữ liệu nếu nó chưa tồn tại, và cũng không khóa được một thứ không có thật.
Nếu logic của bạn yêu cầu kiểm tra điều kiện trên nhiều bảng hoặc nhiều dòng khác nhau cùng lúc (ví dụ: tổng tiền các hóa đơn không vượt quá mức cho phép), câu `UPDATE` đơn giản sẽ không đủ an toàn. 

Lúc này bạn cần:
- Dùng các ràng buộc (Constraint) như `UNIQUE`, `CHECK` trong Database.
- Có một bảng hoặc một dòng riêng dùng để đếm số (bộ đếm).
- Cân nhắc chuyển sang dùng khóa bi quan (Pessimistic locking - `SELECT ... FOR UPDATE`).

## Chạy nhiều server cùng lúc (Multi-instance)

Khi ứng dụng của bạn chạy trên nhiều server (ví dụ: có 5 con server Spring Boot cùng chạy), đừng lo lắng. Tất cả các lệnh `UPDATE` đều sẽ đẩy về chung một Database chính (Primary PostgreSQL). 
Bản thân Database sẽ tự sắp xếp hàng đợi và khóa dòng dữ liệu lại một cách an toàn. Bạn KHÔNG CẦN phải dùng khóa trên code Java (như `synchronized` hay các thư viện Lock phân tán) cho trường hợp này.

Tuy nhiên, nếu có quá nhiều server cùng đập vào một dòng dữ liệu (dữ liệu quá "hot"), Database sẽ bị quá tải ở hàng đợi khóa. Lúc này hệ thống sẽ chậm đi, bạn cần theo dõi thời gian chờ và tình trạng kết nối.

## Tóm lại: Khi nào nên dùng cách này?
Đây là cách đơn giản và rất mạnh mẽ khi:
- Bạn biết rõ ID của dòng cần cập nhật.
- Điều kiện kiểm tra chỉ phụ thuộc vào dữ liệu của chính dòng đó.
- Thao tác là cộng, trừ số lượng hoặc đổi trạng thái đơn giản.
- Khối lượng công việc cập nhật và giữ khóa (lock) ngắn.

*(Nếu bạn cần lấy rất nhiều dữ liệu về Java xử lý tính toán phức tạp rồi mới lưu thì hãy nghĩ tới dùng từ khóa `FOR UPDATE` hoặc dùng khóa lạc quan `@Version` thay thế).*

## Những thông số cần theo dõi (Monitoring)

Để đảm bảo hệ thống khỏe mạnh, bạn nên theo dõi (log/metric) các chỉ số sau:
- Tỉ lệ cập nhật thành công (1 dòng) so với thất bại (0 dòng).
- Các mã lỗi báo kẹt khóa hay deadlock: `55P03`, `40P01`, `40001`.
- Thời gian phải chờ để lấy được khóa và thời gian chạy xong một giao dịch.
- Nhớ đừng bao giờ in các thông tin nhạy cảm (như mật khẩu, số thẻ) ra log. 

## Liên kết tài liệu tham khảo

- [LOCK-004 — Conditional atomic UPDATE](../locking/conditional-atomic-update/README.md)
- [DB-001 — Lost update dưới MVCC](../postgresql/lost-update-mvcc/README.md)
- [DB-004 — Phantom capacity check](../postgresql/phantom-capacity-check/README.md)
- [DB-007 — Row/table lock lifecycle](../postgresql/row-table-lock-lifecycle/README.md)
- [LOCK-003 — Pessimistic write lock](../locking/pessimistic-write-for-update/README.md)
- [PostgreSQL MVCC](postgresql-mvcc.md)
- [PostgreSQL locks](postgresql-locks.md)
- [Ranh giới transaction trong Spring](spring-transaction-boundaries.md)
- [Tính lũy đẳng và uniqueness](idempotency-and-uniqueness.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
