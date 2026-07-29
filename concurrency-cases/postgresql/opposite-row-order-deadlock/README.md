# DB-008 — PostgreSQL deadlock do khóa row ngược thứ tự

## Tóm tắt

Hai request chuyển giá trị giữa cùng hai account nhưng theo hai hướng ngược nhau.
Mỗi transaction khóa source trước rồi mới khóa destination:

```text
T1: transfer A → B, giữ row A rồi chờ row B
T2: transfer B → A, giữ row B rồi chờ row A
```

PostgreSQL nhìn thấy một **chu trình chờ** (`wait-for cycle`), chọn một
transaction làm victim và hủy statement của nó với SQLSTATE `40P01`. Transaction
còn lại tiếp tục sau khi victim rollback và giải phóng lock.

Case bảo vệ ba quy tắc:

```text
Mọi code path cần khóa nhiều account phải acquire row locks theo cùng một
canonical order dựa trên stable unique key.

Một transfer attempt hoặc commit đủ debit + credit, hoặc rollback toàn bộ.

Nếu retry deadlock victim, mỗi attempt phải chạy trong transaction mới, reload
state mới và dừng theo attempt cap/deadline.
```

> **Nói ngắn gọn:** transaction không tự ngăn deadlock; mọi actor phải khóa cùng
> resource theo cùng thứ tự, còn retry chỉ là lớp phục hồi có giới hạn.

## Actor và trạng thái dùng chung

Bảng `account` là authoritative shared state:

| Account | ID | Balance ban đầu |
| --- | ---: | ---: |
| A | `101` | `1_000` |
| B | `202` | `1_000` |

Hai request đi qua hai application instance cũng tạo cùng race:

| Actor | Command | Thứ tự broken |
| --- | --- | --- |
| T1 trên App-1 | chuyển `100` từ A sang B | khóa `101`, rồi `202` |
| T2 trên App-2 | chuyển `70` từ B sang A | khóa `202`, rồi `101` |

Điểm tranh chấp (`contention point`) là lần `SELECT ... FOR UPDATE` thứ hai.
Mỗi actor đã giữ một row-level lock mà actor còn lại cần.

## Ranh giới transaction

Một attempt đúng có đúng một Spring transaction:

```text
BEGIN
  lock account có ID nhỏ hơn
  lock account có ID lớn hơn
  validate source balance trên state đã khóa
  debit source
  credit destination
  flush
COMMIT
```

`@Transactional` nằm trên một worker bean được gọi qua Spring proxy. Retry
coordinator ở ngoài transaction; vì vậy exception của attempt trước được
rollback hoàn toàn trước khi attempt mới bắt đầu.

Case giả định PostgreSQL `READ COMMITTED`, mặc định của Spring/PostgreSQL. Mỗi
`SELECT ... FOR UPDATE` lấy statement snapshot và chờ incompatible row lock khi
cần. Row lock được giữ đến `COMMIT` hoặc `ROLLBACK`, không kết thúc khi repository
method trả về.

## Invariant và kết quả mong đợi

Case dùng account để minh họa lock cycle, không định nghĩa đầy đủ mô hình banking.
Trong phạm vi ví dụ:

- tổng balance của A và B luôn là `2_000`;
- transfer đã báo thành công phải áp dụng cả debit và credit;
- deadlock victim không được để lại thay đổi dở dang;
- hai command hợp lệ cuối cùng phải hoàn tất hoặc trả một kết quả exhaustion rõ
  ràng sau bounded retry;
- không external side effect nào được phát trước commit nếu chưa có
  outbox/idempotency design.

Broken implementation mong hai transfer tự serialize. Thực tế PostgreSQL abort
một transaction; nếu application nuốt lỗi, retry trong transaction cũ hoặc trả
success sớm thì contract bị phá.

## Thuật ngữ cần biết

| Thuật ngữ | Ý nghĩa trong case |
| --- | --- |
| bế tắc (`deadlock`) | Các transaction chờ lock lẫn nhau theo vòng kín |
| đồ thị chờ (`wait-for graph`) | Quan hệ transaction nào đang chờ transaction nào |
| thứ tự khóa chuẩn (`canonical lock order`) | Total order ổn định; account ID nhỏ hơn luôn được khóa trước |
| victim | Transaction bị PostgreSQL abort để phá cycle |
| SQLSTATE `40P01` | Mã lỗi PostgreSQL cho `deadlock_detected` |
| transaction bị hủy (`aborted transaction`) | Transaction không chạy tiếp được cho tới khi rollback |
| bounded retry | Retry có attempt cap, backoff/jitter và overall deadline |
| fresh attempt | Lần chạy lại trong transaction/persistence context mới |

## Điều hướng

- [Broken Spring/JPA implementation](broken-code.md)
- [Timeline, detector và rollback analysis](analysis.md)
- [Canonical ordering và safe retry](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [PostgreSQL locks và lock lifetime](../../concepts/postgresql-locks.md)
- [Deadlock và retry an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trong production

- một request nhận `40P01`, transaction bị rollback và latency tăng;
- retry sai boundary lặp lại trên persistence context/transaction đã doomed;
- retry đồng bộ không backoff có thể tạo retry storm trên hot accounts;
- lock wait giữ connection, làm cạn pool trước khi CPU database quá tải;
- event/email/HTTP call phát trước commit có thể bị lặp dù database attempt bị
  rollback;
- nhiều code path dùng ordering khác nhau làm deadlock tái xuất hiện khó đoán;
- scale-out tăng số actor cạnh tranh; local `synchronized` không bảo vệ rows
  dùng chung.

## Hướng sửa khuyến nghị

1. Chuẩn hóa hai account ID thành `firstId = min(fromId, toId)` và
   `secondId = max(fromId, toId)`.
2. Khóa từng row bằng `FOR UPDATE` theo đúng order đó ở mọi code path.
3. Chỉ sau khi giữ đủ locks mới map lại vai trò source/destination, validate và
   mutate.
4. Giữ transaction ngắn; không gọi remote service trong lock lifetime.
5. Phân loại chính xác `40P01`; rollback attempt cũ rồi retry toàn bộ command
   trong transaction mới với giới hạn rõ.
6. Đặt `lock_timeout`, `statement_timeout` và application deadline theo cùng một
   latency budget; timeout không thay thế ordering.

Canonical ordering ngăn cycle này theo thiết kế. Bounded retry vẫn cần vì hệ
thống thực có thể chứa code path khác, foreign-key/index locking, maintenance
hoặc cycle nhiều hơn hai resource.

## Phạm vi

Case tập trung vào PostgreSQL database lock cycle, victim abort và transaction
retry. Semantics chi tiết của ledger, hold, overdraft, idempotent transfer và
reconciliation thuộc `BANK-003`. JVM intrinsic-lock deadlock thuộc `JVM-007`;
`SERIALIZABLE` abort và SSI thuộc `DB-009`.
