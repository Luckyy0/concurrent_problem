# JVM-004 — Thay đổi ArrayList dùng chung trong tác vụ song song

## Tóm tắt

Một Spring service nhận batch yêu cầu báo giá rồi gửi từng item sang các task chạy
song song. Các task cùng gọi `results.add(...)` trên một `ArrayList`; progress
endpoint có thể đồng thời traverse list để hiển thị kết quả tạm thời.

`ArrayList` không hỗ trợ concurrent mutation. Hai lần `add` có thể cùng dùng một
vị trí nội bộ hoặc ghi đè cập nhật `size`, làm mất result dù mọi task đều hoàn
tất. Iterator còn có thể ném `ConcurrentModificationException`; nếu coordinator
đọc trước completion barrier, nó cũng có thể thấy state cũ hoặc chỉ có một phần.

Case bảo vệ các **quy tắc luôn phải đúng** (`business invariant`) trong một JVM:

```text
Mỗi request hợp lệ tạo đúng một QuoteResult tương ứng.
Kết quả cuối không thiếu, không trùng và không chứa phần tử null.
Không công bố final result trước khi mọi task thành công hoặc batch đã đi vào trạng thái thất bại.
Nếu API cam kết input order, output phải giữ cùng thứ tự dù task hoàn tất khác thứ tự.
```

> **Nói ngắn gọn:** chạy công việc song song không có nghĩa nhiều task phải cùng
> sửa một collection; cách an toàn hơn là để mỗi task trả value cho một owner duy
> nhất tổng hợp.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| list không an toàn cho nhiều luồng (`non-thread-safe list`) | List không có contract cho nhiều thread cùng thay đổi cấu trúc |
| thay đổi cấu trúc (`structural modification`) | Thêm, xóa hoặc resize làm thay đổi số phần tử/cấu trúc collection |
| cô lập quyền sở hữu (`thread confinement`) | Mutable object chỉ được một thread hoặc một scope tuần tự truy cập |
| completion barrier | Mốc xác nhận các task đã kết thúc trước khi coordinator đọc kết quả |
| collector hỗ trợ song song (`concurrent collector`) | Cơ chế gom kết quả có contract phù hợp với parallel execution |
| thứ tự đầu vào (`encounter order`) | Thứ tự phần tử do source stream hoặc danh sách input xác định |
| fail-fast iterator | Iterator có thể phát hiện structural modification và ném exception, nhưng không phải cơ chế synchronization |
| partial result | Tập kết quả mới hoàn thành một phần, chưa đủ điều kiện coi là final |

## Bối cảnh nghiệp vụ

Ứng dụng ecommerce/fintech cần báo giá một batch giao dịch:

- request chứa nhiều `QuoteRequest` theo thứ tự người dùng gửi;
- mỗi task gọi `QuoteClient` độc lập và có latency khác nhau;
- coordinator chờ batch hoàn tất rồi trả `List<QuoteResult>`;
- endpoint vận hành có thể xem tiến độ trong lúc batch chạy;
- API yêu cầu final result giữ input order để caller ghép đúng item.

Việc một task hoàn tất sớm không cho phép nó tự quyết định vị trí logic bằng cách
append vào shared list. Completion order và input order là hai khái niệm khác
nhau.

## Trạng thái dùng chung và điểm tranh chấp

| Thành phần | Giá trị |
| --- | --- |
| Object dùng chung | Một `ArrayList<QuoteResult>` của batch |
| Writer | N task gọi `add` đồng thời |
| Reader | Coordinator hoặc progress endpoint traverse/copy list |
| Operation gây lỗi | `ArrayList.add`, resize backing array, `size++` và iteration |
| Điểm tranh chấp | Nhiều task đọc/cập nhật cùng internal size và backing array |
| Completion barrier | Chỉ xuất hiện khi mọi `Future` đã hoàn tất thành công |
| Ranh giới transaction | Không có database transaction |
| Phạm vi invariant | Một batch trong một application instance |

`Future.get()` thành công tạo quan hệ happens-before từ task sang thread chờ. Nó
bảo đảm visibility sau completion, nhưng không hoàn tác các lần ghi đè đã xảy ra
khi nhiều task mutate `ArrayList`.

## Phạm vi của case

Case tập trung vào result aggregation trong memory:

- chọn owner cho mutable collection;
- phân biệt thread-safe operation với snapshot/ordering semantics;
- xử lý completion, timeout, cancellation và partial failure;
- kiểm thử missing/duplicate/order invariant.

Case không đi sâu vào composition graph của `CompletableFuture`; nội dung đó
thuộc `JVM-009`. Database transaction và uniqueness giữa nhiều node cũng nằm
ngoài phạm vi.

## Điều hướng

- [Cách triển khai bị lỗi](broken-code.md)
- [Dòng thời gian tranh chấp và nguyên nhân](analysis.md)
- [Code đã sửa và các phương án lựa chọn](solutions.md)
- [Cách kiểm thử đồng thời](experiments.md)
- Kiến thức nền:
  [Java Memory Model và tính nguyên tử](../../concepts/java-memory-model-and-atomicity.md)
- Kiến thức nền:
  [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong môi trường thực tế

### Hậu quả kỹ thuật

- `results.size()` nhỏ hơn số task thành công;
- một result bị ghi đè hoặc backing array chứa state không nhất quán;
- progress snapshot ném `ConcurrentModificationException`;
- output mang completion order thay vì input order;
- batch timeout nhưng task còn chạy và tiếp tục mutate list đã được trả về.

### Hậu quả nghiệp vụ

- thiếu báo giá cho một item nhưng không có error tương ứng;
- caller ghép quote sai transaction do order thay đổi;
- tổng tiền hoặc báo cáo batch tính trên tập result thiếu;
- retry toàn batch tạo thêm remote call hoặc duplicate side effect;
- lỗi chỉ xuất hiện dưới concurrency nên khó đối chiếu log.

## Hướng sửa được khuyến nghị

Mặc định dùng **thread confinement**: mỗi task trả về một `QuoteResult`; chỉ
coordinator tạo và mutate `ArrayList` sau khi nhận kết quả từ các `Future`. Danh
sách future theo input order giúp output giữ đúng order mà không cần shared
append.

Với CPU-bound stream transformation thuần túy, dùng pipeline
`parallelStream().map(...).toList()` để framework quản lý partition và merge;
không dùng external mutable accumulator.

Chỉ dùng concurrent collection khi nhiều producer thực sự phải publish result
trong lúc batch đang chạy và contract chấp nhận completion order hoặc weakly
consistent progress view.

## Khi nào nên dùng từng giải pháp

- Task trả value + coordinator merge: lựa chọn mặc định, cần final result đầy đủ,
  failure policy rõ và input order.
- Per-task indexed slot: số input cố định, mỗi task sở hữu một index riêng và
  coordinator chỉ đọc sau completion barrier.
- Parallel stream collector: tác vụ CPU-bound, không blocking I/O và phù hợp với
  execution policy của stream.
- `ConcurrentLinkedQueue`: cần nhiều producer publish completion-order result;
  final snapshot chỉ tạo sau khi producer hoàn tất.
- `Collections.synchronizedList`: chỉ khi mọi access, kể cả iteration, tuân thủ
  cùng monitor; thường khó duy trì hơn ownership rõ ràng.
