# Bài toán JVM-008 — Ách tắc luồng thực thi (Executor saturation), nạn đói tài nguyên (Starvation) và vòng lặp chờ lồng nhau (Nested blocking)

## 1. Tóm tắt vấn đề (Overview)

Thiết lập một bộ điều phối giới hạn (Bounded executor) sở hữu 2 luồng công nhân (worker). Hệ thống phát sinh hai tác vụ cha (parent task) chiếm dụng toàn bộ cả hai luồng này. Mấu chốt sai lầm nằm ở việc mỗi tác vụ cha tiếp tục đẩy (submit) các tác vụ con (child task) ngược trở lại chính bộ điều phối đó và tự đưa mình vào trạng thái đóng băng chờ đợi qua lệnh `Future.get()`. 
Kết quả: Các tác vụ con bị dồn vào hàng đợi (queue) nhưng hoàn toàn không còn bất kỳ luồng công nhân nào rảnh rỗi để xử lý chúng. Hệ thống sụp đổ vào trạng thái **đói tài nguyên (starvation)** mặc dù không hề tồn tại bất kỳ vòng lặp khóa (lock cycle) kinh điển nào.

Quy tắc Bất biến (Business Invariant) yêu cầu nghiêm ngặt:

```text
- Các tác vụ đã được tiếp nhận (Accepted work) bắt buộc phải vươn tới một trong ba trạng thái: hoàn tất, thất bại hoặc bị hủy bỏ, trong giới hạn Hạn mức thời gian vận hành (Operation deadline).
- Tuyệt đối cấm luồng công nhân tự khóa chờ một tác vụ con mà tác vụ con đó chỉ có thể được thi hành trên chính bộ điều phối đã cạn kiệt tài nguyên (exhausted executor) này.
- Hàng đợi (Queue) bắt buộc phải có giới hạn (bounded), và trạng thái quá tải (overload) phải có chính sách từ chối (rejection) / giảm áp (backpressure) minh bạch.
- Tín hiệu hủy tác vụ (Task cancellation) phải được truyền dẫn xuyên suốt xuống các luồng phụ thuộc đang nằm chờ.
```

> **Nguyên tắc kỹ thuật:** Hàng đợi còn không gian trống không bảo chứng cho việc hệ thống còn khả năng tiến bước; các luồng công nhân đang chiếm giữ tài nguyên hệ thống rất có thể chính là nguyên nhân phong tỏa các tác vụ đang mòn mỏi chờ đợi dưới hàng đợi.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Ách tắc (`saturation`) | Mọi luồng công nhân đều bận rộn và sức chứa tiếp nhận tác vụ mới đã cạn kiệt. |
| Đói tài nguyên (`starvation`) | Tác vụ bị tước đoạt quyền cấp phát luồng/tài nguyên, không thể tiếp tục thực thi. |
| Chờ lồng nhau (`nested blocking`) | Luồng công nhân ủy thác tác vụ con rồi tự đóng băng để chờ đồng bộ kết quả từ chính tác vụ con đó. |
| Giảm áp (`backpressure`) | Ép bộ phận sản xuất (Producer) giảm tốc hoặc từ chối dịch vụ thay vì tích tụ hàng đợi vô hạn độ. |
| Chính sách từ chối (`rejection policy`) | Giao thức xử lý khi hệ thống Pool/Queue từ chối tiếp nhận thêm tác vụ. |
| Kiểm soát đầu vào (`admission control`) | Hàng rào định mức số lượng tiến trình được phép xâm nhập vào hệ thống. |
| Độ trễ hàng đợi (`queueing delay`) | Khoảng thời gian chết mà tác vụ phải chịu đựng trong hàng đợi trước khi được thi hành. |
| Ngân sách độ trễ (`latency budget`) | Tổng hạn mức thời gian của toàn bộ yêu cầu, không phải mốc thời gian quá hạn lẻ tẻ của từng bước. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Hệ thống xử lý lệnh làm giàu dữ liệu hàng loạt (Batch enrichment) vận hành tác vụ cha trên `enrichmentExecutor`. Tác vụ cha lại tiếp tục nhồi các tác vụ con định giá (pricing) hoặc hồ sơ (profile) vào cùng một không gian Pool và gọi lệnh `get`. Với giới hạn Pool Size là 2, chỉ cần hai yêu cầu song song là đủ khả năng đóng băng toàn bộ hệ thống luồng công nhân.

| Thành phần | Vai trò và Trạng thái |
| --- | --- |
| Tài nguyên chia sẻ | Vị trí Luồng (Worker slots) và Hàng đợi giới hạn (Bounded queue) |
| Tác vụ Cha (Parent) | Đang vận hành (RUNNING) nhưng đóng băng tại khối chờ Child Future |
| Tác vụ Con (Child) | Đang xếp hàng (QUEUED), khao khát luồng công nhân của cùng hệ thống Pool |
| Vòng lặp bế tắc | Cha giam giữ Luồng chờ Con; Con kẹt dưới Hàng đợi chờ Luồng |
| Phạm vi | Ranh giới một máy ảo JVM; Nạn đói kết nối JDBC thuộc chuyên đề `SPR-007` |

## 4. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 5. Tác Động Tới Hệ Thống (Production Impact)

Hàng loạt Yêu cầu bốc hơi vì quá hạn (Timeout), hàng đợi phình to vô độ, tín hiệu từ chối (Rejection) phản hồi quá muộn, dữ kiện luồng/ngữ cảnh (Context) bị giam hãm, kéo theo bão Thử lại (Retry storm). Cảnh báo: Mức tiêu thụ CPU có thể tụt xuống cực thấp, khiến các bảng giám sát (Dashboard) báo hiệu "CPU bình thường" che đậy hoàn toàn một cuộc khủng hoảng sụp đổ hệ thống (Outage) đang diễn ra.

## 6. Khuyến Nghị Áp Dụng (Best Practices)

- Tiên quyết: Loại bỏ hoàn toàn mô hình đệ trình lồng nhau (Nested submission): Tác vụ cha tự thân thực thi trực tiếp các truy vấn phụ thuộc, hoặc bộ Điều phối ngoài (Orchestrator) đệ trình các tác vụ lá (Leaf task) rồi tự tổng hợp bên ngoài Worker Pool.
- Chỉ tiến hành phân tách Executor khi và chỉ khi Cấu trúc phụ thuộc (Dependency) / Ngân sách tài nguyên (Resource budget) thực sự biệt lập.
- Áp đặt kỷ luật sắt: Dùng hàng đợi giới hạn (Bounded queue), minh bạch chính sách Giảm áp/Từ chối, áp dụng Hạn mức thời gian chung (Deadline) và thiết lập Metric giám sát chặt chẽ các chỉ số: Luồng vận hành (Active) / Hàng đợi (Queue) / Chờ đợi (Wait) / Từ chối (Rejection).
- Khai thác Luồng Ảo (Virtual thread) giúp cắt giảm hao tổn khi luồng bị đóng băng, nhưng KHÔNG tự động tăng cường hạn mức kết nối JDBC, băng thông mạng, hay dung lượng RAM (Heap); Cấu trúc Kiểm soát đầu vào (Admission control) vẫn là bức tường thành bắt buộc.
