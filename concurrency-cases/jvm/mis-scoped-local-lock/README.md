# JVM-006 — synchronized và ReentrantLock đặt sai phạm vi

## Tóm tắt

Một Spring service tạo settlement artifact rồi ghi vào shared object store theo
`artifactKey`. Hai request cùng key không được render và ghi đồng thời. Developer
đã thêm `synchronized` hoặc `ReentrantLock`, nhưng tạo lock mới trong mỗi lần gọi,
khóa nhầm object, hoặc tin rằng lock của một node bảo vệ được node khác.

Lock chỉ tạo mutual exclusion khi các actor cạnh tranh **cùng lock identity** và
critical section bao trọn shared state cần bảo vệ. Hai object khác nhau không trở
thành cùng monitor chỉ vì `equals()` trả `true`; hai JVM cũng không thể chia sẻ
monitor hoặc `ReentrantLock` trong heap.

Case bảo vệ các **quy tắc luôn phải đúng** (`business invariant`):

```text
Trong một JVM, tối đa một generation workflow cho cùng artifactKey đi qua critical section.
Check tồn tại, render và publish phải nằm trong cùng coordination boundary.
Lock luôn được release khi success, exception, timeout hoặc interruption.
Local lock không được tuyên bố bảo vệ uniqueness giữa nhiều application instance.
```

> **Nói ngắn gọn:** code có từ khóa `synchronized` chưa đủ; mọi actor phải khóa
> đúng cùng một “ổ khóa” và ổ khóa đó phải nằm trong cùng phạm vi với shared state.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| định danh monitor (`monitor identity`) | Object reference cụ thể được dùng bởi `synchronized` |
| phạm vi khóa (`lock scope`) | Tập actor và state thực sự chịu sự bảo vệ của lock |
| vùng tới hạn (`critical section`) | Đoạn code phải chạy loại trừ lẫn nhau để giữ invariant |
| khóa theo key (`keyed lock`) | Cùng logical key luôn được ánh xạ tới cùng lock |
| striped lock | Một số lock cố định; key được băm vào một stripe để tránh map lock tăng vô hạn |
| mutual exclusion | Tại một thời điểm chỉ một actor giữ cùng lock |
| local lock | Lock chỉ tồn tại trong heap của một process/JVM |
| conditional create | Operation ở authoritative store chỉ tạo object nếu key chưa tồn tại |
| fencing token | Số thứ tự giúp authoritative resource từ chối writer cũ sau lease expiry |

## Bối cảnh nghiệp vụ

Ứng dụng cuối ngày tạo file settlement:

- request hoặc scheduler gọi `generate("settlement/2026-07-29.csv")`;
- service kiểm tra artifact đã tồn tại chưa;
- nếu chưa, renderer đọc dữ liệu, tạo bytes rồi publish vào object store;
- request trùng key phải nhận artifact đã có thay vì render lần hai;
- deployment có thể chạy nhiều application instance.

Render có thể tốn CPU/DB read và publish có remote I/O. Duplicate generation làm
tăng tải; hai writer còn có thể ghi đè metadata hoặc nội dung khác nhau nếu input
snapshot không giống nhau.

## Trạng thái dùng chung và điểm tranh chấp

| Thành phần | Giá trị |
| --- | --- |
| Logical resource | Một `artifactKey` |
| Shared state trong node | Renderer, local cache và lock registry |
| Shared state giữa node | Object/artifact store |
| Compound action | `exists(key) → render(key) → put(key, bytes)` |
| Actor | Request, scheduler hoặc retry thread |
| Critical section local | Toàn bộ decision và publish cho cùng key |
| Phạm vi local lock | Một JVM/application instance |
| Authoritative boundary | Store operation/DB constraint nếu cần multi-node uniqueness |

Lock mới mỗi call không có contention với lock của call khác. Lock global đúng
identity nhưng serialize mọi key. Keyed/striped lock cân bằng correctness và
concurrency trong một JVM.

## Phạm vi của case

Case giải thích:

- monitor identity và `equals` khác reference identity;
- lock field so với lock local variable;
- critical section quá hẹp;
- keyed/striped locking và lifecycle của lock registry;
- timeout, interruption, exception và unlock;
- giới hạn tuyệt đối của JVM-local coordination.

Case không phát triển đầy đủ distributed lease/fencing protocol; nội dung đó
thuộc `DIST-001`. Database unique constraint hoặc object-store conditional write
chỉ được nêu như authoritative alternative.

## Điều hướng

- [Cách triển khai bị lỗi](broken-code.md)
- [Dòng thời gian tranh chấp và nguyên nhân](analysis.md)
- [Code đã sửa và các phương án lựa chọn](solutions.md)
- [Cách kiểm thử đồng thời](experiments.md)
- Kiến thức nền: [Java Memory Model và khóa](../../concepts/java-memory-model-and-atomicity.md)
- Kiến thức nền: [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong môi trường thực tế

### Hậu quả kỹ thuật

- duplicate render và duplicate remote write;
- last-writer-wins che mất artifact trước;
- global lock gây head-of-line blocking giữa các key độc lập;
- lock map tăng vô hạn hoặc remove lock quá sớm tạo hai lock cho cùng key;
- thread chờ vô hạn nếu không có timeout;
- exception làm lock không được release nếu thiếu `finally`.

### Hậu quả nghiệp vụ

- settlement file không khớp snapshot kỳ vọng;
- downstream nhận nhiều event “artifact ready”;
- tăng tải database/object store trong thời điểm batch cao;
- một node hoạt động đúng trong test nhưng deployment nhiều node vẫn duplicate;
- retry sau timeout ghi đè artifact đã publish thành công.

## Hướng sửa được khuyến nghị

Trong một JVM:

- dùng private final monitor hoặc `ReentrantLock` field cho coarse-grained lock;
- dùng striped locks khi key space lớn và muốn các key khác nhau chạy song song;
- dùng lock map ổn định nếu key set bounded và lifecycle được quản lý rõ;
- critical section phải bao trọn check và publish nếu đó là invariant local.

Giữa nhiều node, đưa conflict detection về authoritative resource: conditional
create ở object store, database unique constraint/idempotency record, hoặc một
distributed lock/lease có fencing khi thật sự cần. Local lock vẫn có thể giảm
duplicate trong node nhưng không phải correctness boundary toàn hệ thống.

## Khi nào nên dùng từng giải pháp

- Private final monitor: ít contention, mọi key có thể serialize chung.
- `ReentrantLock`: cần `tryLock`, timeout, interruptible wait hoặc condition.
- Striped lock: nhiều key, chấp nhận hash collision làm một số key serialize.
- Stable per-key lock map: key set bounded hoặc có thư viện lifecycle đáng tin.
- Authoritative conditional create/unique constraint: uniqueness qua nhiều node.
- Distributed lease + fencing: operation dài, không thể biểu diễn bằng atomic
  store operation đơn giản; triển khai chi tiết ở `DIST-001`.
