# JVM-003 — Thay đổi HashMap đồng thời và công bố không an toàn

## Tóm tắt

Một Spring singleton giữ bảng định tuyến payment trong memory. Request thread đọc
rule để chọn provider, còn scheduled refresh thread tải cấu hình mới rồi cập nhật
cùng một `HashMap`.

Nếu refresh dùng `clear()` rồi `putAll()`, request có thể nhìn thấy map rỗng, một
snapshot mới chỉ có một phần, hoặc gặp lỗi trong lúc iterate. Nếu developer đổi
sang tạo map mới rồi gán reference nhưng field không có cơ chế công bố an toàn,
reader vẫn tham gia một data race và không được Java Memory Model bảo đảm nhìn
thấy snapshot mới.

Case này bảo vệ các **quy tắc luôn phải đúng** (`business invariant`) trong một
JVM:

```text
Mỗi request phải đọc trọn vẹn một generation của bảng định tuyến.
Request không được quan sát map rỗng hoặc snapshot trộn giữa hai generation.
Một PaymentRoute đã công bố không được bị thay đổi từ phía sau.
```

> **Nói ngắn gọn:** thread refresh phải thay cả bảng rule trong một bước có điểm
> hiệu lực rõ ràng; không được tháo bảng cũ rồi lắp từng phần trước mặt request.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| collection không an toàn cho nhiều luồng (`non-thread-safe collection`) | Collection không bảo vệ cấu trúc nội bộ khi nhiều thread cùng đọc và ghi |
| công bố an toàn (`safe publication`) | Đưa object cho thread khác qua một cơ chế bảo đảm object và state đã khởi tạo được nhìn thấy |
| quan hệ xảy ra-trước (`happens-before`) | Quy tắc của Java Memory Model bảo đảm kết quả ghi ở một thread có thể được thread khác quan sát |
| snapshot bất biến (`immutable snapshot`) | Một bản state hoàn chỉnh, không còn bị sửa sau khi được công bố |
| thay reference nguyên tử (`atomic reference swap`) | Đổi từ snapshot cũ sang snapshot mới bằng một thao tác duy nhất |
| iteration nhất quán yếu (`weakly consistent iteration`) | Iterator an toàn trước thay đổi đồng thời nhưng có thể thấy một phần update mới |
| structural modification | Thao tác làm thay đổi cấu trúc collection như thêm, xóa hoặc `clear()` |
| generation | Phiên bản logic của toàn bộ bảng rule sau mỗi lần refresh |

## Bối cảnh nghiệp vụ

Ứng dụng fintech định tuyến payment theo `merchantId`:

- request thread gọi `selectRoute(merchantId)` để chọn provider;
- scheduled refresh thread tải toàn bộ rule từ config service mỗi 30 giây;
- endpoint vận hành có thể yêu cầu refresh thủ công;
- mỗi lần tải trả về một generation hoàn chỉnh, ví dụ generation 41 hoặc 42.

Việc refresh phải có tính all-or-nothing: request có thể dùng generation cũ hoặc
mới, nhưng không được dùng một bảng đang được dựng dở.

`PaymentRoute` trong ví dụ là một immutable `record`. Nếu value vẫn mutable thì
một map immutable chỉ khóa cấu trúc map, không ngăn field bên trong value thay
đổi.

## Trạng thái dùng chung và điểm tranh chấp

| Thành phần | Giá trị |
| --- | --- |
| Object dùng chung | Singleton `PaymentRoutingRegistry` |
| State dùng chung | Bảng `merchantId → PaymentRoute` |
| Reader | Request thread và health/diagnostic thread |
| Writer | Scheduled refresh hoặc manual refresh thread |
| Chuỗi gây lỗi | `clear() → put(...) → put(...)` trên cùng `HashMap` |
| Điểm tranh chấp | Khoảng thời gian map chỉ chứa một phần generation mới |
| Ranh giới transaction | Không có database transaction; đây là state trong một JVM |
| Phạm vi invariant | Một application instance |

Spring công bố singleton bean an toàn sau khi tạo xong. Điều đó không tự động làm
cho mọi lần mutate field của bean trong lúc chạy trở nên thread-safe.

## Phạm vi của case

Case này giải quyết:

- an toàn cấu trúc của map;
- visibility khi đổi snapshot;
- tính nhất quán của một lần đọc toàn bộ bảng rule;
- lựa chọn collection theo semantics mà nghiệp vụ cần.

Case không giải quyết transaction nghiệp vụ trên nhiều key và không đồng bộ cấu
hình giữa nhiều application instance. Mỗi node vẫn có thể đang dùng một
generation khác nhau nếu nguồn cấu hình và cơ chế refresh không cung cấp version
hoặc coordination phù hợp.

## Điều hướng

- [Cách triển khai bị lỗi](broken-code.md)
- [Dòng thời gian tranh chấp và nguyên nhân](analysis.md)
- [Code đã sửa và các phương án lựa chọn](solutions.md)
- [Cách kiểm thử đồng thời](experiments.md)
- Kiến thức nền:
  [Java Memory Model và công bố object](../../concepts/java-memory-model-and-atomicity.md)
- Kiến thức nền:
  [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong môi trường thực tế

### Hậu quả kỹ thuật

- request nhận `null` dù route hợp lệ tồn tại ở generation cũ và mới;
- iterator ném `ConcurrentModificationException` hoặc đọc một tập key không
  nhất quán;
- reader tiếp tục dùng snapshot cũ vì update không được công bố đúng cách;
- metric, health check và routing request nhìn thấy các state khác nhau;
- value mutable bị thay đổi sau khi map đã được công bố.

### Hậu quả nghiệp vụ

- payment bị từ chối sai hoặc rơi vào provider fallback;
- merchant bị định tuyến bằng rule không cùng generation;
- thay đổi khẩn cấp như disable provider có hiệu lực không đồng đều;
- sự cố phụ thuộc timing, khó tái hiện bằng unit test tuần tự.

## Hướng sửa được khuyến nghị

Với workload đọc nhiều và refresh thay toàn bộ bảng, dùng **snapshot bất biến**
(`immutable snapshot`) rồi công bố qua `AtomicReference` hoặc một field
`volatile`. Writer dựng snapshot ngoài đường đọc và đổi reference đúng một lần.

Dùng `ConcurrentHashMap` khi mỗi key thật sự được cập nhật độc lập và nghiệp vụ
chấp nhận iterator nhất quán yếu. Nó không biến một loạt update nhiều key thành
một snapshot nguyên tử.

Dùng `ReentrantReadWriteLock` khi bắt buộc giữ cấu trúc mutable và reader cần khóa
để quan sát một trạng thái nhất quán. Cách này tạo contention và yêu cầu mọi
đường truy cập cùng tuân thủ lock.

## Khi nào nên dùng từng giải pháp

- `AtomicReference<Map<...>>`: refresh toàn bộ bảng, read-heavy, cần snapshot
  all-or-nothing và muốn lưu thêm metadata như generation.
- `volatile Map<...>`: cùng mô hình snapshot nhưng state chỉ là một field đơn
  giản, không cần compare-and-set hoặc update function.
- `ConcurrentHashMap`: update theo từng key độc lập, không cần snapshot toàn bảng.
- `ReentrantReadWriteLock`: compound read/write trên mutable state, có thể chấp
  nhận reader bị block trong lúc refresh.
- Database hoặc distributed configuration protocol: nhiều node phải cùng tuân
  thủ một generation tại một thời điểm nghiệp vụ xác định.
