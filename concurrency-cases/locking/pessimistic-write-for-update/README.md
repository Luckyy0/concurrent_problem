# LOCK-003: "Xin Lỗi, Ghế Này Đã Có Chủ!" - Khóa Bi Quan với `FOR UPDATE`

## 1. Tóm tắt câu chuyện

Tưởng tượng hai vị khách (A và B) cùng nhắm trúng cái ghế "VIP A-10" ở rạp chiếu phim. Nếu bạn chỉ cho hệ thống bốc dữ liệu lên kiểm tra một cách ngây thơ (xài lệnh `SELECT` chay), thì ở cùng một tích tắc, cả A và B đều thấy ghế đang `AVAILABLE` (còn trống). Hệ quả là cả 2 cùng nhấn nút Đặt Ghế, hệ thống báo thành công, thế là rạp phim phải ăn "cú phốt" bán 1 ghế cho 2 người.

Đó là lúc Khóa Bi Quan (Pessimistic Write Lock) xuất trận! Hệ thống sẽ khóa chặt cái ghế đó lại ngay từ cái nhìn đầu tiên:

```text
Khách A: Tải ghế và KHÓA LẠI (SELECT FOR UPDATE) → Kiểm tra → Báo đặt thành công (COMMIT)
Khách B: Cũng tải đúng ghế đó và ĐÒI KHÓA → BỊ HỆ THỐNG BẮT ĐỨNG CHỜ!
         → Đợi A chốt sổ xong → B mới được đọc lại trạng thái → Thấy HELD (Đã bán) → Đi về mâm!
```

> **Nói ngắn gọn:** "Khóa Dòng" (Row Lock) ép 2 kẻ đang giành nhau chung một nguồn tài nguyên phải ngoan ngoãn xếp hàng 1-1. Kẻ đến sau bắt buộc phải đưa ra quyết định dựa trên kết quả mà kẻ đến trước vừa chốt sổ.

## 2. Dàn diễn viên và Đạo cụ (Actor và trạng thái)

| Thành phần | Vai trò |
| --- | --- |
| Dữ liệu `show_seat` | Cái ghế `A-10` suất `42`, đang để trạng thái `AVAILABLE`. |
| Request của Khách A | Khách mã `501`, gọi lệnh `hold-a`, đang chạy bên Server 1. |
| Request của Khách B | Khách mã `902`, gọi lệnh `hold-b`, đang chạy bên Server 2. |
| Dữ liệu `seat_hold` | Cuốn sổ ghi lại lịch sử ghế này đã thuộc về ai. |
| Quản gia PostgreSQL | Nơi lưu dữ liệu gốc và cầm cái chìa khóa để phân bua. |

Điểm nóng tranh chấp ở đây chính là câu truy vấn khóa cái Khóa Chính `(show_id, seat_no)`. Lưu ý: Chúng ta đang khóa **Một cái ghế cụ thể đã tồn tại**, chứ không phải khóa khơi khơi kiểu "cấm ai mua bất kỳ ghế trống nào".

## 3. Những Định Luật Thép (Invariant)

```text
- Tại mọi thời điểm, mỗi ghế chỉ được có tối đa 1 người Gắn Trạng Thái ĐANG GIỮ (ACTIVE hold).
- Cột hold_id của ghế phải khớp với đúng người đang giữ ghế đó trong sổ lịch sử.
- Một lệnh đặt chỗ không được phép lỡ tay sinh ra 2 dòng lịch sử giữ ghế.
- Khách hàng chỉ được thông báo "Đặt xong rồi" KHI VÀ CHỈ KHI Database đã chốt sổ (Commit).
```

Khóa dòng sẽ bảo vệ chu trình: Đọc -> Quyết Định -> Lưu. Còn cái luật Ràng buộc Duy nhất (Unique constraint) dưới DB sẽ chặn đứng việc khách spam Click 2 lần vào 1 lệnh. Mỗi vũ khí trị một bệnh khác nhau!

## 4. Xây Ranh Giới Giao Dịch (Transaction)

Người điều phối lệnh (`SeatHoldCoordinator`) sẽ KHÔNG tự mở Giao Dịch, mà nó sai vặt một thằng cu Proxy của Spring (`SeatHoldTx`) đi làm việc đó. Một kịch bản chuẩn sẽ như sau:

1. Đặt đồng hồ đếm ngược (`lock_timeout`) cho Giao Dịch (Quá thời gian là đuổi).
2. Tải cái ghế lên và KHÓA (`PESSIMISTIC_WRITE`).
3. Nếu giật được Khóa, bắt đầu kiểm tra trạng thái ghế.
4. Ghi tên khách vào sổ `seat_hold`, đổi trạng thái ghế rồi Xả (`flush`).
5. Chốt sổ (Commit) hoặc Xóa sổ (Rollback) trước khi báo kết quả ra ngoài.

Nhớ là: Những trò như Gọi qua mạng (Remote I/O) hay Chờ khách trả lời tuyệt đối KHÔNG ĐƯỢC nhét vào trong cái vòng Giao Dịch này, nó sẽ làm kẹt cả hệ thống.

## 5. Cuộc đời của một chiếc Khóa

Khi bạn gõ từ khóa `LockModeType.PESSIMISTIC_WRITE` trong Spring Data JPA, Hibernate sẽ tự dịch nó sang câu SQL có đính kèm chữ `FOR UPDATE` thần thánh:

```sql
select show_id, seat_no, state, hold_id, hold_until
from show_seat
where show_id = :showId
  and seat_no = :seatNo
for update; -- Khóa lại cho tôi!
```

Giao dịch sẽ giật Khóa **Ngay lúc câu lệnh SQL này chạy dưới DB**, chứ không phải lúc bạn gọi hàm Java.
Và cái Khóa này sẽ tồn tại cho đến khi Giao Dịch Chốt Sổ (commit) hoặc Bị Hủy (rollback). Bất kỳ ông nào đòi giật Khóa trên cùng một ghế sẽ phải Đứng Nhìn, Bỏ Cuộc, hoặc Bị Đuổi (Timeout).

(Lưu ý: Mấy hàm `SELECT` bình thường để xem danh sách ghế sẽ không bị kẹt bởi cái Khóa này đâu, chúng vẫn thấy dữ liệu cũ bình thường).

## 6. Người Thua Cuộc Sẽ Về Đâu?

| Số phận kẻ nắm Khóa (A) | Số phận kẻ phải Chờ (B) |
| --- | --- |
| A chốt sổ `HELD` thành công | B tỉnh dậy, đọc thấy ghế đã `HELD`, văng lỗi `ALREADY_HELD` |
| A đổi ý, Rollback (Hủy) | B tỉnh dậy, giật được Khóa, thấy ghế vẫn `AVAILABLE` -> Mua được! |
| Thời gian chờ vượt quá `lock_timeout` | Lệnh của B nổ tung, tự Rollback và báo `BUSY` (Hệ thống đang bận) |
| Vướng lỗi Khóa Chéo (Deadlock victim) | Lệnh tự Hủy (Rollback), chỉ cho làm lại nếu quy trình cho phép. |
| Mất mạng / Chết App giữa chừng | PostgreSQL thấy đứt kết nối liền tự Hủy kết quả của A và nhả Khóa. |

Bài học: Xếp hàng xong không có nghĩa là được mua! Sau khi tỉnh dậy từ hàng chờ, bạn LUÔN PHẢI KIỂM TRA LẠI trạng thái kinh doanh.

## 7. Bách Khoa Toàn Thư Thuật Ngữ

| Thuật ngữ | Ý nghĩa dân dã |
| --- | --- |
| Khóa Bi quan (Pessimistic locking) | Tính đa nghi. Cho rằng kiểu gì cũng có kẻ giành giật nên "Xí chỗ" trước rồi tính sau. |
| `PESSIMISTIC_WRITE` | Cờ của JPA, dịch ra là: Bắt thằng DB khóa lại để tôi ghi dữ liệu. |
| `FOR UPDATE` | Đuôi SQL của PostgreSQL để thực hiện việc khóa dòng đó. |
| Người giữ khóa (Lock holder) | Ông Giao dịch đang giữ cái ghế và không cho ai đụng tới. |
| Kẻ chờ khóa (Waiter) | Ông Giao dịch đến sau phải đứng chầu chực. |
| Tuổi thọ khóa (Lock lifetime) | Sống từ lúc xin được Khóa đến lúc Giao dịch kết thúc. |
| `lock_timeout` | Đứng chờ quá 3 giây thì dẹp đi, đừng chờ nữa. |
| Tái thẩm định (Revalidation) | Nhận được ghế rồi thì phải check xem nó còn trống không (Lỡ ông trước mua mất). |
| Thứ tự Khóa (Lock ordering) | Muốn khóa 5 cái ghế thì phải xếp từ nhỏ tới lớn để tránh kẹt xe chéo nhau. |

## 8. Bản Đồ Kho Báu (Điều hướng)

- [Code đọc-quyết-ghi bị hỏng như thế nào?](broken-code.md)
- [Soi Time-line và các vụ kẹt xe](analysis.md)
- [Tuyệt chiêu code JPA](solutions.md)
- [Đập thử bằng Testcontainers](experiments.md)
- [Tổng quan Khóa Bi Quan](../../concepts/pessimistic-locking.md)
- [Các loại Khóa của PostgreSQL](../../concepts/postgresql-locks.md)
- [Cách test đa luồng](../../concepts/concurrency-testing.md)
- [DB-007 — Vòng đời Khóa Dòng/Bảng](../../postgresql/row-table-lock-lifecycle/README.md)

## 9. Hậu Quả Kinh Hoàng Nếu Code Ẩu

- 2 khách cùng cầm vé vào rạp tranh nhau 1 cái ghế A-10.
- Lịch sử đặt ghế đẻ ra 2 dòng hợp lệ cho cùng 1 ghế.
- Viết hàm Transaction chạy quá lâu làm hàng ngàn người phía sau phải xếp hàng, kéo sập cả cái App.
- Chờ quá lâu mà không cấu hình Timeout, làm màn hình điện thoại người dùng quay mòng mòng.
- `Catch` lỗi Khóa bên trong Transaction rồi chạy tiếp lôm côm làm vỡ toàn bộ dữ liệu.
- Giật Khóa hàng loạt ghế mà không xếp thứ tự làm 2 Giao Dịch kẹt chết cứng nhau (Deadlock).
- Dùng `synchronized` của Java. Rất ngầu nhưng chỉ chạy đúng khi có 1 Server, deploy lên 2 Server là toang!

## 10. Khi Nào Nên Lôi Món Này Ra Xài?

Xài `PESSIMISTIC_WRITE` (Khóa Bi Quan) khi:

- Bạn đã trỏ đích danh một Dòng Dữ Liệu có thật.
- Quyết định thay đổi cần chạy qua 7749 bước kiểm tra phức tạp dựa trên Dữ liệu Mới Nhất.
- Tranh giành nhau đánh lộn xảy ra NHƯ CƠM BỮA (nếu dùng Khóa Lạc Quan thì suốt ngày văng lỗi).
- Đoạn code xử lý chỉ tốn 1 chớp mắt, không làm hàng chờ dồn ứ.
- Kẻ đến sau văng Lỗi có ý nghĩa kinh doanh (Ví dụ báo: "Xin lỗi, ghế vừa có người đặt").

(Lưu ý nhỏ: Nếu bài toán chỉ đơn giản là trừ đi 1 số lượng tồn kho, hãy dùng `UPDATE` kèm điều kiện - cái này nằm ở `LOCK-004`).

## 11. Giới Hạn Của Bài Viết

Phần này chỉ nói về chuyện Tóm Cổ 1 Dòng Dữ Liệu Rõ Ràng bằng `FOR UPDATE`. Các môn phái khác như Xài `SKIP LOCKED` cho hàng đợi (Worker) thì xem `DB-010`; Đụng độ Deadlock xếp ngược thì đọc `DB-008`; hoặc tối ưu hóa khi Server quá rát thì xem `LOCK-005`.
