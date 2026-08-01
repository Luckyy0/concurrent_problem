# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

```text
nextSequence  = 41
lastCustomerId = null

T1 input = "alice"
T2 input = "bob"
```

Biểu thức `++nextSequence` nhìn như một câu lệnh, nhưng thực tế gồm ba bước:

```text
read nextSequence → add 1 → write nextSequence
```

## Thứ tự thực thi xen kẽ

```text
T1: alice                                  T2: bob
----------------------------------------   ----------------------------------
lastCustomerId = "alice"
read nextSequence -> 41
                                             lastCustomerId = "bob"
                                             read nextSequence -> 41
                                             add 1 -> 42
                                             write nextSequence = 42
add 1 -> 42
write nextSequence = 42
read lastCustomerId -> "bob"
return ReceiptDraft(42, "bob")
                                             read lastCustomerId -> "bob"
                                             return ReceiptDraft(42, "bob")
```

## Kết quả mong đợi và kết quả thực tế

```text
Expected:
  alice -> ReceiptDraft(42, "alice")
  bob   -> ReceiptDraft(43, "bob")
  unique sequences = 2

Actual:
  alice -> ReceiptDraft(42, "bob")
  bob   -> ReceiptDraft(42, "bob")
  unique sequences = 1
```

Hai quy tắc bắt buộc cùng bị phá:

- sequence `42` được cấp phát hai lần vì một cập nhật đã bị ghi đè (`lost update`);
- dữ liệu khách hàng của T1 bị thay bằng dữ liệu của T2.

> **Nói ngắn gọn:** hai request cùng đọc sequence cũ, sau đó cùng ghi một giá
> trị mới; đồng thời request B thay thế dữ liệu khách hàng đang được request A sử dụng.

## Nguyên nhân theo từng lớp

### Vòng đời Spring bean

`@Service` mặc định có singleton scope trong một `ApplicationContext`. Spring
không tạo service mới cho từng HTTP request và cũng không buộc các lời gọi phương thức
phải chạy lần lượt. Việc bean được công bố an toàn sau khởi tạo không bảo vệ các
field tiếp tục bị thay đổi trong lúc xử lý request.

### JVM và Java Memory Model

Có **tranh chấp dữ liệu** (`data race`) vì hai luồng cùng truy cập một field, ít
nhất một luồng thực hiện ghi, nhưng không có cơ chế đồng bộ tạo quan hệ
xảy ra-trước giữa chúng.

Hai lỗi độc lập:

1. `++nextSequence` là thao tác read-modify-write không nguyên tử;
2. `lastCustomerId` là dữ liệu request được lưu trong field dùng chung rồi đọc lại sau
   một khoảng xen kẽ (interleaving).

Chuỗi thao tác gây lỗi chính xác là:

```text
shared field write → interleaving → compound update/read shared field
```

Vì vậy, nguyên nhân không chỉ là “có nhiều request cùng lúc”, mà là các request
cùng sửa trạng thái qua một chuỗi không nguyên tử.

### Spring transaction, Hibernate và database

Case này không truy cập database nên không có persistence context, MVCC hoặc
row lock. Ngay cả khi method có `@Transactional`, mỗi request vẫn chạy trong một
transaction riêng nhưng cùng truy cập Java object. Database rollback cũng không
khôi phục giá trị của field trong Java heap.

## Ảnh hưởng của commit, rollback, timeout và crash

- **Commit/rollback:** không áp dụng cho các field cục bộ; không có transaction log
  để phục hồi chúng.
- **Ngoại lệ (Exception):** nếu phương thức lỗi sau khi tăng sequence, sequence bị bỏ trống;
  không tự rollback.
- **Timeout/retry:** phía gọi (client) thử lại sẽ tạo một bản nháp mới; đây không phải là
  luồng công việc an toàn khi lặp lại (idempotent workflow).
- **Tiến trình gặp sự cố/khởi động lại:** bộ đếm cục bộ trở về giá trị ban đầu và có thể tái sử
  dụng sequence cũ.

Các đặc tính này cho thấy bộ đếm cục bộ không phù hợp làm định danh nghiệp vụ bền vững.

## Khi có nhiều application instance

Nếu production có hai application instance:

```text
App A: nextSequence = 41, monitor/AtomicLong A
App B: nextSequence = 41, monitor/AtomicLong B
```

Ngay cả khi dùng `synchronized` hoặc `AtomicLong`, hai node vẫn có thể cấp cùng
sequence. Cơ chế phối hợp cục bộ chỉ có hiệu lực trong JVM đang sở hữu nó.

> **Nói ngắn gọn:** khóa trong App A không khóa được code đang chạy ở App B.

## Hậu quả

### Hậu quả kỹ thuật

- trùng lặp sequence và mất cập nhật (lost update);
- rò rỉ dữ liệu chéo giữa các request;
- log và dữ liệu tương quan không đáng tin;
- hành vi thay đổi theo bộ lập lịch (scheduler);
- không có cam kết phục hồi sau sự cố.

### Hậu quả nghiệp vụ

- khách hàng nhận bản nháp chứa định danh của khách hàng khác;
- hệ thống phía sau (downstream) khi loại bỏ trùng lặp/tương quan có thể gộp sai thao tác;
- kiểm toán và tái hiện sự cố sai;
- rủi ro riêng tư nếu field dùng chung chứa dữ liệu nhạy cảm.

## Vì sao lỗi khó tái hiện bằng kiểm thử đơn vị thông thường

Một kiểm thử đơn vị (unit test) tuần tự luôn tạo ra thứ tự hoàn chỉnh:

```text
call A hoàn tất → call B bắt đầu
```

Test như vậy không tạo được cửa sổ giữa bước đọc và bước ghi. Kiểm thử hồi quy
cần barrier hoặc latch để chủ động điều phối thứ tự xen kẽ; xem
[phần kiểm thử](experiments.md).
