# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

Một batch có hai input theo thứ tự:

```text
index 0 → request-a
index 1 → request-b
```

`results` là `ArrayList` rỗng đã được cấp capacity 2:

```text
size = 0
backing array = [null, null]
```

Task T1 tạo result A, task T2 tạo result B. Cả hai gọi `results.add(...)` mà không
có synchronization.

## Thứ tự thực thi làm mất phần tử

Bảng dưới mô tả logic quan trọng của `add`: lấy `size` hiện tại làm insertion
index, ghi element, rồi công bố size mới. Đây không phải một operation atomic.

| Bước | Task T1 | Task T2 | State logic |
| --- | --- | --- | --- |
| 1 | đọc `size = 0`, chọn index 0 | | cả hai slot còn `null` |
| 2 | tạm dừng | đọc `size = 0`, chọn index 0 | cả hai task cùng sở hữu nhầm index 0 |
| 3 | ghi result A vào slot 0 | | `[A, null]` |
| 4 | | ghi result B vào slot 0 | `[B, null]` |
| 5 | ghi/công bố `size = 1` | ghi/công bố `size = 1` | list chỉ có một phần tử |
| 6 | `Future.get()` hoàn tất | `Future.get()` hoàn tất | coordinator thấy list size 1 |

Cách triển khai nội bộ có thể thay đổi giữa các JDK, nhưng `ArrayList` không đưa
ra thread-safety contract cho concurrent structural modification. Ứng dụng
không được dựa vào một interleaving hoặc biểu hiện cụ thể để coi code là an toàn.

`Future.get()` tạo visibility cho action của task trước khi nó hoàn tất. Tuy
nhiên, state được công bố ở bước 6 đã mất result A; memory barrier không thể phục
hồi dữ liệu bị ghi đè.

> **Nói ngắn gọn:** chờ đủ task chỉ bảo đảm nhìn rõ kết quả cuối của cuộc tranh
> chấp, không biến các lần `ArrayList.add` trước đó thành atomic.

## Thứ tự thực thi làm progress snapshot không ổn định

| Bước | Writer T1 | Progress reader T3 |
| --- | --- | --- |
| 1 | bắt đầu `add(resultA)` | |
| 2 | structural modification đang diễn ra | bắt đầu `List.copyOf(results)` |
| 3 | tiếp tục thay backing array hoặc size | traverse source list |
| 4 | hoàn tất add | có thể nhận partial view hoặc exception |

`ConcurrentModificationException` là fail-fast signal best-effort. Iterator
không bắt buộc phát hiện mọi race; việc không có exception không chứng minh
snapshot nhất quán.

## Kết quả mong đợi và kết quả thực tế

| Khía cạnh | Mong đợi | Broken implementation |
| --- | --- | --- |
| Cardinality | Số result bằng số input thành công | Có thể thiếu result dù mọi future thành công |
| Identity | Mỗi `requestId` xuất hiện đúng một lần | Có thể mất một ID; retry có thể tạo duplicate ngoài hệ thống |
| Null safety | Final result không có `null` | Internal backing array có thể chứa slot chưa được công bố hợp lệ |
| Ordering | Output theo input order | Shared append tạo completion order hoặc order không xác định |
| Progress | Snapshot có contract rõ | Traverse cạnh tranh với writer |
| Failure | Không công bố final list khi batch chưa quyết định | Timeout có thể xảy ra trong khi task khác tiếp tục mutate |

## Nguyên nhân theo từng lớp

### ArrayList

`ArrayList` tối ưu cho access tuần tự, không serialize structural modification.
Capacity, backing array, `size` và modification count là mutable state phối hợp
với nhau. Bảo vệ riêng một field không đủ giữ invariant của toàn cấu trúc.

### Java Memory Model

Các lần read/write cạnh tranh trên state nội bộ tạo data race. `final` reference
chỉ hỗ trợ công bố reference ban đầu; `volatile` reference chỉ bảo vệ việc thay
reference. Chúng không tạo mutual exclusion cho method `add` trên object được
tham chiếu.

Completion action của `Future` và `Future.get()` cung cấp happens-before cho
reader sau khi task hoàn tất. Đây là completion barrier hữu ích cho lời giải
đúng, nhưng không sửa concurrent mutation sai đã xảy ra bên trong task.

Xem thêm
[Java Memory Model và tính nguyên tử](../../concepts/java-memory-model-and-atomicity.md).

### Spring và executor

Spring singleton `BrokenBatchQuoteService` được nhiều request thread dùng chung,
nhưng list của mỗi batch còn được chia sẻ cho các executor worker. Bean scope và
executor lifecycle không áp đặt ownership cho object được capture trong lambda.

### Collection thread-safe không quyết định business semantics

Ngay cả khi mọi `add` được bảo vệ, append vẫn theo completion order. Nếu contract
yêu cầu input order, cần giữ index hoặc để coordinator collect future theo thứ
tự input. Thread safety và ordering là hai yêu cầu độc lập.

### Transaction

Không có database transaction, MVCC, commit hoặc rollback. `@Transactional`
không hoàn tác mutation của list. Nếu remote quote call có side effect, việc
cancel hoặc retry batch còn cần idempotency ở ranh giới remote; local collection
không giải quyết phần đó.

## Điểm hiệu lực của lời giải đúng

Với task trả value và coordinator merge:

1. coordinator submit một future cho mỗi input theo input order;
2. worker chỉ tạo và trả `QuoteResult`, không biết output list;
3. coordinator chờ từng future với deadline chung;
4. chỉ coordinator append vào list riêng của nó;
5. `List.copyOf(results)` là điểm công bố final result.

Mutable list được **cô lập quyền sở hữu** (`thread confinement`) trong coordinator.
Không cần concurrent list vì không còn concurrent access.

## Failure, timeout, cancellation và partial result

Batch API phải chọn một policy rõ ràng:

- **all-or-nothing:** một task fail hoặc timeout thì cancel task còn lại, không
  trả final result một phần;
- **per-item outcome:** mỗi input trả `Success` hoặc `Failure`, cardinality output
  vẫn bằng cardinality input;
- **streaming progress:** result được publish qua concurrent channel/queue với
  contract completion order, không giả làm final ordered list.

`Future.cancel(true)` chỉ gửi interrupt; nó không bảo đảm remote call dừng. Client
phải có connect/read/request timeout và tôn trọng interruption khi có thể. Không
được trả mutable accumulator rồi để task nền tiếp tục thay đổi nó.

Deadline phải áp dụng cho toàn batch. Nếu mỗi future được chờ đủ cùng một timeout,
tổng thời gian có thể tăng theo số input.

## Khi có nhiều application instance

Một batch request thông thường được một node điều phối, nên local ownership đủ
bảo vệ accumulator của batch đó. Nếu caller có thể poll progress qua node khác,
`ConcurrentHashMap` local không chia sẻ job state; cần sticky routing hoặc lưu
progress trong một external store có semantics riêng.

JVM lock hoặc concurrent collection không cung cấp exactly-once remote call giữa
nhiều node. Đó là invariant phân tán nằm ngoài case này.

## Hậu quả

### Hậu quả kỹ thuật

- mất phần tử, cardinality sai và output order không ổn định;
- exception khi copy/iterate progress;
- task tiếp tục ghi sau timeout hoặc sau khi caller đã nhận response;
- executor saturation nếu remote call thiếu timeout;
- diagnostic khó đọc vì từng future đều báo success.

### Hậu quả nghiệp vụ

- batch tổng hợp thiếu quote nhưng bị đánh dấu hoàn tất;
- quote bị ghép sai request;
- retry làm tăng tải hoặc side effect ở provider;
- progress và final result mâu thuẫn;
- reconciliation phải tìm lỗi không có exception rõ ràng.

## Phạm vi không được giải quyết trong case này

Case không thiết kế graph composition phức tạp, fan-out/fan-in bằng
`CompletableFuture`, hoặc structured concurrency; các chủ đề đó thuộc `JVM-009`.
Nó cũng không xử lý transaction database hay deduplication giữa nhiều node.
