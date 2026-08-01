# Phân Tích Cơ Chế SSI, Lỗi `40001` Và Phương Pháp Thử Lại (Transaction retry)

## 1. Điểm Xuất Phát (Trạng thái ban đầu)

```text
Hạn mức của merchant 7 (limit) = 100
Tổng tiền cọc ACTIVE đã commit = 60
Giao dịch C1 yêu cầu = 30
Giao dịch C2 yêu cầu = 30
```

Cả T1 và T2 đều chạy ở mức cô lập `SERIALIZABLE`. Mỗi tiến trình sử dụng một kết nối cơ sở dữ liệu riêng, giao dịch riêng, bộ đệm (persistence context) riêng và một Command ID riêng.

## 2. Diễn biến thời gian (Timeline)

| Bước | Luồng T1 (Xử lý C1) | Luồng T2 (Xử lý C2) |
| ---: | --- | --- |
| 1 | Thực thi `BEGIN SERIALIZABLE` | Thực thi `BEGIN SERIALIZABLE` |
| 2 | Kiểm tra lịch sử xử lý C1 (không có) | Kiểm tra lịch sử xử lý C2 (không có) |
| 3 | Đọc hạn mức `100`, tổng active `60` | |
| 4 | | Đọc hạn mức `100`, tổng active `60` |
| 5 | Đánh giá: `60 + 30 <= 100` -> Cho phép | Đánh giá: `60 + 30 <= 100` -> Cho phép |
| 6 | Thêm bản ghi đặt cọc C1 | Thêm bản ghi đặt cọc C2 |
| 7 | Ghi nhận quyết định `ACCEPTED` | Ghi nhận quyết định `ACCEPTED` |
| 8 | Gọi lệnh `flush/commit` | Gọi lệnh `flush/commit` |
| 9 | Giao dịch Commit thành công | Xảy ra lỗi `40001`, giao dịch Rollback |
| 10 | Tổng active tăng lên `90` | Thực hiện thử lại (Retry), đọc tổng là `90`, ghi nhận quyết định `REJECTED` |

PostgreSQL có thể hủy (abort) T1 hoặc T2. Lỗi này có thể xuất hiện tại bất kỳ thời điểm nào, kể cả trước khi gọi lệnh `COMMIT`. Mã nguồn không nên phụ thuộc vào việc giao dịch nào sẽ thành công.

> **Ghi chú quan trọng:** SSI không đảm bảo giao dịch nào đến trước sẽ hoàn thành trước. Nó chỉ bảo đảm rằng lịch sử của các giao dịch đã commit (committed history) tương đương với một chuỗi thực thi tuần tự (serial execution).

## 3. Mong đợi và Thực tế (Expected vs Broken states)

| Khía cạnh | Trạng thái hợp lệ (Correct) | Trạng thái lỗi (Broken) |
| --- | --- | --- |
| Ràng buộc nghiệp vụ (Invariant) | Tổng active chốt là `90` | Phát sinh lỗi `40001` không xử lý, hoặc thử lại sai cách |
| Kết quả lệnh C1/C2 | Một lệnh `ACCEPTED`, một lệnh `REJECTED` sau khi thử lại | Một lệnh không có kết quả xác định |
| Giao dịch thất bại (Failed attempt) | Rollback hoàn toàn | Ứng dụng tiếp tục truy vấn gây ra lỗi `25P02` |
| Ảnh chụp dữ liệu (Snapshot) | Mở giao dịch mới và đọc trạng thái đã cập nhật của tiến trình chiến thắng | Giữ nguyên bộ đệm và snapshot cũ (đọc lại giá trị `60`) |
| Hiệu ứng phụ (Side effect) | Chỉ gửi thông báo nếu commit thành công | Gửi thông báo trong khi giao dịch bị abort |

Sử dụng `READ COMMITTED` hoặc `REPEATABLE READ` trong kịch bản này sẽ làm cả 2 giao dịch commit thành công, đẩy tổng giá trị lên `120`, phá vỡ ràng buộc toàn vẹn của ứng dụng.

## 4. Cơ chế Snapshot của `SERIALIZABLE` (Serializable Snapshot)

Mức cô lập `SERIALIZABLE` cung cấp một góc nhìn dữ liệu ổn định (stable transaction snapshot). T1 và T2 không thể nhìn thấy những thay đổi chưa commit của nhau.
Điểm khác biệt của SSI là nó theo dõi các chuỗi phụ thuộc (dependency) để ngăn các giao dịch vi phạm trật tự tuần tự. Cơ chế này KHÔNG chuyển đổi mọi thao tác đọc điều kiện (predicate read) thành khóa chặn (blocking lock) và không ép các giao dịch phải chờ đợi nhau.
Trạng thái cập nhật chỉ được xác nhận là hợp lệ SAU KHI thao tác commit hoàn thành thành công.

## 5. Khóa theo dõi `SIReadLock` (Predicate tracking)

Khi ứng dụng thực thi truy vấn điều kiện (`SUM ... WHERE merchant_id=7 AND status='ACTIVE'`), PostgreSQL đánh dấu vùng dữ liệu đã đọc bằng các khóa điều kiện (predicate locks) hiển thị là `SIReadLock` trong `pg_locks`.
Các khóa này:
- KHÔNG chặn (không block) các thao tác ghi (writer) như `FOR UPDATE`.
- Được sử dụng để giám sát xem có luồng nào đang ghi đè vào vùng dữ liệu mà luồng hiện tại vừa đọc hay không.
- Không gây nghẽn kết nối, do đó không nên dùng `lock_timeout` để điều tiết. Phải dùng giới hạn thời gian (deadline) và giới hạn thử lại (bounded retry) tại lớp ứng dụng.

## 6. Xung đột Đọc/Ghi (Read/write dependencies)

Ký hiệu `T_read --> T_write` chỉ ra rằng giao dịch đọc không thấy dữ liệu mà giao dịch ghi đã đưa vào, trong khi giao dịch ghi lại làm ảnh hưởng đến vùng điều kiện (predicate) của giao dịch đọc.

Trong kịch bản này:
```text
T1 đọc -> T2 ghi vào vùng dữ liệu T1 đã đọc
T2 đọc -> T1 ghi vào vùng dữ liệu T2 đã đọc
```
Hai sự phụ thuộc này tạo ra một chu kỳ (serialization cycle). PostgreSQL sẽ ngay lập tức hủy (abort) một giao dịch (ném lỗi `40001`) để ngăn chặn sự cố.

Điều này khác với Deadlock truyền thống:
- **SSI Conflict (`40001`)**: Xung đột logic giữa các snapshot; `SIReadLock` không gây block; Xảy ra khi vi phạm tính tuần tự; Cần thử lại (retry) trong một giao dịch mới.
- **Deadlock (`40P01`)**: Xung đột vật lý khi hai luồng chờ nhau mở khóa (wait-for cycle); Cần giải quyết bằng thứ tự khóa nhất quán.

## 7. Lý do phải thực hiện lại toàn bộ giao dịch (Whole-transaction retry)

Khi một giao dịch thất bại, dữ liệu mà nó đọc ban đầu (`active=60`) đã không còn chính xác. Sau khi luồng kia commit, tổng đã là `90`.
Để thử lại đúng cách, ứng dụng cần thực hiện lại toàn bộ luồng:
1. Đọc hạn mức (limit).
2. Tính lại tổng (active predicate).
3. Kiểm tra tính hợp lệ của lệnh.
4. Ghi nhận kết quả `ACCEPTED/REJECTED`.
5. Thực hiện `COMMIT`.

Hệ quản trị cơ sở dữ liệu không thể tự động thực hiện việc này. Bộ điều phối (coordinator) ở tầng ứng dụng phải kiểm soát toàn bộ chu kỳ này.

## 8. Trạng thái Giao dịch Lỗi (Transaction failed state)

Khi xuất hiện lỗi `40001`:
- Các bản ghi nháp chưa commit sẽ bị hủy bỏ.
- Giao dịch bị đánh dấu là hỏng. Mọi lệnh SQL tiếp theo trong giao dịch đó sẽ sinh lỗi `25P02` (doomed transaction).
- Hibernate `EntityManager` sẽ ở trạng thái không ổn định.
- Không thể sử dụng `EntityManager.clear()` để sửa lỗi này. Quá trình xử lý thử lại (backoff và retry) phải được thực hiện BÊN NGOÀI ranh giới giao dịch hiện hành, đảm bảo một kết nối và snapshot hoàn toàn mới.

## 9. Điểm phát sinh xung đột (Conflict origins)

Lỗi `40001` có thể phát sinh tại các thời điểm:
- Khi thực thi lệnh query.
- Khi gọi `flush()`.
- Khi kết thúc giao dịch (`commit`).

Vì vậy, lớp Điều phối (Coordinator) phải bắt lỗi ở bên ngoài proxy `@Transactional`, chứ không phải xung quanh từng lệnh query cụ thể. Cần sử dụng mã SQLSTATE `40001` để nhận diện nguyên nhân chính xác.

## 10. Đảm bảo mức cô lập trong Spring (Effective isolation)

Annotation `@Transactional(isolation = Isolation.SERIALIZABLE)` chỉ có tác dụng khi tạo ra một giao dịch vật lý mới:
- Việc gọi hàm phải thông qua Spring Proxy.
- Propagation `REQUIRED` sẽ chạy chung trong giao dịch bên ngoài và không thể thay đổi mức cô lập của giao dịch bên ngoài. Cần sử dụng `REQUIRES_NEW`.
- Cần có các bài test tự động chạy lệnh `select current_setting('transaction_isolation');` để xác minh mức cô lập hiện tại.

## 11. Các kết quả kết thúc (Commit, rollback và retry outcomes)

- **Giao dịch thành công (Winner commit)**: Bản ghi đặt cọc và quyết định được lưu nhất quán. `SIReadLock` có thể tồn tại thêm một thời gian để phục vụ theo dõi, điều này là bình thường.
- **Giao dịch thất bại (Victim rollback)**: Trạng thái chưa hoàn thiện sẽ biến mất. Ứng dụng phải trả về lỗi tạm thời hoặc thực hiện retry; không được trả về `ACCEPTED`.
- **Thử lại (Fresh retry)**: Ứng dụng mở giao dịch mới, đọc lại dữ liệu, phát hiện tổng là `90` và ghi nhận `REJECTED`.
- **Hết số lần thử (Retry exhaustion)**: Khi chạm giới hạn thử lại, hệ thống trả về lỗi (ví dụ: `LimitContentionException`). Ứng dụng không được giữ lại trạng thái hỏng.

## 12. Tính lũy đẳng và Kết quả mập mờ (Idempotency and ambiguous outcomes)

Lỗi `40001` là lỗi thất bại rõ ràng (known abort). Nguy hiểm hơn là lỗi mất kết nối (connection loss) trong quá trình commit, gây ra kết quả mập mờ (ambiguous outcome).
Việc thiết kế bảng `limit_command_decision` với `command_id` là khóa chính giúp giải quyết vấn đề:
- Đảm bảo một lệnh không bao giờ được cấp hai lần.
- Cho phép luồng thử lại an toàn thông qua kiểm tra trùng lặp (`23505` unique violation).
- Lưu giữ kết quả (outcome) lâu dài để phản hồi cho Client trong trường hợp Client gọi lại (replay).

Tính lũy đẳng (Idempotency) KHÔNG thay thế cơ chế `SERIALIZABLE`. Cả hai kết hợp để duy trì tính toàn vẹn hệ thống.

## 13. Cơ chế trễ hẹn và giới hạn (Backoff, jitter và deadline)

Thử lại ngay lập tức (Immediate retry) sẽ gây ra xung đột liên tục. Cần thiết lập:
- Bộ lọc lỗi chỉ áp dụng cho mã `40001` (allowlist).
- Giới hạn số lần thử tối đa (attempt cap).
- Thời gian trễ ngẫu nhiên (exponential backoff with random jitter).
- Giới hạn thời gian tổng (overall deadline/cancellation).

Cấu hình các thông số này phụ thuộc vào mức độ tương tranh (concurrency), kích thước connection pool, và cam kết chất lượng dịch vụ (SLO) của ứng dụng.

## 14. Giao dịch Đọc trì hoãn (`SERIALIZABLE READ ONLY DEFERRABLE`)

Đối với các tác vụ báo cáo dài hạn (long-running reports), sử dụng:
```sql
begin isolation level serializable read only deferrable;
```
Chế độ này cho phép PostgreSQL tự động tính toán một Snapshot an toàn trước khi thực thi lệnh. Các truy vấn đọc sẽ KHÔNG BAO GIỜ gây ra lỗi `40001` và không cần phải thử lại. Chế độ này không áp dụng cho các giao dịch thay đổi dữ liệu (read-write).

## 15. Phân loại lỗi Timeout và Deadlock

Mức cô lập `SERIALIZABLE` không giải quyết các vấn đề chờ khóa hàng (row-level lock waits):
- `40P01`: Deadlock do thứ tự khóa sai.
- `55P03`: Lock acquisition timeout (hết thời gian chờ cấp khóa).
- `57014`: Lệnh bị hủy (statement timeout).
- `40001`: Lỗi cơ chế SSI.

Mỗi loại lỗi này đòi hỏi một chiến lược nhận dạng và xử lý (retry/backoff) riêng biệt, không nên đánh đồng.

## 16. Hiệu ứng phụ bên ngoài (External side effect)

TUYỆT ĐỐI KHÔNG thực hiện các lệnh gọi RPC (ví dụ: HTTP call) hoặc gửi tin nhắn (publish message) bên trong hàm `@Transactional` trước khi xác nhận commit. Nếu giao dịch bị lỗi `40001` và rollback, các hiệu ứng phụ này không thể thu hồi.
Cần sử dụng kỹ thuật Transactional Outbox: Lưu thông điệp vào bảng Outbox cùng với giao dịch cơ sở dữ liệu. Sau khi commit thành công, một tiến trình khác (consumer) sẽ đảm nhiệm việc gửi thông báo.

## 17. Sự cố treo máy và mất kết nối (Crash và connection loss)

Khi tiến trình bị sập, kết nối bị ngắt, PostgreSQL sẽ rollback giao dịch hiện tại. Nếu ứng dụng gặp lỗi ngắt mạng ngay lúc nhận kết quả commit, kết quả đó sẽ bị mập mờ (ambiguous outcome). Việc có một Command ID bảo vệ quá trình retry sẽ giúp ứng dụng đọc lại (replay) được quyết định hợp lệ thay vì tạo ra dữ liệu trùng lặp.

## 18. Môi trường đa máy chủ (Multi-instance behavior)

Cơ chế SSI quản lý tại lớp dữ liệu (Database level), do đó nó có khả năng điều phối xung đột trên tất cả các instance ứng dụng. Không sử dụng `synchronized` của Java để điều tiết tiến trình, vì nó chỉ có tác dụng trong một JVM đơn lẻ và hoàn toàn vô nghĩa trong kiến trúc phân tán.

Tuyệt đối ngăn chặn các thao tác can thiệp trực tiếp vào dữ liệu (như truy vấn thủ công bằng `READ COMMITTED`) để không làm phá vỡ các giả định an toàn của `SERIALIZABLE`. Mọi tương tác trên tập dữ liệu này phải tuân theo một tiêu chuẩn chung.

## 19. Xác định nguyên nhân gốc rễ theo phân tầng (Root cause by layer)

| Lớp (Layer) | Vai trò và Rủi ro |
| --- | --- |
| Ứng dụng (Application) | Cung cấp logic thử lại (Retry coordinator). Lỗi nếu thiết lập ranh giới giao dịch sai. |
| Spring | Quản lý Transaction Proxy. Lỗi nếu cố gắng thử lại bên trong giao dịch thất bại. |
| Hibernate (ORM) | Quản lý persistence context. Lỗi nếu không làm sạch ngữ cảnh trước lần thử lại mới. |
| PostgreSQL | Giám sát xung đột theo dependency và phát mã lỗi `40001`. Đây là tính năng chuẩn (expected behavior). |
| Môi trường cục bộ (JVM) | Cần sử dụng Command ID để kết nối nhất quán trạng thái giữa nhiều instance ứng dụng độc lập. |

Lỗi `40001` là dấu hiệu hệ thống đang bảo vệ tính toàn vẹn dữ liệu. Đó không phải là lỗi máy chủ (Database bug), mà là một tín hiệu để ứng dụng thực thi quy trình điều phối phục hồi.

## 20. Giám sát hệ thống (Observability)

Các số liệu cần quan sát:
- Tần suất xuất hiện mã lỗi `40001` theo từng chức năng.
- Số lần thử lại (attempt count) cho mỗi giao dịch thành công.
- Tỷ lệ giao dịch thất bại hoàn toàn sau nhiều lần thử lại (exhaustion rate).
- Trạng thái cấp khóa `SIReadLock` trong `pg_locks` (dùng để chẩn đoán, không dùng trong logic nghiệp vụ).
- Ghi log rõ ràng quá trình xử lý thử lại. Không ghi các dữ liệu nhạy cảm (PII) vào log.

Sử dụng View `pg_stat_database.deadlocks` chỉ giúp phát hiện lỗi Deadlock truyền thống, nó KHÔNG thống kê lỗi Serialization Failure. Cần thiết lập Metrics ở tầng ứng dụng để quản lý tỷ lệ lỗi này.
