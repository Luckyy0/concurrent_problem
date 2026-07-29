# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

Registry đang giữ generation 41:

```text
merchant-a → Provider-X, generation=41
merchant-b → Provider-Y, generation=41
```

Config service trả về generation 42 với hai route hợp lệ:

```text
merchant-a → Provider-Z, generation=42
merchant-b → Provider-Y, generation=42
```

Writer áp dụng generation mới bằng `routes.clear()` rồi `routes.putAll(loaded)`.
`putAll` không phải một lần thay thế snapshot; về mặt semantics nó là nhiều
mutation trên cùng map.

## Thứ tự thực thi xen kẽ

| Bước | Refresh thread T1 | Request T2 | Diagnostic thread T3 | State quan sát được |
| --- | --- | --- | --- | --- |
| 1 | tải xong generation 42 | | | map chứa đủ generation 41 |
| 2 | gọi `routes.clear()` | | | map rỗng |
| 3 | tạm dừng | gọi `get("merchant-b")` | | nhận `null`, có thể chạy fallback |
| 4 | thêm route của `merchant-a` | | | map chỉ có một phần generation 42 |
| 5 | tạm dừng | | iterate `entrySet()` | báo cáo chỉ thấy `merchant-a` |
| 6 | thêm route của `merchant-b` | | | map chứa đủ generation 42 |

Không cần hai writer để vi phạm invariant. Một writer và một reader là đủ. Nếu
iteration thực sự chồng lên structural modification, `HashMap` còn có thể ném
`ConcurrentModificationException`; fail-fast là best-effort nên không được dùng
như một cơ chế bảo đảm phát hiện.

> **Nói ngắn gọn:** kết quả cuối cùng có thể đúng, nhưng request đã ra quyết định
> nghiệp vụ trong lúc state chỉ hoàn thành một nửa.

## Kết quả mong đợi và kết quả thực tế

| Khía cạnh | Mong đợi | Thực tế với broken code |
| --- | --- | --- |
| Tính đầy đủ | Reader thấy đủ generation 41 hoặc 42 | Reader có thể thấy map rỗng hoặc một phần generation 42 |
| Tính nhất quán | Các route trong một lần đọc thuộc cùng generation | Một chuỗi đọc có thể ghép state trước và sau refresh |
| Visibility | Refresh thành công được request sau đó nhìn thấy | Không có contract visibility nếu snapshot được gán qua field thường |
| Iteration | Diagnostic có một snapshot ổn định | Iterator có thể thiếu entry, thấy update xen kẽ hoặc ném exception |
| Failure | Refresh lỗi giữ nguyên state tốt gần nhất | Mutate tại chỗ có thể để lại map rỗng hoặc dở dang nếu lỗi xảy ra giữa chừng |

## Nguyên nhân theo từng lớp

### HashMap

`HashMap` không hỗ trợ concurrent structural modification. Khi không có external
synchronization, đọc và ghi đồng thời là data race; hành vi quan sát được không
có thread-safety contract. Không nên xây correctness dựa trên việc một phiên bản
JDK cụ thể “có vẻ chạy được”.

### Compound action

Nghiệp vụ coi refresh là một operation duy nhất, nhưng code triển khai bằng
`clear()` và nhiều `put()`. Không có điểm hiệu lực duy nhất (`linearization
point`) cho toàn bộ generation, nên reader có thể chen vào giữa.

### Java Memory Model

Nếu code chuyển sang copy-and-swap nhưng field giữ reference là field thường,
writer và reader vẫn không có quan hệ happens-before. Việc build map trước dòng
gán là đúng theo program order của writer, nhưng chưa đủ để công bố update cho
thread khác.

`volatile`, monitor lock và atomic variable đều có thể tạo cạnh happens-before
phù hợp. Chi tiết nền tảng nằm trong
[Java Memory Model và công bố object](../../concepts/java-memory-model-and-atomicity.md).

### Spring

Spring singleton là scope vòng đời, không phải synchronization policy. Container
công bố bean đã khởi tạo một cách an toàn, nhưng scheduled method và controller
method vẫn có thể chạy trên các thread khác nhau và truy cập cùng mutable field.

### Transaction

Không có database row, MVCC snapshot, commit hoặc rollback nào tham gia. Thêm
`@Transactional` không bảo vệ memory state. Nếu refresh mutate map rồi ném
exception, transaction rollback cũng không hoàn tác các mutation Java đã xảy ra.

## Điểm hiệu lực của lời giải đúng

Với immutable snapshot, writer thực hiện ba pha:

1. tải và validate dữ liệu trong biến cục bộ;
2. tạo `Map.copyOf(...)` hoàn chỉnh, không còn mutate;
3. công bố snapshot bằng `AtomicReference.set` hoặc một volatile write.

Bước 3 là điểm hiệu lực duy nhất. Reader lấy reference đúng một lần nên toàn bộ
quyết định của request dùng cùng snapshot.

Nếu nhiều refresh writer có thể chạy đồng thời, atomic swap chỉ bảo vệ tính toàn
vẹn chứ chưa bảo vệ độ mới. Generation 42 tải chậm có thể ghi đè generation 43.
Khi invariant yêu cầu generation tăng đơn điệu, writer phải dùng compare-and-set
và từ chối snapshot cũ hơn.

## Lỗi, timeout và vòng đời process

- Nếu config client timeout hoặc trả dữ liệu không hợp lệ trước bước publish,
  giữ nguyên snapshot tốt gần nhất.
- Không `clear()` state đang phục vụ để chuẩn bị cho dữ liệu chưa validate.
- Nếu publish bằng một lần atomic swap, không có rollback một phần cần thực hiện.
- Nếu process crash trước swap, snapshot cũ mất cùng process; sau restart phải
  bootstrap lại từ nguồn cấu hình.
- Nếu process crash sau swap, request trong process đó đã thấy snapshot mới,
  nhưng node khác không tự động nhận cùng generation.
- Retry tải cấu hình phải có backoff và không được cho một response cũ ghi đè
  generation mới hơn.

## Khi có nhiều application instance

Mỗi JVM có một `AtomicReference` riêng. Lời giải local bảo đảm mỗi request trên
một node đọc snapshot hoàn chỉnh, nhưng không bảo đảm node A và node B cùng dùng
một generation.

Nếu thay đổi như “disable provider ngay lập tức” cần có hiệu lực đồng bộ toàn hệ
thống, cần thêm protocol ở nguồn cấu hình: generation có thứ tự, event delivery,
acknowledgement/readiness hoặc một ranh giới database/distributed phù hợp. Không
nên mở rộng JVM lock với kỳ vọng nó khóa được node khác.

## Hậu quả

### Hậu quả kỹ thuật

- data race trên cấu trúc map và reference snapshot;
- state một phần lọt vào request path;
- exception hoặc metric không nhất quán khi iterate;
- refresh cũ ghi đè refresh mới nếu có nhiều writer;
- khó chẩn đoán vì log sau cùng chỉ cho thấy map đã đầy đủ.

### Hậu quả nghiệp vụ

- payment bị route sai, fallback sai hoặc từ chối sai;
- cấu hình disable khẩn cấp có thể bị bỏ qua;
- merchant trong cùng batch quan sát các generation khác nhau;
- retry ở tầng request có thể tạo thêm duplicate attempt phía provider.

## Phạm vi không được giải quyết trong case này

Case không làm cho nhiều thay đổi nghiệp vụ trên nhiều database row trở thành
atomic. Nó cũng không thay thế versioning và coordination giữa các node. Trọng
tâm là cấu trúc map, visibility và snapshot semantics trong một JVM.
