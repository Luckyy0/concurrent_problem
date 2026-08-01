# Bài toán JVM-009 — Đứt gãy luồng chia sẻ trên nền tảng CompletableFuture (Shared Aggregation Race)

## 1. Tóm tắt vấn đề (Overview)

Chào bạn, để mình giải thích vấn đề này nhé. Khi hệ thống của chúng ta chạy nhiều luồng (thread) để xử lý dữ liệu cùng lúc, các luồng này lại cùng gọi hàm `add` để nhét kết quả vào chung một cái `ArrayList`. Rủi ro ở đây là gì? Nếu một luồng bị lỗi, các luồng khác vẫn cứ hồn nhiên nhét data vào mảng đó. Hậu quả là người gọi hàm (caller) sẽ nhận về một mớ data thiếu hụt, chắp vá.

Nguyên tắc nghiệp vụ của mình yêu cầu rất chặt chẽ:
- Có input thì phải có output tương ứng.
- Chỉ báo "Thành công" khi 100% các luồng đã xử lý xong xuôi.
- Nếu có lỗi hay bị hủy, tuyệt đối không trả về cái mảng dở dang kia.
- Dữ liệu trả về phải đúng thứ tự như lúc đưa vào (nếu API có hứa hẹn điều đó).

> **Nhớ nè:** Thằng `Future` sinh ra để chứa kết quả hoàn thành, chứ nó KHÔNG phải là cái khóa (lock) để bảo vệ dữ liệu dùng chung (shared object) khi bị nhiều hàm callback chọc vào đâu nhé!

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh này |
| --- | --- |
| Giai đoạn hoàn tất (`completion stage`) | Đoạn code sẽ chạy sau khi một `Future` làm xong việc của nó. |
| Hợp nhất giá trị (`value composition`) | Gom các giá trị trả về lại với nhau thay vì đi sửa data của một biến nằm tít bên ngoài. |
| Rào cản hợp nhất (`allOf`) | Cái chốt chặn bắt tất cả các `Future` con phải chạy xong thì mới được đi tiếp. |
| Kết thúc có ngoại lệ (`exceptional completion`) | Khi `Future` tạch và quăng ra lỗi. |
| Cô lập sở hữu (`confinement`) | Chỉ giao quyền sửa đổi kết quả cho đúng một luồng quản lý (coordinator) thôi. |
| Khả kiến phân mảnh (`partial visibility`) | Lỗi do code để lộ ra trạng thái dữ liệu đang build dở dang cho bên ngoài thấy. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Tưởng tượng hệ thống của mình phải tẻ nhánh ra (fan-out) để làm giàu dữ liệu, ví dụ như đi lấy giá (Price) và check rủi ro (Risk) cùng lúc. Khi làm xong, các callback lại chui vào dùng chung một cái List. Cái dở ở đây là cái rào `allOf`, luồng báo lỗi và luồng báo hủy lại tranh nhau nhảy vào thay đổi đúng cái List chứa kết quả đó. Rất dễ toang!

## 4. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 5. Tác Động Tới Hệ Thống và Hướng Khắc Phục (Impact & Mitigation)

**Hậu quả nếu để kệ:**
- Dữ liệu rớt mất hoặc bị duplicate.
- Thứ tự lung tung beng hết cả lên.
- Trả về cái đống data rác rưởi, chắp vá.
- Task chạy ngầm cứ chạy quài tốn tài nguyên dù đã hết giờ (timeout).

**Cách mình fix:**
1. Bắt mỗi `Future` phải tự trả về giá trị của riêng nó.
2. Đặt một anh "Điều phối viên" (Coordinator) đứng chờ sau cái rào `allOf`. Anh này sẽ đọc lần lượt các `Future` theo thứ tự ban đầu và đóng gói thành một cái List không cho sửa (Immutable list).
3. Đặt ra luật rõ ràng: bao lâu thì timeout, khi nào thì hủy (cancel), và lỗi thì xử lý ra sao.
