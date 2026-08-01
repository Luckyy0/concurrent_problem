# Bài toán LOCK-002 — Thử lại có giới hạn (Bounded Retry) với khóa lạc quan

## 1. Tóm tắt vấn đề

Hãy xem xét kịch bản: Có nhiều lệnh (command) đồng thời yêu cầu cộng điểm thưởng vào cùng một ví. Ví này được bảo vệ bằng khóa lạc quan (optimistic lock) thông qua annotation `@Version`.

Vì bản chất của việc cộng điểm là thao tác tính toán dựa trên số dư hiện tại (số dư mới = số dư hiện tại + điểm cộng thêm), nên nếu một thao tác thất bại do xung đột phiên bản, thao tác đó hoàn toàn có thể được thử lại (retry) một cách an toàn. Tuy nhiên, nếu vòng lặp thử lại được thiết kế không có khoảng nghỉ, các thread thất bại sẽ liên tục cạnh tranh và tạo ra hiện tượng "bão thử lại" (retry storm), gây quá tải database.

Một quy trình thử lại chuẩn xác phải tuân thủ các bước sau:

```text
Lần thử thứ N (Tx-N): Tải thông tin ví + Ghi nhận lịch sử lệnh.
→ Tính toán số điểm mới + Cập nhật database (flush).
→ Xảy ra xung đột phiên bản: Hoàn tác (rollback) toàn bộ transaction.
→ Kiểm tra giới hạn số lần thử và thời gian cho phép.
→ Thoát khỏi ngữ cảnh transaction, tạm dừng thread với thời gian trễ ngẫu nhiên (backoff with jitter).
→ Bắt đầu lần thử thứ N+1 (Tx-N+1): Tạo transaction mới và tải lại dữ liệu ví từ đầu.
```

> **Ghi chú quan trọng:** `@Version` đảm bảo tính toàn vẹn của dữ liệu; trong khi đó, vòng lặp thử lại có giới hạn bảo vệ hệ thống khỏi tình trạng cạn kiệt tài nguyên do tranh chấp.

## 2. Các thực thể và trạng thái chia sẻ

| Thực thể | Trạng thái ban đầu |
| --- | --- |
| Ví điểm (`reward_wallet`) | Ví số `77`, số dư `100`, phiên bản `10` |
| Các lệnh cộng điểm (C1…Cn) | Mỗi lệnh mang một mã định danh duy nhất (Command ID) và số điểm dương cần cộng. |
| Bảng lịch sử (`reward_credit`) | Lưu trữ lịch sử thao tác để chống trùng lặp theo Command ID. |
| Máy chủ ứng dụng (App-1…App-N) | Các tiến trình chạy song song tiếp nhận yêu cầu. |

Điểm tranh chấp tập trung tại câu lệnh `UPDATE` có kiểm tra phiên bản trên cùng một bản ghi ví. Khi tiến trình trong trạng thái tạm dừng, nó KHÔNG ĐƯỢC phép giữ connection với database. Tuy nhiên, mỗi lần tiến hành thử lại, hệ thống sẽ tiêu tốn thêm một connection và một lượt truy vấn.

## 3. Các quy tắc bất biến

```text
Số dư cuối cùng = Số dư ban đầu + Tổng số điểm của các lệnh cộng điểm hợp lệ đã commit.

Mỗi mã lệnh (Command ID) chỉ được phép tạo ra tối đa một bản ghi lịch sử trong bảng reward_credit.

Mỗi lần thử lại BẮT BUỘC phải mở một transaction hoàn toàn mới (khởi tạo lại persistence context) và phải truy xuất lại dữ liệu ví từ database.

Quá trình thử lại phải tự động chấm dứt khi vượt quá số lần cho phép hoặc hết hạn thời gian tối đa (timeout). Nếu tình trạng cạn kiệt xảy ra, hệ thống không được báo cáo là thành công.
```

## 4. Ranh giới transaction

Lớp điều phối `RewardCreditCoordinator` hoàn toàn KHÔNG sử dụng annotation `@Transactional`. Nó hoạt động bên ngoài ngữ cảnh transaction để gọi phương thức xử lý `RewardCreditAttempt.creditOnce()` thông qua cơ chế proxy của Spring.

Phương thức thực thi (`creditOnce`) được cấu hình với mức lan truyền `REQUIRES_NEW` (Bắt buộc mở transaction mới). Trong transaction này, hệ thống sẽ: kiểm tra tính lũy đẳng, truy xuất dữ liệu ví, cập nhật số dư, lưu trữ lịch sử, đồng bộ dữ liệu (`flush`), và cuối cùng là commit hoặc rollback.

Thời gian tạm dừng (backoff) được quản lý bởi lớp điều phối sau khi transaction của phương thức thực thi đã được rollback xong. Nhờ thiết kế này, database connection không bị giữ lại vô ích. Phía gọi (caller) chỉ nhận được kết quả sau khi tiến trình đã commit thành công hoặc khi đã cạn kiệt nỗ lực thử lại.

## 5. Điều kiện an toàn khi thử lại

Thao tác `Cộng thêm 10 điểm` là một phép tính tương đối dựa trên trạng thái mới nhất, kết hợp với một mã `Command ID` không đổi, do đó việc thử lại (dù nhiều lần) vẫn đảm bảo an toàn. TUY NHIÊN, không được áp dụng cơ chế thử lại cho: các lệnh gán giá trị tuyệt đối (ví dụ: `Gán số dư bằng 80`), các lệnh gọi hệ thống ngoại vi không hỗ trợ cơ chế chống trùng lặp, hoặc các thao tác bị từ chối do vi phạm quy tắc nghiệp vụ (ví dụ: vượt quá hạn mức tối đa).

Trong mỗi lần thử lại, hệ thống phải xác thực lại toàn bộ ngữ cảnh:

- Lệnh này đã được xử lý thành công trước đó chưa (nhằm phòng tránh lỗi mạng dẫn đến mất phản hồi)?
- Trạng thái của ví có còn hợp lệ hay đã bị khóa?
- Giá trị điểm cộng thêm có còn thỏa mãn chính sách hệ thống không?
- Yêu cầu đã hết hạn (timeout) hoặc bị người dùng hủy chưa?

## 6. Các thuật ngữ chuyên ngành

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Khuếch đại tải | Một yêu cầu đầu vào tạo ra nhiều vòng lặp truy vấn/cập nhật do cơ chế thử lại. |
| Bão thử lại | Số lượng lớn các thread thất bại đồng loạt thử lại, gây nghẽn cục bộ database. |
| Thử lại từ đầu | Mở mới transaction, lấy dữ liệu mới và bộ nhớ đệm mới. |
| Thử lại có giới hạn | Quy định rõ ràng về số lần thử tối đa và tổng thời gian tối đa cho một chuỗi xử lý. |
| Thời gian trễ lũy thừa | Khoảng thời gian chờ tăng theo cấp số nhân sau mỗi lần thử thất bại. |
| Độ lệch ngẫu nhiên | Bổ sung giá trị ngẫu nhiên vào thời gian chờ để phân tán các thread cùng lúc bị đánh thức. |
| Suy kiệt tiến trình | Tình trạng một lệnh liên tục bị lệnh khác chiếm tài nguyên cho đến khi cạn kiệt thời gian. |
| Lịch sử lũy đẳng | Bản ghi lưu trữ nhằm đảm bảo một mã lệnh không được thực thi thay đổi quá một lần. |
| Trạng thái cạn kiệt | Vượt quá giới hạn số lần thử hoặc thời gian cho phép mà thao tác vẫn chưa thành công. |

## 7. Điều hướng tài liệu

- [Phân tích lỗi thiết kế và tác động của bão thử lại (broken-code.md)](broken-code.md)
- [Phân tích chuyên sâu các chuỗi transaction và lỗi hệ thống (analysis.md)](analysis.md)
- [Giải pháp cấu trúc điều phối và độ trễ (solutions.md)](solutions.md)
- [Thực nghiệm với PostgreSQL Testcontainers (experiments.md)](experiments.md)
- [Khóa lạc quan và xung đột phiên bản](../../concepts/optimistic-locking.md)
- [Ranh giới transaction trong Spring](../../concepts/spring-transaction-boundaries.md)
- [Kiểm thử tương tranh](../../concepts/concurrency-testing.md)

## 8. Tác động tới hệ thống

Thiết kế vòng lặp thử lại không đúng cách có thể dẫn đến các hệ quả nghiêm trọng:
- Mức độ sử dụng CPU, số lượng truy vấn và lưu lượng ghi tăng vọt bất thường.
- Cạn kiệt connection pool do các thread chờ đợi chiếm dụng.
- Gia tăng độ trễ đối với các bản ghi có tính tranh chấp cao, dẫn đến tỷ lệ lỗi cạn kiệt gia tăng.
- Đặt vòng lặp thử lại BÊN TRONG một transaction sẽ khiến bộ đệm JPA bị đánh dấu `rollback-only`, làm mọi nỗ lực thử lại sau đó đều vô hiệu.
- Việc tự động tái tạo Command ID mới trong mỗi lần thử lại sẽ dẫn đến rủi ro thực thi trùng lặp.
- Thiết lập thời gian trễ cố định (không có độ lệch ngẫu nhiên) dễ dẫn đến tình trạng suy kiệt tiến trình.
- Bỏ qua cơ chế chống trùng lặp (lũy đẳng) khi xử lý các yêu cầu gửi lại từ phía gọi.

## 9. Khuyến nghị áp dụng

1. **Giới hạn mục tiêu:** CHỈ kích hoạt cơ chế thử lại đối với các ngoại lệ phát sinh do xung đột phiên bản (optimistic lock).
2. **Kiến trúc tách biệt:** Tách rời lớp điều phối (không có transaction) và lớp xử lý (transaction độc lập).
3. **Cập nhật ngữ cảnh:** Trong MỖI lần thử, bắt buộc phải tải lại thực thể từ database.
4. **Định danh bền vững:** GIỮ NGUYÊN Command ID ban đầu và kết hợp ràng buộc duy nhất để đảm bảo tính lũy đẳng.
5. **Chiến lược tạm dừng:** Triển khai cơ chế chờ lũy thừa kết hợp với độ lệch ngẫu nhiên. Thiết lập giới hạn về số lần và thời gian.
6. **Quan sát:** Ghi log chi tiết về số lần thử, thông báo khi đạt giới hạn cạn kiệt và theo dõi tần suất xung đột.
7. **Kiểm soát tải:** Nếu tranh chấp tài nguyên ở mức quá cao, cần chuyển đổi sang các giải pháp phân tán như xử lý qua hàng đợi thay vì tăng số lần thử lại.

## 10. Phạm vi áp dụng

Phương pháp này tối ưu cho các hệ thống có mức độ tranh chấp từ thấp đến trung bình và đối với các tác vụ an toàn khi thực hiện lại. Đối với các kỹ thuật xử lý mức độ tranh chấp cao, vui lòng tham khảo bài `LOCK-005`. Chi tiết về thứ tự thực thi của Spring Advisor được phân tích tại `SPR-006`.
