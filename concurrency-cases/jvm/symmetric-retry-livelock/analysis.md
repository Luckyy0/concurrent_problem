# Phân tích progress failure và retry symmetry

## Trạng thái ban đầu

Tưởng tượng Lock-A và Lock-B đang nằm không. T1 thì muốn lấy A rồi lấy B, T2 thì muốn B rồi đến A. Chưa có dữ liệu nào bị thay đổi cả, vì chúng ta cẩn thận chỉ cho phép đổi khi lấy đủ cả 2 khóa.

## Interleaving tạo livelock

Dưới đây là màn "tấu hài" khiến hệ thống bị kẹt (livelock):

| Bước (Phase) | Anh T1 | Anh T2 | Kết quả (Progress) |
| --- | --- | --- | --- |
| 1 | lấy được A | lấy được B | mỗi anh ôm khư khư một khóa |
| 2 | tạch khi lấy B (`tryLock(B)`) | tạch khi lấy A (`tryLock(A)`) | chả đổi chác được gì |
| 3 | chán nản nhả A | buồn bã nhả B | vẫn thức, state chẳng thay đổi |
| 4 | chờ một tí (fixed backoff) | chờ một tí y chang (fixed backoff) | hai anh cùng nhịp |
| 5 | lại lấy A | lại lấy B | lặp lại từ đầu, nhức cái đầu! |

Trường hợp này không ai giữ khóa lâu (không có wait-for cycle), ai cũng nhả ra rồi chạy tiếp. Nên nếu bạn chụp thread dump, bạn sẽ thấy tụi nó báo là đang chạy (RUNNABLE) hoặc đang chờ (TIMED_WAITING) chứ chả thấy BLOCKED đâu. Khổ nỗi, chạy mãi mà công việc xong vẫn bằng 0.

> **Nói ngắn gọn:** Deadlock là hai ông cứng đầu đứng nhìn nhau không ai nhường ai, còn Livelock là hai ông khách sáo nhường nhau hoài nên kẹt luôn ở cửa.

## Kết quả mong đợi và thực tế

| Khía cạnh | Điều ta muốn (Mong đợi) | Thực tế đắng lòng (Broken loop) |
| --- | --- | --- |
| Kết quả cuối cùng | Xong việc hoặc báo lỗi đàng hoàng trước khi hết giờ | Thử lại mãi mãi không lối thoát |
| Tài nguyên CPU & Thread | Chạy tốn sức thì phải ra việc | Tốn hơi sức chỉ để đụng nhau và thử lại |
| Hủy bỏ (Cancellation) | Có interrupt thì phải dừng | Bỏ qua interrupt, loop cứ chạy phà phà |
| Thay đổi dữ liệu (Mutation) | Chỉ khi thành công mới cập nhật dữ liệu | Không cập nhật bậy bạ, nhưng cũng chả làm được gì |
| Dễ theo dõi (Observability) | Rõ ràng ngân sách và kết quả | Log rác một đống, nhưng việc hoàn thành thì giậm chân tại chỗ |

## Nguyên nhân gốc rễ (Root cause)

Cái hàm `tryLock` chỉ giúp bạn an toàn không bị kẹt cứng (deadlock), nhưng nó chả hứa hẹn là sẽ giúp bạn hoàn thành công việc. Hai luồng chạy y chang nhau, nhận tín hiệu như nhau, thời gian chờ cũng bằng nhau, quyết định cũng giống nhau... thì đụng nhau là tất yếu (cái này gọi là symmetry - tính đối xứng).

Cho tụi nó chờ (backoff) chỉ có ích khi bạn làm "trật nhịp" của tụi nó đi. Thời gian chờ ngẫu nhiên (Jitter) sẽ giúp một anh có cơ hội ra tay trước. Nhưng nhớ nhé, phải kèm thêm cả deadline và giới hạn số lần thử, chứ chờ ngẫu nhiên không thôi thì lỡ xui nó vẫn thử mãi mãi đấy.

## Livelock, deadlock và starvation

Để rõ ràng hơn, mình phân biệt 3 bệnh này nhé:
- **Deadlock**: Các anh khóa nhau thành vòng tròn, ôm nhau chết chìm.
- **Livelock**: Các anh cứ nhường nhau, chạy lui chạy tới mà công việc chung không nhúc nhích.
- **Starvation**: Một anh bị xui, toàn bị mấy anh khác hớt tay trên, đói meo mốc mỏ.

Nếu bạn cho thời gian chờ ngẫu nhiên, bạn có thể trị được bệnh Livelock toàn cục, nhưng chả bảo đảm anh nào xui thì vẫn bị Starvation. Nếu bạn muốn "chơi đẹp" (fairness), hãy nghĩ đến chuyện dùng hàng đợi (queue) hoặc các khóa công bằng (fair lock).

## Ordering là kỹ thuật symmetry breaking mạnh hơn

Sắp xếp thứ tự (Stable total order) là "trùm cuối". Ví dụ dựa vào `channelId`, bắt buộc anh nào cũng phải lấy A trước. Lúc này, 1 anh sẽ ăn A, anh kia đành phải đứng chờ ở cửa A chứ không thể chạy qua lấy B được. Thế là hết chuyện đụng nhau lãng nhách!

Ghi chú nhỏ: Cấu trúc so sánh (Comparator) phải chuẩn xác nhé. Không lấy những cái hay thay đổi như chủ sở hữu để đem ra sắp xếp.

## Retry budget và deadline

Khi đã quyết định làm cơ chế Retry, phải trang bị tận răng:
- Đặt deadline cho toàn bộ quá trình (bao gồm thời gian lấy lock, chờ và xử lý).
- Phải có max attempts (giới hạn số lần thử tối đa) để chống lỗi config linh tinh.
- Chặn mức chờ tối đa (exponential cap) để thời gian chờ không bị bung nóc (overflow).
- Thêm jitter (chờ ngẫu nhiên) cho nó xáo trộn nhịp đi.
- Luôn kiểm tra xem luồng có bị interrupt hay huỷ không.
- Hết giờ hay hết lượt thì quăng lỗi `CONTENDED` hay `TIMEOUT` trả về cho bên gọi.

Đừng bao giờ reset deadline sau mỗi lần thử. Và làm ơn, đừng nhả log bừa bãi mỗi khi kẹt khóa, hỏng hết ổ cứng; hãy dùng biểu đồ (histogram) đếm số lần kẹt là đủ.

## Failure, side effect và cleanup

Lần thử nào tạch thì phải đảm bảo nhả hết khóa trong block `finally`. Do mình chỉ đổi dữ liệu khi đã nắm đủ 2 khóa, nên không cần rollback gì cả. Nhưng nếu trước đó bạn lỡ gọi một API bên ngoài nào đó, thì bạn phải lo liệu vụ idempotent (gọi nhiều lần không sao). Cái này thì không còn là bài toán khóa nội bộ đơn giản nữa rồi.

Nhớ nha, `LockSupport.parkNanos` không quăng `InterruptedException` đâu. Bạn phải tự lấy tay check `Thread.currentThread().isInterrupted()` rồi mới quyết định dừng cuộc chơi.

## Multi-instance và scope

Lưu ý là bài toán này chỉ nằm trong 1 máy ảo Java (JVM) thôi nhé. Mỗi máy có khóa riêng. Nếu bạn định xử lý vấn đề trên nhiều máy hoặc liên quan tới Database, thì phải xem các mẫu như `LOCK-002` hoặc `DB-009`.

## Hậu quả và khả năng quan sát (observability)

Để biết hệ thống có bị Livelock hay không, hãy để ý: số lần retry bắn lên nóc, nhưng số lệnh hoàn thành thì bẹp dí bằng 0, version dữ liệu chả thay đổi. Nếu thấy cảnh này, chụp vài cái thread dump liên tục (theo thời gian) sẽ giúp bạn nhìn rõ bệnh hơn là chỉ chụp 1 tấm rồi ngồi đoán.
