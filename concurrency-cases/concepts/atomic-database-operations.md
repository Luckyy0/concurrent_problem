# Phép toán nguyên tử trong cơ sở dữ liệu và cập nhật có điều kiện

## Mục đích

Phép cập nhật nguyên tử có điều kiện (`conditional atomic update`) đặt điều kiện
nghiệp vụ và thao tác thay đổi dữ liệu trong cùng một câu lệnh SQL. Cách làm này
phù hợp khi quy tắc cần bảo vệ có thể diễn đạt bằng các cột của dòng hiện tại,
không cần đọc dữ liệu về Java rồi mới quyết định.

## Thuật ngữ cần biết

| Cách gọi trong tài liệu | Tên tiếng Anh hoặc API | Giải thích |
| --- | --- | --- |
| cập nhật có điều kiện | conditional `UPDATE` | Chỉ cập nhật khi mệnh đề `WHERE` còn đúng |
| số dòng bị ảnh hưởng | affected-row count | Số dòng thực sự được câu lệnh thay đổi |
| kiểm tra lại điều kiện | predicate recheck | PostgreSQL đánh giá lại `WHERE` sau khi phải chờ giao dịch khác |
| phiên bản hiện tại của dòng | current row version | Dữ liệu mới nhất mà câu lệnh cập nhật được phép xử lý |
| không thay đổi dữ liệu | no-op | Câu lệnh chạy thành công nhưng không cập nhật dòng nào |
| trả lại dữ liệu sau cập nhật | `RETURNING` | PostgreSQL trả các giá trị của dòng vừa được cập nhật |
| phép cộng hoặc trừ có điều kiện | guarded delta | Thay đổi một lượng trên giá trị hiện tại, kèm điều kiện giới hạn |
| DML hàng loạt | bulk DML | Câu lệnh cập nhật trực tiếp trong cơ sở dữ liệu, không đi qua cơ chế phát hiện thay đổi (`dirty checking`) của thực thể (`entity`) |

## Mẫu cơ bản

```sql
update inventory_item
set available_quantity = available_quantity - :quantity,
    reserved_quantity = reserved_quantity + :quantity
where product_id = :productId
  and :quantity > 0
  and available_quantity >= :quantity;
```

Quy ước kết quả:

```text
Số dòng bị ảnh hưởng = 1 → yêu cầu đã được áp dụng.
Số dòng bị ảnh hưởng = 0 → không có dữ liệu nào bị thay đổi.
```

Ứng dụng phải xử lý cả hai trường hợp. Nếu bỏ qua kết quả trả về, cơ sở dữ liệu
có thể từ chối cập nhật đúng cách nhưng API vẫn báo thành công hoặc vẫn ghi tác
dụng phụ.

> **Nói ngắn gọn:** `WHERE` là điều kiện nghiệp vụ tại đúng thời điểm ghi; số
> dòng bị ảnh hưởng là câu trả lời bắt buộc phải xử lý.

## Gửi ý định thay đổi, không gửi giá trị tính từ dữ liệu cũ

Không nên gửi một giá trị tuyệt đối đã được tính từ lần đọc trước:

```sql
update inventory_item
set available_quantity = :valueCalculatedFromOldRead
where product_id = :productId;
```

Nên gửi trực tiếp ý định cần thực hiện:

```sql
update inventory_item
set available_quantity = available_quantity - :quantity
where product_id = :productId
  and available_quantity >= :quantity;
```

Phép trừ sử dụng giá trị hiện tại trong cơ sở dữ liệu. Điều kiện kiểm tra và thao
tác thay đổi nằm trong cùng một câu lệnh nên không có khoảng trống để giao dịch
khác chen vào giữa hai bước.

## Cách PostgreSQL xử lý tại `READ COMMITTED`

Hai câu lệnh `UPDATE` cùng nhắm tới một dòng vẫn sử dụng khóa mức dòng:

1. Giao dịch A thấy điều kiện đúng, khóa dòng và cập nhật.
2. Giao dịch B cũng nhắm tới dòng đó nên phải chờ A.
3. Nếu A chốt giao dịch (`commit`), B xử lý phiên bản mới của dòng và đánh giá
   lại `WHERE`.
4. Nếu A hoàn tác giao dịch (`rollback`), thay đổi của A biến mất và B xử lý dữ
   liệu ban đầu.
5. B cập nhật một dòng hoặc không cập nhật dòng nào tùy kết quả kiểm tra lại.

Cơ chế này phù hợp với điều kiện đơn giản trên một dòng đã biết trước. Với điều
kiện phức tạp dựa trên nhiều dòng, `READ COMMITTED` không bảo đảm mọi dữ liệu mà
câu lệnh nhìn thấy đều thuộc cùng một ảnh chụp nhất quán.

`lock_timeout`, bế tắc (`deadlock`) và lỗi tuần tự hóa
(`serialization failure`) vẫn có thể xảy ra. Đây là lỗi của câu lệnh hoặc giao
dịch, không phải trường hợp “số dòng bị ảnh hưởng bằng 0”.

## Dùng `RETURNING` khi cần giá trị sau cập nhật

```sql
update inventory_item
set available_quantity = available_quantity - :quantity,
    revision = revision + 1
where product_id = :productId
  and available_quantity >= :quantity
returning available_quantity, revision;
```

Một dòng được trả về nghĩa là cập nhật đã thắng; không có dòng nào được trả về
nghĩa là không có thay đổi. `RETURNING` giúp lấy đúng giá trị sau cập nhật từ
chính câu lệnh đó, thay vì chạy thêm một `SELECT` rồi suy đoán.

Spring Data `@Modifying` phù hợp khi chỉ cần số dòng bị ảnh hưởng. Nếu cần ánh
xạ các cột do `RETURNING` trả về, JDBC, jOOQ hoặc một lớp truy cập dữ liệu viết
SQL trực tiếp thường thể hiện ý định rõ hơn.

## Giới hạn số dòng được cập nhật

Với một yêu cầu nhắm tới khóa duy nhất, kết quả mong đợi là:

```text
Số dòng bị thay đổi ∈ {0, 1}
```

Nếu kết quả lớn hơn `1`, truy vấn hoặc ranh giới của đối tượng nghiệp vụ đang sai. Nếu kết quả
bằng `0`, ứng dụng phải biết điều kiện nào có thể đã không thỏa:

- không còn đủ số lượng;
- dòng cần cập nhật không tồn tại;
- trạng thái hoặc đơn vị sở hữu dữ liệu (`tenant`) không khớp;
- mã kiểm soát (`token`) đã cũ;
- dữ liệu đầu vào chưa hợp lệ.

Không nên ánh xạ mọi kết quả `0` thành cùng một lỗi nghiệp vụ nếu câu lệnh SQL
không bảo đảm ý nghĩa đó.

## Đặt toàn bộ thay đổi liên quan trong cùng giao dịch

Một câu lệnh chỉ nguyên tử trong phạm vi của chính nó. Một nghiệp vụ hoàn chỉnh
thường còn nhiều bước:

```text
Giành quyền xử lý mã yêu cầu (command ID)
→ cập nhật có điều kiện
→ lưu kết quả và dữ liệu kiểm toán (audit)
→ ghi bản ghi hộp thư đi (outbox) chờ phát
→ chốt giao dịch (commit)
```

Các bước ghi vào cơ sở dữ liệu phải dùng cùng một giao dịch. Nếu bước sau cập
nhật thất bại, việc hoàn tác phải hoàn nguyên cả bộ đếm và các bản ghi liên quan.
Lời gọi ra hệ thống ngoài hoặc việc phát thông điệp không tự tham gia giao dịch
PostgreSQL; dùng mẫu hộp thư đi (`outbox`) hoặc quy trình xử lý phù hợp.

Khóa do `UPDATE` lấy vẫn được giữ đến khi giao dịch kết thúc. Vì vậy, gọi dịch vụ
từ xa sau câu lệnh cập nhật vẫn kéo dài thời gian giữ khóa.

## Chống xử lý lặp không thay thế an toàn cập nhật

Tính lũy đẳng (`idempotency`) ngăn cùng một yêu cầu bị áp dụng nhiều lần. Cập
nhật có điều kiện lại ngăn nhiều yêu cầu khác nhau cùng vượt quá giới hạn tài
nguyên.

Một bản ghi duy nhất dùng để giành quyền xử lý mã yêu cầu có thể trả lại kết quả
đã lưu khi yêu cầu bị gửi trùng. Tuy nhiên, bản ghi này không ngăn hai yêu cầu
khác nhau cạnh tranh cùng một dòng. Ngược lại, điều kiện tồn kho không nhận ra
cùng một yêu cầu đang được gửi lại khi dữ liệu vẫn còn đủ.

Hệ thống cần cả hai cơ chế nếu phải bảo vệ cả hai quy tắc.

## Ràng buộc là lớp bảo vệ bổ sung

```sql
check (available_quantity >= 0)
check (reserved_quantity >= 0)
check (available_quantity + reserved_quantity = on_hand_quantity)
```

Các ràng buộc (`constraint`) chặn giá trị không hợp lệ từ mọi đường ghi.
PostgreSQL `CHECK` chỉ nên dựa trên dữ liệu của chính dòng đang được kiểm tra,
không dùng như một khẳng định trải qua nhiều bảng.

Ràng buộc không thay thế giao dịch nguyên tử giữa bộ đếm và dữ liệu kiểm toán.
Hệ thống vẫn cần đối soát (`reconciliation`) để phát hiện hai nguồn dữ liệu bị lệch.

## Lưu ý với Spring Data JPA

Phương thức trong lớp truy cập dữ liệu (`repository`) có thể trả trực tiếp số dòng bị ảnh hưởng:

```java
@Modifying
@Query(value = """
        update inventory_item
        set available_quantity = available_quantity - :quantity
        where product_id = :productId
          and available_quantity >= :quantity
        """, nativeQuery = true)
int reserveIfEnough(long productId, int quantity);
```

Câu lệnh DML hàng loạt hoặc SQL viết trực tiếp cập nhật thẳng cơ sở dữ liệu,
không tự đồng bộ thực thể đang được quản lý:

- ngữ cảnh lưu trữ (`persistence context`) có thể vẫn giữ thực thể với dữ liệu cũ;
- một lần `flush()` hoặc `merge()` sau đó có thể ghi đè kết quả vừa cập nhật;
- hàm gọi lại của thực thể (`entity callback`) và cơ chế phát hiện thay đổi
  (`dirty checking`) thông thường không đại diện cho câu lệnh này.

Có ba cách thiết kế an toàn:

1. Không nạp thực thể đích trong cùng giao dịch.
2. `flush()` các thay đổi đang chờ trước câu lệnh và `clear()` sau đó nếu lớp dịch
   vụ sở hữu toàn bộ ngữ cảnh lưu trữ.
3. `refresh()` đúng thực thể nếu vẫn cần dùng thực thể đó.

`clearAutomatically` và `flushAutomatically` là công cụ quản lý vòng đời ngữ
cảnh lưu trữ, không thay thế một ranh giới giao dịch rõ ràng.

## Chốt giao dịch, hoàn tác và thử lại

| Kết quả | Ý nghĩa |
| --- | --- |
| Một dòng bị thay đổi, rồi chốt giao dịch | Thay đổi đã bền vững |
| Không có dòng bị thay đổi, rồi chốt giao dịch | Không có thay đổi; có thể lưu kết quả từ chối |
| Một dòng bị thay đổi, sau đó hoàn tác | Thay đổi bị hoàn nguyên |
| SQLSTATE `55P03` | Chờ khóa thất bại; giao dịch phải hoàn tác |
| SQLSTATE `40P01` | Giao dịch bị chọn làm nạn nhân của bế tắc |
| SQLSTATE `40001` | Giao dịch gặp lỗi tuần tự hóa |

Chỉ thử lại (`retry`) lỗi kỹ thuật khi yêu cầu có thể phát lại an toàn. Mỗi lần
thử phải dùng giao dịch mới, có giới hạn số lần, thời hạn tổng và khoảng chờ
hữu hạn. Không tự động thử lại một kết quả từ chối nghiệp vụ.

## Cập nhật nhiều dòng

Một câu lệnh `UPDATE` có thể thay đổi nhiều dòng, nhưng khi đó cần phân tích thêm:

- thứ tự lấy khóa và nguy cơ bế tắc (`deadlock`);
- số dòng bị ảnh hưởng không còn chỉ là `0` hoặc `1`;
- ý nghĩa nghiệp vụ nếu chỉ một phần các dòng thỏa điều kiện;
- `UPDATE ... FROM` phải không nối nhiều dòng nguồn vào cùng một dòng đích;
- thứ tự dữ liệu trả về không tự được bảo đảm.

Nếu nghiệp vụ yêu cầu toàn bộ một tập dòng đã biết phải cùng thành công hoặc cùng
thất bại, giao dịch vẫn bảo đảm tính nguyên tử nhưng ứng dụng phải so số dòng
bị ảnh hưởng với số dòng mong đợi.

## Dòng không tồn tại và quy tắc trải trên nhiều dòng

Câu lệnh `UPDATE` có điều kiện không tạo mới hoặc khóa một dòng không tồn tại.
Nó cũng không tự bảo vệ quy tắc “không có dòng nào thỏa điều kiện” hoặc giới hạn
được tính từ nhiều dòng con, trừ khi hệ thống có một dòng kiểm soát hoặc bộ đếm
ổn định làm nguồn dữ liệu có thẩm quyền.

Với các quy tắc như vậy, cần cân nhắc:

- `UNIQUE`, `CHECK` hoặc ràng buộc loại trừ (`exclusion constraint`);
- một dòng kiểm soát hoặc bộ đếm ổn định;
- mức cô lập `SERIALIZABLE`;
- quy ước khóa bi quan (`pessimistic locking`);
- thiết kế lại lược đồ dữ liệu.

## Nhiều phiên bản ứng dụng

Trong mô hình nhiều phiên bản chạy song song (`multi-instance`), mọi yêu cầu đều
gửi điều kiện và phép cập nhật tới cùng máy chủ PostgreSQL chính (`primary`).
Khóa dòng, dữ liệu hiện tại và số dòng bị ảnh hưởng đều nằm tại nguồn dữ liệu có
thẩm quyền, nên khóa loại trừ lẫn nhau (`mutex`) trong từng JVM không cần tham
gia bảo vệ quy tắc này.

Tính đúng đắn giữa nhiều máy không đồng nghĩa thông lượng luôn tốt. Một dòng quá
nóng có thể tạo hàng đợi khóa; cần theo dõi thời gian chờ, áp lực lên vùng kết
nối (`connection pool`) và tỷ lệ câu lệnh không thay đổi dữ liệu.

## Khi nên dùng cách này

SQL cập nhật nguyên tử có điều kiện thường là lựa chọn gọn nhất khi:

- dòng đích đã tồn tại và được biết trước;
- điều kiện chỉ dựa trên dữ liệu của dòng đó;
- thay đổi là phép cộng, trừ hoặc chuyển trạng thái;
- ứng dụng ánh xạ được trường hợp số dòng bị ảnh hưởng bằng `0`;
- phần công việc giữ khóa cần ngắn.

Dùng `FOR UPDATE` khi Java cần đọc nhiều dữ liệu rồi mới quyết định. Dùng
`@Version` khi chỉnh sửa đối tượng nghiệp vụ phức tạp và xung đột hiếm. Dùng
ràng buộc khi cơ sở dữ liệu có thể biểu diễn trực tiếp quy tắc cần bảo vệ.

## Những gì cần theo dõi

- số lần cập nhật một dòng và số lần không cập nhật dòng nào;
- tỷ lệ `RETURNING` không trả dòng;
- SQLSTATE `55P03`, `40P01`, `40001`;
- thời gian chờ khóa và thời lượng giao dịch;
- kết quả đối soát giữa bộ đếm và dữ liệu kiểm toán;
- số yêu cầu được phát lại và số dấu vân tay yêu cầu (`fingerprint`) không khớp;
- lỗi do ngữ cảnh lưu trữ giữ thực thể cũ.

Không ghi nhật ký giá trị tham số nhạy cảm. Nên ghi chỉ số đo lường (`metric`) cho
cả kết quả nghiệp vụ và kết quả của câu lệnh SQL để phát hiện mã nguồn bỏ qua số
dòng bị ảnh hưởng.

## Liên kết

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
