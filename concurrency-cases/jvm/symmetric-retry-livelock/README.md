# JVM-010 — Livelock do retry đối xứng

## Tóm tắt

Tưởng tượng bạn có hai worker cần lấy hai cái khóa (lock) cùng lúc để làm việc. Anh T1 thử lấy khóa A rồi đến B, còn anh T2 thì thử lấy B rồi đến A. Chuyện gì xảy ra nếu mỗi anh lấy được một khóa? Cả hai đều không lấy được khóa thứ hai, nên rất "lịch sự" nhả khóa đầu tiên ra, chờ một chút rồi thử lại. Khổ nỗi, vì cả hai cùng chờ và cùng thử lại giống hệt nhau, nên cứ va chạm nhau hoài mà chẳng ai làm xong việc cả.

Để hệ thống thực sự "chạy tiến lên" (progress invariant), chúng ta cần:

```text
Mỗi operation phải hoàn tất hoặc trả về một terminal failure trước deadline.
Retry phải có attempt limit, deadline và cơ chế phá vỡ tính đối xứng (symmetry breaking).
Mọi lock của attempt thất bại phải được release trước khi retry.
Không thực hiện business mutation trước khi acquire đủ resource.
```

> **Nói ngắn gọn:** Hệ thống trông có vẻ đang bận rộn, nhưng thực ra chả làm được gì sất. Hai worker cứ nhường nhau mãi mà không ai tới đích.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| livelock | Chương trình vẫn chạy, vẫn đổi state nhưng chả có việc nào xong. Giống như hai người đi ngược chiều cứ nhường đường qua lại hoài. |
| deadlock | Các luồng (thread) đứng chờ nhau tạo thành vòng tròn, cứng ngắc luôn. |
| starvation | Một luồng xui xẻo cứ phải chờ hoài không đến lượt, trong khi các luồng khác vẫn tà tà làm việc. |
| progress guarantee | Lời hứa rằng kiểu gì cái operation này cũng sẽ kết thúc (thành công hoặc báo lỗi) trong thời gian cho phép. |
| bounded retry | Thử lại nhưng có điểm dừng (giới hạn số lần thử hoặc deadline). |
| randomized backoff | Chờ một khoảng thời gian ngẫu nhiên để các luồng không bị "đụng hàng" nhau ở lần thử kế tiếp. |
| symmetry breaking | Làm cách nào đó để các luồng đưa ra quyết định khác nhau khi xảy ra va chạm. |
| retry budget | Tổng quỹ thời gian hoặc số lần cho phép để thử lại. |

## Bối cảnh nghiệp vụ

Ở đây, hai worker đang cố đổi chủ (ownership) của hai in-memory channel cho nhau:

- T1 khoái lấy channel A trước, rồi mới tới B.
- T2 thì ngược lại, thích lấy B trước, A sau.
- Cả hai đều xài `tryLock` kiểu không block để né deadlock.
- Hễ đụng nhau là nhả khóa ra liền rồi thử lại.
- Oái oăm là cả hai có chung một khoảng thời gian chờ (fixed backoff), vô tình làm chúng nó cứ lặp đi lặp lại nhịp điệu va chạm.

## Trạng thái dùng chung và contention point

| Thành phần | Dành cho người mới |
| --- | --- |
| Shared resource | Khóa Lock-A và Lock-B. |
| Actor | Hai worker (hoặc thread xử lý request). |
| Conflict | Mỗi anh cầm một khóa, anh nào cũng móm không lấy được khóa còn lại. |
| Lỗi (Broken reaction) | Cùng nhả khóa, cùng chờ, cùng thử lại một lượt. |
| Trạng thái quan sát được | CPU chạy vèo vèo, log báo retry liên tục nhưng số việc làm xong bằng 0. |
| Phạm vi (Scope) | Nằm trong một máy ảo Java (JVM). Đừng nhầm với database retry storm nhé. |

## Điều hướng

- [Vòng lặp retry bị lỗi (Broken retry loop)](broken-code.md)
- [Phân tích progress failure](analysis.md)
- [Giải pháp ordering và bounded backoff](solutions.md)
- [Các thử nghiệm tất định (Deterministic experiments)](experiments.md)
- [Deadlock và retry an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trên production

- CPU vọt lên tận nóc, retry đếm không xuể nhưng chả có request nào được xử lý xong.
- Request cứ bị ngâm trong vòng lặp retry đến khi hết hạn (timeout).
- Log và metric về conflict tăng đột biến, nhìn báo động đỏ lòm.
- Dù chưa gọi gì tới hệ thống phía sau (downstream) nhưng thread của bạn đã bị chiếm dụng sạch.
- Các request rủ nhau retry đồng loạt tạo ra một làn sóng tranh chấp cực lớn.

## Hướng sửa khuyến nghị

Ngon nhất là ép tụi nó lấy khóa theo một thứ tự duy nhất (deterministic total order) - ai cũng phải bốc khóa A trước chẳng hạn. Đây là cách giải quyết gọn gàng, chắc cú. 
Nếu vì lý do gì đó mà bạn không thể sắp xếp thứ tự, thì bắt buộc phải giới hạn số lần thử lại (bounded attempts), đặt deadline cho operation và thêm chút thời gian chờ ngẫu nhiên (exponential backoff with jitter). Hết "ngân sách" thì quăng lỗi overload hoặc conflict luôn, đừng bắt hệ thống ráng sức dã tràng nữa.

## Khi nào dùng từng phương án

- **Total order:** Dùng khi các khóa nội bộ có ID duy nhất và ổn định.
- **Single owner / Queue:** Dùng khi tài nguyên có thể xếp hàng tuần tự giải quyết.
- **Bounded randomized retry:** Khi va chạm chỉ là tạm thời, việc chạy lại không làm sai dữ liệu và bạn không thể quy định thứ tự lấy khóa.
- **Fail-fast / Admission control:** Khi hệ thống đã quá tải nặng, từ chối luôn cho lẹ.
- **Database retry policy:** Xài khi lỗi ở tầng DB hoặc transaction, không dùng cho vòng lặp lấy lock nội bộ kiểu này.
