# Phân Tích Chuyên Sâu: Cơ Chế Khóa Dòng `FOR UPDATE`

## 1. Trạng Thái Khởi Điểm (Initial state)

```text
Thực thể `show_seat` "A-10", Suất chiếu "42":
  Trạng thái (state): AVAILABLE
  Định danh giữ chỗ (hold_id): null
  Thời hạn giữ (hold_until): null

Dữ liệu `seat_hold`: Rỗng []
```

Xem xét kịch bản hai máy chủ ứng dụng (App-1 và App-2) tiếp nhận đồng thời hai yêu cầu độc lập để đặt chỗ trên cùng ghế `A-10`. Cả hai ngữ cảnh giao dịch đều khởi tạo tiến trình xác nhận cấp phát.

## 2. Kịch Bản Lỗi: Truy Xuất Thiếu Đồng Bộ (Lost Update Timeline)

Phân tích luồng thực thi khi sử dụng lệnh Đọc tiêu chuẩn (`SELECT` thuần túy):

| Bước | Máy chủ 1 (Giao dịch A) | Máy chủ 2 (Giao dịch B) | Trạng thái Cơ sở dữ liệu |
| --- | --- | --- | --- |
| 1 | Khởi tạo Giao dịch | | |
| 2 | Đọc dữ liệu: `AVAILABLE` | | Không cấp phát khóa bảo vệ. |
| 3 | | Khởi tạo Giao dịch | |
| 4 | | Đọc dữ liệu: `AVAILABLE` | Hai giao dịch nhận chung trạng thái snapshot. |
| 5 | Đánh giá nghiệp vụ: Hợp lệ | Đánh giá nghiệp vụ: Hợp lệ | Tính toán rẽ nhánh dựa trên dữ liệu độc lập. |
| 6 | Gửi lệnh `UPDATE` cho A | | CSDL thiết lập Row Lock cho phiên của A. |
| 7 | Hoàn tất (Commit) | | Trạng thái thay đổi do A được ghi nhận. |
| 8 | | Gửi lệnh `UPDATE` cho B | B chèn đè (overwrite) lên kết quả của A. |
| 9 | | Hoàn tất (Commit) | Xảy ra lỗi phân bổ trùng lặp (Double booking). |

Mặc dù PostgreSQL tự động kích hoạt Khóa Dòng tại khâu `UPDATE`, cơ chế này không kiểm soát được độ trễ giữa pha Đọc (Read) và pha Ghi (Write). Sự logic hóa nghiệp vụ trên lớp Ứng dụng đã diễn ra trước khi Khóa CSDL được thiết lập, dẫn đến vi phạm tính toàn vẹn (Inconsistent Read).

## 3. Đối Chiếu Kết Quả (Expected vs Actual)

| Chỉ Tiêu | Kế Hoạch Đề Ra | Hệ Quả Thực Tế (Không Khóa) |
| --- | --- | --- |
| Số lượng bản ghi `ACTIVE` | 1 bản ghi duy nhất | 2 bản ghi xung đột |
| Dữ liệu tham chiếu | Trỏ tới giao dịch hợp lệ duy nhất | Trỏ tới giao dịch thực thi `UPDATE` cuối cùng |
| Phản hồi luồng cạnh tranh | Lỗi từ chối `ALREADY_HELD` | Xác nhận Thành công sai lệch |
| Tính toàn vẹn hệ thống | Duy trì trạng thái nhất quán | Dữ liệu rác và sai lệch kiểm toán |

## 4. Kịch Bản Tiêu Chuẩn: Dòng Thời Gian Khóa Bi Quan

Phân tích luồng thực thi khi áp dụng `SELECT ... FOR UPDATE`:

| Bước | Máy chủ 1 (Giao dịch A) | Máy chủ 2 (Giao dịch B) | Trạng thái Cơ sở dữ liệu |
| --- | --- | --- | --- |
| 1 | Khởi tạo Giao dịch | | |
| 2 | Yêu cầu `FOR UPDATE` | | CSDL cấp phát Exclusive Row Lock cho A. |
| 3 | Đọc dữ liệu: `AVAILABLE` | | Khóa thuộc quyền sở hữu của A. |
| 4 | | Yêu cầu `FOR UPDATE` | Xung đột Khóa, Giao dịch B chuyển sang trạng thái Lock Wait (Chờ). |
| 5 | Gửi lệnh `UPDATE` cho A | (Đang trong hàng chờ) | |
| 6 | Hoàn tất (Commit) | (Đang trong hàng chờ) | Lưu thay đổi của A. Giải phóng Row Lock. |
| 7 | | Được cấp Khóa | B đọc dữ liệu Mới Nhất do CSDL cung cấp (`HELD`). |
| 8 | | Tái thẩm định: Lỗi `ALREADY_HELD`| B từ chối cập nhật do vi phạm điều kiện nghiệp vụ. |
| 9 | | Hoàn tác (Rollback) | Giao dịch kết thúc, trả lại Khóa. |

Nguyên lý thiết yếu: **Giao dịch cạnh tranh (Tiến trình chờ) bắt buộc phải tiến hành Tái thẩm định (Revalidation) trên bộ dữ liệu vừa được cập nhật bởi Giao dịch tiền nhiệm.**

## 5. Hiện Tượng Ảo Giác Snapshot Và `READ COMMITTED`

Với các truy vấn không đi kèm Khóa (Plain `SELECT`), hệ thống MVCC cấp phát một Snapshot cách ly. Truy vấn này không bị cản trở bởi `FOR UPDATE` và sẽ đọc trạng thái đã xác nhận gần nhất.

Tuy nhiên, cấu trúc `SELECT ... FOR UPDATE` sở hữu các đặc tính khác biệt:
1. Xác định bản ghi mục tiêu qua Khóa Chính (hoặc Index).
2. Phát hiện Khóa Xung Đột (Incompatible Lock) và chuyển sang trạng thái chờ.
3. Khi Khóa được giải phóng, cơ chế nội bộ của PostgreSQL cấp phát lại dữ liệu **MỚI NHẤT (Latest version)** cho tiến trình chờ (hoặc loại bỏ kết quả nếu bản ghi đã bị xóa).
4. Tầng Ứng dụng tiếp nhận đối tượng đã cập nhật để tái đánh giá logic (ví dụ: kiểm tra `state`).

Kỹ thuật này tạo ra chuỗi thao tác "Chờ - và - Thẩm Định", đem lại sự ưu việt về thông lượng so với việc lạm dụng cấp độ cách ly `REPEATABLE READ` (thường gây ra lượng lớn ngoại lệ `Serialization Failure`).

## 6. Phạm Vi Tác Động Của Khóa (Lock Scope)

Truy vấn `SELECT ... FOR UPDATE` thiết lập hai lớp Khóa: Khóa bảo vệ cấu trúc bảng (Table-level Intent Lock) ngăn chặn các lệnh thay đổi lược đồ (DDL) và Khóa Dòng Độc Quyền (Row-level Lock) trên các bản ghi được trả về. Các giao dịch yêu cầu `UPDATE`, `DELETE`, hoặc `SELECT ... FOR SHARE/UPDATE` trên cùng bản ghi sẽ bị đẩy vào hàng chờ.

Lưu ý: Khóa Dòng không phong tỏa toàn bộ Bảng. Nó cũng không khóa các dòng không tồn tại trong kết quả truy vấn. Khuyến nghị phân tích chi tiết lệnh SQL được sinh ra bởi Hibernate nhằm đảm bảo tính chính xác của tập bản ghi bị khóa.

## 7. Tuổi Thọ Của Khóa (Lock Lifetime)

Chu trình sống của Khóa Bi Quan:

```text
BẮT ĐẦU GIAO DỊCH (BEGIN)
  Cấu hình giới hạn chờ (lock_timeout)
  SELECT ... FOR UPDATE  ← THỜI ĐIỂM CẤP PHÁT KHÓA
  Tái thẩm định (Revalidate)
  Cập nhật trạng thái
XÁC NHẬN / HOÀN TÁC (COMMIT/ROLLBACK) ← THỜI ĐIỂM GIẢI PHÓNG KHÓA
```

Khóa **CHƯA ĐƯỢC GIẢI PHÓNG** trong các trường hợp:
- Phương thức Repository hoàn tất.
- Đối tượng Java thoát khỏi phạm vi quản lý (Scope).
- Phát sinh tương tác ngoại vi (Remote API Call) bên trong Giao dịch.
- Lệnh `UPDATE` đã xả (flush) xuống CSDL nhưng Giao dịch chưa được Commit.

Kết luận: Biên độ của Giao dịch vật lý tỷ lệ thuận trực tiếp với Thời gian giữ khóa.

## 8. Ảnh Hưởng Từ Spring Proxy Và Quá Trình Đồng Bộ (Flush)

Annotation `@Lock` chỉ khai báo siêu dữ liệu. Ranh giới quản trị Giao dịch thuộc về `@Transactional`, thường được điều phối bởi AOP Proxy ở cấp độ Service:

```text
Client kích hoạt Service
  → Proxy khởi tạo Giao Dịch (BEGIN)
  → Lệnh `FOR UPDATE` cấp Khóa Dòng
  → Xử lý logic và thay đổi Trạng thái đối tượng
  → Xả lệnh cập nhật (flush)
  → Proxy xác nhận Giao Dịch (COMMIT)
  → Giải phóng tài nguyên và Trả kết quả
```

Các cuộc gọi nội bộ (Self-invocation) bỏ qua Proxy sẽ không hình thành Giao dịch, khiến Khóa bị mất tác dụng. Việc chủ động gọi `flush()` dời thời điểm phát hiện lỗi SQL lên sớm hơn trong khối hàm, nhưng sự thay đổi dữ liệu chỉ được chốt cứng sau quá trình Commit của Proxy.

## 9. Phân Tích Các Kịch Bản Thất Bại (Failure Resolution)

### Giao Dịch Sở Hữu Khóa Hoàn Tất (Commit)
Dữ liệu của luồng A được lưu trữ vĩnh viễn. Luồng B tiếp quản Khóa, đọc trạng thái mới (`HELD`), và từ chối xử lý dựa trên ràng buộc vòng đời (State Machine). Luồng B tuyệt đối không được sử dụng lại Snapshot trước đó.

### Giao Dịch Sở Hữu Khóa Hủy Bỏ (Rollback)
Dữ liệu được hoàn tác về trạng thái gốc (`AVAILABLE`). Luồng B tiếp quản Khóa, xác minh tính khả dụng, và tiến hành cập nhật thành công.

### Sự Cố Mất Kết Nối (Connection Drop)
Nếu máy chủ xử lý luồng A gặp sự cố (Crash), tiến trình của PostgreSQL sẽ tự động kích hoạt Rollback cho Giao dịch treo và giải phóng Khóa. Chú ý: Việc phía gọi mất tín hiệu phản hồi (Timeout) không đồng nghĩa với việc Giao dịch trên CSDL đã Rollback. Cần sử dụng Idempotency ID để đối soát.

## 10. Quản Trị Hàng Chờ Khóa Và Timeout (Lock Timeout)

Thuộc tính `lock_timeout` định nghĩa thời gian tối đa Giao dịch được phép tồn tại ở trạng thái Lock Wait. Khi vượt quá giới hạn, PostgreSQL sẽ ngắt truy vấn và báo lỗi SQLSTATE `55P03` (`lock_not_available`). Việc phân tích mã lỗi nguyên thủy giúp phân biệt giữa quá tải hệ thống và lỗi nghiệp vụ.

Phát sinh lỗi CSDL đồng nghĩa Giao dịch hiện tại đã chuyển sang trạng thái Bắt Buộc Hoàn Tác (`aborted` / `rollback-only`). Toàn bộ truy vấn tiếp theo trong ngữ cảnh này sẽ bị từ chối:

```text
Vượt giới hạn thời gian (Timeout)
→ Ngoại lệ SQL kích hoạt
→ Spring đánh dấu Giao dịch thành Hủy bỏ (Rollback)
→ Thoát khỏi Proxy, định tuyến trả mã HTTP tương ứng (503 Dịch vụ không khả dụng / 409 Xung đột).
```

Cần thiết lập `lock_timeout` nhỏ hơn ngân sách thời gian tổng thể (request timeout) để hệ thống có khả năng tự phục hồi.

## 11. Các Chiến Lược Xử Lý Đợi Khóa (Lock Wait Policies)

| Cơ Chế | Hành Vi Chấp Hành | Khuyến Nghị Áp Dụng |
| --- | --- | --- |
| **Đợi Cấp Phép Có Thời Hạn (Bounded wait)** | Xếp hàng và kiểm tra điều kiện | Cạnh tranh danh tính cụ thể, giao dịch xử lý cực nhanh |
| **Loại Bỏ Ngay Lập Tức (`NOWAIT`)** | Từ chối ngay nếu tài nguyên đang bị khóa | Cần phản hồi tức thì (Fail-fast), có cơ chế xử lý lỗi |
| **Bỏ Qua Khóa Kế Tiếp (`SKIP LOCKED`)** | Bỏ qua các dòng đang bị khóa, tìm dòng tự do | Mô hình phân phối công việc hàng đợi (Tiến trình xử lý tác vụ) |
| **Đợi Vô Thời Hạn (Unbounded wait)** | Phụ thuộc hoàn toàn vào luồng khóa | **Tuyệt đối tránh** sử dụng làm cấu hình mặc định |

Trong bài toán đặt chỗ đích danh, `SKIP LOCKED` sẽ gây ra phản hồi sai lệch (VD: Báo cáo ghế không tồn tại thay vì ghế đang được xử lý). `Bounded wait` hoặc `NOWAIT` là các lựa chọn kiến trúc chuẩn.

## 12. Ngăn Chặn Khóa Chéo Do Khóa Đa Dòng (Deadlock Prevention)

Khi một Giao dịch yêu cầu khóa đồng thời nhiều bản ghi (VD: Đặt 2 ghế), **bắt buộc** phải sắp xếp tập hợp theo một trật tự xác định tuyến tính (ví dụ: Sort theo ID tăng dần).

```text
Sắp xếp tập ID ghế (A-10, A-11)
→ Lặp qua danh sách và xin Khóa tuần tự
→ Tái thẩm định toàn bộ tập
→ Cập nhật thay đổi
```

Nếu Giao dịch 1 yêu cầu (A-10, A-11) và Giao dịch 2 yêu cầu (A-11, A-10), hệ thống sẽ hình thành Deadlock. PostgreSQL sẽ tự động ngắt một Giao dịch (Deadlock victim - SQLSTATE `40P01`). Sắp xếp tuần tự khóa là biện pháp kỹ thuật giảm thiểu tối đa rủi ro này (Chi tiết tham khảo `DB-008`).

## 13. Phân Định Khóa Dòng Và Khóa Trùng Lặp Logic

Ràng buộc Unique Constraint (`command_id`) có nhiệm vụ chặn các yêu cầu gửi đúp (Idempotency), không cho phép 1 lệnh tạo 2 bản ghi lịch sử.
Khóa Dòng (Row Lock) có nhiệm vụ ngăn chặn 2 lệnh riêng biệt cạnh tranh trên cùng 1 tài nguyên hệ thống. Hai tầng bảo vệ hoạt động bổ trợ nhau trên không gian quản trị khác biệt.

## 14. Sự Vô Hiệu Của Khóa Ứng Dụng Trên Kiến Trúc Phân Tán (Multi-instance Fallacy)

Các rào cản luồng nội bộ (VD: `synchronized`, `ReentrantLock` trong Java) chỉ có giá trị cục bộ trong một JVM. Trên môi trường Cụm (Cluster), các máy chủ không thể chia sẻ trạng thái Khóa này, dẫn đến xung đột ghi đè. Cấu trúc `FOR UPDATE` triển khai cơ chế khóa tại tầng Lưu trữ (Storage Engine), bảo đảm tính toàn vẹn độc lập với mô hình mở rộng Ứng dụng.

## 15. Giám Sát Hiện Tượng Tắc Nghẽn Giao Dịch (Lock Convoy & Observability)

Khi thời gian giữ khóa kéo dài, số lượng kết nối chuyển sang trạng thái chờ gia tăng đột biến, gây cạn kiệt Connection Pool. Việc giám sát (Observability) là bắt buộc:
- Theo dõi tỷ lệ ngoại lệ Timeout (`55P03`) và Deadlock (`40P01`).
- Giám sát độ trễ luồng xử lý bên trong ranh giới Giao dịch.
- Đo lường tình trạng Utilized của Connection Pool.
- Phân tích trạng thái cảnh báo các kết nối Idle-in-transaction.

## 16. Phân Tích Lớp Trách Nhiệm Gây Lỗi (Root Cause Mapping)

- **Application Layer**: Không thiết lập chuỗi `Đọc-Khóa -> Tái Thẩm Định -> Cập Nhật`, phán quyết logic trên dữ liệu đã lỗi thời.
- **Spring Layer**: Đặt `@Transactional` sai vị trí hoặc tổ chức phạm vi Giao dịch không bao trọn quy trình xử lý, khiến Khóa bị giải phóng sớm.
- **ORM (Hibernate/JPA)**: Phụ thuộc vào Dirty Checking để phát sinh `UPDATE` vào cuối chu kỳ mà không cấp phát Khóa Bi Quan từ đầu thông qua phương thức Repository.
- **PostgreSQL**: Tại mức cách ly `READ COMMITTED`, mọi yêu cầu đọc không khóa đều trả về dữ liệu tại thời điểm quá khứ (Snapshot). Phải áp dụng explicit lock (`FOR UPDATE`) để kích hoạt cơ chế đọc nhất quán (Read-latest).

## 17. Giới Hạn Áp Dụng (Out of Scope)

Cơ chế `FOR UPDATE` chỉ có tác dụng bảo vệ **Các bản ghi đã tồn tại trong CSDL**. Đối với yêu cầu nghiệp vụ ngăn chặn sự hình thành các bản ghi mới vi phạm điều kiện (Ví dụ: Chống đặt lịch lặp khung giờ - Phantom Rows), phương pháp Khóa Dòng không khả thi. Cần chuyển hướng sang các giải pháp Khóa Bảng, Constraint Dữ liệu Phức hợp, Khóa Phân Tán (Distributed Lock), hoặc mức cách ly `SERIALIZABLE`.
