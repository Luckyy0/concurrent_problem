# DB-007 — Vòng Đời Của Khóa Cấp Dòng, Cấp Bảng và Tính Phân Lập (Row lock, table lock and lock lifetime)

## 1. Tóm tắt (Overview)

Tưởng tượng một hệ thống quản lý có một Khách Hàng (Tenant) mã `T-42` với hạn mức (quota) hiện tại là `10`. Một quản trị viên (Admin A) khởi tạo giao dịch, đọc mức quota bằng lệnh `SELECT` thông thường, sau đó thực hiện quá trình thẩm định nghiệp vụ cục bộ (local validation). Đội ngũ phát triển (Dev Team) nhầm tưởng rằng: câu lệnh `SELECT` đó đã tự động khóa (row lock) dữ liệu, và bất kỳ ai cố gắng thay đổi dữ liệu này sẽ phải chờ đợi. Tuy nhiên, quản trị viên khác (Admin B) vẫn có thể thực thi lệnh `UPDATE` cập nhật quota thành `8` và hoàn tất giao dịch (commit) thành công trước khi Admin A kịp thay đổi.

Khi phát hiện sai sót, đội ngũ bổ sung cờ `FOR UPDATE` vào lệnh `SELECT`. Lần này họ tin rằng lệnh đọc thông thường cũng sẽ bị chặn lại. Sự thực là, khi Admin A đang giữ khóa (row lock) và cập nhật quota thành `12` (chưa commit):

- Bất kỳ Admin B nào muốn cập nhật (`UPDATE`) hoặc lấy khóa (`SELECT ... FOR UPDATE`) trên cùng dòng dữ liệu ĐỀU phải chờ đợi hoặc bị quá hạn chờ khóa (timeout).
- Các hệ thống hiển thị như Bảng điều khiển (Dashboard C) sử dụng lệnh `SELECT` thông thường VẪN có thể đọc dữ liệu mà không bị chặn, và dữ liệu chúng nhận được là trạng thái đã được commit trước đó (`8`), không phải giá trị `12` đang chờ xử lý của Admin A.
- Khóa (`lock`) CHỈ được giải phóng (release) khi giao dịch cơ sở dữ liệu kết thúc (lúc thực thi `COMMIT` hoặc `ROLLBACK`), KHÔNG PHẢI khi phương thức truy vấn (repository method) trả về kết quả.

**Nguyên tắc thiết kế cốt lõi (Invariant/contract):**

Khi cần thay đổi trạng thái dữ liệu dựa trên giá trị hiện tại của nó (read-modify-write), ứng dụng bắt buộc phải thu được khóa độc quyền (authoritative conflict lock) trước khi ra quyết định. Khóa này phải được duy trì cho đến khi ranh giới giao dịch vật lý khép lại.

Các phiên bản đọc thông thường (Plain observers) sẽ chỉ thấy phiên bản dữ liệu đã được chốt của MVCC (committed MVCC state). Tuyệt đối không có khái niệm "lấy khóa cấp dòng cho phép đọc dữ liệu chưa commit của giao dịch khác".

> **Ghi chú quan trọng:** Khóa cấp dòng (Row lock) sinh ra để giải quyết xung đột khi có nhiều tiến trình muốn cập nhật (conflicting mutations) trên cùng một bản ghi, buộc chúng phải xếp hàng. Nó không làm tê liệt toàn bộ các thao tác đọc MVCC không yêu cầu khóa.

## 2. Các thực thể và Trạng thái chia sẻ (Actors and shared state)

| Thực thể (Actor/State) | Vai trò (Role) |
| --- | --- |
| Admin A | Luồng đọc và cập nhật dữ liệu (quota) trong một giao dịch. |
| Admin B | Luồng thao tác cạnh tranh (competing writer) hoặc tranh chấp khóa (locking reader). |
| Dashboard C | Hệ thống đọc dữ liệu để hiển thị (plain committed-state observer). |
| `tenant_quota` | Bảng chứa trạng thái cần bảo vệ độc quyền (authoritative state). |
| PostgreSQL MVCC | Cơ chế cung cấp phiên bản dữ liệu đã chốt cho các luồng đọc thông thường. |
| Spring Transaction | Lớp vỏ quản lý vòng đời giao dịch và thời gian sống của khóa (lock lifetime). |

Trạng thái dữ liệu ban đầu (Initial committed row):

```text
tenant_id = T-42
quota     = 10
revision  = 5
```

## 3. Ranh giới giao dịch (Transaction boundaries)

### Sự sai lầm về nhận thức khóa (Broken assumption):

```text
Luồng A: BEGIN -> SELECT quota=10 (ngó chay) -> Thẩm định -> UPDATE
Luồng B: BEGIN -> UPDATE trên cùng dòng dữ liệu -> (Kỳ vọng B sẽ bị chặn)
```

Thực tế: Lệnh `SELECT` thông thường của Luồng A KHÔNG giữ bất kỳ khóa bảo vệ dòng nào. Luồng B hoàn toàn có thể thực thi lệnh `UPDATE` và `COMMIT` thành công.

### Khóa Tường minh (Explicit locking):

```text
Luồng A: BEGIN -> Lệnh SELECT ... FOR UPDATE -> Lệnh UPDATE quota=12 -> Chờ đợi ... -> COMMIT
Luồng B: Cố gắng gọi UPDATE hoặc SELECT ... FOR UPDATE trên cùng dữ liệu -> Bị chặn (waits) hoặc hết hạn chờ (timeout).
Luồng C: Gọi lệnh SELECT thông thường -> Không bị chặn, trả về phiên bản cũ (committed version).
```

Điểm tranh chấp (Contention point) xảy ra tại thời điểm các câu lệnh `UPDATE` hoặc `SELECT ... FOR UPDATE` được phát ra. Khóa được kích hoạt (acquired) ở thời điểm phát lệnh và giải phóng (released) tại thời điểm kết thúc giao dịch vật lý.

## 4. Bảng mức độ xung đột khóa (Lock matrix rút gọn)

| Hành động của Luồng A (Holder) | Hành động của Luồng B (Contender) | Kết quả mong đợi (Expected behavior) |
| --- | --- | --- |
| Lệnh đọc `plain SELECT` | Phát lệnh `UPDATE` | B được thực thi ngay lập tức. |
| Giữ khóa cấp dòng `FOR UPDATE` | Phát lệnh `UPDATE` trên cùng dòng | B bị chặn, chờ đợi hoặc timeout. |
| Giữ khóa cấp dòng `FOR UPDATE` | Phát lệnh `SELECT ... FOR UPDATE` | B bị chặn, chờ đợi hoặc timeout. |
| Giữ khóa cấp dòng `FOR UPDATE` | Lệnh đọc `plain SELECT` | C đọc được phiên bản dữ liệu cũ (committed version) của A. |
| Giữ khóa toàn bảng `ACCESS EXCLUSIVE` | Lệnh đọc `plain SELECT` | Lệnh của C bị chặn chờ đợi hoặc timeout. |

Bảng này trình bày các tình huống phổ biến trong bài toán hiện tại. PostgreSQL có nhiều cấp độ khóa bảng (table lock modes) với độ phức tạp cao hơn.

## 5. Thuật ngữ quan trọng (Terminology)

| Khái niệm | Giải thích |
| --- | --- |
| Khóa cấp dòng (row-level lock) | Ngăn chặn các cập nhật đồng thời trên một hàng dữ liệu cụ thể. |
| Khóa cấp bảng (table-level lock mode) | Cơ chế khóa bảo vệ toàn bộ bảng, tự động áp dụng tương ứng với loại lệnh SQL được gọi. |
| Người giữ khóa (lock holder) | Giao dịch đang giữ các khóa khiến các giao dịch khác có yêu cầu khóa xung đột (incompatible lock) phải chờ. |
| Người chờ khóa (waiter) | Giao dịch đang tạm dừng để chờ một người giữ khóa khác hoàn thành. |
| Đọc MVCC không khóa (plain SELECT) | Truy vấn dữ liệu từ ảnh chụp (snapshot) MVCC mà không kèm các cờ yêu cầu khóa (như `FOR UPDATE`). |
| Khóa bi quan độc quyền (FOR UPDATE) | Cờ khóa dòng mạnh nhất để bảo vệ ý định cập nhật dữ liệu (row-level intent). |
| Vòng đời khóa (lock lifetime) | Thời gian từ khi khóa được cấp cho đến khi giao dịch chứa lệnh cấp khóa thực hiện commit/rollback. |
| Thời gian chờ khóa (lock_timeout) | Thời hạn giới hạn việc một giao dịch phải đứng chờ để xin được khóa. |
| Khóa toàn bảng loại trừ (ACCESS EXCLUSIVE) | Cấp độ khóa cao nhất, có khả năng ngăn chặn mọi giao dịch, bao gồm cả đọc thông thường. |

## 6. Hướng dẫn tham khảo (Navigation)

- [Trí Tưởng Tượng Sai Lệch Về Khóa Đọc (Broken SELECT-lock assumptions)](broken-code.md)
- [Phân Tích Cấp Khóa, Hiển Thị Dữ Liệu Và Xả Khóa (Lock acquisition, visibility and release analysis)](analysis.md)
- [Giải Pháp Khóa Bi Quan, Nguyên Tử và Lạc Quan (Pessimistic, atomic and optimistic solutions)](solutions.md)
- [Thực Nghiệm Chặn Khóa Trong PostgreSQL (Deterministic PostgreSQL lock experiments)](experiments.md)
- [Cơ Bản Về Khóa Pessimistic (Pessimistic locking)](../../concepts/pessimistic-locking.md)
- [Tổng Quan Cơ Chế Khóa Của PostgreSQL (PostgreSQL locks)](../../concepts/postgresql-locks.md)
- [Kiến Trúc Đa Phiên Bản MVCC (PostgreSQL MVCC)](../../concepts/postgresql-mvcc.md)
- [Nguyên Tắc Kiểm Thử Tương Tranh (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Tác động tới hệ thống (Production impact)

- Các tiến trình ghi dữ liệu (Writer) có thể vô tình áp dụng logic nghiệp vụ trên dữ liệu đã cũ (stale state) nếu chúng phụ thuộc vào truy vấn đọc thông thường (`plain SELECT`), dẫn đến các lỗi tính toán như cấp phát vượt hạn mức (quota overrun).
- Khi hệ thống triển khai trên môi trường phân tán (multi-node), các giải pháp đồng bộ cấp JVM (như `synchronized`) trở nên vô nghĩa, dẫn đến lỗi bất đồng bộ trạng thái.
- Đặt các lời gọi I/O bên ngoài (ví dụ gọi API bên thứ 3) trong phạm vi khối giữ khóa cấp dòng sẽ khiến các yêu cầu khóa khác bị treo lâu dài, có thể làm cạn kiệt Connection Pool (wait queue/pool exhaustion).
- Việc lạm dụng khóa (`FOR UPDATE`) không đúng chỗ (ví dụ như ở chức năng truy vấn hiển thị Dashboard) sẽ tạo ra tắc nghẽn (needless serialization), gây giảm sút hiệu năng của hệ thống.
- Thiết lập ranh giới giao dịch (transaction boundary) sai cách khiến thời gian sống của khóa ngắn hơn hoặc dài hơn yêu cầu của nghiệp vụ, dẫn đến các sai sót khó dự báo.
- Việc khóa thủ công (explicit table lock mode) ở quy mô toàn bảng sẽ đình trệ tất cả các thao tác liên quan, ảnh hưởng chéo đến các Khách hàng (tenant) khác hoàn toàn không liên quan (unrelated tenants bị block).

## 8. Khuyến nghị khắc phục (Remediation steps)

1. Thiết lập Cấu trúc Giao Dịch Ghi (read-modify-write serialization): Sử dụng cờ `@Lock(LockModeType.PESSIMISTIC_WRITE)` của JPA kết hợp với giới hạn ranh giới giao dịch hẹp (short transaction), hoặc áp dụng lệnh SQL cập nhật nguyên tử có điều kiện (atomic conditional SQL) nếu logic cập nhật đủ đơn giản.
2. Thiết lập Cấu trúc Đọc (Observer): Sử dụng các câu lệnh `SELECT` không khóa cho các chức năng báo cáo, hiển thị (như Dashboard), chấp nhận trạng thái dữ liệu có thể trễ vài mili-giây (committed-staleness contract).
3. Đánh giá tính phù hợp của Cơ Chế Tranh Chấp Lạc Quan (Optimistic Locking): Áp dụng đánh dấu `@Version` để ngăn lỗi đè dữ liệu. Thích hợp cho các trường hợp tỷ lệ tranh chấp thấp, ứng dụng có thể chủ động Retry hoặc Reject giao dịch mà không làm giảm trải nghiệm người dùng.
4. Tránh lạm dụng lệnh khóa cấp bảng: Chỉ dùng các hình thức khóa cấp bảng (explicit table lock) đối với các sự kiện ảnh hưởng tới tính toàn vẹn của cấu trúc bảng (schema operation) hoặc thay đổi cấp độ hệ thống.
5. Luôn xác lập các thông số thời gian chờ `lock_timeout` một cách rõ ràng. Cần áp dụng kỹ thuật thử nghiệm tất định (deterministic testing) cho các lỗi liên quan đến quản lý nhiều dòng (multi-row lock) để giảm rủi ro.

## 9. Phạm vi giới hạn (Scope)

Chủ đề này làm rõ sự tương tác, các loại khóa (lock mode) cơ bản, khả năng hiển thị dữ liệu của MVCC và vòng đời khóa (lock lifetime) trong PostgreSQL.
Các vấn đề liên quan đến hàng đợi công việc bằng cách sử dụng `SKIP LOCKED` sẽ được mô tả tại `DB-010`.
Các mẫu thiết kế (patterns) chọn lựa cơ chế khóa phức tạp hơn được đặt ở `LOCK-003`.
Vấn đề xung đột khóa chéo dẫn đến vòng lặp vô tận (deadlock) được phân tích chi tiết tại `DB-008`.
