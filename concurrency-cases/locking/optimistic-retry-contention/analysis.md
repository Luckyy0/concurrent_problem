# Phân tích hiện tượng khuếch đại tải và tiến độ hệ thống

## 1. Trạng thái ban đầu

Giả định hệ thống có một ví mang mã `77`: Số dư hiện tại là `100` điểm, phiên bản `10`.
Có 8 lệnh đồng thời yêu cầu cộng `10` điểm vào ví này.

## 2. Diễn biến xen kẽ

| Giai đoạn | Luồng thành công (1 thread) | Các luồng thất bại (7 thread) |
| --- | --- | --- |
| Truy xuất | Nhận dữ liệu `100` điểm, phiên bản `v10` | Nhận dữ liệu `100` điểm, phiên bản `v10` |
| Đồng bộ | Thay đổi `1` bản ghi, cập nhật thành `v11` | Lỗi do phiên bản dữ liệu database đã là `v11` (Thay đổi `0` dòng) |
| Kích hoạt thử lại | Hoàn tất commit | Đồng loạt bắt đầu transaction mới, truy xuất `v11` |
| Đồng bộ lần 2 | 1 thread thành công cập nhật lên `v12` | 6 thread tiếp tục gặp xung đột và thất bại |

Phân tích trên cho thấy, nếu các thread thất bại thực hiện thử lại cùng một lúc, số lượng lần xử lý hệ thống phải đáp ứng sẽ lớn hơn theo cấp số nhân so với lượng yêu cầu ban đầu. Cơ chế phiên bản bảo đảm tính toàn vẹn dữ liệu, nhưng có thể làm suy giảm nghiêm trọng hiệu suất tổng thể và làm tăng đột biến độ trễ.

> **Ghi chú quan trọng:** Khóa lạc quan (optimistic lock) là cơ chế bảo vệ tính đúng đắn của dữ liệu hiệu quả, nhưng nhạy cảm với tình trạng tranh chấp cao. Việc thiết kế chính sách thử lại là yếu tố quyết định khả năng chịu tải của hệ thống.

## 3. Tính an toàn, khả năng hoạt động và tính công bằng

- **Tính an toàn:** Đảm bảo hệ thống không bỏ sót hay tính trùng giá trị. Số phiên bản luôn phải đồng biến với số lần commit thành công.
- **Khả năng hoạt động:** Mỗi yêu cầu phải có một kết quả xác định trong khoảng thời gian hữu hạn: Thành công, bị từ chối do trùng lặp, hoặc lỗi do cạn kiệt nỗ lực.
- **Tính công bằng:** Hệ thống không cam kết ưu tiên thời gian truy cập. Một thread có thể liên tục gặp xung đột và không thể hoàn thành cho đến khi vượt quá giới hạn thử lại (suy kiệt).

Chỉ đo lường tỷ lệ thành công là không đủ. Trong kiểm thử và vận hành, bắt buộc phải giám sát số lần thử và tỷ lệ cạn kiệt.

## 4. Bắt buộc sử dụng ranh giới transaction mới

Khi ngoại lệ khóa lạc quan xảy ra:

- Transaction hiện tại đã bị database đánh dấu yêu cầu hoàn tác (rollback-only).
- Bộ đệm của ORM chứa trạng thái đối tượng không còn hiệu lực.
- Lệnh chèn dữ liệu lịch sử chống trùng lặp của vòng lặp cũng đã bị vô hiệu hóa.
- Quá trình thử lại yêu cầu khởi tạo lại bản ghi để tiếp nhận trạng thái mới nhất từ database.

Bộ điều phối bắt buộc phải nằm ngoài phạm vi `@Transactional` để bắt lỗi một cách an toàn sau khi proxy đã xử lý rollback. Quá trình tạm dừng phải diễn ra bên ngoài ranh giới transaction nhằm tránh lãng phí connection. Trong transaction mới, hệ thống phải thực hiện lại khâu kiểm tra tính lũy đẳng trước khi tải lại dữ liệu ví.

## 5. Mô hình khuếch đại tải

Nếu hệ thống nhận `R` yêu cầu và cấu hình số lần thử tối đa là `A`, khối lượng công việc trên database sẽ xấp xỉ `R × A`, chưa bao gồm các lệnh `SELECT` bổ sung.

Tăng giới hạn thử lại đồng nghĩa với việc mở rộng tải cho database. Thiết lập thử lại không có độ trễ gây ra tình trạng xung đột đồng thời. Giải pháp tạm dừng lũy thừa giúp phân tán tần suất yêu cầu, trong khi độ lệch ngẫu nhiên ngăn chặn sự thức giấc đồng loạt của các thread. Cần lưu ý, cơ chế backoff chỉ làm dịu hệ thống trong ngắn hạn; nếu tốc độ phát sinh yêu cầu liên tục vượt quá năng lực xử lý của hệ thống, database vẫn sẽ quá tải.

## 6. Chính sách giới hạn tần suất và thời gian

Chính sách thử lại cần hai tham số cốt lõi:

- **Giới hạn số lần:** Ngăn chặn các vòng lặp vô hạn gây tiêu hao tài nguyên cục bộ.
- **Thời hạn xử lý:** Đảm bảo việc thử lại không vượt quá ngân sách thời gian của API, ngăn chặn các thread bị kẹt do trễ mạng hoặc thời gian chờ quá dài.
- **Nút ngắt:** Xử lý kịp thời trường hợp phía gọi đã chủ động ngắt connection trước khi hoàn tất transaction.

Nguyên tắc: Hệ thống phải kiểm tra thời hạn còn lại trước mỗi lần thử và trước khi bắt đầu chu kỳ tạm dừng. Thời gian chờ tuyệt đối không được vượt quá thời gian còn lại của thời hạn.

## 7. Cơ chế chống trùng lặp (tính lũy đẳng)

Khóa chính `command_id` của bảng `reward_credit` được xử lý trong cùng một transaction vật lý với lệnh cập nhật ví.

- Nếu transaction rollback: Cả điểm được cộng và dữ liệu lịch sử đều được khôi phục.
- Nếu phía gọi gửi lại lệnh: Điểm chỉ được ghi nhận một lần nhờ ràng buộc khóa chính.
- Trễ connection: Trạng thái đã commit sẽ được bảo tồn, phía gọi gửi lại `command_id` sẽ nhận lại kết quả cũ.
- Khi có các truy cập cùng mã đồng thời: Chỉ một thread được xử lý, thread còn lại bị từ chối bởi lỗi vi phạm khóa duy nhất, và có thể truy vấn lại để tiếp tục tiến trình.

Cần lưu ý, bảng lũy đẳng chỉ xử lý chống lặp cho cùng một mã lệnh. Nó không ngăn chặn hiện tượng tranh chấp nguồn tài nguyên giữa các mã lệnh khác nhau.

## 8. Xác nhận lại quy tắc nghiệp vụ

Transaction thử lại không chỉ đơn thuần tải lại phiên bản mới nhất mà còn phải thực hiện kiểm duyệt lại toàn bộ quy tắc nghiệp vụ:
Ví dụ: Một thread khác có thể đã cộng điểm vượt qua hạn mức tối đa của ví. Thread thử lại sau khi đánh thức phải xác nhận và phản hồi lỗi từ chối nghiệp vụ thay vì mù quáng thử lại thay đổi.

## 9. Phân loại trạng thái xử lý

Hệ thống CHỈ được phép khởi động cơ chế thử lại đối với lỗi `ObjectOptimisticLockingFailureException` (hoặc các ngoại lệ xung đột tương đương).
**TUYỆT ĐỐI KHÔNG** áp dụng thử lại với các nhóm lỗi sau:

- Dữ liệu tham chiếu không hợp lệ hoặc bị đình chỉ.
- Vi phạm ràng buộc duy nhất mà không phải xuất phát từ tiến trình của cùng một Command ID.
- Lỗi thời gian chờ chưa được cấu hình cho phép thử lại an toàn.
- Yêu cầu đã bị hủy bởi tiến trình cha.
- Các lỗi về định dạng hoặc ánh xạ đối tượng.
- Các lỗi khi giao tiếp hệ thống ngoại vi mà chưa xác định rõ trạng thái dữ liệu đã được xử lý hay chưa.

Ngoại lệ thường chỉ được ném ra khi EntityManager gọi hàm `flush()` hoặc khi kết thúc transaction, do đó khối bắt lỗi cần bao trọn lời gọi của proxy thay vì chỉ kiểm tra lời gọi thay đổi thuộc tính trên RAM.

## 10. Quản lý vòng đời connection trong quá trình chờ

Khi transaction rollback, database sẽ giải phóng lock trên dòng và trả connection về pool TRƯỚC KHI ứng dụng chuyển sang trạng thái tạm dừng.
Trong thời gian chờ, hệ thống chỉ tạm giữ thread hoặc lịch biểu, tuyệt đối không được phép duy trì transaction đang mở. Mặc dù công nghệ thread ảo (Virtual threads - Java 21) hỗ trợ tạm dừng hiệu quả, tài nguyên của database connection pool vẫn bị giới hạn nghiêm ngặt.

## 11. Xử lý tình huống sập nguồn và mập mờ trạng thái commit

Trường hợp lỗi sập nguồn trước khi commit, lần transaction đó được rollback triệt để.
Trường hợp lỗi sập nguồn ngay SAU KHI commit nhưng trước khi trả về kết quả cho phía gọi, transaction rơi vào trạng thái mập mờ. Trong trường hợp này, việc thử lại từ phía gọi với cùng `command_id` phải dựa vào database lũy đẳng (`reward_credit`) để trả lại kết quả. Việc dùng biến đếm vòng lặp để tạo mã định danh là lỗi thiết kế nghiêm trọng, làm vô hiệu hóa khả năng phục hồi dữ liệu.

Đối với các tác vụ ngoại vi phát sinh từ transaction, kiến trúc yêu cầu sử dụng mô hình hộp thư (transactional outbox). Các transaction thử lại tuyệt đối không gọi API ngoại vi khi chưa đảm bảo trạng thái commit hoàn chỉnh tại database.

## 12. Quản trị trong môi trường đa phiên bản

Database đảm nhận vai trò trọng tài duy nhất cho hệ thống phân tán nhờ tính năng phiên bản và ràng buộc khóa.
Các tham số độ lệch ngẫu nhiên phải được tính toán độc lập tại từng node. Việc sử dụng cơ chế đồng bộ hóa bộ nhớ trong chỉ hỗ trợ giới hạn tải cho cá thể hệ thống, không thay thế được khả năng bảo toàn dữ liệu đồng bộ tổng thể.

Việc mở rộng ngang hệ thống làm gia tăng tỷ lệ tranh chấp cục bộ trên các thực thể đang hoạt động mạnh. Nếu lỗi do cạn kiệt thử lại phát sinh liên tục, kiến trúc cần chuyển đổi sang cấu trúc luân chuyển qua hàng đợi, hoặc chiến lược khóa bi quan.

## 13. Phân lớp trách nhiệm sự cố

| Lớp | Trách nhiệm xử lý |
| --- | --- |
| Ứng dụng | Định nghĩa số lần thử, quản trị lũy đẳng, thiết lập ranh giới thời gian và xác minh luật nghiệp vụ. |
| Khung làm việc (Spring) | Khởi tạo và cô lập ranh giới transaction thông qua cơ chế proxy/advisor. |
| ORM (Hibernate) | Quản lý phiên bản tự động và đánh dấu rollback khi phát hiện xung đột. |
| Database | Giữ lock dòng, xử lý mệnh đề cập nhật nguyên tử, và báo cáo lỗi nếu có vi phạm phiên bản/khóa. |

## 14. Tổng hợp các kết quả hoạt động

| Trạng thái | Diễn biến hệ thống |
| --- | --- |
| Thành công lần đầu | Hoàn tất transaction lập tức, không kích hoạt cơ chế chờ. |
| Thành công sau thử lại | Rollback, kích hoạt chờ, khởi tạo transaction mới và commit thành công. |
| Gửi lại cùng ID | Từ chối cập nhật mới, tiến hành truy xuất và trả kết quả cũ. |
| Từ chối nghiệp vụ | Hủy bỏ hoặc kết thúc theo thiết kế lỗi, DỪNG toàn bộ quy trình thử lại. |
| Cạn kiệt | Dừng xử lý và trả về lỗi kỹ thuật hệ thống (quá tải/timeout). |
| Chấm dứt ngang | Truyền tiếp ngoại lệ để đóng các tiến trình không cần thiết. |

## 15. Kiểm soát cạn kiệt nỗ lực

Việc bổ sung độ lệch ngẫu nhiên hỗ trợ phân tán tần suất thử lại nhưng không giải quyết triệt để tính bất bình đẳng trong phân bố thứ tự truy cập. Hệ thống cần được thiết lập cơ chế giám sát vòng đời lệnh, sự phân bố số lần lặp và tần suất phát sinh lỗi cạn kiệt.
Với các thực thể thường xuyên rơi vào cảnh tranh chấp, các kỹ thuật kiểm soát tiếp nhận và cấu trúc phân mảnh là các thiết kế bắt buộc.

## 16. Yêu cầu giám sát kỹ thuật

Các chỉ số cơ bản cần được đo lường:

- Tổng lượng transaction hệ thống so sánh với tỷ lệ lệnh được nhận.
- Mức độ phát sinh ngoại lệ khóa lạc quan.
- Phân bố thành công dựa trên số lần lặp.
- Tỷ lệ lỗi kỹ thuật: Cạn kiệt, vượt quá thời hạn, chấm dứt ngang.
- Phân phối thời gian chờ.
- Số lượng yêu cầu trùng lặp.
- Trạng thái connection pool và thời gian thực thi transaction.
- Tránh sử dụng định danh thực thể trực tiếp trong nhãn metric để hạn chế độ phân giải cao, dẫn tới tràn bộ nhớ. Sử dụng các công cụ truy vết để phát hiện thực thể đang bị tranh chấp hẹp.

Tỷ lệ thành công đầu cuối cao có thể che giấu đi sự thật rằng hệ thống đang bị lạm dụng tài nguyên bên trong, gây đe dọa trực tiếp tới mức khả dụng của toàn cụm dịch vụ.
