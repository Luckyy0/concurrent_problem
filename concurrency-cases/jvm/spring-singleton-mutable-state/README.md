# JVM-001 — Trạng thái thay đổi trong Spring Singleton

## Tóm tắt

Trong Spring, bean gắn `@Service` mặc định chỉ có một instance trong mỗi
`ApplicationContext`. Nhiều luồng xử lý request sẽ gọi chung instance này. Nếu
service lưu dữ liệu riêng của từng request vào field có thể thay đổi, request
sau có thể ghi đè dữ liệu của request trước.

Case này bảo vệ hai **quy tắc luôn phải đúng** (`business invariant`):

```text
Mỗi draft ID phải là duy nhất trong phạm vi đã cam kết.
result.customerId phải thuộc đúng request đã tạo result.
```

Phạm vi case chỉ là các luồng trong **một JVM**. Trường hợp entity JPA bị mất cập
nhật và cách database transaction xử lý xung đột thuộc `DB-001`.

> **Nói ngắn gọn:** Spring chỉ tạo một service object, nhưng nhiều request vẫn có
> thể gọi object đó cùng lúc.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| `shared mutable state` | Trạng thái dùng chung mà nhiều luồng có thể thay đổi |
| `race condition` | Kết quả phụ thuộc vào thứ tự các luồng chạy xen kẽ |
| `lost update` | Một cập nhật bị cập nhật khác ghi đè và biến mất |
| `business invariant` | Quy tắc nghiệp vụ bắt buộc luôn đúng |
| `read-modify-write` | Chuỗi ba bước: đọc giá trị, sửa giá trị, rồi ghi lại |
| `atomic operation` | Thao tác không thể bị luồng khác chen vào giữa chừng |
| `happens-before` | Quan hệ giúp một luồng nhìn thấy thay đổi của luồng khác theo thứ tự hợp lệ |

## Bối cảnh nghiệp vụ

API tạo một `ReceiptDraft` trước khi checkout:

- request A thuộc customer `alice`;
- request B thuộc customer `bob`;
- Tomcat hoặc Jetty xử lý hai request đồng thời;
- `ReceiptDraftService` lưu sequence và customer gần nhất trong field của
  service.

Draft ID bị trùng có thể làm log, file tạm hoặc mã liên kết với hệ thống phía sau
bị gộp sai. Nghiêm trọng hơn, dữ liệu customer của request này có thể xuất hiện
trong kết quả của request khác.

## Trạng thái dùng chung và điểm tranh chấp

| Thành phần | Giá trị |
| --- | --- |
| Object dùng chung | Một instance của `ReceiptDraftService` |
| Field dùng chung | `nextSequence`, `lastCustomerId` |
| Tác nhân đồng thời | Luồng xử lý request T1 và T2 |
| Ranh giới transaction | Không có database transaction |
| Điểm tranh chấp | `++nextSequence` và thao tác đọc/ghi `lastCustomerId` |
| Phạm vi bảo vệ | Một JVM/application instance |

`@Transactional` không phải là khóa cho Java object. Nếu thêm annotation này,
hai request vẫn có thể chạy đồng thời trong hai transaction riêng và cùng truy
cập field của singleton.

> **Nói ngắn gọn:** Database transaction không tự động làm cho field trong Java
> service trở nên an toàn khi nhiều luồng cùng truy cập.

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

- sequence bị trùng do mất cập nhật;
- dữ liệu bị lẫn giữa các request;
- log và mã liên kết không còn đáng tin cậy;
- test tuần tự vẫn pass nhưng hệ thống lỗi dưới tải đồng thời.

### Hậu quả nghiệp vụ

- draft của customer A có thể chứa identifier của customer B;
- lệnh gửi xuống hệ thống sau có thể tham chiếu nhầm draft;
- audit trail khó dùng để điều tra sự cố;
- nếu field chứa dữ liệu nhạy cảm, lỗi trở thành data leakage.

## Hướng sửa được khuyến nghị

Thiết kế service theo hướng **không giữ trạng thái của request** (`stateless`):

- dữ liệu request chỉ nằm trong parameter hoặc local variable;
- kết quả là immutable object;
- ID được tạo bởi generator an toàn cho nhiều luồng và phù hợp với phạm vi cần
  bảo đảm uniqueness;
- nếu ID là business identity dùng chung giữa nhiều node, database hoặc
  authoritative store phải bảo vệ uniqueness bằng constraint phù hợp.

## Khi nào nên dùng từng giải pháp

- Dùng stateless singleton làm lựa chọn mặc định cho Spring service xử lý
  request.
- Dùng `AtomicLong` cho metric hoặc sequence cục bộ khi chấp nhận việc reset sau
  restart và trùng giữa các application instance.
- Dùng `synchronized` cho critical section nhỏ trong một JVM, khi không thể loại
  bỏ trạng thái thay đổi và mức tranh chấp thấp.
- Dùng database sequence cùng unique constraint khi ID là định danh nghiệp vụ
  bền vững và dùng chung giữa nhiều node.
- Không dùng `ThreadLocal` để che giấu dữ liệu request nếu parameter hoặc local
  variable đã diễn tả data flow rõ ràng hơn.
