# JVM-007 — Deadlock do acquire lock ngược thứ tự

## Tóm tắt

Chào bạn, lỗi deadlock này rất kinh điển. Tưởng tượng hai thread chuyển tiền giữa hai tài khoản. Thread 1 chuyển từ A sang B nên giữ khoá A và chờ khoá B. Cùng lúc, Thread 2 chuyển từ B sang A nên giữ B và chờ A. Thế là cả hai đứng nhìn nhau, tạo thành vòng lặp chờ vô tận (wait-for cycle).

Tình huống này nhắc nhở chúng ta:

```text
Mọi thao tác cần nhiều account lock phải acquire theo cùng một total order.
Không thread nào được chờ vô hạn ngoài ràng buộc độ trễ (latency) và thời hạn (deadline).
Khi acquire thất bại hoặc bị ngắt (interrupt), mọi lock đã giữ phải được release.
Thao tác chuyển hoàn tất phải bảo toàn tổng balance trong phạm vi in-memory của tình huống.
```

> **Nói ngắn gọn:** Không phải cứ "khóa cả hai" là an toàn. Bạn phải khóa theo một thứ tự cố định, bất kể chiều chuyển khoản là gì.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| deadlock | Tình trạng tắc nghẽn, các thread chờ nhau thành vòng tròn. |
| wait-for cycle | Cái vòng lẩn quẩn: T1 chờ T2, T2 lại chờ T1. |
| lock ordering | Quy tắc "xếp hàng": quy định rõ lock nào phải lấy trước. |
| intrinsic lock | Kiểu lock tích hợp sẵn dùng trong `synchronized`. |
| ownable synchronizer | Các lock xịn như `ReentrantLock`, JVM biết ai đang giữ nó. |
| interruptible acquisition | Xin lock nhưng có thể bị huỷ giữa chừng nếu thread bị ngắt. |
| timed acquisition | Hàm `tryLock` có cài đặt thời gian chờ tối đa. |
| livelock | Thread không ngủ nhưng cứ chạy đi chạy lại mà chả giải quyết được việc gì. |

## Bối cảnh và contention point

T1 chạy `transfer(A, B, 10)`, T2 chạy `transfer(B, A, 20)`. Cả hai đang tranh nhau cập nhật trạng thái trong RAM với `ReentrantLock`.

| Thành phần | Giá trị |
| --- | --- |
| Shared state | Hai đối tượng `LocalAccount` và số dư |
| Tác nhân | Hai thread thực thi lệnh |
| Broken order | Sai lầm ở chỗ: cứ hễ chuyển từ đâu thì khoá thằng đó trước. |
| Cycle | T1 có A đòi B; T2 có B đòi A. Đứng hình! |
| Transaction | Đây là bộ nhớ, không có transaction như Database. |
| Scope | Giới hạn trong 1 ứng dụng JVM thôi nhé. Deadlock database để sau. |

## Điều hướng

- [Cách triển khai bị lỗi](broken-code.md)
- [Wait-for cycle và nguyên nhân](analysis.md)
- [Code đã sửa và lựa chọn timeout](solutions.md)
- [Cách tái hiện và phát hiện deadlock](experiments.md)
- [Deadlock và thử lại an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả production

- Hệ thống đứng cmn hình, thread pool và connection cạn kiệt.
- Gọi timeout từ phía ngoài cũng chả ăn thua nếu thread đang kẹt cứng ở intrinsic lock.
- Client thử gọi lại chỉ làm rác thêm hệ thống.
- Endpoint báo health check vẫn xanh nhưng app thì ngắc ngoải.
- Chỉ có cách khởi động lại ứng dụng, nhưng lại mất dữ liệu đang làm dở trên RAM.

## Hướng sửa khuyến nghị

Dùng ID của tài khoản để tạo luật xếp hàng (total order): thằng nào ID nhỏ hơn thì luôn bị khóa trước, không quan trọng nó là nguồn hay đích. Ngoài ra, nên dùng kiểu xin lock có thể ngắt (interruptible) hoặc có thời gian chờ (timeout). Lỡ có lỗi thì nhớ nhả lock theo thứ tự ngược lại trong khối `finally`.

Nhớ nhé: Timeout không thể thay thế cho việc xếp hàng đúng thứ tự lock. Nó chỉ là phao cứu sinh để tránh bị treo vĩnh viễn.

## Phạm vi

Bài này chỉ bàn chuyện xử lý JVM lock và bộ nhớ trong. Không áp dụng lock kiểu này cho dữ liệu lưu ở database (như bài `DB-008`).

## Khi nào dùng giải pháp nào

- Định hướng tất định (Deterministic order): Bắt buộc khi cần giữ nhiều lock cùng lúc.
- Coarse single lock: Dùng một lock bự cho dễ hiểu, nếu hệ thống nhỏ, ít tranh chấp.
- `tryLock` với deadline: Khi bạn có giới hạn thời gian rõ ràng và thà thất bại còn hơn treo máy.
- Thiết kế không giữ hai lock: Tìm cách chuyển qua kiến trúc message passing hoặc phân chia quyền sở hữu.
- Database transaction: Dùng khi dữ liệu cần lưu dai dẳng và có nhiều server cùng chọc vào.
