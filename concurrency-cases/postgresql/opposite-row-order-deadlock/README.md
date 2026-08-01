# DB-008 — Deadlock Trong PostgreSQL Do Yêu Cầu Khóa Ngược Trật Tự (PostgreSQL opposite row-order deadlock)

## 1. Tóm tắt (Overview)

Tưởng tượng hệ thống xử lý hai lệnh chuyển tiền chạy đồng thời giữa hai tài khoản, nhưng theo chiều ngược nhau.
Mỗi giao dịch (transaction) tiến hành khóa tài khoản nguồn (source) trước, sau đó mới yêu cầu khóa tài khoản đích (destination):

```text
Giao dịch T1: Chuyển từ tài khoản A -> tài khoản B. Khóa tài khoản A, chờ khóa tài khoản B.
Giao dịch T2: Chuyển từ tài khoản B -> tài khoản A. Khóa tài khoản B, chờ khóa tài khoản A.
```

Tình trạng này dẫn đến một vòng lặp chờ đợi lẫn nhau (wait-for cycle) hay còn gọi là Deadlock.
Khi PostgreSQL phát hiện vòng lặp này thông qua cơ chế deadlock detector, nó sẽ buộc phải hủy (abort) một trong các giao dịch (gọi là victim) để giải phóng vòng chờ. Giao dịch bị hủy sẽ nhận lỗi SQLSTATE `40P01`. Giao dịch còn lại sẽ được tiếp tục thực thi sau khi giao dịch bị hủy giải phóng các khóa (locks) đang nắm giữ.

Bài học này nhấn mạnh 3 nguyên tắc thiết kế quan trọng:

```text
1. Khi hệ thống yêu cầu khóa nhiều tài nguyên cùng lúc, BẮT BUỘC phải tuân thủ một trật tự chuẩn (canonical order) thống nhất trên toàn hệ thống (ví dụ: so sánh ID, khóa ID nhỏ trước, ID lớn sau).

2. Trong các giao dịch tài chính, việc cập nhật số dư phải tuân thủ tính nguyên tử (atomicity). Bất kỳ lỗi nào xảy ra cũng phải dẫn đến thao tác rollback toàn bộ giao dịch.

3. Khi xử lý lỗi deadlock thông qua cơ chế thử lại (retry), thao tác thử lại phải được thực hiện trong một giao dịch cơ sở dữ liệu hoàn toàn mới (new transaction). Cần cấu hình số lần thử lại tối đa (bounded retry) và tải lại trạng thái dữ liệu mới nhất.
```

> **Ghi chú quan trọng:** Annotation `@Transactional` không tự động ngăn chặn deadlock. Mọi luồng xử lý (worker) phải khóa tài nguyên theo CÙNG MỘT TRẬT TỰ. Cơ chế tự động thử lại (retry) chỉ là giải pháp phục hồi thứ cấp, không thể thay thế cho một thiết kế trật tự khóa chuẩn xác.

## 2. Các thực thể và Trạng thái chia sẻ (Actors and shared state)

Thực thể `account` đóng vai trò là trạng thái dùng chung (authoritative shared state):

| Tài khoản (Account) | Mã Số (ID) | Số dư (Balance) |
| --- | ---: | ---: |
| A | `101` | `1_000` |
| B | `202` | `1_000` |

Hai luồng chuyển tiền độc lập thực thi đồng thời:

| Luồng (Actor) | Thao tác (Command) | Trình tự khóa gây lỗi (Broken order) |
| --- | --- | --- |
| Luồng T1 | Chuyển `100` từ A sang B | Khóa `101`, sau đó yêu cầu khóa `202` |
| Luồng T2 | Chuyển `70` từ B sang A | Khóa `202`, sau đó yêu cầu khóa `101` |

Điểm tranh chấp (contention point) xảy ra tại thời điểm các luồng gọi lệnh `SELECT ... FOR UPDATE` thứ hai.
Tại thời điểm này, mỗi luồng đều đang giữ một khóa độc quyền cấp dòng (row-level lock) mà luồng kia đang cần.

## 3. Ranh giới giao dịch (Transaction boundary)

Một thao tác chuyển tiền tiêu chuẩn chỉ được thực hiện trong một giao dịch cơ sở dữ liệu duy nhất:

```text
BEGIN TRANSACTION
  Khóa tài khoản có ID NHỎ HƠN
  Khóa tài khoản có ID LỚN HƠN
  Xác thực số dư của tài khoản chuyển (trên dữ liệu vừa khóa)
  Trừ tiền tài khoản chuyển
  Cộng tiền tài khoản nhận
  Ghi nhận dữ liệu (flush)
COMMIT TRANSACTION
```

Trong Spring Framework, annotation `@Transactional` nên được đặt tại lớp dịch vụ chịu trách nhiệm xử lý nghiệp vụ chính (worker bean). Lớp điều phối thử lại (retry coordinator) phải nằm NGOÀI giao dịch này. Cấu trúc này đảm bảo ngoại lệ của giao dịch trước đó kích hoạt rollback hoàn toàn trước khi phiên giao dịch thử lại được khởi tạo.

Bài kiểm thử này sử dụng mức cô lập (isolation level) mặc định của PostgreSQL là `READ COMMITTED`. Ở cấp độ này, lệnh `SELECT ... FOR UPDATE` sẽ truy xuất statement snapshot hiện tại và yêu cầu khóa cấp dòng. Khóa này được duy trì cho đến khi giao dịch kết thúc (commit hoặc rollback), không phải nhả ra ngay khi kết thúc phương thức repository.

## 4. Các yêu cầu toàn vẹn (Invariants and expected outcomes)

Trong khuôn khổ của bài toán chuyển tiền cơ bản, hệ thống phải đảm bảo các quy tắc sau:

- Tổng số dư của A và B luôn bằng `2_000` tại mọi thời điểm.
- Khi giao dịch thành công, số dư của A và B phải phản ánh chính xác số tiền đã chuyển.
- Giao dịch bị hủy (deadlock victim) tuyệt đối KHÔNG ĐƯỢC để lại dữ liệu không nhất quán.
- Sau quá trình xử lý (bao gồm các lần thử lại hợp lệ), cả hai lệnh chuyển tiền đều phải thành công, hoặc bị từ chối với lỗi rõ ràng (exhaustion).
- Các tác vụ ngoại ứng (side effects) như gửi email, gọi API hệ thống khác không được thực hiện trước khi giao dịch COMMIT thành công (cần sử dụng các cơ chế như Outbox pattern hoặc Idempotency).

Việc không tuân thủ quy tắc trật tự khóa có thể khiến PostgreSQL hủy bỏ (abort) một giao dịch. Nếu ứng dụng nuốt lỗi (swallow exception), thử lại bên trong giao dịch hiện tại đã bị hủy, hoặc trả về phản hồi sai, các yêu cầu toàn vẹn dữ liệu sẽ bị vi phạm nghiêm trọng.

## 5. Thuật ngữ quan trọng (Terminology)

| Thuật ngữ | Giải nghĩa |
| --- | --- |
| Deadlock (Bế tắc) | Tình trạng các giao dịch chờ đợi tài nguyên của nhau, tạo thành một vòng lặp không thể tự giải quyết. |
| Wait-for graph | Đồ thị biểu diễn mối quan hệ chờ đợi khóa giữa các tiến trình trong cơ sở dữ liệu. |
| Canonical lock order | Trình tự chuẩn mực và nhất quán khi khóa nhiều tài nguyên (ví dụ: luôn khóa tài nguyên có ID nhỏ nhất trước). |
| Deadlock victim | Giao dịch bị hệ quản trị cơ sở dữ liệu (RDBMS) buộc phải hủy bỏ (abort) để phá vỡ vòng lặp chờ đợi. |
| SQLSTATE `40P01` | Mã lỗi chuẩn của PostgreSQL khi phát hiện deadlock (`deadlock_detected`). |
| Aborted transaction | Giao dịch không thể thực thi tiếp các lệnh SQL và bắt buộc phải thực hiện ROLLBACK. |
| Bounded retry | Cơ chế thử lại giao dịch thất bại nhưng có giới hạn số lần, kết hợp với độ trễ (backoff). |
| Fresh attempt | Khởi tạo một giao dịch vật lý hoàn toàn mới, làm sạch (clear) bộ đệm và tải lại dữ liệu từ đầu. |

## 6. Hướng dẫn tham khảo (Navigation)

- [Mã nguồn lỗi sử dụng Spring/JPA](broken-code.md)
- [Phân tích chu trình kẹt khóa và quá trình rollback](analysis.md)
- [Giải pháp trật tự khóa chuẩn và thử lại an toàn](solutions.md)
- [Các thực nghiệm với Testcontainers PostgreSQL](experiments.md)
- [Tổng quan về cơ chế khóa trong PostgreSQL](../../concepts/postgresql-locks.md)
- [Xử lý Deadlock và cơ chế Retry](../../concepts/deadlocks-and-retries.md)
- [Nguyên tắc kiểm thử tương tranh](../../concepts/concurrency-testing.md)

## 7. Tác động tới hệ thống Production (Production impact)

- Người dùng gặp lỗi hệ thống liên tục do mã lỗi `40P01`, làm tăng độ trễ (latency) của dịch vụ.
- Thiết kế retry sai cách (ví dụ: thử lại trên persistence context hiện tại) có thể dẫn đến việc lưu trữ dữ liệu rác, gây sai lệch thông tin.
- Việc thực hiện retry mà không có khoảng lùi (backoff) gây ra hiện tượng "retry storm", làm quá tải cơ sở dữ liệu.
- Các giao dịch bị treo sẽ giữ kết nối JDBC trong thời gian dài, có khả năng làm cạn kiệt Connection Pool và khiến toàn bộ ứng dụng bị ngưng trệ.
- Gửi thông báo thành công (ví dụ: email) trước khi cơ sở dữ liệu hoàn tất commit có thể gây ra tranh chấp thông tin nếu giao dịch sau đó bị rollback.
- Mã nguồn thực hiện khóa tài nguyên không theo chuẩn khiến việc xác định nguyên nhân và khắc phục lỗi khóa chéo trên hệ thống phân tán trở nên vô cùng khó khăn.
- Giải pháp sử dụng từ khóa `synchronized` trong Java không có tác dụng trong môi trường triển khai đa máy chủ (multi-instance).

## 8. Hướng dẫn khắc phục (Remediation steps)

1. Khi cần thao tác trên hai đối tượng cùng loại, luôn thực hiện sắp xếp ID: `firstId = min(fromId, toId)` và `secondId = max(fromId, toId)`.
2. Yêu cầu khóa (`FOR UPDATE`) theo thứ tự đã sắp xếp: ID nhỏ trước, ID lớn sau.
3. Chỉ sau khi đã thu thập đủ tất cả các khóa cần thiết, ứng dụng mới tiến hành xác minh logic kinh doanh và cập nhật dữ liệu.
4. Tối thiểu hóa thời gian giữ khóa. Không thực hiện lệnh gọi I/O từ xa (như gọi API ngoài) trong phạm vi giao dịch cơ sở dữ liệu.
5. Cần cấu trúc khối lệnh `try-catch` hoặc công cụ AOP phù hợp để nhận diện mã lỗi `40P01`, sau đó xử lý rollback và kích hoạt cơ chế retry trong một giao dịch mới với số lần giới hạn.
6. Thiết lập các thông số thời gian chờ (`lock_timeout`, `statement_timeout`) một cách cẩn trọng để phù hợp với Service Level Objective (SLO). Chú ý rằng các tham số timeout này dùng để ngăn chặn hệ thống bị treo, không phải là phương pháp xử lý gốc rễ của việc thiết kế khóa sai trật tự.

Trật tự khóa chuẩn (Canonical ordering) giải quyết vấn đề deadlock tại cơ sở dữ liệu. Tuy nhiên, một chiến lược thử lại (Bounded retry) an toàn vẫn cần thiết để đối phó với các lỗi ngẫu nhiên và tranh chấp khác (ví dụ: khóa khóa ngoại, bảo trì, vòng lặp phức tạp).

## 9. Phạm vi giới hạn (Scope)

Tài liệu này tập trung phân tích hiện tượng khóa ngược trật tự dẫn đến deadlock ở PostgreSQL (`40P01`), hành vi hủy giao dịch (victim abort) của hệ quản trị cơ sở dữ liệu, và cách thức thiết kế vòng lặp retry an toàn.
Các nghiệp vụ kế toán phức tạp hơn như Sổ cái (Ledger) hay hệ thống luân chuyển chứng từ sẽ được mô tả tại `BANK-003`.
Vấn đề tranh chấp khóa phần mềm (JVM-level locking) được trình bày tại `JVM-007`.
Việc xử lý lỗi phân lập tuần tự (Serialization anomaly) và SSI sẽ được hướng dẫn chi tiết tại `DB-009`.
