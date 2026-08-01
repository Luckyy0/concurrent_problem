# DB-004 — Lỗi Bóng Ma (Phantom Row) Phá Vỡ Kiểm Tra Giới Hạn (Capacity Check)

## Tóm tắt

Một processing pool `P-42` có sức chứa tối đa (capacity) là `10` và hiện tại đang có `9` slot được cấp phát (trạng thái `ACTIVE`). Khách hàng A và B đồng thời gửi yêu cầu cấp phát. Cả hai yêu cầu đều chạy trong các giao dịch (transactions) hoàn toàn độc lập và thực hiện câu lệnh đếm:

```sql
select count(*)
from slot_allocation
where pool_id = :poolId
  and status = 'ACTIVE';
```

Cả A và B đều nhận được kết quả `9 < 10`, do đó cả hai đều quyết định tiếp tục và chèn (insert) các dòng cấp phát (allocation rows) mới với các ID khác nhau. Cả hai giao dịch cùng commit thành công, dẫn đến tổng số slot `ACTIVE` thực tế trở thành `11`.

Điều kiện bất biến (Invariant):

```text
Với bất kỳ pool nào:
  active count <= capacity

Cụ thể với P-42:
  nếu capacity = 10
  và active count = 9
  thì chỉ một request được chấp nhận, request còn lại phải bị từ chối (rejected).
```

> **Nói ngắn gọn:** Hành động khóa dòng (row lock) khi INSERT một dòng mới không thể bảo vệ được một điều kiện lọc áp dụng trên nhiều dòng (predicate invariant) như kiểm tra giới hạn tổng sức chứa.

## Các thành phần tham gia (Actors và shared state)

| Thành phần | Vai trò |
| --- | --- |
| Allocator A | Đọc số lượng hiện tại và cấp phát slot `A-101`. |
| Allocator B | Đọc số lượng hiện tại và cấp phát slot `B-202`. |
| Bảng `processing_pool` | Lưu trữ cấu hình giới hạn (capacity). |
| Bảng `slot_allocation` | Lưu các slot đã được cấp phát cùng trạng thái. |
| PostgreSQL MVCC | Quản lý snapshot visibility cho các giao dịch. |
| Hibernate | Thực thi các lệnh flush/insert riêng biệt. |

Trạng thái ban đầu (Initial committed state):

```text
processing_pool(P-42): capacity=10
slot_allocation: 9 dòng ACTIVE
```

Trạng thái bị lỗi cuối cùng (Broken final state):

```text
A-101 = ACTIVE
B-202 = ACTIVE
Tổng số dòng ACTIVE: 11
Cả hai request đều ghi nhận là ACCEPTED.
```

## Ranh giới giao dịch và điểm tranh chấp (Transaction boundary và contention point)

Service method `allocate()` thực thi trong một Spring proxy transaction:

```text
BEGIN
  Đọc capacity của pool
  COUNT(*) các dòng ACTIVE của pool
  So sánh COUNT < capacity
  INSERT dòng allocation mới
COMMIT
```

Vấn đề ở đây là trình tự read-then-write này không có tính nguyên tử (non-atomic). Điểm tranh chấp (contention point) không phải là khóa chính (primary key) của dòng cấp phát mới, mà là điều kiện lọc (predicate):

```text
pool_id=P-42 and status=ACTIVE
```

Bởi vì A insert `A-101` và B insert `B-202`, các ràng buộc khóa duy nhất (unique constraints) không hề bị xung đột. Và câu lệnh `COUNT` thông thường không hề tạo ra bất kỳ khóa (lock) nào để giữ chỗ.

## Hành vi theo các mức độ cô lập (Isolation levels)

| Mức cô lập (Isolation) | Hành vi khi đọc lại (Read behavior) | Xung đột kiểm tra giới hạn (Capacity race) |
| --- | --- | --- |
| `READ COMMITTED` | Có thể thấy dòng mới (visible phantom) khi giao dịch khác đã commit. | Hai giao dịch cùng đếm ra `9` và cùng commit thành công. |
| `REPEATABLE READ` | Snapshot ổn định, đọc lại vẫn đếm ra `9`. | Vẫn cho phép cả hai commit, tạo ra `11` dòng. |
| `SERIALIZABLE` | Phát hiện xung đột phụ thuộc (predicate dependency). | Sẽ hủy (abort) một giao dịch với mã lỗi `40001`; ứng dụng phải tự retry. |

Tính ổn định của `REPEATABLE READ` trong PostgreSQL chỉ giúp bạn không thấy dữ liệu thay đổi khi truy vấn nhiều lần trong cùng một giao dịch, nhưng nó **không** tự động bảo vệ điều kiện tổng quát (aggregate capacity invariant) khi bạn ghi thêm dữ liệu mới.

## Kỳ vọng và Thực tế (Expected và actual)

| Bước | Allocator A | Allocator B | Kết quả cuối cùng (Final) |
| --- | --- | --- | --- |
| Đếm số (COUNT) | 9 | 9 | |
| Quyết định | ACCEPT | ACCEPT | |
| Chèn dữ liệu (INSERT) | `A-101` | `B-202` | |
| Cập nhật DB | Commit | Commit | Đếm lại: 11 |
| KỲ VỌNG | Cấp phát thành công | Bị từ chối (FULL) | Đếm lại: 10 |

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| Phantom row | Một dòng dữ liệu xuất hiện hoặc biến mất khi truy vấn cùng một điều kiện trong một giao dịch. |
| Predicate invariant | Điều kiện đúng sai áp dụng trên một tập hợp các dòng dựa trên điều kiện lọc (chứ không phải 1 dòng cụ thể). |
| Check-then-insert | Anti-pattern: kiểm tra trạng thái rồi mới chèn dữ liệu, dễ bị sai lệch nếu trạng thái thay đổi ở giữa. |
| Stable snapshot | Trạng thái dữ liệu nhất quán được nhìn thấy bởi giao dịch `REPEATABLE READ`. |
| Predicate dependency | Sự phụ thuộc read/write trên một tập hợp các dòng thỏa mãn điều kiện nhất định. |
| SSI | Serializable Snapshot Isolation, cơ chế PostgreSQL dùng để đảm bảo mức `SERIALIZABLE`. |
| Serialization failure | Lỗi khi PostgreSQL hủy một giao dịch để tránh sai lệch dữ liệu (mã lỗi `40001`). |
| Authoritative counter | Biến đếm được cập nhật một cách nguyên tử tại database thay vì đếm số dòng. |

## Điều hướng

- [Mã nguồn gây lỗi do count-then-insert (Broken code)](broken-code.md)
- [Phân tích MVCC và điều kiện lọc (MVCC and predicate analysis)](analysis.md)
- [Giải pháp dùng bộ đếm nguyên tử, khóa cha và Serializable (Solutions)](solutions.md)
- [Các bài kiểm thử tái hiện trên PostgreSQL (PostgreSQL experiments)](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Các mức độ cô lập giao dịch (Isolation levels)](../../concepts/isolation-levels.md)
- [Kiểm thử tương tranh (Concurrency testing)](../../concepts/concurrency-testing.md)

## Hậu quả trên môi trường Production

- Vượt quá sức chứa thực tế, có thể gây quá tải bộ nhớ, CPU hoặc làm cạn kiệt tài nguyên của hệ thống xử lý bên dưới.
- Cả hai request đều nhận được kết quả thành công, không bên nào biết để tự xử lý lỗi.
- Giao diện Dashboard hiển thị số lượng vượt mức tối đa dù từng request đơn lẻ đều hợp lệ.
- Constraints dạng `UNIQUE` không giúp ích được gì trong trường hợp vi phạm tập hợp (aggregate) này.
- Khi tăng số lượng instance của ứng dụng (scale out), lỗi này càng dễ xảy ra do số lượng giao dịch đồng thời tăng lên.
- Xóa thủ công dữ liệu rác dễ dẫn đến lỗi trừ kép (double-decrement) nếu bộ đếm không được thiết kế kỹ.
- Logic tự động retry sai cách có thể tạo thêm các dòng rác nếu không đảm bảo tính toàn vẹn (idempotency).

## Hướng sửa chữa khuyến nghị

Biến kiểm tra giới hạn sức chứa thành một rào cản có thể gây ra xung đột rõ ràng ở tầng database:

1. **Cập nhật bộ đếm nguyên tử (Atomic conditional counter):** Thêm một biến đếm `used_slots` ở bảng cha (`processing_pool`) và sử dụng lệnh UPDATE kèm điều kiện để giành slot, thực hiện cùng transaction với lệnh chèn dữ liệu.
2. **Khóa dòng cha (Lock parent):** Dùng lệnh `SELECT ... FOR UPDATE` trên bảng cha (pool) trước khi thực hiện COUNT. Mọi giao dịch muốn cấp phát phải đi qua nút thắt này.
3. **Tạo sẵn slot trống (Pre-create slot rows):** Nếu số lượng slot là nhỏ và cố định, hãy chèn sẵn các dòng trống. Dùng `FOR UPDATE SKIP LOCKED` để cấp phát slot an toàn.
4. **Sử dụng SERIALIZABLE:** Nếu điều kiện đếm quá phức tạp để áp dụng các cách trên, hãy cấu hình giao dịch ở mức `SERIALIZABLE` và bổ sung logic tự động retry (bounded retry) tại ứng dụng khi gặp lỗi `40001`.

**Lưu ý:** Ràng buộc tính duy nhất (Unique request constraint) vẫn cần thiết để tránh trùng lặp cùng một yêu cầu, nhưng nó giải quyết bài toán khác, không thay thế được kiểm tra giới hạn.

## Phạm vi của vấn đề

Tài liệu này tập trung vào bài toán cấp phát dùng chung (generic single-pool capacity) và các lỗi xung đột trên một tập hợp dữ liệu (predicate capacity).
- Các bài toán cấp phát slot cụ thể có trạng thái riêng rẽ (như phòng khách sạn) sẽ được phân tích ở `BOOK-001`.
- Lỗi write skew ảnh hưởng đến logic định tuyến sẽ được phân tích ở `DB-005`.
