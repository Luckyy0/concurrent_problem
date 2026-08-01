# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

```text
limit = 10
active = 9
pending = 0
used = active + pending = 9
```

Hai request T1 và T2 đồng thời muốn giữ một slot để tạo connection. Cả hai gọi
`tryReserveCreation()` trên cùng một Spring singleton.

## Interleaving thứ nhất: kiểm tra rồi increment

| Bước | T1 | T2 | State |
| --- | --- | --- | --- |
| 1 | đọc `active = 9` | | `active=9, pending=0` |
| 2 | đọc `pending = 0`, tổng 9 nên vượt qua (pass) | | chưa reserve |
| 3 | tạm dừng | đọc `active = 9` và `pending = 0`, pass | cả hai cùng tin còn slot |
| 4 | `pending.incrementAndGet()` → 1 | | tổng 10 |
| 5 | | `pending.incrementAndGet()` → 2 | tổng 11, vượt limit |
| 6 | trả về `true` | trả về `true` | hai creation cùng bắt đầu |

Mỗi `get` và `incrementAndGet` đều có atomic/visibility guarantee của riêng nó.
Lỗi nằm ở operation nghiệp vụ gồm việc kiểm tra (check) tổng rồi mới giữ chỗ (reserve).

> **Nói ngắn gọn:** hai thread không tranh chấp bên trong một lần increment;
> chúng tranh chấp quyền quyết định rằng “slot cuối cùng vẫn còn trống”.

## Interleaving thứ hai: capacity gap trong transition

State ban đầu đã đầy:

```text
active = 9, pending = 1, used = 10
```

T1 hoàn tất handshake và chuyển một connection từ pending sang active. Đoạn code lỗi (broken
code) decrement counter này rồi increment counter kia:

| Bước | Completion T1 | New request T2 | State |
| --- | --- | --- | --- |
| 1 | `pending.decrementAndGet()` → 0 | | `active=9, pending=0`, xuất hiện gap giả |
| 2 | tạm dừng | đọc tổng 9, reserve pending → 1 | `active=9, pending=1` |
| 3 | `active.incrementAndGet()` → 10 | | `active=10, pending=1` |
| 4 | hoàn tất | được chấp nhận | tổng 11 |

Về nghiệp vụ, pending connection chỉ đổi trạng thái; nó không trả lại permit rồi lấy
permit mới. Hai quá trình cập nhật counter tách rời đã làm lộ một state không tồn tại trong
mô hình logic.

## Kết quả mong đợi và kết quả thực tế

| Khía cạnh | Mong đợi | Mã triển khai lỗi (Broken implementation) |
| --- | --- | --- |
| Capacity | `active + pending <= limit` ở mọi thời điểm | Tổng có thể vượt limit |
| Reservation | Chỉ một tác nhân (actor) thắng slot cuối | Nhiều actor cùng trả về `true` |
| Transition | Pending sang active bảo toàn tổng | Lộ gap giả giữa decrement và increment |
| Snapshot | Health view là một state nhất quán | Hai `get()` có thể thuộc hai thời điểm khác nhau |
| Release | Một vòng đời (lifecycle) trả đúng một slot | Callback lặp có thể làm counter âm hoặc tạo capacity giả |
| Failure | Quá trình tạo thất bại trả lại pending slot đã sở hữu | Aggregate counter không xác nhận định danh (identity) của reservation |

## Nguyên nhân theo từng lớp

### Volatile

Volatile read nhìn thấy volatile write theo ordering của Java Memory Model. Nhưng
`used++` gồm việc đọc, cộng và ghi. Một thread có thể chen vào giữa; visibility không
đồng nghĩa với atomicity.

### AtomicInteger

`AtomicInteger` cung cấp operation atomic cho một integer. Nó không tạo transaction
trên hai instance và không kéo một biểu thức bên ngoài như
`active.get() + pending.get()` vào cùng một linearization point.

Ngay cả `view()` cũng đọc active rồi pending riêng biệt. Nếu transition xảy ra giữa
hai lần đọc, view có thể kết hợp hai giá trị chưa từng cùng tồn tại trong một
logical state.

Xem nền tảng tại
[Java Memory Model, volatile và atomic variable](../../concepts/java-memory-model-and-atomicity.md).

### Compound invariant

Invariant thuộc về toàn bộ `BudgetState`, nên ranh giới đồng bộ hóa (synchronization boundary) cũng phải bao
trọn state đó. Có ba cách phổ biến:

- mã hóa (encode) nhiều field trong một immutable object và CAS reference;
- dùng một lock bảo vệ mọi field/transition;
- giảm mô hình về một permit counter duy nhất nếu nghiệp vụ chỉ cần tổng.

### Spring

Singleton chỉ khiến mọi phía gọi (caller) dùng cùng một budget; nó không serialize các method call.
Scheduled health check, request và connection callback có thể chạy trên các
thread khác nhau.

### Transaction

Không có database transaction hoặc rollback. `@Transactional` không khóa heap
state và exception sau quá trình increment không tự trả lại slot. Việc dọn dẹp (cleanup) phải là một phần của
connection lifecycle.

## Linearization point của lời giải CAS

Với `AtomicReference<BudgetState>`:

1. thread đọc một immutable state chứa cả active và pending;
2. kiểm tra invariant và tạo state kế tiếp trong local variable;
3. `compareAndSet(current, next)` thử công bố toàn bộ transition;
4. nếu state đã đổi, CAS thất bại và thread đọc lại thay vì dùng quyết định cũ.

CAS thành công là linearization point. Tại đó việc kiểm tra capacity và reserve pending
cùng có hiệu lực. Transition từ pending sang active cũng đổi cả hai field trong một
lần swap reference nên không làm lộ capacity gap.

Hàm tính state mới có thể chạy lại nhiều lần khi có contention. Nó phải thuần túy,
không gọi remote service, không ghi log nghiệp vụ một lần duy nhất và không tạo
side effect ra bên ngoài.

## Kẻ thua cuộc (Loser), thử lại (retry) và contention

- Actor thua CAS sẽ retry với state mới nhất.
- Nếu state mới đã đầy, actor trả về `false`; đây là sự từ chối do quá tải capacity (capacity rejection), không phải
  lỗi kỹ thuật (technical error).
- CAS loop không bảo đảm tính công bằng (fairness). Một thread có thể retry nhiều lần dưới
  tình trạng contention cao.
- Không retry vô hạn quá trình tạo remote connection; CAS chỉ retry các transition ngắn
  trong memory.
- Nếu contention/fairness quan trọng hơn khả năng tiến triển không bị chặn (non-blocking progress), lock hoặc
  semaphore thường dễ vận hành hơn.

## Thất bại (Failure), timeout, crash và double callback

Reservation phải tồn tại trước khi bắt đầu remote handshake:

- handshake thành công: chuyển pending thành active, không đổi tổng used;
- handshake thất bại/timeout: giảm pending đúng một lần;
- connection đóng: giảm active đúng một lần;
- callback trùng lặp: phải bị từ chối hoặc không có tác dụng (no-op) dựa theo một lifecycle token, không được
  trừ aggregate counter thêm lần nữa.

Aggregate counter chỉ biết số lượng, không biết callback thuộc về reservation nào.
Nếu duplicate/out-of-order callback là khả năng thực tế có thể xảy ra, dùng permit handle có
`AtomicBoolean released`, hoặc state machine được định danh (keyed) theo reservation ID.

Process crash làm mất local counter. Hệ điều hành (OS) thường đóng connection của process,
nhưng provider có thể cần timeout để nhận biết. Đây không phải là durable lease và
không có log phục hồi (recovery log).

## Khi có nhiều application instance

Mỗi JVM có budget riêng. Nếu ba node đều cấu hình limit là 10, toàn bộ deployment có
thể mở tới khoảng 30 connection. Local CAS, lock và semaphore không phối hợp giữa
các node.

Nếu provider quota là 10 cho toàn bộ deployment, phải chia quota tĩnh theo node
hoặc dùng sự phối hợp (coordination) bên ngoài với lease/expiry/fencing phù hợp. Tình huống này chỉ
bảo vệ capacity của một process.

## Hậu quả

### Hậu quả kỹ thuật

- đăng ký vượt mức (oversubscription), connection storm và bị provider throttling;
- metric được ghép từ hai thời điểm khác nhau;
- counter âm hoặc capacity rò rỉ (leak) do lifecycle callback sai;
- CAS spin hoặc lock contention khi hệ thống bão hòa (saturated);
- lỗi khó tái hiện vì chỉ xảy ra khi sát limit.

### Hậu quả nghiệp vụ

- request latency tăng và tỷ lệ lỗi (error rate) lan sang các luồng (flow) khác;
- quota của provider bị vượt, ảnh hưởng mọi người bán (merchant) trên node;
- autoscaling/alerting phản ứng theo số liệu không nhất quán;
- retry sau timeout có thể tạo connection dư thừa;
- sự cố cục bộ bị hiểu nhầm là provider outage.

## Phạm vi không được giải quyết trong tình huống này

Tình huống không bảo vệ inventory bền vững, account balance, database row hoặc quota
phân tán. Nó cũng không cung cấp lease/fencing sau khi crash. Trọng tâm là compound
state trong một JVM.
