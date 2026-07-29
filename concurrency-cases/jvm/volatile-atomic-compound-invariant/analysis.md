# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

```text
limit = 10
active = 9
pending = 0
used = active + pending = 9
```

Hai request T1 và T2 đồng thời muốn giữ một slot để tạo connection. Cả hai gọi
`tryReserveCreation()` trên cùng Spring singleton.

## Interleaving thứ nhất: check rồi increment

| Bước | T1 | T2 | State |
| --- | --- | --- | --- |
| 1 | đọc `active = 9` | | `active=9, pending=0` |
| 2 | đọc `pending = 0`, tổng 9 nên pass | | chưa reserve |
| 3 | tạm dừng | đọc `active = 9` và `pending = 0`, pass | cả hai cùng tin còn slot |
| 4 | `pending.incrementAndGet()` → 1 | | tổng 10 |
| 5 | | `pending.incrementAndGet()` → 2 | tổng 11, vượt limit |
| 6 | trả `true` | trả `true` | hai creation cùng bắt đầu |

Mỗi `get` và `incrementAndGet` đều có atomic/visibility guarantee của riêng nó.
Lỗi nằm ở operation nghiệp vụ gồm check tổng rồi reserve.

> **Nói ngắn gọn:** hai thread không tranh chấp bên trong một lần increment;
> chúng tranh chấp quyền quyết định rằng “slot cuối cùng vẫn còn trống”.

## Interleaving thứ hai: capacity gap trong transition

State ban đầu đã đầy:

```text
active = 9, pending = 1, used = 10
```

T1 hoàn tất handshake và chuyển một connection từ pending sang active. Broken
code decrement counter này rồi increment counter kia:

| Bước | Completion T1 | New request T2 | State |
| --- | --- | --- | --- |
| 1 | `pending.decrementAndGet()` → 0 | | `active=9, pending=0`, xuất hiện gap giả |
| 2 | tạm dừng | đọc tổng 9, reserve pending → 1 | `active=9, pending=1` |
| 3 | `active.incrementAndGet()` → 10 | | `active=10, pending=1` |
| 4 | hoàn tất | được chấp nhận | tổng 11 |

Về nghiệp vụ, pending connection chỉ đổi trạng thái; nó không trả permit rồi lấy
permit mới. Hai counter update tách rời đã làm lộ một state không tồn tại trong
mô hình logic.

## Kết quả mong đợi và kết quả thực tế

| Khía cạnh | Mong đợi | Broken implementation |
| --- | --- | --- |
| Capacity | `active + pending <= limit` mọi thời điểm | Tổng có thể vượt limit |
| Reservation | Chỉ một actor thắng slot cuối | Nhiều actor cùng trả `true` |
| Transition | Pending sang active bảo toàn tổng | Lộ gap giả giữa decrement và increment |
| Snapshot | Health view là một state nhất quán | Hai `get()` có thể thuộc hai thời điểm khác nhau |
| Release | Một lifecycle trả đúng một slot | Callback lặp có thể làm counter âm hoặc tạo capacity giả |
| Failure | Creation fail trả pending slot đã sở hữu | Aggregate counter không xác nhận identity của reservation |

## Nguyên nhân theo từng lớp

### Volatile

Volatile read nhìn thấy volatile write theo ordering của Java Memory Model. Nhưng
`used++` gồm đọc, cộng và ghi. Một thread có thể chen vào giữa; visibility không
đồng nghĩa atomicity.

### AtomicInteger

`AtomicInteger` cung cấp operation atomic cho một integer. Nó không tạo transaction
trên hai instance và không kéo một biểu thức bên ngoài như
`active.get() + pending.get()` vào cùng linearization point.

Ngay cả `view()` cũng đọc active rồi pending riêng. Nếu transition xảy ra giữa
hai lần đọc, view có thể kết hợp hai giá trị chưa từng cùng tồn tại trong một
logical state.

Xem nền tảng tại
[Java Memory Model, volatile và atomic variable](../../concepts/java-memory-model-and-atomicity.md).

### Compound invariant

Invariant thuộc toàn bộ `BudgetState`, nên synchronization boundary cũng phải bao
trọn state đó. Có ba cách phổ biến:

- encode nhiều field trong một immutable object và CAS reference;
- dùng một lock bảo vệ mọi field/transition;
- giảm mô hình về một permit counter duy nhất nếu nghiệp vụ chỉ cần tổng.

### Spring

Singleton chỉ khiến mọi caller dùng cùng budget; nó không serialize method call.
Scheduled health check, request và connection callback có thể chạy trên các
thread khác nhau.

### Transaction

Không có database transaction hoặc rollback. `@Transactional` không khóa heap
state và exception sau increment không tự trả slot. Cleanup phải là một phần của
connection lifecycle.

## Linearization point của lời giải CAS

Với `AtomicReference<BudgetState>`:

1. thread đọc một immutable state chứa cả active và pending;
2. kiểm tra invariant và tạo state kế tiếp trong local variable;
3. `compareAndSet(current, next)` thử công bố toàn bộ transition;
4. nếu state đã đổi, CAS fail và thread đọc lại thay vì dùng quyết định cũ.

CAS thành công là linearization point. Tại đó check capacity và reserve pending
cùng có hiệu lực. Transition pending sang active cũng đổi cả hai field trong một
reference swap nên không lộ capacity gap.

Hàm tính state mới có thể chạy lại nhiều lần khi contention. Nó phải thuần túy,
không gọi remote service, không ghi log nghiệp vụ một lần duy nhất và không tạo
side effect bên ngoài.

## Loser, retry và contention

- Actor thua CAS retry với state mới nhất.
- Nếu state mới đã đầy, actor trả `false`; đây là capacity rejection, không phải
  technical error.
- CAS loop không bảo đảm fairness. Một thread có thể retry nhiều lần dưới
  contention cao.
- Không retry vô hạn remote connection creation; CAS chỉ retry transition ngắn
  trong memory.
- Nếu contention/fairness quan trọng hơn non-blocking progress, lock hoặc
  semaphore thường dễ vận hành hơn.

## Failure, timeout, crash và double callback

Reservation phải tồn tại trước khi bắt đầu remote handshake:

- handshake success: chuyển pending thành active, không đổi used;
- handshake failure/timeout: giảm pending đúng một lần;
- connection close: giảm active đúng một lần;
- callback trùng: phải bị từ chối hoặc no-op theo một lifecycle token, không được
  trừ aggregate counter lần nữa.

Aggregate counters chỉ biết số lượng, không biết callback thuộc reservation nào.
Nếu duplicate/out-of-order callback là khả năng thực tế, dùng permit handle có
`AtomicBoolean released`, hoặc state machine keyed theo reservation ID.

Process crash làm mất local counters. OS thường đóng connection của process,
nhưng provider có thể cần timeout để nhận biết. Đây không phải durable lease và
không có recovery log.

## Khi có nhiều application instance

Mỗi JVM có budget riêng. Nếu ba node đều cấu hình limit 10, toàn deployment có
thể mở tới khoảng 30 connection. Local CAS, lock và semaphore không phối hợp giữa
các node.

Nếu provider quota là 10 cho toàn bộ deployment, phải chia quota tĩnh theo node
hoặc dùng coordination bên ngoài với lease/expiry/fencing phù hợp. Case này chỉ
bảo vệ capacity của một process.

## Hậu quả

### Hậu quả kỹ thuật

- oversubscription, connection storm và provider throttling;
- metric ghép từ hai thời điểm;
- counter âm hoặc capacity leak do lifecycle callback sai;
- CAS spin hoặc lock contention khi saturated;
- lỗi khó tái hiện vì chỉ xảy ra sát limit.

### Hậu quả nghiệp vụ

- request latency tăng và error rate lan sang flow khác;
- quota provider bị vượt, ảnh hưởng mọi merchant trên node;
- autoscaling/alerting phản ứng theo số liệu không nhất quán;
- retry sau timeout có thể tạo connection dư;
- sự cố cục bộ bị hiểu nhầm là provider outage.

## Phạm vi không được giải quyết trong case này

Case không bảo vệ inventory bền vững, account balance, database row hoặc quota
phân tán. Nó cũng không cung cấp lease/fencing sau crash. Trọng tâm là compound
state trong một JVM.
