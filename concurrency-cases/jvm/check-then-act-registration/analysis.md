# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

```text
resources = {}
resourceKey = "tenant-a"
factory open count = 0

T1 gọi register("tenant-a")
T2 gọi register("tenant-a")
```

Quy tắc cần bảo vệ:

```text
active resource cho "tenant-a" <= 1
mọi caller nhận cùng resource đang được registry quản lý
factory open count cho lần đăng ký này = 1
```

## Thứ tự thực thi xen kẽ

```text
T1                                             T2
---------------------------------------------  ----------------------------------
resources.get("tenant-a") -> null
                                                resources.get("tenant-a") -> null
factory.open("tenant-a") -> Resource A
                                                factory.open("tenant-a") -> Resource B
resources.put("tenant-a", Resource A)
return Resource A
                                                resources.put("tenant-a", Resource B)
                                                return Resource B
```

Map cuối cùng:

```text
resources = { "tenant-a" -> Resource B }
```

Nhưng process vẫn có thể chứa cả Resource A và Resource B. Resource A đã được
trả cho T1 nhưng không còn nằm trong registry, vì vậy shutdown/cleanup đi qua
registry không biết để đóng nó.

## Kết quả mong đợi và kết quả thực tế

```text
Expected:
  factory calls     = 1
  active resources  = 1
  T1 result         = Resource A
  T2 result         = Resource A
  registry value    = Resource A

Actual:
  factory calls     = 2
  active resources  = 2
  T1 result         = Resource A
  T2 result         = Resource B
  registry value    = Resource B
  orphan resources  = Resource A
```

> **Nói ngắn gọn:** map không bị hỏng cấu trúc, nhưng business invariant vẫn bị
> phá vì hai resource đã được tạo trước khi map chọn value cuối cùng.

## Nguyên nhân theo từng lớp

### Vòng đời Spring bean

`ManagedResourceRegistry` là singleton nên mọi request trong cùng
`ApplicationContext` dùng chung một map. Spring không buộc các lời gọi
`register(...)` phải chạy lần lượt.

### ConcurrentHashMap

`get` và `put` riêng lẻ đều an toàn cho nhiều luồng. Tuy nhiên, registry cần một
operation logic lớn hơn:

```text
nếu key vắng mặt thì tạo value và công bố value đó
```

Đây là kiểu **kiểm tra rồi hành động** (`check-then-act`). Không có điểm hiệu lực
duy nhất cho toàn bộ chuỗi. T1 và T2 đều có thể quan sát cùng trạng thái “chưa có
key” và cùng đưa ra quyết định hợp lệ dựa trên snapshot cục bộ đó.

### Java Memory Model

`ConcurrentHashMap` công bố value đã được đưa vào map một cách an toàn. Vấn đề
không phải T2 nhìn thấy một object được khởi tạo dở dang; vấn đề là hai object
hoàn chỉnh đã được tạo ra.

An toàn công bố (`safe publication`) và tính nguyên tử (`atomicity`) là hai yêu
cầu khác nhau:

- safe publication giúp thread khác nhìn thấy state hoàn chỉnh của value;
- atomicity bảo đảm chỉ một actor thực hiện chuỗi tạo và đăng ký.

### Spring transaction và database

Case không truy cập database. `@Transactional` không quản lý Java map, không
rollback việc factory đã mở socket/thread và không khóa singleton field.

## Điểm hiệu lực duy nhất

Một cách sửa đúng cần tạo **điểm hiệu lực duy nhất** (`linearization point`):
thời điểm operation được xem là đã thắng và mọi actor phải đồng ý với kết quả
đó.

- Với `computeIfAbsent`, điểm này nằm trong atomic map operation.
- Với `putIfAbsent`, điểm này là lần insert thành công; resource tạo dư vẫn cần
  cleanup.
- Với placeholder `FutureTask`, điểm này là lần placeholder được đưa vào map;
  chỉ actor thắng chạy factory.

## Lỗi, timeout và dọn dẹp tài nguyên

- Nếu factory throw trước khi trả về, map chưa có entry. Factory phải tự dọn phần
  resource đã mở dở dang.
- Nếu dùng `computeIfAbsent` và mapping function throw, map không ghi value;
  caller sau có thể thử lại.
- Nếu dùng `FutureTask`, entry thất bại phải được remove có điều kiện trước khi
  cho phép lần đăng ký sau retry.
- Nếu process crash, toàn bộ local registry biến mất; đây không phải durable
  state.
- Nếu resource có thao tác unregister/close đồng thời, cần một case lifecycle
  riêng để tránh đóng resource đang được caller khác sử dụng.

## Khi có nhiều application instance

```text
App A: registry A -> Resource A
App B: registry B -> Resource B
```

`computeIfAbsent`, `putIfAbsent`, `synchronized` và `FutureTask` chỉ phối hợp các
thread trong cùng JVM. Chúng không bảo đảm toàn cluster chỉ có một resource cho
mỗi key.

> **Nói ngắn gọn:** registry cục bộ có thể đúng trong từng node nhưng vẫn tạo một
> resource ở mỗi node.

## Hậu quả

### Hậu quả kỹ thuật

- tạo resource dư và rò rỉ tài nguyên;
- hai caller giữ hai object khác nhau cho cùng key;
- cleanup không quản lý được resource đã bị overwrite;
- thread/socket/file descriptor tăng theo các lần tranh chấp;
- lỗi không thể phát hiện chỉ bằng `map.size()`.

### Hậu quả nghiệp vụ

- một tenant có nhiều consumer hoặc connection ngoài dự kiến;
- event/callback có thể được xử lý lặp;
- chi phí kết nối và quota phía ngoài tăng;
- deploy dưới tải có thể gây cạn tài nguyên nhanh hơn.

## Phạm vi không được giải quyết trong case này

Case chỉ bảo vệ việc đăng ký resource cục bộ. Nó không giải quyết:

- uniqueness bền vững sau restart;
- uniqueness giữa nhiều application instance;
- distributed lease hoặc fencing;
- database check-then-insert;
- lifecycle phức tạp giữa register, unregister và close.
