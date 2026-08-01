# Phân Tích Tầm Nhìn MVCC (Phân tích khả năng hiển thị ở PostgreSQL)

## 1. Trạng Thái Khởi Đầu (Initial state)

```text
job_id          = IMPORT-42
status          = RUNNING
progress        = 20
generation      = 7
last commit     = Tx-0 (Giao dịch số 0 đã commit trước đó)
```

Luồng Processor (A) và luồng Watchdog (B) sử dụng hai kết nối/giao dịch độc lập đến PostgreSQL.

## 2. Kỳ Vọng Theo Thiết Kế Lỗi (Expected theo broken design)

```text
Processor A cập nhật tiến độ lên 80 nhưng chưa commit.
Watchdog B truy vấn với cấu hình Đọc Bẩn (READ_UNCOMMITTED).
Watchdog B đọc được giá trị 80.
Watchdog B kết luận: "Tiến trình vẫn đang hoạt động (HEALTHY)".
```

## 3. Thực Tế Hoạt Động Ở PostgreSQL (PostgreSQL actual)

```text
Processor A cập nhật tiến độ lên 80 nhưng chưa commit.
Watchdog B truy vấn với cấu hình Đọc Bẩn (READ_UNCOMMITTED).
Watchdog B chỉ nhận được giá trị 20 (phiên bản đã commit gần nhất).
Watchdog B kết luận sai lệch: "Tiến trình bị kẹt! Kích hoạt khôi phục (START_RECOVERY)".
```

Nếu Processor A gặp lỗi và Hủy giao dịch (Rollback), giá trị cuối cùng vẫn được giữ nguyên là `20`. Nếu Processor A hoàn tất việc commit `80` *sau khi* B đã đọc dữ liệu, thì giao dịch của B cũng không tự động cập nhật được con số mới. Chỉ các câu lệnh `SELECT` mới, được khởi tạo sau khi A đã commit, mới thấy được giá trị `80`.

## 4. Dòng Thời Gian Tương Tranh (Timeline ba actor)

| Bước | Processor A — Tx-A | Watchdog B — Tx-B | Recovery C |
| --- | --- | --- | --- |
| T0 | BẮT ĐẦU Giao Dịch | | |
| T1 | CẬP NHẬT (UPDATE) progress=80 | | |
| T2 | LƯU XUỐNG DB (FLUSH), tiếp tục xử lý | | |
| T3 | | BẮT ĐẦU xin Đọc Bẩn (RU) | |
| T4 | | Truy vấn (SELECT) -> Nhận 20 | |
| T5 | | Quyết định: Kích hoạt báo động | |
| T6 | | ĐÓNG GIAO DỊCH (COMMIT) | Bắt đầu chạy trùng lặp (duplicate work) |
| T7 | Tiếp tục công việc hoặc lỗi | | |
| T8 | ĐÓNG GIAO DỊCH (COMMIT) hoặc ROLLBACK | | |

Tại bước T4, cơ sở dữ liệu không hề cung cấp dữ liệu chưa commit (Đọc Bẩn). Lỗi thiết kế nghiệp vụ (Business failure) xảy ra khi Watchdog B sử dụng việc "tiến độ không tăng" làm bằng chứng xác định "Tiến trình bị kẹt", trong khi thực tế Processor A vẫn đang làm việc bên trong một giao dịch dài.

> **Khắc cốt ghi tâm:** "Dữ liệu đã commit" và "Tín hiệu hoạt động (Heartbeat/Liveness)" là hai khái niệm khác biệt. Một giao dịch cơ sở dữ liệu đang mở (open transaction) không thể thay thế cho một cơ chế nhịp tim giám sát (heartbeat mechanism) rõ ràng.

## 5. Cơ Chế Đa Phiên Bản Của MVCC (MVCC tuple versions)

Khi Processor A thực thi lệnh `UPDATE`, PostgreSQL không trực tiếp ghi đè lên dòng dữ liệu cũ, mà tạo ra một phiên bản mới (new tuple version):

```text
Phiên bản cũ: progress=20, tạo ra bởi Giao dịch Tx-0 đã commit.
Phiên bản mới: progress=80, tạo ra bởi Giao dịch Tx-A chưa commit.
```

Bức ảnh chụp dữ liệu (Snapshot) của Watchdog B sẽ xử lý như sau:
- Nhìn thấy phiên bản cũ `20` hợp lệ.
- Giao dịch Tx-A chưa commit nên phiên bản mới `80` hoàn toàn vô hình.
- Lệnh đọc thông thường (plain `SELECT`) của B không cần phải chờ đợi khóa (lock) từ A, mà trực tiếp sử dụng phiên bản cũ.

Processor A luôn thấy được dữ liệu của chính nó trong cùng giao dịch, nhưng điều này không đồng nghĩa với việc giao dịch B có thể đọc được dữ liệu đó.

## 6. Xử Lý Mức Độ Cô Lập Trong PostgreSQL (PostgreSQL isolation mapping)

Tiêu chuẩn SQL cho phép hiện tượng Đọc Bẩn ở mức `READ_UNCOMMITTED`. Tuy nhiên, PostgreSQL được thiết kế chặt chẽ hơn: Yêu cầu `READ_UNCOMMITTED` luôn được âm thầm xử lý tương đương với `READ_COMMITTED` (Chỉ đọc dữ liệu đã commit).

Cần phân biệt rõ:
1. **Mức độ được báo cáo (Reported/Requested label):** Cấu hình `transaction_isolation` hoặc hàm API JDBC có thể trả về giá trị `read uncommitted` để giữ tính tương thích mã nguồn.
2. **Hành vi thực tế (Effective visibility semantics):** Các lệnh đọc thông thường vĩnh viễn không thể thấy được dữ liệu của giao dịch khác chưa commit.

Khi viết kiểm thử tự động, bạn phải đối chiếu trực tiếp giá trị nghiệp vụ (`progress`) trả về, không nên chỉ kiểm tra giá trị của cờ isolation và kết luận hệ thống có hỗ trợ Đọc Bẩn.

## 7. Ảnh Chụp Dữ Liệu Từng Câu Lệnh (Statement snapshot)

Lệnh `SELECT` thông thường của B áp dụng cơ chế "chụp ảnh dữ liệu tại thời điểm câu lệnh bắt đầu thực thi":

```text
Bắt đầu SELECT trước khi A commit -> Chỉ nhìn thấy 20.
A hoàn thành commit.
Một lệnh SELECT mới bắt đầu -> Lúc này sẽ thấy 80.
```

Sự khác biệt khi chạy hai lệnh `SELECT` liên tiếp và nhận hai kết quả đã commit khác nhau được gọi là `Non-repeatable read`, hiện tượng này hoàn toàn khác biệt với Đọc Bẩn (Dirty Read) và sẽ được phân tích ở bài `DB-003`.

## 8. Đọc Thông Thường (Plain SELECT) Và Khóa Dòng (Row Lock)

Lệnh `UPDATE` của A chiếm giữ khóa cấp dòng (row-level lock) cho đến khi commit hoặc rollback.
Lệnh đọc thông thường của B:
- Không yêu cầu cấp phát khóa đối nghịch (như `FOR UPDATE`).
- Truy cập vào phiên bản dữ liệu cũ.
- Không bị chặn đứng (block) bởi quá trình ghi của A.

Tuy nhiên, nếu B sử dụng cơ chế đọc có khóa:
```sql
select *
from job_run
where job_id = :id
for update;
```
Câu lệnh này sẽ bị chặn (block) và đứng chờ A. Sau khi A xử lý xong:
- A commit: B nhận được dữ liệu mới nhất (`80`).
- A rollback: B tiếp tục lấy giá trị cũ (`20`).

Tóm lại: Đọc có khóa (Locking Read) là cơ chế "đợi kết quả cuối cùng", nó không sinh ra hiện tượng Đọc Bẩn.

## 9. Rollback Và Quản Lý Dữ Liệu Rác (Aborted versions)

Điều gì xảy ra nếu Processor A gặp lỗi và Rollback?
```text
Phiên bản mới 80 bị hủy -> Trở thành dữ liệu rác (aborted/dead tuple).
Phiên bản cũ 20 -> Tiếp tục là phiên bản chính thức.
```
Cơ chế dọn dẹp vật lý của PostgreSQL (như `VACUUM`) sẽ xử lý các dòng dữ liệu rác này một cách bất đồng bộ. Về mặt logic, PostgreSQL cấm tuyệt đối các giao dịch khác nhìn thấy hoặc sử dụng dữ liệu từ các giao dịch đã bị hủy. 

Đây là rủi ro lớn nhất của cơ chế Đọc Bẩn (nếu sử dụng CSDL khác có hỗ trợ): Bạn có thể ra quyết định khôi phục hệ thống dựa trên một dữ liệu sẽ bị xóa bỏ ngay sau đó.

## 10. Ranh Giới Giữa Lệnh Cập Nhật (Flush) Và Xác Nhận (Commit)

Quy trình đồng bộ của các ORM framework (như Hibernate):
- Nhận diện sự thay đổi trạng thái của Entity (dirty-check).
- Đẩy câu lệnh `UPDATE` xuống cơ sở dữ liệu (`flush`).
- PostgreSQL ghi nhận thao tác và tạo phiên bản dữ liệu dở dang (chưa commit).
- ORM tiếp tục duy trì giao dịch và khóa dòng dữ liệu.
- Dữ liệu này chỉ được nhìn thấy bởi chính giao dịch hiện hành.

Việc gọi phương thức `saveAndFlush()` không đưa dữ liệu ra phạm vi toàn cục. Chỉ khi giao dịch kết thúc bằng lệnh `COMMIT`, sự thay đổi mới chính thức hiển thị cho các giao dịch khác.

## 11. Đánh Giá Trách Nhiệm Từng Tầng (Root cause theo layer)

| Tầng Hệ Thống | Hành Vi Bị Lỗi | Đánh Giá Khách Quan |
| --- | --- | --- |
| Spring Framework | Chuyển tiếp yêu cầu cấu hình Isolation | Annotation không thể thay đổi hành vi cốt lõi của RDBMS. |
| JDBC/Driver | Ánh xạ và báo cáo lại Isolation Label | Việc hiển thị label không tác động đến cơ chế MVCC. |
| PostgreSQL | Từ chối yêu cầu `READ_UNCOMMITTED` | Tuân thủ triết lý bảo vệ dữ liệu mạnh mẽ, tránh dirty-read. |
| Hibernate | Đẩy lệnh (`flush`) nhưng trì hoãn `commit` | Hành vi tiêu chuẩn của một khối giao dịch, không phải lỗi ORM. |
| Application | Sử dụng Đọc Bẩn để làm tín hiệu giám sát | Lỗi thiết kế nghiệp vụ cơ bản (Root design error). |

## 12. Ảo Tưởng Về Tính Tương Thích (Portability)

Việc giả định rằng cùng một thiết lập mức độ cô lập (Isolation Level) sẽ mang lại hành vi giống hệt nhau trên mọi hệ quản trị CSDL là một sai lầm phổ biến. Các RDBMS có quyền cài đặt các cơ chế bảo vệ nghiêm ngặt hơn chuẩn SQL quy định.

Nếu hệ thống được thiết kế phụ thuộc vào khả năng Đọc Bẩn, ngay cả khi triển khai trên RDBMS có hỗ trợ tính năng này, ứng dụng vẫn đối mặt với rủi ro:
- Đọc dữ liệu không nhất quán (partial/inconsistent).
- Ra quyết định dựa trên dữ liệu có khả năng bị Rollback.
- Giao diện báo cáo tiến độ thay đổi bất thường (tăng rồi tụt giảm).
- Các luồng xử lý đồng thời tạo ra side-effect không thể vãn hồi.

Thiết kế phần mềm chuyên nghiệp (Correct contract) cần dựa trên cơ chế truyền thông điệp rõ ràng (explicit messaging) hoặc trạng thái dữ liệu đã commit, thay vì khai thác một đặc tính không an toàn của CSDL.

## 13. Cơ Chế Giám Sát Chuẩn Mực (Watchdog semantics)

Báo cáo tiến độ và xác nhận hoàn thành công việc là hai hoạt động khác biệt:
```text
Cơ chế Nhịp Tim (Heartbeat/Liveness):
  Sử dụng giao dịch ngắn, độc lập, chốt sổ ngay lập tức. Cần tích hợp định danh (Generation) và hợp đồng thuê mướn (Lease).

Kết quả cuối cùng (Final status):
  Chỉ lưu trữ lâu dài (durable) khi toàn bộ nghiệp vụ đã xử lý thành công.
```

Luồng giám sát (Watchdog) cần kiểm tra các thông tin cụ thể, thay vì chỉ dựa vào tiến độ phần trăm:
- Phiên bản (Generation) và Mã chủ sở hữu (Owner Token) của tiến trình hiện tại.
- Thời điểm nhận nhịp tim gần nhất (Last seen at).
- Thời gian trễ tối đa cho phép (Timeout/Deadline).
- Tiến trình đã kết thúc chưa (Terminal status).

Thay vì tự quyết định khôi phục, Watchdog nên dùng câu lệnh ghi nguyên tử (atomic claim) để giành quyền giải cứu một cách an toàn, tránh tình trạng nhiều Watchdog cùng khởi chạy tiến trình khôi phục.

## 14. Tác Hại Của Giao Dịch Chạy Dài (Long transaction effect)

Nếu Processor A sử dụng một giao dịch duy nhất cho toàn bộ quá trình xử lý phức tạp:
- Trạng thái hệ thống trở nên mù mờ (stale visibility).
- Tài nguyên cơ sở dữ liệu (Kết nối, Khóa, Snapshot) bị chiếm dụng quá lâu, gây áp lực lên bộ nhớ.
- Khi gặp lỗi Rollback, mọi tính toán trung gian đều bị hủy bỏ gây lãng phí lớn.
- Thời gian giám sát bị kéo dài dễ dẫn đến báo động nhầm.

Nên chia nhỏ quy trình nghiệp vụ (workflow) thành các Điểm dừng (checkpoints) hoặc áp dụng kiến trúc Máy trạng thái (state transitions) để rút ngắn chu kỳ commit.

## 15. So Sánh Timeout Và Hỏi Vòng (Timeout vs Polling)

Kỹ thuật hỏi vòng (Polling) liên tục bằng lệnh `SELECT` sẽ chỉ trả về dữ liệu cũ (`20`) cho đến khi giao dịch kia được commit. Thiết lập Timeout cho truy vấn không giải quyết được vấn đề tầm nhìn dữ liệu.

Khi Watchdog phát hiện tiến trình vượt quá thời gian chờ (timeout):
- Không nên tự động chạy lại tiến trình một cách mù quáng.
- Nên cố gắng giành quyền điều khiển thông qua Hợp đồng (Atomic lease/claim).
- Kiểm tra tính xác thực (Generation/Status) trong chính câu lệnh cập nhật.
- Cần triển khai cơ chế tự miễn dịch (Idempotency) để xử lý an toàn nếu Processor A khôi phục hoạt động và tiếp tục xử lý.

## 16. Phân Tích Lỗi Hệ Thống (Crash behavior)

- Processor A bị lỗi hỏng (Crash) trước khi commit: Kết nối ngắt, giao dịch tự động Rollback, dữ liệu trở về `20`.
- Processor A commit `80` thành công nhưng ứng dụng lỗi trước khi phản hồi: Lệnh `SELECT` tiếp theo của Watchdog sẽ đọc được `80`.
- Watchdog lỗi sau khi giành quyền khôi phục: Yêu cầu quản lý trạng thái Lease (cho phép hết hạn hợp đồng) để nhường quyền cho các Watchdog khác.
- Việc phụ thuộc vào Đọc Bẩn hoàn toàn không giúp hệ thống chống chọi tốt hơn với các sự cố đứt gãy giữa chừng.

## 17. Lưu Ý Với Sequence (Sequence caveat)

Cơ chế sinh chuỗi tự động (Sequence/Auto-increment) trong PostgreSQL hoạt động ngoài phạm vi giao dịch và không bị Rollback nếu giao dịch thất bại. Không nên nhầm lẫn tính năng này với khả năng Đọc Bẩn dữ liệu nghiệp vụ của RDBMS.

## 18. Xử Lý Tương Tranh Giữa Nhiều Máy Chủ (Multi-instance)

Các giải pháp sử dụng biến bộ nhớ (in-memory flags) để báo cáo tiến độ chỉ hoạt động trong phạm vi một JVM duy nhất. Trong kiến trúc phân tán (multi-instance), cơ chế giám sát bắt buộc phải dựa vào dữ liệu đã được chốt xuống CSDL, hàng đợi thông điệp (durable message queue), hoặc kho lưu trữ hợp đồng (Lease store) có tính nhất quán cao.

## 19. Chỉ Số Quan Sát Vận Hành (Observability)

Cần giám sát các chỉ số sau để chẩn đoán hệ thống:
- Mức độ cô lập yêu cầu (Requested Isolation) và nhãn trả về (Reported Label).
- Giá trị tiến độ và thế hệ (Generation) thực tế đọc được.
- Thời gian tồn tại trung bình của giao dịch (Transaction duration) từ phía Processor.
- Trạng thái Nhịp Tim và Kẻ nắm giữ Hợp đồng thuê hiện hành.
- Tần suất Watchdog kích hoạt cơ chế khôi phục và số lần thất bại do mất quyền (claim rejection).
- Tần suất Rollback của các giao dịch chia nhỏ (Checkpoints).

Không thiết lập các hệ thống cảnh báo (Alert) nhạy cảm dựa trên sự chênh lệch dữ liệu giữa trạng thái bộ nhớ và CSDL đã commit, vì đây là hành vi thiết kế chủ đích của MVCC, không phải lỗi đồng bộ (Replication lag) hay hỏng hóc CSDL.
