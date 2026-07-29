# JVM-010 — Livelock do retry đối xứng

## Tóm tắt

Hai worker cần giữ đồng thời hai local resource lock. Worker T1 thử `A → B`, còn
T2 thử `B → A`. Khi không lấy được lock thứ hai, cả hai “lịch sự” release lock đầu,
back off cùng một khoảng rồi retry. Do hành vi hoàn toàn đối xứng, chúng có thể
liên tục va chạm mà không operation nào hoàn tất.

Case bảo vệ các **progress invariant**:

```text
Mỗi operation phải hoàn tất hoặc trả một terminal failure trước deadline.
Retry phải có attempt limit, deadline và cơ chế phá đối xứng.
Mọi lock của attempt thua phải được release trước retry.
Không thực hiện business mutation trước khi acquire đủ resource.
```

> **Nói ngắn gọn:** hệ thống có hoạt động không đồng nghĩa có tiến triển; hai
>worker có thể liên tục nhường nhau nhưng cùng không tới đích.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| livelock | Actor vẫn chạy và đổi state nhưng không operation nào hoàn tất |
| deadlock | Actor đứng chờ nhau theo cycle và không thể tự chạy tiếp |
| starvation | Một actor lâu dài không được cấp resource dù actor khác có thể tiến |
| progress guarantee | Cam kết operation hoàn tất hoặc kết thúc có giới hạn |
| bounded retry | Retry bị giới hạn bởi số attempt và/hoặc deadline |
| randomized backoff | Thời gian chờ có jitter để actor không lặp cùng nhịp |
| symmetry breaking | Làm actor đưa ra quyết định khác nhau khi conflict |
| retry budget | Phần deadline/attempt được phép tiêu cho retry |

## Bối cảnh nghiệp vụ

Hai worker cục bộ cần hoán đổi ownership của hai in-memory channel:

- T1 ưu tiên channel A rồi B;
- T2 ưu tiên channel B rồi A;
- cả hai dùng non-blocking `tryLock` để tránh deadlock;
- khi conflict, chúng release ngay và retry;
- fixed backoff giống nhau vô tình đồng bộ các lần thử tiếp theo.

## Trạng thái dùng chung và contention point

| Thành phần | Giá trị |
| --- | --- |
| Shared resource | Lock-A và Lock-B |
| Actor | Hai worker/request thread |
| Conflict | Mỗi actor giữ một lock, không lấy được lock còn lại |
| Broken reaction | Cùng release, cùng delay, cùng retry |
| Observable state | Retry/CPU tăng nhưng completed operation bằng 0 |
| Scope | Một JVM; database retry storm thuộc `LOCK-002`/`DB-009` |

## Điều hướng

- [Broken retry loop](broken-code.md)
- [Phân tích progress failure](analysis.md)
- [Giải pháp ordering và bounded backoff](solutions.md)
- [Deterministic experiments](experiments.md)
- [Deadlock và retry an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả production

- CPU/retry count cao nhưng throughput bằng 0 hoặc rất thấp;
- request dùng hết deadline trong local retry;
- log/metric conflict tăng mạnh;
- downstream chưa bị gọi nhưng thread capacity vẫn bị tiêu thụ;
- retry đồng bộ giữa nhiều request tạo contention wave.

## Hướng sửa khuyến nghị

Nếu có thể, dùng deterministic total order để mọi worker chọn cùng lock trước;
đây là symmetry breaking có thể chứng minh. Khi conflict vốn không thể order,
dùng bounded attempts, operation deadline và exponential backoff có jitter. Sau
khi hết budget, trả overload/conflict thay vì retry vô hạn.

## Khi nào dùng từng phương án

- Total order: nhiều local lock có stable unique key.
- Single owner/queue: resource có thể được serialize theo ownership.
- Bounded randomized retry: conflict tạm thời, operation idempotent và ordering
  không biểu diễn được.
- Fail-fast/admission control: contention cho thấy hệ thống đã saturated.
- Database retry policy: conflict thuộc transaction/database layer, không dùng
  local loop của case này.
