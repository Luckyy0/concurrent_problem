# JVM-001 — Trạng thái thay đổi trong Spring Singleton

## Tóm tắt

Trong Spring, một class được gắn `@Service` theo mặc định sẽ chỉ có một object (instance) duy nhất chạy trong toàn bộ ứng dụng. Khi có nhiều người dùng gọi API cùng lúc, các luồng (thread) xử lý request sẽ nhảy vào xài chung cái object này. Nếu bạn thiết kế service lưu dữ liệu riêng của từng request vào các biến (field) trên class, rủi ro cực cao là dữ liệu của người này sẽ bị ghi đè bởi người kia.

Trong bài toán này, chúng ta cần bảo vệ hai **quy tắc sống còn** (business invariant):

```text
Mỗi draft ID phải là duy nhất trong phạm vi đã cam kết.
result.customerId phải thuộc đúng request đã tạo kết quả.
```

Phạm vi của case này chỉ tập trung vào các luồng chạy bên trong **một máy chủ (JVM)**. Nếu bạn quan tâm đến việc mất cập nhật trong database (JPA) và cách xử lý xung đột, hãy xem case `DB-001` nhé.

> **Nói ngắn gọn:** Spring chỉ tạo một cục service thôi, và hàng tá request có thể lao vào dùng chung cục service đó cùng lúc.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| `shared mutable state` | Dữ liệu dùng chung mà luồng nào cũng có thể sửa được. Dễ toang lắm! |
| `race condition` | Tình trạng "đua xe" - kết quả chạy ra hên xui tuỳ thuộc vào luồng nào chạy nhanh hơn. |
| `lost update` | Dữ liệu bạn vừa ghi vào chưa kịp ngáp đã bị một luồng khác ghi đè lên làm mất sạch. |
| `business invariant` | Các quy tắc nghiệp vụ bất di bất dịch, lúc nào cũng phải đúng. |
| `read-modify-write` | Thao tác 3 bước: đọc ra, sửa lại, rồi ghi vào. Cực kỳ dễ bị luồng khác nhảy vào phá đám giữa chừng. |
| `atomic operation` | Thao tác "một phát ăn luôn", chạy một mạch không luồng nào chen ngang được. |
| `happens-before` | Quy tắc đảm bảo một luồng chạy sau sẽ nhìn thấy mọi thay đổi mà luồng chạy trước đã làm. |

## Bối cảnh nghiệp vụ

Tưởng tượng chúng ta có một API tạo `ReceiptDraft` (bản nháp biên lai) trước khi khách hàng thanh toán:

- Request A của khách hàng `alice` gọi tới;
- Request B của khách hàng `bob` cũng vừa tới;
- Tomcat hoặc Jetty (web server) cho phép hai request này chạy song song;
- `ReceiptDraftService` lại lưu ID đếm tiến (sequence) và tên khách hàng vào biến chung của class.

Nếu ID bị trùng, hệ thống có thể gom nhầm log, lưu sai file tạm, hoặc báo sai mã cho hệ thống khác. Tệ hơn nữa là thông tin của ông khách này tự nhiên lại lọt vào hóa đơn của bà khách kia.

## Trạng thái dùng chung và điểm tranh chấp

| Thành phần | Giá trị |
| --- | --- |
| Object dùng chung | Chỉ có 1 instance của `ReceiptDraftService` |
| Field dùng chung | Biến `nextSequence` và `lastCustomerId` |
| Tác nhân đồng thời | Hai luồng T1 và T2 cùng lúc xử lý request |
| Ranh giới transaction| Không dùng đến database transaction ở đây |
| Điểm tranh chấp | Thao tác `++nextSequence` và lúc đọc/ghi `lastCustomerId` |
| Phạm vi bảo vệ | Chỉ xét trong một máy ảo Java (JVM) |

Nhiều bạn nghĩ rằng gắn `@Transactional` vào là yên tâm. Sự thật là `@Transactional` không hề khóa (lock) cái Java object của bạn lại đâu! Hai request vẫn chạy song song trong hai transaction khác nhau, và vẫn dẫm chân lên nhau khi đụng vào biến chung.

> **Nói ngắn gọn:** Database transaction chỉ quản lý dữ liệu trong DB, nó không giúp các biến trong code Java của bạn tự động an toàn khi chạy đa luồng đâu!

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

- Bị trùng sequence do một luồng ghi đè lên dữ liệu của luồng kia;
- Dữ liệu của request này "đi lạc" sang request khác;
- Log hệ thống và mã liên kết bị loạn, không còn tin được;
- Chạy test bình thường thì xanh (pass), nhưng cứ lên thực tế chịu tải cao là lăn ra chết.

### Hậu quả nghiệp vụ

- Khách hàng A tự nhiên thấy mã khách hàng B trong giỏ hàng của mình;
- Các hệ thống đứng sau nhận sai thông tin bản nháp;
- Rất khó trace log để tìm nguyên nhân sự cố;
- Rò rỉ thông tin cá nhân của khách hàng - đây là lỗi cực kỳ nghiêm trọng!

## Hướng sửa được khuyến nghị

Cách tốt nhất là thiết kế service kiểu "không nhớ gì cả" (**stateless**):

- Dữ liệu của request nào thì để gọn trong tham số (parameter) hoặc biến chạy trong hàm (local variable) của request đó thôi;
- Khi trả về kết quả, hãy dùng các object không thể sửa đổi (immutable object);
- Cần tạo ID? Hãy xài một công cụ tạo ID an toàn trong môi trường đa luồng;
- Nếu ID này là mã quan trọng dùng cho nhiều máy chủ, hãy để database lo việc đảm bảo tính duy nhất.

## Khi nào nên dùng từng giải pháp

- **Luôn ưu tiên làm service stateless** (không giữ trạng thái) khi code Spring service.
- Dùng `AtomicLong` nếu bạn chỉ cần một bộ đếm hoặc sequence nhẹ nhàng, và không ngại việc nó bị reset khi restart app hay trùng với máy chủ khác.
- Dùng từ khoá `synchronized` nếu chỉ có một đoạn code nhỏ cần chạy tuần tự trong 1 JVM, lượng truy cập thấp, và bạn không thể bỏ được trạng thái dùng chung.
- Dùng sequence của database (kèm unique constraint) khi ID phải tuyệt đối duy nhất giữa tất cả các máy chủ.
- Hạn chế xài `ThreadLocal` để lén giấu dữ liệu request, cứ truyền thẳng qua tham số hàm là rõ ràng và dễ bảo trì nhất!
