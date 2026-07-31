# SPR-005 — Bắt lỗi Checked Exception nhưng giao dịch vẫn vô tình commit

## Tóm tắt

Hàm `PayoutPreparationService.prepare()` chuyển trạng thái giao dịch chi trả (payout) sang `PROCESSING`, thực hiện giữ tiền (hold) trong ví và thêm một bản ghi sổ cái (ledger entry). Tuy nhiên, sau đó, chính sách cục bộ phát hiện ra người thụ hưởng bị chặn và ném ra ngoại lệ `BeneficiaryRejectedException` — một ngoại lệ có kiểm tra (checked exception).

Mặc dù phương thức gọi (caller) nhận được ngoại lệ và đinh ninh rằng toàn bộ thao tác đã bị hủy (rollback), nhưng mặc định Spring chỉ tự động rollback với các ngoại lệ không kiểm tra (unchecked exception) và `Error`. Kết quả là trình chặn giao dịch (transaction interceptor) của Spring vẫn tiến hành commit các thay đổi vào cơ sở dữ liệu rồi mới truyền ngoại lệ có kiểm tra ra bên ngoài.

Bài toán này giúp bảo vệ rào chắn tính đúng đắn (invariant) sau:

```text
Nếu hàm prepare() kết thúc bằng lỗi BeneficiaryRejectedException:
  - Giao dịch chi trả không được phép ở trạng thái có thể thực thi (executable).
  - Số dư khả dụng (available balance) và số tiền đang bị giữ (hold) không được phép thay đổi.
  - Tuyệt đối không có bản ghi giữ tiền (PAYOUT_HOLD) nào được lưu vào sổ cái.
  - Tiến trình điều phối (dispatcher) không được phép nhìn thấy giao dịch này như một công việc hợp lệ.
```

> **Nói ngắn gọn:** Kết quả của hàm (method outcome) thì nói là "bị từ chối" (rejected), nhưng kết quả của giao dịch (transaction outcome) lại là "đã commit". Lỗi ở đây hoàn toàn do cách phân loại rollback (rollback classification) của Spring, chứ không phải lỗi của cơ sở dữ liệu PostgreSQL.

## Các thành phần và trạng thái dùng chung (Actors và shared state)

| Thành phần | Vai trò |
| --- | --- |
| Yêu cầu A (Request A) | Chuẩn bị chi trả và vô tình nhận được ngoại lệ nghiệp vụ có kiểm tra |
| Tiến trình điều phối B (Dispatcher B) | Đọc các giao dịch chi trả đã commit ở trạng thái `PROCESSING` để đẩy vào luồng xử lý |
| Bảng `payout_request` | Chứa trạng thái của quá trình chi trả |
| Bảng `wallet_account` | Chứa thông tin số dư khả dụng và số tiền bị giữ (hold projection) |
| Bảng `ledger_entry` | Nơi lưu vết kiểm toán chỉ được phép thêm mới (append-only) cho việc giữ tiền |
| Spring transaction proxy | Quyết định sẽ commit hay rollback dựa trên quy tắc phân loại ngoại lệ |
| PostgreSQL | Chứa trạng thái đã commit chuẩn xác mà mọi máy chủ ứng dụng đều cùng nhìn thấy |

## Ranh giới giao dịch và điểm tranh chấp (Transaction boundary và contention point)

Hàm `prepare()` là một phương thức công khai được gọi thông qua Spring proxy, sử dụng chính sách lan truyền (propagation) là `REQUIRED`. Nó sẽ tạo ra một giao dịch vật lý thực sự (physical transaction) dưới PostgreSQL. Chính sách kiểm tra người thụ hưởng diễn ra cục bộ, không thực hiện bất kỳ lệnh gọi I/O từ xa nào trong lòng giao dịch này.

Điểm tranh chấp (Contention point) nằm ở chỗ nhiều tiến trình cùng truy cập vào các dòng dữ liệu của giao dịch chi trả và ví, cùng với điều kiện tìm kiếm `status = 'PROCESSING'` của tiến trình điều phối. Khóa dòng (row lock) có thể giúp sắp xếp tuần tự hai tiến trình ghi (writer) chạy cùng lúc, nhưng nó không thể sửa được quy tắc rollback: trạng thái sai lệch vẫn sẽ bị commit và trở thành trạng thái hợp lệ để các tiến trình khác nhìn thấy sau khi khóa được giải phóng.

## Kỳ vọng và Thực tế

| | Phía Caller | Dữ liệu sau khi gọi hàm | Phía Dispatcher |
| --- | --- | --- | --- |
| Điều mong đợi | Nhận lỗi bị từ chối | Trạng thái ban đầu hoặc `REJECTED`, không giữ tiền | Không điều phối |
| Lỗi thực tế | Nhận lỗi bị từ chối | Trạng thái `PROCESSING`, số dư giảm, có bản ghi giữ tiền | Vẫn có thể điều phối |

Khi proxy của giao dịch xử lý ngoại lệ có kiểm tra, thao tác commit đã hoàn tất từ trước khi ngoại lệ kịp ném ra ngoài (rethrow) tới tay người gọi. Vì vậy, các cơ chế thử lại (retry), cảnh báo (alert) hoặc các luồng đền bù (compensation) nếu chỉ dựa vào việc bắt ngoại lệ thì sẽ hoạt động trên một giả định hoàn toàn sai lầm.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| Checked exception | Ngoại lệ có kiểm tra, bắt buộc phải dùng `try-catch` hoặc khai báo `throws`; không kế thừa từ `RuntimeException` |
| Unchecked exception | Ngoại lệ không kiểm tra, tức là `RuntimeException` hoặc các lớp con của nó |
| Rollback rule | Quy tắc quyết định việc ánh xạ từ một loại ngoại lệ sang hành động commit hay rollback |
| Rollback-only | Cờ đánh dấu cho biết giao dịch này không còn được phép commit nữa |
| Failure contract | Cam kết đã công bố về mối liên hệ giữa kết quả của hàm và trạng thái dữ liệu đã lưu trữ |
| Logical scope | Phạm vi hiệu lực của `@Transactional` trên một hàm được chặn bởi proxy |
| Physical transaction | Giao dịch thực sự dưới cơ sở dữ liệu sẽ bị commit hoặc rollback |
| Executable state | Trạng thái mà tiến trình điều phối coi là đủ điều kiện để mang đi xử lý |

## Hướng dẫn điều hướng

- [Đường dẫn lỗi do checked exception](broken-code.md)
- [Phân tích phân loại rollback](analysis.md)
- [Thiết lập rào chắn lỗi tường minh và cách sửa](solutions.md)
- [Các thí nghiệm với PostgreSQL Testcontainers](experiments.md)
- [Ranh giới giao dịch trong Spring](../../concepts/spring-transaction-boundaries.md)
- [Kiểm thử các vấn đề đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production nếu làm sai

- Tiền bị giữ lại (hold) dù giao dịch chi trả đã bị báo là từ chối (rejected).
- Tiến trình điều phối vẫn hì hục chạy một công việc mà phương thức gọi đinh ninh là đã thất bại.
- Nếu client thử lại (retry) do nhận ngoại lệ, có thể tạo ra các lệnh trùng lặp hoặc xung đột với trạng thái dữ liệu đã lỡ commit.
- Hệ thống đối soát (Reconciliation) thấy có log báo ngoại lệ nhưng trong sổ cái lại tồn tại một bản ghi hợp lệ về mặt kỹ thuật.
- Khi chạy đa máy chủ (multi-instance), độ lệch sẽ càng hiển hiện rõ vì máy chủ khác ngay lập tức nhìn thấy kết quả đã commit.
- Cuối cùng, bạn bắt buộc phải viết thêm các luồng đền bù (compensation) rắc rối cho một lỗi lẽ ra chỉ cần rollback nguyên tử (atomic rollback) là xong.

## Hướng sửa chữa (Khuyến nghị)

Nếu ngoại lệ có kiểm tra đại diện cho việc toàn bộ khối công việc phải thất bại, hãy khai báo tường minh `rollbackFor = BeneficiaryRejectedException.class` ngay trên ranh giới giao dịch công khai, và cứ để ngoại lệ đó thoát ra ngoài qua proxy.

Nếu việc bị từ chối là một kết quả nghiệp vụ (business outcome) nằm trong dự kiến và cần được lưu trữ lại, đừng dùng ngoại lệ với hàm ý ép buộc rollback. Hãy ghi nhận trạng thái là `REJECTED`, tuyệt đối không tạo ra các trạng thái có thể thực thi hay thao tác giữ tiền, commit một kết quả nghiệp vụ rõ ràng và trả về thông báo `Rejected`.

## Khi nào nên áp dụng

Áp dụng cho các hàm dịch vụ (service methods) có khai báo sẽ ném ra ngoại lệ nghiệp vụ có kiểm tra sau khi đã thay đổi dữ liệu cơ sở dữ liệu. Bài toán này không áp dụng để xử lý các vấn đề như mạng bị chậm quá thời gian (remote timeout ambiguity), kết quả không xác định từ bên thứ ba thanh toán (unknown outcome) hay việc đền bù cho các tác động gọi hệ thống bên ngoài (compensation for external side effect); vì đó là một mô hình thiết kế xử lý lỗi hoàn toàn khác.
