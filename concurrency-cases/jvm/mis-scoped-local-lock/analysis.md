# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

Artifact `settlement/day-1.csv` chưa tồn tại. Hai request T1 và T2 chạy trên cùng
node, cùng gọi `generate`, nhưng mỗi call tạo một `ReentrantLock` mới.

## Interleaving với lock mới mỗi call

| Bước | T1 | T2 | Shared store |
| --- | --- | --- | --- |
| 1 | tạo Lock-A, acquire thành công | | chưa có artifact |
| 2 | | tạo Lock-B, acquire thành công | chưa có artifact |
| 3 | `exists(key)` → false | | chưa có |
| 4 | | `exists(key)` → false | chưa có |
| 5 | render content A | render content B | chưa có |
| 6 | `put(key, A)` | | chứa A |
| 7 | | `put(key, B)` | B ghi đè A hoặc duplicate write |
| 8 | unlock Lock-A | unlock Lock-B | cả hai báo created |

Hai lock đều hoạt động đúng với chính chúng; không actor nào cạnh tranh cùng lock.

> **Nói ngắn gọn:** lock không hỏng—mapping từ logical resource sang lock mới
> hỏng. Một key đã bị ánh xạ thành hai ổ khóa khác nhau.

## Interleaving khi critical section quá hẹp

Ngay cả khi T1/T2 dùng cùng monitor:

| Bước | T1 | T2 |
| --- | --- | --- |
| 1 | acquire, check false, release | chờ |
| 2 | bắt đầu render ngoài lock | acquire, check false, release |
| 3 | render | render |
| 4 | put | put |

Lock chỉ serialize check chứ không serialize decision “nếu chưa có thì tạo”.
Critical section phải khớp compound action hoặc write phải dùng atomic conflict
detection ở authoritative store.

## Interleaving khi remove keyed lock quá sớm

1. T1 giữ Lock-X cho key K.
2. T2 lấy reference Lock-X từ map và chờ.
3. T1 unlock rồi remove K.
4. T2 acquire Lock-X.
5. T3 `computeIfAbsent(K)` tạo Lock-Y và acquire.
6. T2 và T3 cùng ở critical section cho K.

`ConcurrentHashMap` chỉ làm map operation an toàn; nó không quản lý lifecycle của
lock value và waiter đã giữ reference.

## Kết quả mong đợi và kết quả thực tế

| Khía cạnh | Mong đợi | Broken implementation |
| --- | --- | --- |
| Lock identity | Cùng key trên node dùng cùng lock | Lock mới/caller key tạo identity khác |
| Critical section | Bao trọn check-render-publish | Chỉ check được khóa |
| Per-key concurrency | Key khác nhau chạy song song | `synchronized(this)` serialize mọi key |
| Lock lifecycle | Không có hai live lock cho cùng key | Remove sớm tạo lock cũ và mới |
| Multi-node | Conflict được store/DB phát hiện | Mỗi node tự acquire local lock |
| Failure | Unlock ở mọi exit path | Thiếu `finally` có thể giữ lock vô hạn |

## Nguyên nhân theo từng lớp

### Monitor identity

`synchronized (object)` dùng reference identity, không gọi `equals()` để gom
monitor. Quan hệ happens-before từ unlock tới lock sau chỉ có ý nghĩa khi hai
actor dùng cùng monitor.

### ReentrantLock

`ReentrantLock` là object có identity giống monitor. Tạo trong local variable mỗi
call làm lock không được chia sẻ. Lock phải được unlock bởi thread đã acquire;
không được acquire trước async boundary rồi unlock trong callback thread khác.

### Lock scope và shared state

Synchronization boundary phải rộng ít nhất bằng invariant boundary. Nếu shared
state là một artifact key, mọi path check/publish key đó phải dùng cùng policy.
Nếu shared state nằm trong object store giữa node, heap lock có scope nhỏ hơn
shared state và không thể là correctness boundary.

### Spring

Spring singleton giúp các request trong một application context dùng cùng field
lock. Prototype bean, manual `new`, nhiều context hoặc nhiều JVM phá giả định đó.
Proxy/`@Transactional` không thay đổi monitor identity và không khóa object store.

### Distributed boundary

Node A và B không chia heap. Static field cũng chỉ static trong một classloader
của một JVM. Multi-node correctness cần conditional operation tại store, unique
record tại database, hoặc distributed protocol có lease/fencing.

Xem thêm [Java Memory Model và khóa](../../concepts/java-memory-model-and-atomicity.md).

## Linearization point của các lời giải

- Local striped lock: acquire stripe là điểm actor được quyền đánh giá compound
  action; `put` hoàn tất là điểm artifact được công bố trong node workflow.
- Conditional create: authoritative store quyết định `CREATED` hoặc `EXISTS` tại
  một atomic write; đây là linearization point giữa các node.
- Database unique constraint: unique index conflict tại insert/commit quyết định
  winner theo database semantics.
- Distributed lease: acquire lease chưa đủ nếu old owner có thể tiếp tục; fencing
  token phải được authoritative resource kiểm tra trên write.

## Timeout, interruption, failure và crash

- Validate lock timeout dương và dùng `tryLock`/`lockInterruptibly` khi không muốn
  chờ vô hạn.
- Khi bị interrupt, khôi phục interrupt status rồi trả lỗi phù hợp.
- Unlock trong `finally`, chỉ khi acquire thành công.
- Render failure không được publish artifact; lock vẫn phải release.
- Store timeout có outcome không chắc chắn: write có thể đã thành công. Retry phải
  đọc lại hoặc dùng idempotent/conditional operation.
- JVM crash tự giải phóng local lock nhưng không rollback external write hoặc xóa
  partial artifact.
- Không giữ local lock trong khi chờ một future sẽ cần cùng lock để hoàn tất.

## Contention và granularity

Coarse lock dễ chứng minh nhưng gây head-of-line blocking. Per-key lock cho
concurrency tốt nhưng lifecycle khó. Striped lock có số object cố định, không
remove race; đổi lại hai key khác nhau có thể hash vào cùng stripe và bị serialize.

Fair `ReentrantLock` không sửa correctness và có thể giảm throughput. Chỉ chọn
fairness khi starvation là requirement đã quan sát/đo.

## Khi có nhiều application instance

Hai instance dùng hai stripe arrays riêng. Local lock vẫn hữu ích để giảm công
việc trùng trong từng node, nhưng cả hai node có thể render đồng thời. Correctness
phải dựa vào `putIfAbsent`/conditional request, database uniqueness hoặc protocol
phân tán.

Nếu conditional create trả `EXISTS`, loser bỏ content đã render và đọc artifact
winner khi cần. Nếu render rất đắt và cần single-flight toàn cluster, xem
`DIST-001`; lease cần expiry, ownership và fencing, không chỉ Redis-style mutex
không có token.

## Hậu quả

### Hậu quả kỹ thuật

- duplicate CPU/DB work và write amplification;
- overwrite, metadata inconsistency hoặc duplicate event;
- lock timeout/head-of-line blocking không cần thiết;
- memory leak từ unbounded lock map;
- deadlock/leak khi unlock sai path hoặc sai thread.

### Hậu quả nghiệp vụ

- settlement artifact không xác định theo writer nào;
- downstream nhận notification lặp;
- batch window kéo dài do render trùng;
- single-node test pass nhưng production nhiều replica sai;
- retry sau ambiguous timeout làm che mất nguyên nhân gốc.

## Phạm vi không được giải quyết trong case này

Case chỉ giải thích lựa chọn/scope của local lock và chỉ ra authoritative boundary
cho multi-node. Thiết kế lease renewal, fencing, clock/partition failure thuộc
`DIST-001`.
