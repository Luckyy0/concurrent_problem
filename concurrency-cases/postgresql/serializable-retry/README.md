# DB-009 — `SERIALIZABLE` abort và retry an toàn

## Tóm tắt

Một merchant có hạn mức reservation `100`. Tổng reservation đang hoạt động là
`60`. Hai command đồng thời đều muốn reserve thêm `30`:

```text
T1 đọc tổng 60 → quyết định ACCEPTED → insert reservation C1
T2 đọc tổng 60 → quyết định ACCEPTED → insert reservation C2
```

Nếu cả hai commit, tổng thành `120` và vượt hạn mức. PostgreSQL
`SERIALIZABLE` dùng **Serializable Snapshot Isolation** (`SSI`) để theo dõi quan
hệ read/write. Khi concurrent history không thể tương đương với một thứ tự chạy
tuần tự, PostgreSQL abort một transaction với SQLSTATE `40001`.

`SERIALIZABLE` vì vậy bảo đảm cho các transaction **đã commit**, không hứa mọi
attempt đều thành công. Application phải rollback, backoff và chạy lại toàn bộ
decision trong transaction mới.

> **Nói ngắn gọn:** `SERIALIZABLE` biến anomaly thành một failure có kiểm soát;
> retry boundary mới biến failure đó thành kết quả nghiệp vụ đúng.

## Actor và trạng thái dùng chung

| Thành phần | Trạng thái |
| --- | --- |
| `merchant_limit` | merchant `7`, limit `100` |
| `credit_reservation` | một reservation `ACTIVE` trị giá `60` |
| T1 trên App-1 | command C1, yêu cầu `30` |
| T2 trên App-2 | command C2, yêu cầu `30` |
| Authoritative store | PostgreSQL |

Điểm tranh chấp không phải cùng một row được update. Cả hai transaction đọc cùng
predicate:

```sql
select coalesce(sum(amount), 0)
from credit_reservation
where merchant_id = 7
  and status = 'ACTIVE';
```

Sau đó mỗi transaction insert một row khác nhau nhưng row mới đều làm thay đổi
kết quả predicate mà transaction kia đã đọc.

## Invariant

```text
Tổng amount của ACTIVE reservations cho một merchant không vượt limit đã commit.

Mỗi command ID có một durable decision duy nhất: ACCEPTED hoặc REJECTED.

Một retry phải chạy lại toàn bộ read → decide → write trong transaction mới và
không phát external side effect từ attempt đã rollback.
```

Sau khi T1 commit, fresh retry của T2 đọc tổng `90`, nhận ra `90 + 30 > 100` và
ghi decision `REJECTED`. Final state có tổng `90`, C1/C2 mỗi command có đúng một
decision.

## Ranh giới transaction

Retry coordinator không có transaction. Mỗi call qua `SerializableAttemptService`
proxy tạo một physical transaction:

```text
attempt N
  BEGIN ISOLATION LEVEL SERIALIZABLE
  check durable decision by command_id
  read merchant limit and active SUM
  decide
  insert reservation khi accepted
  insert command decision
  flush
  COMMIT hoặc 40001 + ROLLBACK
```

Conflict có thể xuất hiện lúc statement, Hibernate flush hoặc commit. Coordinator
chỉ catch sau khi transactional proxy đã rollback attempt. Nó không retry vài SQL
cuối và không reuse `EntityManager`, entities hay snapshot cũ.

## Kết quả mong đợi và thực tế lỗi

| Khía cạnh | Mong đợi đúng | Cách hiểu sai |
| --- | --- | --- |
| `SERIALIZABLE` | Commit history tương đương chạy tuần tự | Mọi transaction tự block rồi commit |
| `40001` | Known rollback, có thể whole-transaction retry | Lỗi database bất ngờ hoặc có thể nuốt |
| Retry | Transaction/snapshot mới, revalidate business rule | Loop trong cùng `@Transactional` method |
| Idempotency | Cùng command ID replay durable decision | Retry tạo command ID mới |
| Exhaustion | Trả explicit temporary failure | Retry vô hạn tới khi success |

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong case |
| --- | --- |
| SSI | Serializable Snapshot Isolation; snapshot isolation cộng dependency checks |
| serialization failure | PostgreSQL abort vì history không thể serialize an toàn |
| SQLSTATE `40001` | Mã chuẩn `serialization_failure` |
| predicate lock | `SIReadLock` dùng để theo dõi dữ liệu/predicate đã đọc; không phải blocking row lock |
| rw-conflict | Transaction đọc một version/predicate mà transaction khác ghi thay đổi |
| dangerous structure | Mẫu dependency có thể tạo serialization cycle |
| whole-transaction retry | Chạy lại cả logic chọn SQL và giá trị trong transaction mới |
| idempotent attempt | Cùng command ID không tạo thêm business effect |
| bounded backoff | Attempt cap, delay có jitter và overall deadline |

## Điều hướng

- [Code lỗi và retry sai boundary](broken-code.md)
- [SSI timeline, snapshot và failure analysis](analysis.md)
- [Fresh-transaction retry và idempotency](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [Isolation level và database visibility](../../concepts/isolation-levels.md)
- [Deadlock và retry an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production

- `40001` leak thành HTTP 500 dù command có thể hoàn tất sau retry;
- retry trong doomed transaction nhận tiếp `25P02`;
- retry không giới hạn làm tăng database load và conflict amplification;
- stale snapshot khiến attempt lặp lại decision cũ thay vì thấy winner;
- notification gửi trước commit bị duplicate dù database rollback;
- response mất sau commit khiến caller retry và tạo reservation thứ hai nếu không
  có durable command ID;
- scale-out tăng số overlapping serializable transactions trên cùng predicate.

## Hướng sửa khuyến nghị

1. Chỉ dùng `SERIALIZABLE` khi invariant/read-write set thật sự cần nó; constraint,
   conditional write hoặc guard row nhỏ hơn vẫn nên được ưu tiên khi đủ.
2. Đặt isolation trên transactional attempt bean và xác minh
   `current_setting('transaction_isolation')`.
3. Để retry coordinator ở ngoài transaction; mỗi attempt qua proxy mới.
4. Phân loại root SQLSTATE `40001`, rollback hoàn toàn rồi reload mọi state.
5. Retry có attempt cap, exponential backoff có jitter và overall deadline.
6. Giữ nguyên command ID; lưu decision/outbox trong cùng successful transaction.
7. Không retry business rejection, invalid input hoặc failure chưa được allowlist.

## Khi phù hợp

`SERIALIZABLE` phù hợp khi invariant trải trên predicate/nhiều rows, nhiều
mutation paths cùng tuân thủ isolation này, và team vận hành được abort/retry.
Với hot key hoặc decision rất đắt, explicit guard/conditional counter/queue có
thể dự đoán latency tốt hơn.

## Phạm vi

Case tập trung vào transaction-level retry sau `40001`. Write skew gốc thuộc
`DB-005`; PostgreSQL deadlock `40P01` thuộc `DB-008`; Spring advisor ordering và
retry trong doomed transaction được đào sâu ở `SPR-006`.
