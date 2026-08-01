# Bài toán JVM-008 — Ách tắc luồng thực thi (Executor saturation), nạn đói tài nguyên (Starvation) và vòng lặp chờ lồng nhau (Nested blocking)

## 1. Tóm tắt vấn đề (Overview)

Tưởng tượng em có một đội 2 công nhân (Bounded executor với 2 threads). Có 2 công việc lớn (tác vụ cha) được giao cho họ, thế là cả 2 anh công nhân đều bận. Nhưng ngặt nỗi, mỗi anh lại tự tạo thêm một việc nhỏ (tác vụ con) quăng lại vào hàng đợi của chính đội mình và đứng im đợi kết quả (`Future.get()`).
Kết quả là gì? Các việc nhỏ thì nằm chờ dưới hàng đợi, còn 2 anh công nhân thì cứ đứng ngóng việc nhỏ hoàn thành. Chẳng ai chịu nhúc nhích! Hệ thống bị **đói tài nguyên (starvation)** dù không hề bị dính deadlock theo kiểu truyền thống.

Một số quy tắc bắt buộc (Business Invariant):

```text
- Các tác vụ đã nhận thì phải xong, fail hoặc bị hủy trong một khoảng thời gian nhất định (deadline).
- Tuyệt đối không cho phép công nhân tự khóa mình lại để chờ một tác vụ con nằm trong chính cái hàng đợi đang hết chỗ đó.
- Hàng đợi phải có giới hạn. Nếu quá tải thì phải có chính sách từ chối hoặc ép giảm tốc độ (backpressure) rõ ràng.
- Hủy tác vụ thì phải báo cho các luồng bên dưới biết để dừng theo.
```

> **Mẹo nhỏ:** Hàng đợi còn chỗ không có nghĩa là mọi thứ đang chạy ổn. Nhiều khi mấy anh công nhân đang ngậm tài nguyên mới chính là nguyên nhân chặn đứng các tác vụ khác đó.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Hiểu đơn giản là |
| --- | --- |
| Ách tắc (`saturation`) | Mọi người đều bận, không nhận thêm việc được nữa. |
| Đói tài nguyên (`starvation`) | Tác vụ bị "bỏ đói", không được cấp luồng chạy nên cứ kẹt mãi. |
| Chờ lồng nhau (`nested blocking`) | Anh công nhân giao việc xong tự đứng im chờ kết quả từ chính việc đó. |
| Giảm áp (`backpressure`) | Bắt mấy ông đẩy việc chậm lại bớt, hoặc từ chối luôn chứ không ôm rơm nặng bụng. |
| Chính sách từ chối (`rejection policy`) | Cách xử lý khi Pool/Queue quyết định nói "Không" với tác vụ mới. |
| Kiểm soát đầu vào (`admission control`) | Hàng rào chặn cửa, giới hạn số lượng request được vào hệ thống. |
| Độ trễ hàng đợi (`queueing delay`) | Thời gian tác vụ phải "ngồi chơi xơi nước" trong hàng đợi. |
| Ngân sách độ trễ (`latency budget`) | Tổng thời gian tối đa cho toàn bộ quá trình, không phải tính lẻ tẻ từng bước. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Hệ thống cần làm giàu dữ liệu hàng loạt. Tác vụ cha chạy trên `enrichmentExecutor` rồi đẩy các tác vụ con như định giá (pricing) vào chung một Pool và gọi `get`. Vì Pool chỉ có 2 luồng, chỉ cần 2 request tới cùng lúc là hệ thống "đứng hình" ngay lập tức.

| Thành phần | Nó làm gì và đang ở đâu? |
| --- | --- |
| Tài nguyên chia sẻ | Luồng công nhân (Worker slots) và Hàng đợi (Bounded queue). |
| Tác vụ Cha (Parent) | Đang chạy nhưng bị treo cứng ngắc vì chờ Child Future. |
| Tác vụ Con (Child) | Đang xếp hàng dài chờ đến lượt mình được chạy. |
| Vòng lặp bế tắc | Cha ôm luồng chờ Con; Con nằm hàng đợi chờ Luồng. Deadlock! |
| Phạm vi | Trong phạm vi một máy ảo JVM. (Lỗi tương tự với JDBC thì xem `SPR-007`). |

## 4. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 5. Tác Động Tới Hệ Thống (Production Impact)

Hàng tá request sẽ chết vì timeout, hàng đợi thì phình to, báo lỗi quá muộn khiến hệ thống retry liên tục tạo thành "cơn bão". Nguy hiểm nhất là CPU lúc này có thể tụt xuống thấp (vì các luồng đang ngủ chờ), khiến monitoring báo "CPU bình thường" nhưng thực chất hệ thống đã tèo (Outage).

## 6. Khuyến Nghị Áp Dụng (Best Practices)

- **Cấm kỵ:** Bỏ ngay kiểu ném việc lồng nhau (Nested submission). Tác vụ cha nên tự gọi thẳng hàm xử lý, hoặc dùng một hệ thống điều phối (Orchestrator) riêng lẻ.
- Tách riêng Pool chỉ khi thật sự cần thiết về mặt logic và tài nguyên.
- Phải dùng hàng đợi có giới hạn (Bounded queue) và rõ ràng khi nào thì từ chối (Rejection). Cài đặt timeout và giám sát kỹ các thông số: Active / Queue / Wait / Rejection.
- Virtual thread (Luồng ảo) có thể giúp giảm chi phí đóng băng luồng, nhưng nó KHÔNG tự sinh ra thêm RAM hay Connection DB đâu nhé. Vẫn phải kiểm soát đầu vào đàng hoàng!
