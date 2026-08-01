# JVM-010 — Livelock do retry đối xứng

## Tóm tắt

Hai worker cần giữ đồng thời hai local resource lock. Worker T1 thử `A → B`, còn T2 thử `B → A`. Khi không lấy được lock thứ hai, cả hai "lịch sự" release lock đầu tiên, chờ (backoff) cùng một khoảng thời gian rồi retry. Do hành vi hoàn toàn đối xứng, chúng có thể liên tục va chạm mà không có operation nào hoàn tất.

Tình huống bảo vệ các **progress invariant**:

```text
Mỗi operation phải hoàn tất hoặc trả về một terminal failure trước deadline.
Retry phải có attempt limit, deadline và cơ chế phá vỡ tính đối xứng (symmetry breaking).
Mọi lock của attempt thất bại phải được release trước khi retry.
Không thực hiện business mutation trước khi acquire đủ resource.
```

> **Nói ngắn gọn:** hệ thống có hoạt động không đồng nghĩa với việc có tiến triển; hai worker có thể liên tục nhường nhau nhưng cùng không tới đích.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| livelock | Actor vẫn chạy và chuyển state nhưng không có operation nào hoàn tất |
| deadlock | Các actor đứng chờ nhau theo vòng (cycle) và không thể tiếp tục chạy |
| starvation | Một actor trong thời gian dài không được cấp resource dù các actor khác có thể tạo ra tiến triển |
| progress guarantee | Cam kết operation sẽ hoàn tất hoặc kết thúc trong một giới hạn nhất định |
| bounded retry | Retry bị giới hạn bởi số attempt và/hoặc deadline |
| randomized backoff | Thời gian chờ có độ trễ ngẫu nhiên (jitter) để actor không lặp lại cùng nhịp |
| symmetry breaking | Làm cho actor đưa ra các quyết định khác nhau khi xảy ra conflict |
| retry budget | Phần deadline hoặc số lượng attempt được phép sử dụng cho retry |

## Bối cảnh nghiệp vụ

Hai worker cục bộ cần hoán đổi quyền sở hữu (ownership) của hai in-memory channel:

- T1 ưu tiên channel A rồi đến B;
- T2 ưu tiên channel B rồi đến A;
- cả hai đều dùng `tryLock` ở chế độ non-blocking để tránh deadlock;
- khi xảy ra conflict, chúng release ngay lập tức và retry;
- thời gian chờ cố định (fixed backoff) giống nhau vô tình đồng bộ hoá các lần thử tiếp theo.

## Trạng thái dùng chung và contention point

| Thành phần | Giá trị |
| --- | --- |
| Shared resource | Lock-A và Lock-B |
| Actor | Hai worker hoặc request thread |
| Conflict | Mỗi actor giữ một lock, không lấy được lock còn lại |
| Lỗi (Broken reaction) | Cùng release, cùng delay, cùng retry |
| Trạng thái quan sát được (Observable state) | Số lần retry và mức sử dụng CPU tăng nhưng số operation hoàn tất bằng 0 |
| Phạm vi (Scope) | Một JVM; database retry storm thuộc về `LOCK-002` hoặc `DB-009` |

## Điều hướng

- [Vòng lặp retry bị lỗi (Broken retry loop)](broken-code.md)
- [Phân tích progress failure](analysis.md)
- [Giải pháp ordering và bounded backoff](solutions.md)
- [Các thử nghiệm tất định (Deterministic experiments)](experiments.md)
- [Deadlock và retry an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trên production

- Mức sử dụng CPU và số lần retry cao nhưng throughput bằng 0 hoặc rất thấp;
- request sử dụng hết deadline trong vòng lặp retry cục bộ (local retry);
- số lượng log và metric liên quan đến conflict tăng mạnh;
- hệ thống downstream chưa bị gọi nhưng thread capacity vẫn bị tiêu thụ;
- retry đồng bộ giữa nhiều request tạo ra làn sóng tranh chấp (contention wave).

## Hướng sửa khuyến nghị

Nếu có thể, hãy dùng một deterministic total order để mọi worker chọn cùng một lock trước; đây là một kỹ thuật symmetry breaking có thể chứng minh được tính đúng đắn. Khi conflict vốn không thể phân định thứ tự (order), hãy dùng bounded attempts, operation deadline và exponential backoff có jitter. Sau khi sử dụng hết ngân sách (budget), trả về lỗi overload hoặc conflict thay vì retry vô hạn.

## Khi nào dùng từng phương án

- Total order: nhiều local lock có khoá định danh duy nhất ổn định (stable unique key).
- Single owner hoặc queue: resource có thể được tuần tự hoá (serialize) theo ownership.
- Bounded randomized retry: conflict mang tính tạm thời, operation có tính idempotent và ordering không thể biểu diễn được.
- Fail-fast hoặc admission control: mức độ contention cho thấy hệ thống đã quá tải (saturated).
- Database retry policy: conflict thuộc tầng transaction hoặc database, không dùng local loop của tình huống này.
