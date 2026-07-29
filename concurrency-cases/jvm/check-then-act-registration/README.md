# JVM-002 — Đăng ký resource theo kiểu kiểm tra rồi hành động

## Tóm tắt

Một registry trong memory lưu một `ManagedResource` cho mỗi `resourceKey`.
Developer dùng `ConcurrentHashMap` nên tin rằng code đã an toàn cho nhiều luồng.
Tuy nhiên, code lại thực hiện hai bước tách rời:

```text
kiểm tra key chưa tồn tại → tạo và đưa resource vào map
```

Hai thread có thể cùng vượt qua bước kiểm tra và cùng tạo resource. Map cuối cùng
chỉ giữ một object, nhưng object còn lại đã được mở và có thể tiếp tục giữ thread,
socket hoặc file handle.

Case bảo vệ các **quy tắc luôn phải đúng** (`business invariant`) trong một JVM:

```text
Mỗi resourceKey có tối đa một ManagedResource đang hoạt động.
Mọi caller đăng ký cùng key phải nhận cùng resource được registry quản lý.
Factory chỉ được thực thi một lần cho key đang được đăng ký.
```

> **Nói ngắn gọn:** từng thao tác trên map có thể an toàn, nhưng chuỗi “kiểm tra
> rồi tạo” vẫn có thể bị thread khác chen vào giữa.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| `check-then-act` | Kiểm tra điều kiện rồi mới hành động, với một cửa sổ để thread khác thay đổi state |
| `compound action` | Một thao tác logic được ghép từ nhiều bước riêng |
| `atomic map operation` | Thao tác trên map có điểm hiệu lực duy nhất, không bị chen ngang |
| `computeIfAbsent` | API tạo value chỉ khi key chưa có, trong một thao tác atomic của map |
| `linearization point` | Thời điểm duy nhất mà operation được xem là đã có hiệu lực |
| `orphan resource` | Resource đã mở nhưng không còn được registry theo dõi và đóng đúng cách |
| `safe publication` | Công bố object để thread khác nhìn thấy state đã khởi tạo hoàn chỉnh |

## Bối cảnh nghiệp vụ

Ứng dụng duy trì một local client cho mỗi tenant hoặc endpoint:

- request T1 đăng ký key `tenant-a`;
- request T2 đồng thời đăng ký cùng key;
- `ManagedResourceFactory.open(...)` có thể tạo worker thread, socket hoặc file
  watcher;
- registry phải tái sử dụng resource đã tồn tại thay vì mở thêm resource mới.

Đây là registry cục bộ của một application instance. Nếu toàn hệ thống yêu cầu
một resource duy nhất giữa nhiều node, local map không phải ranh giới đủ mạnh;
trường hợp đó thuộc `DB-006` và `DIST-001`.

## Trạng thái dùng chung và điểm tranh chấp

| Thành phần | Giá trị |
| --- | --- |
| Object dùng chung | Một singleton `ManagedResourceRegistry` |
| State dùng chung | `ConcurrentMap<String, ManagedResource>` |
| Tác nhân đồng thời | Hai hoặc nhiều request/worker thread |
| Chuỗi gây lỗi | `get(key) → open(key) → put(key, resource)` |
| Điểm tranh chấp | Khoảng thời gian sau `get` và trước `put` |
| Ranh giới transaction | Không có database transaction |
| Phạm vi | Một JVM/application instance |

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

- factory chạy nhiều lần cho cùng key;
- resource thua cuộc không được map quản lý;
- rò rỉ thread, socket, file handle hoặc subscription;
- caller nhận hai object khác nhau dù cùng đăng ký một key;
- cleanup và shutdown không còn biết đầy đủ resource cần đóng.

### Hậu quả nghiệp vụ

- cùng một tenant có thể phát sinh nhiều kết nối hoặc consumer;
- event bị xử lý lặp hoặc rate limit phía ngoài bị vượt;
- hệ thống cạn connection/file descriptor dưới tải;
- sự cố chỉ xuất hiện theo timing nên khó tái hiện.

## Hướng sửa được khuyến nghị

- Dùng `computeIfAbsent` khi mapping function nhanh, không gọi lại chính map và
  không thực hiện remote I/O kéo dài.
- Dùng một placeholder như `FutureTask` khi việc khởi tạo chậm hoặc có side
  effect; chỉ actor thắng mới chạy factory, các actor còn lại chờ cùng kết quả.
- Nếu chấp nhận factory chạy nhiều lần, dùng `putIfAbsent` và phải đóng resource
  thua cuộc một cách an toàn.
- Không dùng local registry để bảo vệ uniqueness giữa nhiều application
  instance.

## Khi nào nên dùng từng giải pháp

- `computeIfAbsent`: cache/registry cục bộ, factory ngắn và ít thất bại.
- `FutureTask` placeholder: khởi tạo đắt, nhiều caller cần chia sẻ cùng kết quả,
  cần loại bỏ entry khi factory thất bại để cho phép retry.
- `putIfAbsent` + cleanup: tạo dư resource là an toàn và có thể đóng ngay.
- `synchronized`: registry rất nhỏ, contention thấp và chấp nhận mọi key bị khóa
  chung.
- Database unique constraint hoặc coordination phân tán: invariant dùng chung
  giữa nhiều node hoặc phải tồn tại sau restart.
