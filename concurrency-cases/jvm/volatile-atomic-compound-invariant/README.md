# JVM-005 — Dùng sai volatile và AtomicInteger cho compound invariant

## Tóm tắt

Một Spring singleton giới hạn số connection cục bộ tới payment provider. Mỗi
connection đang mở được tính vào `active`; mỗi lần tạo connection chưa hoàn tất
được tính vào `pending`. Tổng hai số không được vượt capacity của application
instance.

Developer dùng `AtomicInteger` cho cả hai counter nên tin rằng code đã an toàn.
Tuy nhiên, phép kiểm tra `active.get() + pending.get() < limit` rồi mới
`pending.incrementAndGet()` là một compound operation gồm nhiều atomic operation
riêng. Hai thread có thể cùng vượt qua check và làm tổng vượt limit.

Case bảo vệ các **quy tắc luôn phải đúng** (`compound invariant`):

```text
0 <= active
0 <= pending
active + pending <= limit
Chuyển một slot từ pending sang active không làm thay đổi tổng slot đã giữ.
Creation failure hoặc connection close phải trả lại đúng một slot.
```

> **Nói ngắn gọn:** từng con số có thể được đọc/ghi an toàn, nhưng quy tắc nối
> nhiều con số chỉ an toàn khi toàn bộ lần chuyển state có một điểm hiệu lực duy
> nhất.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| khả năng nhìn thấy (`visibility`) | Thread đọc được giá trị mà thread khác đã ghi theo Java Memory Model |
| tính nguyên tử (`atomicity`) | Operation có hiệu lực như một bước duy nhất, không lộ state ở giữa |
| quy tắc ghép (`compound invariant`) | Điều kiện đúng phụ thuộc đồng thời vào nhiều field hoặc nhiều bước |
| so sánh và hoán đổi (`compare-and-set`, CAS) | Chỉ đổi state nếu state hiện tại vẫn đúng bằng giá trị đã quan sát |
| vòng lặp CAS (`CAS loop`) | Đọc state, tính state mới, thử CAS và đọc lại nếu có thread thắng trước |
| linearization point | Thời điểm duy nhất operation được coi là thành công |
| capacity permit | Quyền chiếm một đơn vị capacity cho tới khi được release |
| transition bảo toàn (`conservation transition`) | Chuyển loại slot nhưng không làm thay đổi tổng slot đang được giữ |
| contention | Nhiều thread cạnh tranh cập nhật cùng state tại một thời điểm |

## Bối cảnh nghiệp vụ

Một application instance duy trì tối đa 10 connection tới provider:

- request cần connection gọi `tryReserveCreation()`;
- nếu được chấp nhận, worker bắt đầu handshake và slot ở trạng thái `pending`;
- handshake thành công chuyển slot từ `pending` sang `active`;
- handshake thất bại trả lại pending slot;
- connection đóng trả lại active slot;
- health endpoint đọc cả hai counter để hiển thị trạng thái.

Capacity này bảo vệ file descriptor, thread, memory và quota kết nối của một JVM.
Nó không phải inventory bền vững và không đảm bảo tổng connection của nhiều node.

## Trạng thái dùng chung và điểm tranh chấp

| Thành phần | Giá trị |
| --- | --- |
| Object dùng chung | Singleton `ProviderConnectionBudget` |
| State dùng chung | `active`, `pending`, `limit` |
| Tác nhân | Request/worker thread tạo, hoàn tất, fail hoặc đóng connection |
| Chuỗi gây lỗi | `get active → get pending → check limit → increment pending` |
| Transition gây lỗi | `decrement pending → increment active` |
| Điểm tranh chấp | Sau check trước reserve, hoặc giữa hai counter update |
| Ranh giới transaction | Không có database transaction |
| Phạm vi invariant | Một application instance/JVM |

`volatile` và `AtomicInteger` giải quyết visibility của từng field. Chúng không tự
động gom một biểu thức nhiều field thành một transaction trong memory.

## Phạm vi của case

Case giải quyết:

- check-and-update trên một capacity counter;
- invariant phụ thuộc hai counter;
- CAS trên immutable state;
- lock và `Semaphore` như các lựa chọn khác;
- timeout, cancellation, underflow và double release trong một JVM.

Case không giải quyết durable inventory, quota dùng chung giữa nhiều node hoặc
reconciliation sau process crash. Các bài toán đó cần database/distributed
coordination tương ứng.

## Điều hướng

- [Cách triển khai bị lỗi](broken-code.md)
- [Dòng thời gian tranh chấp và nguyên nhân](analysis.md)
- [Code đã sửa và các phương án lựa chọn](solutions.md)
- [Cách kiểm thử đồng thời](experiments.md)
- Kiến thức nền:
  [Java Memory Model, volatile và atomic variable](../../concepts/java-memory-model-and-atomicity.md)
- Kiến thức nền:
  [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong môi trường thực tế

### Hậu quả kỹ thuật

- tổng `active + pending` vượt limit;
- provider nhận nhiều handshake hơn capacity dự kiến;
- transition tạo một capacity gap giả và chấp nhận thêm request;
- counter âm do failure/close callback chạy lặp;
- health metric hiển thị một cặp counter chưa từng là state logic hợp lệ;
- CAS retry hoặc lock contention tăng khi gần capacity.

### Hậu quả nghiệp vụ

- vượt connection quota và bị provider throttling hoặc block;
- latency tăng do connection storm;
- request mới được nhận dù hệ thống đã hết resource;
- một callback trùng có thể “tạo” capacity không tồn tại;
- autoscaling dựa trên metric sai đưa ra quyết định không phù hợp.

## Hướng sửa được khuyến nghị

Khi cần đọc chính xác cả `active` và `pending`, gom chúng vào một immutable
`BudgetState` và công bố qua `AtomicReference`. Mỗi transition dùng CAS trên toàn
state, nên check capacity và reserve có cùng một linearization point.

Nếu chỉ cần giới hạn tổng số slot, dùng `Semaphore`: acquire khi bắt đầu creation,
giữ nguyên permit khi chuyển pending thành active, release khi creation fail hoặc
connection đóng.

Nếu transition phức tạp, tần suất contention cao hoặc cần fairness/condition,
dùng một lock bảo vệ các field plain `int`. Atomic counter không phải mục tiêu tự
thân; mục tiêu là bảo vệ invariant dễ review.

## Khi nào nên dùng từng giải pháp

- `AtomicReference<BudgetState>`: cần snapshot chính xác của nhiều counter và
  transition lock-free ngắn.
- Một `AtomicInteger usedSlots`: correctness chỉ phụ thuộc tổng slot, không cần
  phân biệt active/pending trong gate.
- `Semaphore`: capacity là permit có lifecycle acquire/release rõ ràng.
- `ReentrantLock`: nhiều transition, cần condition/fairness hoặc CAS loop khó
  đọc hơn lợi ích nó mang lại.
- Database/distributed limiter: capacity phải dùng chung giữa nhiều node hoặc
  tồn tại qua restart.
