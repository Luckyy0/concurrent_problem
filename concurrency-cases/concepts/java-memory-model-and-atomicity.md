# Java Memory Model, tính nguyên tử và trạng thái dùng chung

## Mục đích

Tài liệu này giải thích nền tảng cho các case có nhiều luồng cùng truy cập một
object trong một JVM. Mục tiêu là phân biệt ba đặc tính thường bị nhầm lẫn:

- **tính nguyên tử** (`atomicity`): thao tác có thể bị luồng khác chen vào giữa
  chừng hay không;
- **khả năng nhìn thấy thay đổi** (`visibility`): một luồng có nhìn thấy giá trị
  mà luồng khác vừa ghi hay không;
- **thứ tự quan sát** (`ordering`): compiler, JVM và CPU được phép sắp xếp hoặc
  quan sát các thao tác theo thứ tự nào.

Một chương trình có thể bảo đảm khả năng nhìn thấy nhưng vẫn sai về tính nguyên
tử. Ví dụ, `volatile long counter` giúp các luồng nhìn thấy giá trị mới hơn,
nhưng `counter++` vẫn gồm ba bước riêng:

```text
đọc counter → cộng 1 → ghi counter
```

## Thuật ngữ chính

| Thuật ngữ | Ý nghĩa trong tài liệu này |
| --- | --- |
| `Java Memory Model` | Bộ quy tắc xác định một luồng được phép nhìn thấy giá trị nào |
| `atomicity` | Tính không thể bị chen ngang giữa chừng |
| `visibility` | Khả năng một luồng nhìn thấy thay đổi của luồng khác |
| `ordering` | Thứ tự mà các thao tác được phép quan sát |
| `happens-before` | Quan hệ bảo đảm visibility và ordering giữa hai action |
| `compound operation` | Thao tác logic được tạo từ nhiều bước nhỏ |
| `safe publication` | Công bố object theo cách các luồng khác nhìn thấy state hoàn chỉnh |
| `thread confinement` | Chỉ cho một luồng sở hữu và thay đổi state |

## Java Memory Model và quan hệ xảy ra-trước

Java Memory Model (JMM) xác định giá trị mà một thao tác đọc được phép quan sát.
Nó dùng quan hệ **xảy ra-trước** (`happens-before`) để nối các action giữa nhiều
luồng.

Một số quan hệ quan trọng:

- thao tác mở khóa một monitor xảy ra-trước lần khóa tiếp theo trên cùng monitor;
- thao tác ghi vào field `volatile` xảy ra-trước lần đọc tiếp theo của field đó;
- action trước `Thread.start()` xảy ra-trước action trong thread mới;
- action của thread đã hoàn tất xảy ra-trước khi thread khác trở về từ
  `Thread.join()`;
- các action trong cùng một thread tuân theo program order của thread đó.

Quan hệ xảy ra-trước giúp truyền giá trị và thứ tự quan sát giữa các luồng. Nó
không tự biến một chuỗi nhiều bước thành một thao tác nguyên tử.

> **Nói ngắn gọn:** nhìn thấy đúng giá trị không đồng nghĩa với việc toàn bộ
> chuỗi xử lý được thực hiện như một bước duy nhất.

## Thao tác nguyên tử và quy tắc gồm nhiều bước

Một lần đọc hoặc ghi reference riêng lẻ có thể là nguyên tử, nhưng quy tắc cần
bảo vệ thường gồm nhiều thao tác:

```java
if (remaining > 0) {
    remaining--;
}
```

Quy tắc `remaining >= 0` trải qua chuỗi đọc–quyết định–ghi. Muốn bảo vệ toàn bộ
chuỗi, mọi actor phải dùng cùng một cơ chế phối hợp, chẳng hạn:

- cùng một monitor qua `synchronized`;
- cùng một `Lock`;
- vòng lặp compare-and-set;
- chỉ cho một luồng sở hữu state;
- loại bỏ hoàn toàn trạng thái dùng chung có thể thay đổi.

Một field an toàn cho nhiều luồng không tự bảo vệ quy tắc gồm nhiều field. Ví dụ,
`AtomicLong sequence` chỉ bảo vệ sequence; nó không bảo vệ cặp
`(sequence, customerId)` nếu `customerId` vẫn là field dùng chung.

## volatile, biến atomic và khóa

| Cơ chế | Điều được bảo đảm | Điều không tự được bảo đảm |
| --- | --- | --- |
| `volatile` | Visibility và ordering quanh một field | Tính nguyên tử của chuỗi nhiều bước |
| `AtomicLong` | Thao tác atomic/CAS trên một value | Quy tắc gồm nhiều field |
| `synchronized` | Mutual exclusion và happens-before trên cùng monitor | Phối hợp giữa nhiều JVM |
| `ReentrantLock` | Mutual exclusion cùng timeout/interrupt policy | Correctness nếu actor dùng lock khác |
| Immutable object | State không đổi sau safe publication | Tính nguyên tử của workflow bên ngoài object |

Các từ trong bảng được giữ bằng tiếng Anh khi chúng là tên API hoặc khái niệm
cần tra cứu trực tiếp. Trong phần diễn giải, có thể hiểu:

- `mutual exclusion`: tại một thời điểm chỉ một luồng được vào vùng bảo vệ;
- `compare-and-set` (CAS): chỉ ghi giá trị mới nếu giá trị hiện tại vẫn đúng như
  lúc đã quan sát;
- immutable object: object không thay đổi state sau khi được tạo hoàn chỉnh.

Ưu tiên thiết kế service không giữ trạng thái request hoặc dùng immutable object
trước khi thêm lock. Không có trạng thái dùng chung thì cũng không còn vùng cần
tranh chấp.

## Công bố object an toàn

Một object chỉ thực sự immutable đối với các luồng khác khi:

1. state được thiết lập hoàn chỉnh trước khi công bố;
2. các field phù hợp được khai báo `final`;
3. reference `this` không thoát khỏi constructor;
4. object được công bố qua một quan hệ xảy ra-trước hợp lệ.

Spring công bố singleton bean an toàn sau khi khởi tạo. Điều đó chỉ bảo đảm các
luồng nhìn thấy bean đã được tạo hoàn chỉnh; nó không làm cho mutable field được
cập nhật trong lúc xử lý request trở nên an toàn.

## Ranh giới giữa một JVM và nhiều application instance

Mỗi application instance có heap, monitor và biến atomic riêng:

```text
Load Balancer
    ├── App A: counter = 42, lock A
    └── App B: counter = 42, lock B
```

`AtomicLong`, `synchronized` hoặc `ReentrantLock` trong App A không phối hợp với
App B. Nếu quy tắc nghiệp vụ dùng chung giữa nhiều node, nó phải được bảo vệ tại
ranh giới authoritative dùng chung, ví dụ:

- database constraint;
- conditional SQL;
- row lock;
- durable idempotency record;
- protocol lease/fencing phù hợp với failure model.

> **Nói ngắn gọn:** khóa Java bảo vệ các luồng trong một tiến trình, không bảo vệ
> toàn bộ cụm application.

## Liên kết

- Case áp dụng trực tiếp: [JVM-001](../jvm/spring-singleton-mutable-state/README.md)
- Kỹ thuật kiểm thử: [Kiểm thử đồng thời](concurrency-testing.md)
- Các case liên quan trong [catalog](../CONCURRENCY_CASE_LIBRARY.md):
  `JVM-005`, `JVM-006`, `DB-001`
