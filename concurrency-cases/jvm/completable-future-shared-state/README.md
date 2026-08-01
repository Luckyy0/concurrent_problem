# Bài toán JVM-009 — Đứt gãy luồng chia sẻ trên nền tảng CompletableFuture (Shared Aggregation Race)

## 1. Tóm tắt vấn đề (Overview)

Kiến trúc hệ thống triển khai nhiều giai đoạn xử lý luồng (stage) hoàn tất độc lập trên các luồng tác vụ (thread) khác nhau, sau đó đồng loạt triệu gọi phương thức `add` vào chung một cấu trúc `ArrayList`. Trong kịch bản này, một giai đoạn có thể phát sinh ngoại lệ (fail) trong khi các luồng khác vẫn tiếp tục ngầm định thay đổi cấu trúc mảng (mutate), dẫn đến hậu quả phía gọi (caller) thu thập một mảnh dữ liệu dang dở (partial result).

Quy tắc Bất biến (Business Invariant) yêu cầu nghiêm ngặt: 
- Mỗi tín hiệu đầu vào phải thu về chính xác một kết quả đầu ra tương ứng;
- Trạng thái chung cuộc "Thành công" (Final success) chỉ được phép phát hành (publish) sau khi 100% các giai đoạn hoàn thành viên mãn;
- Trạng thái Lỗi/Hủy bỏ (Failure/Cancel) cấm tuyệt đối việc hoàn trả một danh sách mảnh vỡ có khả năng biến đổi (mutable partial list);
- Cấu trúc kết quả đầu ra phải tuân thủ nghiêm ngặt trình tự đầu vào (input order) nếu đặc tả API có cam kết.

> **Nguyên tắc kỹ thuật:** Đối tượng Future đóng vai trò như một khoang chứa trạng thái hoàn tất (completion container), KHÔNG phải là một cấu trúc Khóa đồng bộ (Lock) dành cho một đối tượng chia sẻ bị nhiều lệnh gọi (callback) thao túng.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Giai đoạn hoàn tất (`completion stage`) | Phân đoạn mã nguồn được kích hoạt thi hành khi một khối future nội tại đi đến điểm kết thúc. |
| Hợp nhất giá trị (`value composition`) | Kỹ thuật cấu trúc chuỗi giá trị trả về trực tiếp, thay vì ngầm thay đổi một trạng thái nằm ngoài luồng luân chuyển (pipeline). |
| Rào cản hợp nhất (`allOf`) | Điểm kiểm soát đồng bộ chỉ vượt qua khi toàn bộ các future con (children) khép lại vòng đời. |
| Kết thúc có ngoại lệ (`exceptional completion`) | Trạng thái vòng đời Future sụp đổ kèm theo một ngoại lệ hệ thống. |
| Cô lập sở hữu (`confinement`) | Đặc quyền thay đổi cấu trúc tích lũy (accumulator) được giao phó cho duy nhất một luồng điều phối (coordinator). |
| Khả kiến phân mảnh (`partial visibility`) | Lỗ hổng cho phép hệ thống Caller quan sát một trạng thái dữ liệu đang trong quá trình hình thành dở dang. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Hệ thống thiết kế một chu trình quạt rẽ nhánh (Fan-out profile) xử lý phân bổ làm giàu dữ liệu (Batch enrichment), bao hàm các truy vấn giá (Price) và định lường rủi ro (Risk). Tập hợp các Callback chia sẻ chung một cấu trúc List; điểm mấu chốt là rào cản `allOf`, luồng Callback báo lỗi và tín hiệu Hủy bỏ (Cancellation) phát sinh cạnh tranh trực diện trên chính khối tích lũy (accumulator) đó.

## 4. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 5. Tác Động Tới Hệ Thống và Hướng Khắc Phục (Impact & Mitigation)

**Hậu Quả:**
- Thất thoát hoặc nhân bản kết quả xử lý.
- Xáo trộn hoàn toàn trình tự (Order).
- Phản hồi dữ liệu phân mảnh (Partial response) chứa rác.
- Tác vụ nền (Background task) tiếp tục ngốn tài nguyên vô định sau khi hạn mức thời gian (Timeout) đã đóng.

**Chiến Lược Khắc Phục:**
1. Ràng buộc mỗi Future phải tự hoàn trả giá trị độc lập.
2. Thiết lập sau rào cản `allOf`, một Điều phối viên (Coordinator) duy nhất đọc tuần tự hệ thống Future theo trình tự đầu vào gốc và kết xuất một cấu trúc danh sách Bất Biến (Immutable list).
3. Đề ra quy chuẩn minh bạch, rõ ràng cho các chính sách Thời Hạn (Deadline), Hủy Bỏ (Cancel), và Xử lý Lỗi (Failure).
