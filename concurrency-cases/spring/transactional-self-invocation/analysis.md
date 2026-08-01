# Proxy, flush, commit và đan xen (interleaving)

## Trạng thái ban đầu

```text
Account A = 100
Account B = 100
Total     = 200
Transfer  = A → B, amount 10
```

## Timeline self-invocation bị lỗi

| Bước | Luồng ghi T1 | Luồng đọc T2 | Database |
| --- | --- | --- | --- |
| 1 | gọi `transfer` qua proxy | | không có service transaction |
| 2 | `this.executeTransfer` | | interceptor không chạy |
| 3 | repository debit mở Tx-D | | A=100, B=100 |
| 4 | debit A rồi Tx-D commit | | A=90, B=100 |
| 5 | dừng ở probe | đọc tổng | đọc 190 |
| 6 | repository credit mở Tx-C | | |
| 7 | credit B rồi commit | | A=90, B=110 |

Nếu bước 6 thất bại, Tx-D không còn để rollback; tổng 190 tồn tại vĩnh viễn.

> **Nói ngắn gọn:** repository vẫn “có transaction”, nhưng đó là hai chiếc hộp
>commit riêng thay vì một hộp bao trọn transfer.

## Timeline proxy đúng

| Bước | Luồng ghi T1 | Luồng đọc T2 tại READ COMMITTED |
| --- | --- | --- |
| 1 | gọi public transactional entry qua proxy, mở Tx-S | |
| 2 | SQL debit chạy/flush trong Tx-S | vẫn thấy A=100, B=100 |
| 3 | dừng ở probe | tổng=200 từ phiên bản đã commit cũ |
| 4 | credit trong cùng Tx-S | |
| 5 | Tx-S commit | lần đọc sau thấy A=90, B=110 |

Flush không làm debit chưa commit hiển thị cho luồng đọc PostgreSQL thông thường tại
`READ COMMITTED`. Commit chung là ranh giới hiển thị bên ngoài (external visibility boundary).

## Kỳ vọng và thực tế

| Khía cạnh | Intended transaction | Self-invocation |
| --- | --- | --- |
| Atomicity | Debit/credit cùng commit | Mỗi repository method commit riêng |
| Rollback | Lỗi ở credit sẽ rollback debit | Debit đã commit |
| Reader | Trạng thái trước hoặc sau transfer | Có thể thấy trạng thái một phần |
| Probe transaction | Kích hoạt (Active) | Không kích hoạt giữa các lời gọi repository |
| Locks | Giữ tới lúc service commit | Row lock của debit được giải phóng sớm |

## Nguyên nhân gốc rễ theo tầng

### Spring AOP proxy

Proxy intercept lời gọi method từ bên ngoài. Lời gọi từ target tới `this` là lời gọi
trực tiếp; metadata của annotation không được interceptor đọc ở runtime path đó.
JDK proxy/CGLIB có implementation khác nhau nhưng hạn chế về self-invocation của proxy
mode vẫn tồn tại.

### Spring Data repository

Repository là proxy khác. `debit` và `credit` được annotate `@Transactional` vẫn được
intercept, nên lỗi không nhất thiết ném ra ngoại lệ “no transaction”; nó tạo hai transaction
hợp lệ nhưng sai phạm vi.

### Hibernate/JPA

Bulk update thực thi SQL trực tiếp. Persistence context có thể bị cũ (stale) nếu cùng
context đã load entity; repository trên production nên cân nhắc `clearAutomatically`
hoặc tránh đọc entity cũ sau khi bulk update. Vấn đề này tách biệt với proxy
boundary.

### PostgreSQL

MVCC chỉ cung cấp visibility theo từng database transaction. Nó không biết hai
transaction Tx-D/Tx-C thuộc cùng một thao tác nghiệp vụ, nên không thể tự gộp chúng.

## Ngoại lệ và rollback

Với boundary đúng, unchecked exception đánh dấu rollback theo mặc định. Checked
exception chỉ rollback nếu cấu hình phù hợp, ví dụ
`rollbackFor = TransferException.class`. Không catch/bỏ qua (swallow) lỗi rồi kỳ vọng
proxy tự rollback; nếu cần chuyển lỗi, hãy giữ nguyên nguyên nhân (cause) và giao kèo (contract) rollback rõ ràng.

Validation không phụ thuộc vào trạng thái thay đổi có thể chạy trước transaction; việc kiểm tra
và cập nhật số dư phải nằm trong ranh giới cơ sở dữ liệu atomic.

## Đồng thời và đa phiên bản (multi-instance)

Transaction của database hoạt động qua nhiều instance của ứng dụng vì PostgreSQL là
ranh giới quản lý. Tuy nhiên, boundary đúng không tự ngăn lost update,
deadlock hoặc write skew giữa nhiều transfer; isolation/locking thuộc `DB-*`.
`synchronized` cục bộ không thay thế service transaction và không mở rộng qua nhiều node.

## Timeout, retry và tác dụng phụ (side effect)

- Transaction timeout phải phù hợp với query/lock timeout; ngoại lệ do timeout sẽ rollback
  transaction còn active.
- Không retry self-invocation bị lỗi; sửa boundary trước.
- Retry transfer cần tính lũy đẳng (idempotency) và phân loại xung đột (conflict classification).
- Phát hành sự kiện/email sau debit nhưng trước commit có thể bị lệch với database; dùng
  hook after-commit hoặc outbox khi cần tính bền vững (durability).
- Ứng dụng crash giữa hai commit của broken code sẽ giữ trạng thái một phần;
  transaction đúng duy nhất được PostgreSQL rollback nếu chưa commit.

## Quan sát

Log transaction ID/correlation ID và kiểm tra
`TransactionSynchronizationManager.isActualTransactionActive()` trong diagnostic/test,
không dùng nó thay thế kiểm tra nghiệp vụ (business assertion). Theo dõi invariant một phần, rollback,
thời gian của transaction và các lời gọi repository không có outer transaction như kỳ vọng.

Kiến thức nền: [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md).
