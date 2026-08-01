# JVM-005 — Dùng sai volatile và AtomicInteger cho compound invariant

## Tóm tắt

Tưởng tượng bạn có một Spring singleton để giới hạn số kết nối (connection) tới một đối tác thanh toán (payment provider). Mỗi kết nối đang mở thì tính là `active`, còn kết nối đang trong quá trình tạo thì tính là `pending`. Luật ở đây rất đơn giản: tổng số `active` và `pending` không bao giờ được vượt quá sức chứa (capacity) của ứng dụng.

Có một bạn dev quyết định dùng `AtomicInteger` cho cả hai biến đếm này và tự tin rằng code như vậy là an toàn rồi. Nhưng thực tế, việc kiểm tra `active.get() + pending.get() < limit` rồi mới gọi `pending.incrementAndGet()` lại là một chuỗi các hành động riêng lẻ (gọi là compound operation). Hậu quả là, hai luồng (thread) có thể cùng lúc thấy vẫn còn chỗ và rủ nhau tăng biến đếm, làm tổng vượt quá giới hạn lúc nào không hay!

Tình huống này nhằm bảo vệ một tập hợp các quy tắc luôn phải đúng cùng nhau (gọi là `compound invariant`):

```text
0 <= active
0 <= pending
active + pending <= limit
Chuyển một slot từ pending sang active không làm thay đổi tổng số slot đã giữ.
Quá trình tạo thất bại hoặc đóng connection phải trả lại đúng một slot.
```

> **Nói ngắn gọn:** Từng con số thì đọc/ghi an toàn thật đấy, nhưng quy tắc kết hợp nhiều con số chỉ an toàn khi mọi thao tác cập nhật (state transition) diễn ra như một hành động duy nhất, không thể bị chia cắt.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Khả năng nhìn thấy (`visibility`) | Thread này có thể thấy được những gì thread khác vừa ghi vào bộ nhớ. |
| Tính nguyên tử (`atomicity`) | Hành động diễn ra trọn vẹn trong một bước, không ai có thể xen vào giữa chừng để nhìn thấy trạng thái dang dở. |
| Quy tắc ghép (`compound invariant`) | Một điều kiện đúng sai phụ thuộc vào nhiều biến hoặc nhiều bước gộp lại. |
| So sánh và hoán đổi (`compare-and-set`, CAS) | Chiêu thức cập nhật dữ liệu: "Chỉ cập nhật nếu giá trị hiện tại vẫn y hệt như lúc tớ vừa kiểm tra". |
| Vòng lặp CAS (`CAS loop`) | Đọc giá trị, tính toán giá trị mới, thử cập nhật bằng CAS. Nếu có anh nào lanh tay cập nhật trước thì lặp lại từ đầu. |
| Linearization point | Cái chớp mắt duy nhất mà hệ thống công nhận là hành động của bạn đã thành công. |
| Capacity permit | Giống như lấy một cái vé giữ chỗ cho tới khi bạn trả lại. |
| Transition bảo toàn (`conservation transition`) | Đổi trạng thái từ loại này sang loại kia (ví dụ từ pending sang active) nhưng tổng số vé giữ chỗ không đổi. |
| Contention (Tranh chấp) | Cảnh nhiều thread xúm vào giành giật cập nhật cùng một dữ liệu tại một thời điểm. |

## Bối cảnh nghiệp vụ

Giả sử ứng dụng của chúng ta chỉ cho phép tối đa 10 kết nối tới provider:

- Khi có request cần kết nối, hệ thống sẽ xin giữ chỗ bằng `tryReserveCreation()`.
- Nếu được duyệt, worker sẽ bắt đầu quá trình "chào hỏi" (handshake) và vé giữ chỗ này ở trạng thái `pending`.
- Chào hỏi thành công: vé `pending` biến thành `active`.
- Chào hỏi thất bại: trả lại cái vé `pending` đó.
- Xong việc, đóng kết nối: trả lại vé `active`.
- Có một endpoint kiểm tra sức khỏe (health endpoint) sẽ đọc hai con số này ra để báo cáo.

Sức chứa này để bảo vệ tài nguyên của hệ thống (như file descriptor, thread, RAM) trong một máy chủ (JVM). Nó không có tính lưu trữ lâu dài và không đồng bộ giữa nhiều máy chủ khác nhau đâu nhé.

## Trạng thái dùng chung và điểm tranh chấp

| Thành phần | Chi tiết |
| --- | --- |
| Object dùng chung | Singleton `ProviderConnectionBudget` |
| State dùng chung | Các biến đếm `active`, `pending` và giới hạn `limit` |
| Tác nhân | Các request hoặc worker thread tham gia tạo, hoàn tất, hủy hoặc đóng kết nối |
| Chuỗi gây lỗi | `đọc active → đọc pending → check limit → tăng pending` |
| Transition gây lỗi | `giảm pending → tăng active` |
| Điểm tranh chấp | Ngay sau lúc kiểm tra giới hạn nhưng chưa kịp giữ chỗ, hoặc khoảng hở giữa lúc cập nhật hai biến đếm |
| Ranh giới transaction | Không có transaction của database ở đây |
| Phạm vi invariant | Chỉ gói gọn trong một máy chủ/JVM |

Lưu ý là `volatile` và `AtomicInteger` chỉ giúp từng biến đếm hoạt động tốt riêng lẻ. Chúng không có phép thuật để gom nhiều biến đếm thành một cục không thể chia tách (transaction) trong bộ nhớ đâu.

## Phạm vi của tình huống

Trong bài này, chúng ta sẽ xử lý các vấn đề:

- Thao tác kiểm tra-và-cập nhật (check-and-update) đối với một biến đếm sức chứa.
- Quy tắc phụ thuộc vào cả hai biến đếm.
- Cách dùng CAS trên một đối tượng trạng thái không thể thay đổi (immutable state).
- Cách dùng lock và `Semaphore` như các giải pháp thay thế.
- Xử lý các ca khó như timeout, hủy bỏ, số đếm bị âm, hoặc lỡ tay giải phóng 2 lần trong cùng một JVM.

Những thứ chúng ta KHÔNG xử lý ở đây: lưu trữ lâu dài (durable inventory), quota chia sẻ giữa nhiều máy chủ, hoặc chốt sổ sau khi sập hệ thống (process crash). Mấy món đó cần tới database hoặc các công cụ phân tán xịn xò hơn.

## Điều hướng

- [Cách triển khai bị lỗi](broken-code.md)
- [Dòng thời gian tranh chấp và nguyên nhân](analysis.md)
- [Code đã sửa và các phương án lựa chọn](solutions.md)
- [Cách kiểm thử đồng thời](experiments.md)
- Bổ sung kiến thức: [Java Memory Model, volatile và atomic variable](../../concepts/java-memory-model-and-atomicity.md)
- Bổ sung kiến thức: [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong môi trường thực tế

### Hậu quả kỹ thuật

- Tổng số `active + pending` vô tình phình to vượt quá limit.
- Provider phải hứng chịu một rổ handshake nhiều hơn sức tưởng tượng.
- Quá trình chuyển trạng thái vô tình tạo ra một khoảng trống ảo, khiến hệ thống nhận thêm request dù đang đầy.
- Biến đếm bị âm vì mấy callback xử lý lỗi hoặc đóng kết nối bị gọi lặp đi lặp lại.
- Dữ liệu hiển thị (health metric) bị loạn, báo cáo sai trạng thái thực.
- Retry mệt mỏi hoặc bị nghẽn do giành giật lock (lock contention) khi hệ thống gần tới giới hạn.

### Hậu quả nghiệp vụ

- Vượt quota kết nối và bị provider chặn hoặc bóp băng thông (throttling).
- Đứng máy (latency tăng) do bão kết nối.
- Vẫn nhận request mới trong khi nhà đã hết tài nguyên.
- Có thể tự dưng tạo ra sức chứa "ma" do callback chạy đúp.
- Hệ thống tự động mở rộng (autoscaling) đọc dữ liệu sai nên ra quyết định bậy bạ.

## Hướng sửa được khuyến nghị

Nếu bạn thực sự cần biết rõ cả số `active` và `pending` cùng lúc, hãy gom chung chúng lại vào một cục bất biến (immutable object) tên là `BudgetState` và dùng `AtomicReference`. Lúc này, bất kỳ thao tác thay đổi nào cũng dùng CAS trên toàn bộ trạng thái đó. Làm vậy, việc kiểm tra giới hạn và giữ chỗ sẽ dính chặt vào nhau trong một khoảnh khắc duy nhất (linearization point).

Nếu bạn chỉ quan tâm "còn tổng cộng bao nhiêu slot trống", thì cứ `Semaphore` mà táng: giữ chỗ (acquire) lúc bắt đầu, giữ nguyên đó khi kết nối chuyển sang active, và nhả ra (release) khi tạo lỗi hoặc đóng kết nối.

Nếu luật lệ chuyển trạng thái quá rối rắm, mức độ giành giật (contention) cao, hoặc bạn cần sự công bằng (fairness), thì thôi, cứ dùng một cái lock đơn giản ôm trọn các biến `int` thường là ngon nhất. Mục tiêu cuối cùng không phải là cố xài atomic counter cho bằng được, mà là làm sao bảo vệ được quy tắc hệ thống một cách dễ đọc, dễ hiểu.

## Khi nào nên dùng từng giải pháp

- `AtomicReference<BudgetState>`: Dùng khi cần đọc chuẩn xác tình trạng của nhiều biến đếm cùng lúc và các thao tác chuyển đổi diễn ra chớp nhoáng (lock-free).
- Một `AtomicInteger usedSlots`: Khi bạn chỉ màng tới tổng số slot, không rảnh chia chác rạch ròi đâu là active đâu là pending trong bước chặn.
- `Semaphore`: Khi capacity được hiểu là các "tấm vé" có chu trình xin (acquire) và trả (release) rõ ràng.
- `ReentrantLock`: Khi nghiệp vụ phức tạp, cần chờ đợi (condition/fairness), hoặc vòng lặp CAS nhìn rối nách quá, đọc không nổi.
- Database/distributed limiter: Khi sức chứa này phải chia sẻ chung cho nhiều server hoặc cần giữ nguyên vẹn dù app có khởi động lại.
