# Phân Tích Chuyên Sâu: Cơ Chế Hoạt Động Của Statement Snapshot

## 1. Trạng thái khởi điểm (Initial state)

Trước khi quy trình xét duyệt và quản trị viên thực thi, cơ sở dữ liệu PostgreSQL có bản ghi chính sách đã được commit như sau:

```text
merchant_refund_policy(M-42)
  auto_refund_limit = 100.00
  active            = true
  revision          = 7
```

Hệ thống nhận yêu cầu hoàn tiền `80.00`. Ở Phiên bản `7`, hạn mức đủ để tự động phê duyệt; nhưng nếu áp dụng Phiên bản `8` với hạn mức `50.00`, yêu cầu này sẽ bị từ chối.

## 2. Giả định sai lầm trong thiết kế (Broken assumption)

Các lập trình viên thường có suy luận sai lầm như sau:

```text
Vì giao dịch (transaction) ĐÃ BẮT ĐẦU (BEGIN)...
  -> Dữ liệu đọc được sẽ được bảo lưu không thay đổi trong suốt mọi lần truy vấn của giao dịch.
  -> Do đó, phiên bản dùng để đánh giá và phiên bản dùng để lưu audit log sẽ luôn đồng nhất.
```

Điều này chỉ đúng nếu mức cô lập cung cấp một "transaction snapshot" ổn định (như `REPEATABLE READ`). Tuy nhiên, với mức cô lập mặc định của PostgreSQL là `READ COMMITTED`, cơ sở dữ liệu sẽ cấp một snapshot mới ("statement snapshot") cho mỗi câu lệnh SELECT đơn lẻ, khiến giả định trên hoàn toàn sai lệch.

## 3. Diễn biến thực tế (Actual interleaving)

| Bước | Luồng Xét Duyệt (Refund evaluator) | Luồng Quản Trị (Risk administrator) |
| --- | --- | --- |
| 1 | `BẮT ĐẦU GIAO DỊCH (READ COMMITTED)` | |
| 2 | Lệnh SELECT #1 nhận được Snapshot S1 | |
| 3 | Đọc dữ liệu: Hạn mức = `100`, Phiên bản = `7` | |
| 4 | Đánh giá `APPROVED` cho số tiền `80` (Trong RAM) | `BẮT ĐẦU GIAO DỊCH` |
| 5 | | Lệnh UPDATE giảm Hạn mức xuống `50`, Phiên bản lên `8` |
| 6 | | `COMMIT GIAO DỊCH` |
| 7 | Lệnh SELECT #2 nhận được Snapshot mới S2 | |
| 8 | Đọc dữ liệu: Hạn mức = `50`, Phiên bản = `8` | |
| 9 | Lệnh INSERT ghi nhận: Đã Duyệt 80, dựa trên Hạn mức 100, thuộc PHIÊN BẢN 8 | |
| 10 | `COMMIT GIAO DỊCH` | |

Trạng thái hệ thống cuối cùng (Final state):

```text
Chính sách hiện tại: Hạn mức=50, Phiên bản=8
Lịch sử quyết định: Số tiền=80, APPROVED, Hạn mức đối chiếu=100, Phiên bản audit=8
```

Cả hai luồng xử lý đều hoàn thành thành công mà không có lỗi (exception) nào được ném ra. Lý do là hai giao dịch không xảy ra xung đột ghi trên cùng một bản ghi (Luồng xét duyệt thao tác INSERT trên bảng khác, luồng quản trị UPDATE bảng chính sách). Cơ sở dữ liệu không phát hiện xung đột trực tiếp nên không có giao dịch nào bị từ chối.

> **Ghi chú quan trọng:** PostgreSQL đảm bảo không trả về dữ liệu rác (dirty read) ở mức `READ COMMITTED`. Tuy nhiên, nó KHÔNG đảm bảo rằng nhiều lệnh SELECT độc lập trong cùng một giao dịch sẽ trỏ về cùng một trạng thái nhất quán tổng thể. Đó đơn thuần là hai snapshot độc lập ở hai thời điểm khác nhau.

## 4. Cơ chế hoạt động của Statement Snapshot

Ở mức cô lập `READ COMMITTED`, PostgreSQL tạo một snapshot mới chính xác vào thời điểm mỗi câu lệnh bắt đầu thực thi:

```text
Snapshot S1 được tạo TRƯỚC KHI luồng quản trị commit
  -> Quan sát thấy Phiên bản 7

Luồng quản trị commit thay đổi thành Phiên bản 8

Snapshot S2 được tạo SAU KHI luồng quản trị commit
  -> Quan sát thấy Phiên bản 8 mới nhất
```

Lệnh SELECT #1 trả về dữ liệu hoàn toàn hợp lệ tại thời điểm đó (không phải dirty read). Lỗi phát sinh do luồng xử lý ứng dụng (application logic) đã trộn lẫn kết quả của Snapshot S1 với dữ liệu của Snapshot S2 mà không thực hiện xác nhận lại (revalidate) tính hợp lệ của dữ liệu.

## 5. Phiên bản dữ liệu trong MVCC (MVCC tuple versions)

Khi lệnh UPDATE thực thi, PostgreSQL không ghi đè trực tiếp lên bản ghi vật lý cũ, mà tạo ra một phiên bản bản ghi mới (new tuple version):

```text
Bản ghi cũ (old tuple): Hạn mức=100, Phiên bản=7
Bản ghi mới (new tuple): Hạn mức=50, Phiên bản=8
```

Sau khi luồng cập nhật commit:

- Snapshot S1 của luồng xét duyệt vẫn hợp lệ với Bản ghi cũ 7 (dữ liệu vật lý vẫn tồn tại).
- Lệnh SELECT tiếp theo (Snapshot S2) tự động chọn Bản ghi mới 8.
- Bản ghi cũ 7 được giữ lại để phục vụ các snapshot cũ chưa hoàn tất (cho đến khi tiến trình Vacuum thu hồi).
- Ứng dụng ở tầng trên có thể hiểu nhầm hai lần đọc này đại diện cho cùng một trạng thái.

MVCC được thiết kế để hạn chế sự khóa lẫn nhau (đọc không block ghi, ghi không block đọc), không phải để đóng băng trạng thái dữ liệu xuyên suốt mọi mức cô lập.

## 6. Hành vi khóa dữ liệu (Lock behavior)

Khi luồng quản trị thực thi lệnh UPDATE, nó sẽ chiếm khóa độc quyền cấp dòng (row-level lock) và giữ cho đến khi commit.
Ngược lại, lệnh SELECT thông thường của luồng xét duyệt:

- Không yêu cầu khóa (`FOR SHARE` hoặc `FOR UPDATE`).
- Bỏ qua khóa của lệnh UPDATE nhờ cơ chế MVCC cấp snapshot cũ (không bị block).
- Không bảo vệ bản ghi khỏi các thay đổi đồng thời.

Vì lệnh INSERT tiếp theo chỉ yêu cầu khóa trên bảng nhật ký (`refund_decision`), nó không xung đột với khóa trên bảng chính sách. Do đó, cơ sở dữ liệu không thể phát hiện và ngăn chặn tranh chấp này.

## 7. Khối lệnh không nguyên tử (Non-atomic observation)

Quy trình bị phá vỡ do:

```text
Đọc Chính sách (Snapshot V1)
  -> Trong RAM: Đưa ra quyết định dựa trên V1
  -> Giao dịch đồng thời cập nhật DB thành V2
  -> Đọc Chính sách (Snapshot V2)
  -> Ghi Database: Trộn lẫn quyết định V1 với dữ liệu audit V2
```

Annotation `@Transactional` đảm bảo tính nguyên tử cho các lệnh GHI (commit/rollback toàn bộ). Nó KHÔNG đóng gói các lệnh ĐỌC rời rạc thành một quan sát nguyên tử (atomic observation) dưới mức `READ COMMITTED`.

## 8. Nguyên nhân theo từng phân lớp (Root cause analysis)

### Phân lớp PostgreSQL

Mức `READ COMMITTED` được thiết kế để cấp snapshot theo từng câu lệnh. Hiện tượng non-repeatable read là hành vi chuẩn (standard behavior) được phép, không phải là lỗi hệ thống.

### Phân lớp Spring Framework

Annotation `@Transactional(isolation = READ_COMMITTED)` tạo ra một giao dịch vật lý hợp lệ, nhưng không làm thay đổi ngữ nghĩa snapshot của PostgreSQL. Các phương thức con (propagation `REQUIRED`) chỉ đơn giản là tham gia vào cùng một giao dịch.

### Phân lớp Hibernate/JPA

Sử dụng native query hoặc projection (scalar) thường kích hoạt các lệnh SELECT trực tiếp xuống cơ sở dữ liệu thay vì dùng bộ đệm. Hibernate First-Level Cache không can thiệp vào các truy vấn này. Khi Hibernate thực thi lệnh flush, nó không tự động bổ sung điều kiện kiểm tra phiên bản (optimistic locking) vào lệnh INSERT nếu entity không được quản lý đúng cách.

### Phân lớp Ứng dụng (Application code)

Logic ứng dụng thiếu một cơ chế ràng buộc toàn vẹn. Việc sử dụng lại dữ liệu của bảng tham chiếu (authoritative row) mà không kiểm tra lại tính hợp lệ, hoặc không áp dụng cơ chế khóa phù hợp, là nguyên nhân trực tiếp dẫn đến lỗi.

## 9. Lưu ý về First-Level Cache (Persistence context nuance)

Đôi khi ứng dụng không gặp lỗi vì:

```text
Lần 1: findById -> Trả về Entity Java (Phiên bản 7)
Lần 2: findById -> Trả về Entity cũ từ Cache (Vẫn Phiên bản 7, không gọi SQL)
```

Đây chỉ là hành vi của Identity Map trong Hibernate, không phải tính năng bảo vệ của cơ sở dữ liệu. Nếu sử dụng native query, gọi `EntityManager.refresh()`, hoặc chạy trong các transaction context khác nhau, Phiên bản 8 sẽ ngay lập tức được trả về. Do đó, tính đúng đắn của hệ thống không nên phụ thuộc vào hành vi của first-level cache.

## 10. Đánh giá mức `REPEATABLE READ`

Nếu nâng mức cô lập lên `REPEATABLE READ`, PostgreSQL sẽ cấp một transaction snapshot ổn định:

```text
SELECT #1 -> Phiên bản 7
Luồng cập nhật commit Phiên bản 8
SELECT #2 -> Vẫn giữ nguyên Phiên bản 7
```

Mức này loại bỏ hiện tượng "Đọc không lặp lại". Tuy nhiên, giao dịch cập nhật VẪN COMMIT THÀNH CÔNG (do luồng xét duyệt chỉ đọc, không phát sinh write conflict). Kết quả tương đương với:

```text
Quy trình xét duyệt dựa trên Phiên bản 7 hoàn tất.
Sau đó, quy trình quản trị cập nhật chính sách thành Phiên bản 8.
```

Nếu quy tắc kinh doanh cho phép đánh giá dựa trên trạng thái ở đầu giao dịch, thì `REPEATABLE READ` là giải pháp phù hợp. Nếu hệ thống yêu cầu đánh giá phải được áp dụng trên chính sách *mới nhất tại thời điểm chốt*, thì `REPEATABLE READ` không đáp ứng được yêu cầu này.

## 11. Đánh giá mức `SERIALIZABLE`

Mức `SERIALIZABLE` của PostgreSQL áp dụng thuật toán SSI (Serializable Snapshot Isolation) để phát hiện và ngăn chặn các mẫu truy cập bất thường (anomalies). Tuy nhiên, `SERIALIZABLE` đảm bảo tính tuần tự hóa chứ không có nghĩa là "Lệnh đọc luôn lấy dữ liệu được commit gần nhất".

Trường hợp một luồng chỉ Đọc (và ghi bảng khác) và luồng kia Cập nhật, trật tự thực thi tương đương (serialization order) là hợp lệ, do đó cả hai giao dịch đều có thể commit thành công mà không phát sinh lỗi. Để ép buộc sự phụ thuộc chặt chẽ, hệ thống cần áp dụng cơ chế khóa rõ ràng (explicit locking) hoặc thiết kế lại luồng dữ liệu.

## 12. Xác định yêu cầu toàn vẹn (Revalidation semantics)

Cần làm rõ yêu cầu nghiệp vụ (Invariant) thuộc loại nào sau đây:

1. **Snapshot-at-evaluation:** Đánh giá dựa trên chính sách tại thời điểm đọc ban đầu.
2. **Current-at-write:** Tại thời điểm lưu quyết định, chính sách không được phép có bất kỳ thay đổi nào so với lúc đánh giá.
3. **Locked-through-commit:** Yêu cầu độc quyền, không cho phép cập nhật chính sách từ lúc bắt đầu đọc cho đến khi quyết định được commit.

Mỗi loại yêu cầu cần một công cụ tương ứng (ví dụ: truy vấn có điều kiện cho yêu cầu 2, `FOR SHARE` cho yêu cầu 3).

## 13. Kịch bản Commit và Rollback

### Giao dịch cập nhật bị Rollback

Bản ghi Phiên bản 8 sẽ bị hủy (không visible). Lệnh SELECT #2 của quy trình xét duyệt vẫn đọc được Phiên bản 7; hệ thống duy trì tính nhất quán.

### Giao dịch xét duyệt bị Rollback

Quyết định bị hủy. Phiên bản 8 của quy trình cập nhật vẫn được commit thành công và tồn tại độc lập.

### Giao dịch xét duyệt commit trước

Quyết định dựa trên Phiên bản 7 được lưu. Việc này hợp lệ với tuần tự hóa, miễn là thiết kế lịch sử lưu trữ (audit log) hỗ trợ việc tham chiếu lại các phiên bản cũ.

## 14. Quản lý timeout và tranh chấp khóa (Lock wait)

Với lệnh đọc thông thường (plain SELECT), luồng xét duyệt không block luồng cập nhật.
Nếu luồng xét duyệt sử dụng khóa `FOR SHARE`:

- Giao dịch cập nhật sẽ bị block và phải chờ luồng xét duyệt hoàn tất (commit/rollback).
- Có nguy cơ phát sinh lỗi quá thời gian chờ (`lock_timeout`).
- Hoặc gây ra tình trạng bế tắc (deadlock) nếu có nhiều bản ghi bị khóa chéo.

Ứng dụng cần có cơ chế bắt lỗi và xử lý ngoại lệ Timeout một cách an toàn (ví dụ: từ chối xử lý hoặc retry có chủ đích), thay vì cấu hình thời gian chờ không giới hạn.

## 15. Chiến lược Retry an toàn

Lỗi "Non-repeatable read" ở mức `READ COMMITTED` thường không phát sinh exception từ cơ sở dữ liệu. Ứng dụng phải tự phát hiện thông qua kiểm tra logic (so sánh version) hoặc sử dụng truy vấn có điều kiện (affected-row = 0).

Quy trình Retry chuẩn mực:

```text
Thử Lần 1: Phát hiện thay đổi, Rollback giao dịch.
Thực hiện khoảng lùi (Backoff với jitter).
Thử Lần 2: Bắt đầu một GIAO DỊCH MỚI hoàn toàn.
Đọc lại toàn bộ dữ liệu chính sách mới nhất.
Thực hiện lại toàn bộ logic đánh giá.
```

Tuyệt đối không sử dụng lại đối tượng (entity) cũ để thực hiện retry trong cùng một giao dịch. Việc gọi lệnh SELECT lại trong một giao dịch đang sử dụng snapshot ổn định (như `REPEATABLE READ`) sẽ chỉ trả về dữ liệu cũ.

## 16. Tình huống Crash (Crash behavior)

Nếu ứng dụng sập (crash) trước khi commit, PostgreSQL sẽ rollback và dọn dẹp các tài nguyên tạm.
Nếu hệ thống sập ngay sau khi commit nhưng chưa kịp phản hồi cho hệ thống gọi (client), client có thể thực hiện retry. Sử dụng một định danh duy nhất (`command_id`) và cơ chế Idempotency sẽ ngăn chặn việc xử lý trùng lặp.

Điều kiện tiên quyết là Phiên bản chính sách và Thông tin duyệt phải được ghi nhận nguyên tử trong cùng một giao dịch để đảm bảo đội ngũ đối soát luôn có cơ sở dữ liệu hợp lệ để kiểm chứng.

## 17. Môi trường đa máy chủ (Multi-instance concurrency)

Khi các giao dịch được thực thi trên các máy chủ ứng dụng khác nhau, các cơ chế khóa bộ nhớ cục bộ (như `synchronized` hay `ReentrantLock`) hoàn toàn vô hiệu.

Quyền kiểm soát đồng thời phải được thực hiện tại tầng cơ sở dữ liệu:

- Lưu trữ lịch sử (immutable history) với khóa ngoại (Foreign Key).
- Cập nhật có điều kiện (conditional write).
- Khóa cấp dòng (row-level lock).
- Sử dụng mức cô lập (Isolation level) phù hợp với Invariant.

## 18. Khả năng giám sát (Observability)

Cần ghi log các trường thông tin sau:

```text
commandId
merchantId
amount
policyRevisionRead
policyRevisionWritten
decisionOutcome
effectiveIsolation
retryAttempt
```

Các chỉ số (Metrics) cần thiết lập cảnh báo:

- Tần suất bất nhất phiên bản (revision mismatch) / số lần cập nhật hụt (affected-row = 0).
- Số lượng timeout liên quan đến khóa (`lock_timeout`).
- Tần suất lỗi serialization/deadlock.
- Số lượng quyết định không tìm thấy lịch sử tương ứng.
- Tốc độ thực thi giao dịch và áp lực lên Connection Pool.

Trace span cho các lệnh SELECT cần được liên kết với transaction identifier để đội ngũ vận hành có thể phân biệt hiện tượng non-repeatable read với các lệnh truy vấn độc lập.
