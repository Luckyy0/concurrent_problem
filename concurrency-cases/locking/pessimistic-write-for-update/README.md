# Bài toán LOCK-003 — Khóa Bi Quan Với `FOR UPDATE` (Pessimistic Write Lock)

## 1. Tóm tắt vấn đề (Overview)

Xét trường hợp hai người dùng (A và B) cùng cố gắng đặt chỗ cho ghế "A-10" tại một suất chiếu. Nếu hệ thống áp dụng phương thức truy xuất dữ liệu thông thường (lệnh `SELECT` đơn thuần), tại cùng một thời điểm, cả A và B đều nhận được trạng thái ghế là `AVAILABLE` (Đang trống). Hệ quả là hệ thống sẽ xác nhận thành công cho cả hai giao dịch, dẫn đến tình trạng bán trùng một ghế (Double booking).

Giải pháp cho bài toán này là áp dụng Khóa Bi Quan (Pessimistic Write Lock). Hệ thống sẽ thiết lập khóa bảo vệ ngay từ khâu truy xuất dữ liệu ban đầu:

```text
Giao dịch A: Truy xuất dữ liệu và YÊU CẦU KHÓA (SELECT FOR UPDATE) → Kiểm tra điều kiện → Xác nhận cấp phát (COMMIT)
Giao dịch B: Yêu cầu truy xuất cùng bản ghi và YÊU CẦU KHÓA → HỆ THỐNG ĐƯA VÀO TRẠNG THÁI CHỜ
           → Sau khi A hoàn tất (Commit) → B được phép đọc trạng thái mới → Phát hiện ghế đã HELD (Đã cấp phát) → Từ chối giao dịch
```

> **Ghi chú quan trọng:** Cơ chế "Khóa Dòng" (Row Lock) ép buộc các giao dịch cạnh tranh trên cùng một tài nguyên phải xử lý tuần tự. Giao dịch đến sau bắt buộc phải đưa ra quyết định nghiệp vụ dựa trên trạng thái đã được cập nhật bởi giao dịch liền trước.

## 2. Các Thực thể và Trạng thái chia sẻ (Actors and shared state)

| Thành phần | Vai trò trong ngữ cảnh |
| --- | --- |
| Thực thể `show_seat` | Bản ghi đại diện ghế `A-10`, trạng thái hiện tại: `AVAILABLE`. |
| Yêu cầu của A | Lệnh đặt chỗ của mã khách hàng `501`, thực thi trên Instance 1. |
| Yêu cầu của B | Lệnh đặt chỗ của mã khách hàng `902`, thực thi trên Instance 2. |
| Thực thể `seat_hold` | Bảng lưu trữ lịch sử giao dịch cấp phát ghế. |
| Cơ sở dữ liệu (Database) | Đóng vai trò phân xử và quản lý Khóa (Lock Manager). |

Trọng tâm tranh chấp (Contention point) nằm ở câu truy vấn chỉ định Khóa Chính `(show_id, seat_no)`. Lưu ý: Cơ chế này chỉ thực hiện khóa trên **một dòng dữ liệu vật lý đã tồn tại**, không phải khóa toàn cục hay khóa các khoảng dữ liệu trống.

## 3. Các Quy tắc Bất biến (Invariants)

```text
- Tại bất kỳ thời điểm nào, mỗi ghế chỉ được gắn với tối đa một trạng thái ĐANG GIỮ (ACTIVE hold).
- Định danh giữ chỗ (hold_id) của ghế phải đồng nhất với định danh giao dịch hợp lệ lưu trong bảng lịch sử.
- Một yêu cầu đặt chỗ đơn lẻ không được phép sinh ra đa bản ghi lịch sử cấp phát.
- Hệ thống chỉ phản hồi thành công tới phía gọi SAU KHI quá trình Commit tại cơ sở dữ liệu hoàn tất.
```

Khóa dòng đóng vai trò bảo vệ chu trình: `Đọc -> Kiểm tra -> Ghi`. Trong khi đó, ràng buộc duy nhất (Unique constraint) trên CSDL sẽ xử lý các hành vi lặp lại (Idempotency) từ cùng một yêu cầu của phía gọi. Hai cơ chế này giải quyết các nhóm lỗi phân biệt.

## 4. Ranh giới Giao dịch (Transaction Boundaries)

Tiến trình điều phối (`SeatHoldCoordinator`) không trực tiếp quản lý Giao dịch, mà ủy thác cho tầng Proxy (`SeatHoldTx`). Quy trình chuẩn hóa như sau:

1. Thiết lập Giới hạn thời gian chờ khóa (`lock_timeout`) cho Giao dịch.
2. Truy xuất dữ liệu ghế và yêu cầu Cấp Khóa (`PESSIMISTIC_WRITE`).
3. Sau khi nhận Khóa, tiến hành thẩm định trạng thái nghiệp vụ.
4. Ghi nhận giao dịch vào bảng `seat_hold`, cập nhật trạng thái `show_seat` và đồng bộ (Flush).
5. Hoàn tất Commit (hoặc Rollback) trước khi phản hồi kết quả tới phía gọi.

**NGHIÊM CẤM:** Không được phép đưa các tác vụ tương tác ngoại vi (Remote I/O, API Call) vào bên trong phạm vi Giao dịch đang giữ Khóa, nhằm tránh nguy cơ treo hệ thống.

## 5. Vòng đời Của Khóa (Lock Lifecycle)

Khi chỉ định `LockModeType.PESSIMISTIC_WRITE` qua Spring Data JPA, Hibernate sẽ biên dịch thành truy vấn SQL kèm mệnh đề `FOR UPDATE`:

```sql
select show_id, seat_no, state, hold_id, hold_until
from show_seat
where show_id = :showId
  and seat_no = :seatNo
for update; -- Chỉ định yêu cầu cấp Khóa Dòng
```

Giao dịch sẽ sở hữu Khóa **ngay tại thời điểm câu lệnh SQL này thực thi tại CSDL**, không phải tại thời điểm gọi phương thức Java. Khóa này duy trì hiệu lực cho đến khi Giao dịch kết thúc (Commit hoặc Rollback). Các tiến trình cạnh tranh trên cùng một bản ghi bắt buộc phải Chờ đợi, Bỏ qua, hoặc Hủy bỏ do Timeout.

(Các truy vấn đọc thuần túy (Plain `SELECT`) sẽ không bị chặn bởi Khóa này và vẫn tiếp cận được bản lưu (Snapshot) đã xác nhận trước đó).

## 6. Xử Lý Luồng Giao Dịch Thất Bại

| Trạng thái của Giao dịch giữ Khóa (A) | Trạng thái của Giao dịch chờ (B) |
| --- | --- |
| A hoàn tất Commit (Trạng thái `HELD`) | B được cấp Khóa, đọc trạng thái mới là `HELD`, từ chối giao dịch do lỗi Nghiệp vụ (`ALREADY_HELD`). |
| A hoàn tác Rollback | B được cấp Khóa, đọc trạng thái vẫn là `AVAILABLE`, tiến hành cập nhật thành công. |
| Thời gian chờ vượt ngưỡng `lock_timeout` | Giao dịch B bị ngắt, tự Rollback và trả mã lỗi `BUSY` (Hệ thống quá tải). |
| Xảy ra Khóa Chéo (Deadlock victim) | Tiến trình bị CSDL chủ động Rollback, cần cơ chế thử lại (Retry) tại cấp Ứng dụng. |
| Sự cố Mất kết nối (Network/App crash) | CSDL phát hiện mất kết nối, tự động Rollback giao dịch A và giải phóng Khóa cho B. |

## 7. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Khóa Bi Quan (Pessimistic locking) | Chiến lược kiểm soát truy cập đồng thời dựa trên nguyên tắc giả định xung đột sẽ xảy ra, do đó yêu cầu cấp khóa trước khi thao tác. |
| `PESSIMISTIC_WRITE` | Cờ cấu hình JPA, tương đương với yêu cầu Khóa Ghi Độc Quyền (Exclusive Write Lock). |
| `FOR UPDATE` | Mệnh đề SQL tiêu chuẩn để yêu cầu Row-level Lock. |
| Chủ thể giữ khóa (Lock holder) | Giao dịch đang sở hữu khóa trên tài nguyên chỉ định. |
| Tiến trình chờ (Waiter) | Giao dịch bị tạm ngưng và đưa vào hàng đợi cấp khóa. |
| Giới hạn chờ (Lock timeout) | Khoảng thời gian tối đa một tiến trình được phép lưu lại trong hàng đợi trước khi bị hủy bỏ. |
| Tái thẩm định (Revalidation) | Hành động kiểm tra lại các điều kiện nghiệp vụ sau khi được cấp khóa, do dữ liệu có thể đã thay đổi. |
| Thứ tự cấp khóa (Lock ordering) | Nguyên tắc sắp xếp thứ tự các tài nguyên cần khóa (VD: ID tăng dần) nhằm ngăn chặn Deadlock. |

## 8. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống Đặt Chỗ (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình JPA Và `FOR UPDATE` (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Khóa Bi Quan (Pessimistic locking)](../../concepts/pessimistic-locking.md)
- [Cơ Chế Khóa Của PostgreSQL (PostgreSQL locks)](../../concepts/postgresql-locks.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)
- [Phân Tích Vòng Đời Khóa Dòng/Bảng (DB-007)](../../postgresql/row-table-lock-lifecycle/README.md)

## 9. Tác Động Tới Hệ Thống (Production Impact)

Hệ quả của việc bỏ qua Khóa Bi Quan trong các kịch bản cạnh tranh cao:

- Tình trạng phân bổ trùng lặp (Double booking) dẫn tới lỗi bất biến dữ liệu (Inconsistent state).
- Trạng thái rác (Orphan records) phát sinh do bảng lịch sử và bảng thực thể không đồng bộ.
- Xử lý các tác vụ chậm (Long-running tasks) bên trong Giao dịch gây tắc nghẽn hàng đợi Khóa, dẫn đến sụp đổ hệ thống (Cascading failure).
- Không giới hạn `lock_timeout` khiến cạn kiệt Connection Pool và làm mất khả năng phục vụ của máy chủ ứng dụng.
- Xử lý ngoại lệ Khóa sai cách (Catch and continue) phá vỡ toàn vẹn giao dịch.
- Thiếu cơ chế sắp xếp thứ tự khi khóa nhiều tài nguyên gây ra hiện tượng Khóa Chéo (Deadlocks) diện rộng.
- Áp dụng các kỹ thuật khóa cấp JVM (ví dụ: `synchronized`) trên môi trường Đa Máy Chủ (Multi-instance) dẫn đến mất kiểm soát tương tranh.

## 10. Khuyến Nghị Áp Dụng (Best Practices)

Khóa Bi Quan (`PESSIMISTIC_WRITE`) đạt hiệu quả tối ưu khi:

- Thao tác cập nhật nhắm vào một (hoặc một số) bản ghi cụ thể, đã xác định định danh.
- Quy trình ra quyết định phụ thuộc nghiêm ngặt vào Trạng thái Mới nhất của bản ghi.
- Mức độ tranh chấp (Contention rate) cao, việc áp dụng Khóa Lạc Quan sẽ dẫn đến tỷ lệ thất bại và thử lại (Retry) vượt mức chấp nhận.
- Logic xử lý bên trong khoảng thời gian giữ khóa là cực kỳ ngắn gọn, không phát sinh độ trễ.
- Có nhu cầu từ chối nhanh (Fail-fast) với thông điệp nghiệp vụ cụ thể cho tiến trình chậm hơn.

## 11. Phạm Vi Áp Dụng (Scope Boundary)

Tài liệu này tập trung vào kỹ thuật Khóa Dòng Đích Danh thông qua `FOR UPDATE`. Các kịch bản mở rộng như xử lý hàng đợi (Worker queue) bằng `SKIP LOCKED` được đề cập tại `DB-010`; phân tích chuyên sâu về Deadlock do sai tự khóa tại `DB-008`; và kỹ thuật tối ưu hóa thông lượng (Throughput) tại `LOCK-005`.
