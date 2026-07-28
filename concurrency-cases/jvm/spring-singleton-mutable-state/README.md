# JVM-001 — Mutable State in a Spring Singleton

## Tóm tắt

Một Spring `@Service` mặc định là singleton: mọi request trong cùng application
instance gọi chung một object. Nếu service giữ request data hoặc business state
trong mutable field, các request thread có thể ghi đè state của nhau.

Case này bảo vệ hai invariant:

```text
Mỗi draft ID phải unique trong phạm vi được cam kết.
result.customerId phải bằng customerId của chính request tạo result.
```

Case tập trung vào concurrency trong **một JVM**. Lost update trên JPA entity và
database transaction thuộc `DB-001`.

## Business scenario

API tạo một `ReceiptDraft` trước checkout:

- Request A của customer `alice`;
- Request B của customer `bob`;
- cả hai được Tomcat/Jetty xử lý đồng thời;
- `ReceiptDraftService` giữ sequence và customer gần nhất trong field.

Draft ID trùng có thể làm log, file tạm hoặc downstream correlation bị gộp sai.
Customer data bị lẫn giữa request là data leakage, không chỉ là một số đếm
không chính xác.

## Shared state và contention point

| Thành phần | Giá trị |
| --- | --- |
| Shared object | Một instance của `ReceiptDraftService` |
| Shared fields | `nextSequence`, `lastCustomerId` |
| Concurrent actors | Request thread T1 và T2 |
| Transaction boundary | Không có database transaction |
| Contention point | `++nextSequence` và read/write `lastCustomerId` |
| Scope | Một JVM/application instance |

`@Transactional` không phải mutex cho Java object. Nếu thêm annotation đó, hai
request vẫn có thể chạy đồng thời trong hai transaction khác nhau và cùng truy
cập các field của singleton.

## Điều hướng

- [Broken implementation](broken-code.md)
- [Race timeline và root cause](analysis.md)
- [Fixed implementation và trade-offs](solutions.md)
- [Concurrency tests](experiments.md)
- Shared concept:
  [Java Memory Model và atomicity](../../concepts/java-memory-model-and-atomicity.md)
- Shared concept:
  [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

### Technical

- lost update trên local counter;
- duplicate sequence;
- request data leakage;
- non-deterministic log/correlation;
- test pass tuần tự nhưng fail dưới concurrent load.

### Business

- draft của customer A mang dữ liệu customer B;
- downstream command có thể tham chiếu sai draft;
- audit trail khó tin cậy;
- sự cố phụ thuộc timing nên tốn chi phí điều tra.

## Hướng sửa được khuyến nghị

Thiết kế service thành **stateless**:

- request data chỉ nằm trong parameter/local variable;
- result là immutable;
- ID được sinh bởi thread-safe generator phù hợp với phạm vi uniqueness;
- business ID cần global/durable uniqueness phải dùng authoritative store và
  constraint phù hợp, không dùng local counter.

## Khi nào dùng

- Dùng stateless singleton làm mặc định cho Spring service xử lý request.
- Dùng `AtomicLong` cho local metric hoặc local sequence khi reset và
  multi-instance duplication được chấp nhận rõ ràng.
- Dùng `synchronized` cho critical section nhỏ, single-JVM và contention thấp
  khi mutable state thực sự không thể loại bỏ.
- Dùng database sequence/unique constraint khi ID là durable business identity
  dùng chung giữa nhiều node.
- Không dùng `ThreadLocal` để che giấu request state nếu parameter/local
  variable đã giải quyết được vấn đề.

