# Bài toán JVM-006 — Cấu trúc `synchronized` và `ReentrantLock` Đặt Sai Phạm Vi

## 1. Tóm tắt vấn đề (Overview)

Một dịch vụ Spring (Spring service) kiến tạo tệp đối soát (settlement artifact) và ghi vào kho đối tượng chia sẻ (shared object store) theo định danh `artifactKey`. Quy định nghiệp vụ: Hai yêu cầu cùng khóa không được phép kết xuất (render) và ghi đè cùng lúc. Nhà phát triển đã áp dụng từ khóa `synchronized` hoặc cấu trúc `ReentrantLock` nhằm mục đích bảo vệ; tuy nhiên, lỗ hổng nảy sinh từ việc: (1) khởi tạo Khóa (lock) mới trong mỗi lần triệu gọi, (2) khóa nhầm đối tượng tham chiếu, hoặc (3) ảo tưởng rằng một Khóa Nội Bộ (local lock) của một Nút (node) có thể che chắn cho các Nút khác trên mạng phân tán.

Khóa chỉ phát huy tác dụng Trừ Trừ Lẫn Nhau (Mutual exclusion) khi và chỉ khi các luồng cạnh tranh cùng chĩa vào **Một Khóa Đồng Nhất (Lock identity)** và Vùng Tới Hạn (Critical section) phải bao bọc toàn vẹn khối Trạng Thái Chia Sẻ (Shared state). Hai đối tượng khác biệt trên vùng nhớ không bỗng dưng biến thành một Khóa (Monitor) chỉ vì hàm `equals()` trả về `true`; tương tự, hai cỗ máy JVM hoàn toàn không có khả năng chia sẻ một Monitor hoặc một `ReentrantLock` nội bộ.

Bài toán thiết lập ranh giới bảo vệ các **Quy tắc Bất biến (Business Invariant)**:

```text
- Trong một không gian JVM, chỉ duy nhất một Quy trình (Workflow) thuộc cùng một artifactKey được phép xuyên qua Vùng Tới Hạn.
- Khâu Kiểm Tra Tồn Tại, Kết Xuất và Công Bố bắt buộc phải bị nhốt trong cùng một Ranh Giới Phân Xử (Coordination boundary).
- Khóa bắt buộc phải được Phóng Thích (Release) trong mọi trạng thái: Thành công, Ngoại lệ, Quá hạn hay Gián đoạn.
- Nghiêm cấm tuyên bố Khóa Cục Bộ có năng lực bảo vệ tính Độc Nhất (Uniqueness) trên toàn mạng lưới Đa Máy Chủ.
```

> **Nguyên tắc kỹ thuật:** Gắn từ khóa `synchronized` là chưa đủ; 100% tác nhân phải chung tay bóp chung một "Ổ Khóa", và Ổ Khóa đó phải phủ trọn toàn bộ phạm vi sinh tồn của Trạng Thái Chia Sẻ.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Định danh khóa (`monitor identity`) | Trỏ tham chiếu (Reference) vật lý duy nhất của đối tượng được `synchronized` sử dụng. |
| Phạm vi khóa (`lock scope`) | Tập hợp các luồng và vùng trạng thái thực sự nằm dưới tầm bảo vệ của Khóa. |
| Vùng tới hạn (`critical section`) | Vùng mã nguồn cấm đa luồng xâm phạm cùng lúc, nhằm duy trì Quy tắc Bất biến. |
| Khóa theo định danh (`keyed lock`) | Từng Khóa Nghiệp vụ luôn được ánh xạ chuẩn xác tới một Ổ Khóa Tương Ứng. |
| Khóa phân dải (`striped lock`) | Chia số lượng Khóa thành các Dải cố định; Khóa nghiệp vụ bị băm (hash) vào một dải nhằm giới hạn Bộ nhớ. |
| Loại trừ lẫn nhau (`mutual exclusion`) | Tại một sát na, độc tôn một tác nhân được quyền chiếm Khóa. |
| Khóa cục bộ (`local lock`) | Khóa chỉ có vòng đời nội trong Vùng nhớ Heap của một Máy ảo JVM. |
| Khởi tạo có điều kiện (`conditional create`) | Thao tác tại Kho Lưu Trữ uy quyền: Chỉ ghi chèn nếu Khóa chưa từng tồn tại. |
| Thẻ chắn (`fencing token`) | Dấu mộc tuần tự giúp Kho lưu trữ từ chối các Luồng Ghi cũ kỹ sau khi Phiên (Lease) đã hết hạn. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Ứng dụng vận hành quy trình tạo tệp đối soát cuối ngày:
- Tuyến Yêu Cầu (Request) hoặc Tuyến Lịch Trình (Scheduler) gọi lệnh `generate("settlement/2026-07-29.csv")`.
- Dịch vụ xác minh tệp đối soát đã tồn tại hay chưa.
- Nếu vắng mặt, Bộ Kết Xuất (Renderer) đọc dữ liệu, nhào nặn thành byte rồi đẩy vào Kho Đối Tượng (Object store).
- Yêu cầu trùng lặp phải nhận về dữ liệu đã kết xuất thay vì tự chạy lại tiến trình.
- Kiến trúc triển khai bao gồm Đa Máy Chủ (Multi-instance).

Tiến trình Kết xuất tiêu hao kịch liệt CPU/DB; tiến trình Công Bố mang theo độ trễ Mạng (Remote I/O). Việc Kết xuất dư thừa không những làm vỡ Tải, mà hai Tuyến Ghi hoàn toàn có nguy cơ ghi đè Siêu Dữ Liệu (Metadata) trái ngược nhau nếu Bản Chụp Đầu Vào (Input snapshot) có sai lệch.

## 4. Các Thực thể và Trạng thái chia sẻ (Shared state & Contention)

| Thành phần | Vai trò và Trạng thái |
| --- | --- |
| Tài nguyên Logic | Một định danh `artifactKey` |
| Chia sẻ Nội bộ (Local) | Bộ Kết Xuất, Bộ Đệm (Cache) và Sổ Đăng Ký Khóa |
| Chia sẻ Phân Tán (Global) | Kho Đối Tượng (Object store) |
| Chuỗi thao tác khuyết tật | `exists(key) → render(key) → put(key, bytes)` |
| Chủ Thể Cạnh Tranh (Actor) | Yêu cầu, Lịch trình, hoặc Luồng Thử lại (Retry) |
| Vùng Tới Hạn Nội Bộ | Trọn bộ Quy trình Phán Quyết và Công Bố cho cùng Khóa |
| Ranh Giới Uy Quyền (Authoritative) | Tác vụ Store/DB (Ràng buộc Độc Nhất Phân Tán) |

Việc khởi tạo Khóa mới cho mỗi lần triệu gọi tự triệt tiêu Mọi Ranh Giới Cạnh Tranh. Khóa Toàn Cục (Global lock) đúng Định danh nhưng lại thắt cổ chai mọi Khóa Nghiệp vụ khác. Khóa Phân Dải (Striped lock) là nghệ thuật cân bằng giữa Tính Đúng Đắn (Correctness) và Năng lực Đồng Thời (Concurrency) nội trong một JVM.

## 5. Giới hạn Áp dụng (Out of Scope)

Chuyên đề tập trung mổ xẻ: 
- Sự khác biệt giữa Định danh Khóa (Monitor identity) và So sánh Giá trị (`equals`).
- Tham chiếu Khóa Thuộc Tính (Field lock) đối đầu Tham chiếu Biến Cục Bộ.
- Thảm họa Vùng Tới Hạn hẹp.
- Ma trận Khóa Phân dải và Vòng đời của Sổ Đăng Ký Khóa.
- Luật Phóng Thích Khóa đối diện Timeout, Interruption, Exception.
- Ranh giới tuyệt đối của sự Phân Xử Nội Bộ JVM.

Chuyên đề KHÔNG xây dựng Giao thức Thuê Phân Tán (Distributed lease / Fencing protocol) — Chuyên mục đó thuộc về `DIST-001`. Cơ chế Ràng buộc Độc Nhất DB (Unique constraint) hoặc Ghi Có Điều Kiện Kho Đối Tượng (Conditional write) chỉ được nhắc đến như một Nền Tảng Uy Quyền Thay Thế.

## 6. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Bào mòn CPU cho việc Kết Xuất Nhân Bản; Khủng hoảng Băng thông Ghi Đè mạng.
- Định luật Ghi-Sau-Cùng (Last-writer-wins) che đậy các Bản Chụp giá trị trước đó.
- Khóa Toàn Cục gây hiện tượng Kẹt Xe Đầu Ngõ (Head-of-line blocking) cho các Khóa Độc lập.
- Sổ Đăng Ký phình to vỡ Bộ nhớ, hoặc hành vi Xóa Khóa (Remove) quá sớm đẻ ra 2 Khóa song song cho 1 Trọng Tâm.
- Luồng chết cứng vô tận vì thiếu vắng Quản Trị Timeout.
- Ngoại Lệ phá vỡ Luồng và Cầm Tù Khóa vĩnh viễn (Vắng bóng khối `finally`).

### Hệ Quả Nghiệp Vụ
- Tệp Đối Soát (Settlement) vỡ vụn, lệch pha so với Bản Chụp Kỳ Vọng.
- Khối Phân Luồng Hạ Nguồn (Downstream) nhận bão Tín Hiệu "Tệp Đã Sẵn Sàng" trùng lặp.
- Đánh gục Database/Object Store ngay giữa đỉnh Tải Lô (Batch window).
- Hệ thống Mượt mà ở Môi trường Dev (1 Node), Sụp đổ Tan nát trên Production (Đa Node).

## 8. Khuyến Nghị Phân Lớp Áp Dụng (Best Practices)

Nội trong Mạch JVM:
- Cắm chốt Private Final Monitor hoặc `ReentrantLock` Field để bao bọc Khóa Diện Rộng (Coarse-grained).
- Triển khai Khóa Phân Dải (Striped locks) khi không gian Khóa khổng lồ và khao khát Năng Lực Xử Lý Song Song.
- Áp dụng Bản Đồ Khóa Cố Định (Stable Lock Map) với điều kiện Quản trị Vòng Đời Rắn Chắc.
- Vùng Tới Hạn buộc phải ôm trọn khâu Kiểm Tra và Công Bố nếu muốn duy trì Quy tắc Cục Bộ.

Xuyên suốt Mạng Đa Nút (Multi-node):
- Chuyển giao Quyền Phán Xử Xung Đột cho Kho Uy Quyền (Authoritative Resource): Ghi có điều kiện (Conditional create), Ràng Buộc Độc Nhất Database (Unique constraint) hoặc Thuê Phân Tán Cắm Thẻ (Distributed lease with fencing). 
- Khóa Cục Bộ chỉ có vai trò Trợ Thủ (Giảm Rác Xử Lý) — Nó vĩnh viễn không phải là Ranh Giới Tính Đúng Đắn của Toàn Hệ Thống.
