# Phân tích doomed transaction

## Trạng thái ban đầu

```text
inventory_item:
  sku       = BOOK-42
  available = 2
  version   = 7

reservation_record:
  không có bản ghi nào cho command A hoặc B
```

Hai command có ID riêng biệt và mỗi command đặt trước (reserve) số lượng `1`.

## Versioned SQL

Sau khi đọc version 7, Hibernate tạo câu lệnh update tương đương:

```sql
update inventory_item
set available = :newAvailable,
    version = 8
where sku = 'BOOK-42'
  and version = 7;
```

Số lượng dòng bị ảnh hưởng (affected-row count) chính là tín hiệu báo xung đột:

```text
1 dòng -> version kỳ vọng vẫn là hiện tại, update thành công
0 dòng -> version đã thay đổi, quá trình ghi dữ liệu cũ (stale write) bị từ chối
```

PostgreSQL không hiểu `@Version`; Hibernate thêm predicate và kiểm tra số lượng dòng.
Spring biên dịch xung đột này thành ngoại lệ `ObjectOptimisticLockingFailureException` ở lớp abstraction phù hợp.

## Dòng thời gian của hai tiến trình (actor)

| Bước | Command A — Tx-A | Command B — Tx-B |
| --- | --- | --- |
| T0 | Đọc stock 2, version 7 | |
| T1 | | Đọc stock 2, version 7 |
| T2 | Reserve trên bộ nhớ -> 1 | Reserve trên bộ nhớ -> 1 |
| T3 | Flush UPDATE kỳ vọng v7; ảnh hưởng 1 | |
| T4 | Commit stock 1, version 8; ghi nhận A | |
| T5 | | Flush UPDATE kỳ vọng v7; ảnh hưởng 0 |
| T6 | | Hibernate ném ra optimistic conflict; Tx-B trở thành doomed |
| T7 | | Vòng lặp lỗi catch ngoại lệ, xóa context, bắt đầu lần lặp 2 trong Tx-B |
| T8 | | Quá trình tải/thay đổi có thể chạy, nhưng Tx-B không thể commit |
| T9 | | Ranh giới bên ngoài bị rollback hoặc phát ra ngoại lệ muộn |

Luồng tiếp tục (continuation) đúng thay cho T7–T9:

| Bước | Command B |
| --- | --- |
| R1 | Xung đột thoát khỏi ranh giới của lần thử |
| R2 | Tx-B rollback; persistence context đóng |
| R3 | Kiểm tra thời gian chờ/thời hạn (bounded backoff/deadline) bên ngoài transaction |
| R4 | Tx-B2 bắt đầu với persistence context mới |
| R5 | Tải lại stock 1, version 8; kiểm tra lại số lượng |
| R6 | UPDATE kỳ vọng v8 ảnh hưởng 1 dòng |
| R7 | Commit stock 0, version 9; ghi nhận B |

## Kỳ vọng (Expected) so với Thực tế bị lỗi (Broken)

Kỳ vọng:

```text
các lệnh thành công      = A, B
available cuối cùng      = 0
version cuối cùng        = 9
bản ghi reservation      = 2
mỗi lần thử              = transaction độc lập
```

Lỗi:

```text
A commit
B đếm nhiều lần lặp
Các lần lặp của B dùng chung một transaction/context đã hỏng
B không thể tạo ra một commit hợp lệ thứ hai
available/records cuối cùng không khớp với tiến trình retry được báo cáo
```

Một lần lặp (loop iteration) không phải là một lần thử transaction (transaction attempt) nếu ranh giới proxy không được đi qua lại.

> **Nói ngắn gọn:** số lần retry đo lường số lần gọi code; tính đúng đắn cần đo lường số lượng transaction mới và sạch (clean transactions) bắt đầu sau khi transaction trước đó đã rollback.

## Vì sao transaction bị doomed?

Một quá trình flush của JPA thất bại không phải là kết quả có thể phục hồi ở mức câu lệnh (statement-level) để ứng dụng có thể tiếp tục thay đổi trên cùng một persistence context. Lỗi optimistic chỉ ra rằng unit of work đang được tính toán từ một trạng thái lỗi thời; transaction buộc phải rollback.

Có hai lớp trạng thái:

1. **PostgreSQL transaction:** câu lệnh UPDATE có version trả về số lượng dòng là 0, bản thân SQL không nhất thiết đưa PostgreSQL session vào trạng thái hủy bỏ (aborted).
2. **JPA/Hibernate transaction:** lỗi optimistic làm cho unit of work hiện tại không còn hợp lệ để commit và được xử lý như trạng thái rollback-only.

Với lỗi serialization/tiến trình nạn nhân của deadlock, PostgreSQL còn mạnh tay hơn: database hủy transaction; các lệnh tiếp theo sẽ gặp lỗi "current transaction is aborted" cho tới khi rollback.

Chính sách của ứng dụng nên coi cả hai là những lần thử thất bại cần full rollback, không cố gắng phân biệt để sử dụng lại transaction hiện tại.

## Persistence context không phải bộ nhớ đệm có thể đặt lại tùy ý

Sau khi xung đột, persistence context có thể chứa:

- `InventoryItem` với available là `1`, version/trạng thái dựa trên version 7 lỗi thời;
- `ReservationRecord` mới nhưng chưa được commit;
- các hành động đang chờ/thất bại trong action queue của Hibernate;
- trạng thái vòng đời của thực thể không còn đồng bộ với database.

`EntityManager.clear()` sẽ ngắt kết nối (detach) các thực thể đang được quản lý (managed entities), nhưng:

- không rollback JDBC transaction;
- không đặt lại cờ rollback-only;
- không tạo Spring transaction synchronization mới;
- không hoàn tác (undo) các side effect bên ngoài;
- không bảo đảm provider cho phép tiếp tục sau một flush thất bại.

Lệnh `refresh()` cũng không biến một unit of work thất bại thành một unit hợp lệ. Cách dọn dẹp đúng là rollback và đóng context của lần thử đó.

## Isolation ảnh hưởng đến việc tải lại nhưng không sửa ranh giới

Ở mức độ PostgreSQL `READ COMMITTED`, một truy vấn mới trong cùng transaction có thể nhìn thấy version 8 đã commit sau khi context được xóa. Nhưng JPA transaction vẫn là doomed, nên dữ liệu trông có vẻ mới (fresh-looking data) không tạo ra một lần thử có thể commit.

Ở mức độ `REPEATABLE READ`, transaction snapshot còn không nhìn thấy version mới theo cách mà quá trình retry cần. Việc nâng/hạ mức độ cô lập (isolation) không thay đổi yêu cầu cơ bản: transaction mới phải tạo ra một snapshot mới (fresh snapshot).

## Hành vi khóa dòng tại câu lệnh UPDATE có version

Thao tác đọc lạc quan (optimistic read) không giữ khóa dòng (row lock) xuyên suốt quá trình tính toán nghiệp vụ. Khi câu lệnh UPDATE chạy:

- bên cập nhật xin cấp khóa mức dòng;
- nếu A đang cập nhật, B có thể phải chờ A commit/rollback;
- sau khi A commit, PostgreSQL đánh giá lại predicate của B;
- `version = 7` không còn khớp, số lượng dòng bị ảnh hưởng là 0;
- khóa được giải phóng khi transaction B rollback.

Không có lost update (cập nhật bị mất): quá trình ghi dữ liệu cũ (stale write) bị từ chối. Lỗi SPR-006 nằm ở luồng phục hồi sau xung đột, không nằm ở phần phát hiện optimistic.

## Xung đột xuất hiện ở đâu?

Không có lệnh `flush()` tường minh, phương thức đích có thể trả về kết quả trước khi SQL chạy.
Transaction interceptor thực hiện flush/commit sau khi hàm đích hoàn thành; khối `try/catch` cục bộ trong phương thức không bắt được ngoại lệ.

Có lệnh `flush()` tường minh, khối catch thấy xung đột sớm hơn nhưng vẫn ở bên trong transaction bên ngoài. Muốn retry đúng, ngoại lệ phải tiếp tục đi qua transaction interceptor để quá trình rollback hoàn tất, rồi mới được retry coordinator bắt lấy.

Đường dẫn ngoại lệ hợp lệ (Correct exception path):

```text
attempt target
  -> flush conflict
  -> transaction interceptor catches
  -> rollback/close context
  -> rethrow
  -> retry coordinator catches
  -> classify/backoff
  -> next proxy call creates new Tx
```

## Thứ tự Advisor

Khi `@Retryable` và `@Transactional` cùng áp dụng, chuỗi proxy/advisor sẽ quyết định ngữ nghĩa:

```text
Đúng:
RetryInterceptor
  -> TransactionInterceptor
       -> target attempt

Sai:
TransactionInterceptor
  -> RetryInterceptor
       -> target attempt(s)
```

Việc các annotation nằm cạnh nhau không làm cho thứ tự trở nên hiển nhiên. Có thể cấu hình thứ tự này, nhưng chia tách các bean coordinator/worker riêng rẽ sẽ dễ review và test hơn; ranh giới đối tượng làm cho mỗi lần gọi retry đều phải đi qua transaction proxy.

## Lan truyền (Propagation) và transaction bên ngoài

Worker thực hiện lần thử sử dụng `REQUIRED` để tạo transaction mới khi coordinator không có transaction (non-transactional). Nếu một phía gọi (caller) vô tình mở một transaction bên ngoài, worker sẽ join vào và lại làm mất ranh giới của lần thử.

Các biện pháp bảo vệ (guardrails):

- điểm vào của coordinator không nên là transactional;
- worker có thể dùng `REQUIRES_NEW` nếu ngữ nghĩa lần thử độc lập (independent attempt) được chấp nhận và phía gọi bên ngoài thực sự có thể tồn tại;
- hoặc dùng `TransactionTemplate` với lan truyền tường minh;
- kiểm thử kiến trúc/tích hợp (architecture/integration test) phải xác nhận định danh của physical transaction cho mỗi lần thử.

`REQUIRES_NEW` có chi phí riêng: tạm dừng context bên ngoài, dùng thêm connection và thực hiện commit độc lập. Đừng thêm nó mà không phân tích tính nguyên tử (atomicity).

## Phân loại retry

Các lỗi có thể retry:

- xung đột version lạc quan (optimistic version conflict);
- lỗi PostgreSQL serialization, SQLSTATE `40001`;
- tiến trình nạn nhân của deadlock, SQLSTATE `40P01`, nếu thao tác an toàn;
- một số lỗi kết nối/khóa tạm thời được chọn theo một chính sách cụ thể (explicit policy).

Không thể retry:

- không đủ stock trên trạng thái vừa tải lại;
- số lượng không hợp lệ;
- command bị trùng lặp khi đã có kết quả ghi nhận cuối cùng;
- ràng buộc nghiệp vụ không thể thỏa mãn;
- lỗi lập trình/mapping;
- hết thời gian chờ/bị hủy (deadline/cancellation).

Quá trình phân loại dựa trên nguyên nhân/SQLSTATE cụ thể nhất, không dùng catch-all `RuntimeException`.

## Backoff, độ trễ ngẫu nhiên (jitter) và khuếch đại retry

Việc retry ngay lập tức trên các SKU có lượng truy cập cao sẽ làm các tiến trình thua (losers) quay lại cùng lúc, tạo thêm xung đột.
Chính sách (policy) cần:

- `maxAttempts` hữu hạn;
- thời hạn thao tác (operation deadline) bao phủ tất cả các lần thử;
- backoff tăng dần (exponential) hoặc thích ứng (adaptive);
- jitter để làm mất đồng bộ (desynchronize) các tiến trình;
- lan truyền tín hiệu hủy bỏ/ngắt quãng (cancellation/interruption propagation);
- các số liệu (metrics) theo từng lần thử và kết quả cuối cùng.

Thời gian chờ (backoff) phải nằm ngoài transaction để không giữ connection, snapshot hoặc khóa (locks) trong lúc chờ. Không có một khoảng thời gian chờ phổ quát; cần điều chỉnh (tune) từ ngân sách độ trễ và mức độ tranh chấp quan sát được, không sao chép một con số như là một hằng số tính đúng đắn.

## Kiểm tra lại nghiệp vụ (Domain revalidation)

Retry không nên phát lại (replay) các quyết định dựa trên dữ liệu cũ. Lần thử mới phải tải lại và hỏi:

```text
stock còn đủ không?
command đã có bản ghi hoàn tất (terminal record) chưa?
version của giá/quy tắc còn phù hợp không?
thời hạn (deadline) còn không?
```

Nếu command A lấy đơn vị cuối cùng, command B khi retry phải trả về `InsufficientStock` (lỗi không đủ kho), không ép giảm số lượng chỉ vì lần thử đầu từng vượt qua kiểm tra.

## Side effect bên ngoài và tính lũy đẳng (Idempotency)

Không gửi email, thanh toán tiền hoặc publish message không thuộc transaction trước khi lần thử được commit rồi lặp lại chúng. Việc rollback trong database lúc retry sẽ không hoàn tác (undo) side effect bên ngoài.

Sử dụng command ID hoặc unique constraint trong database cho việc phát hiện thông điệp trùng lặp và dùng outbox pattern cho thông điệp sau khi commit khi phù hợp. Tính lũy đẳng (idempotency) và optimistic retry là hai cơ chế kiểm soát khác nhau:

- tính lũy đẳng ngăn cùng một command được áp dụng nhiều lần;
- xung đột version ngăn các thao tác ghi đồng thời (concurrent mutations) riêng biệt ghi đè lên nhau.

## Hành vi khi sự cố (Crash behavior)

- Crash trong một lần thử trước khi commit: connection bị đóng làm transaction rollback.
- Crash sau khi commit nhưng trước khi trả về kết quả (response): bản ghi command / tính lũy đẳng giúp xác định kết quả (outcome).
- Crash trong thời gian chờ (backoff): không transaction nào đang mở.
- Khi retry worker khởi động lại, nó phải đọc trạng thái command/aggregate bền vững (durable), không dựa vào bộ đếm trên bộ nhớ nếu quy trình (workflow) cần retry đáng tin cậy.

## Đa phiên bản (Multi-instance)

JVM lock hoặc bộ đếm retry trên bộ nhớ ở Ứng dụng A không thể phối hợp với Ứng dụng B. Predicate version của PostgreSQL là cơ chế phát hiện xung đột liên ứng dụng (cross-instance conflict detector). Ngân sách retry có thể cục bộ cho từng request đồng bộ, nhưng bảo vệ quá tải toàn hệ thống (global load protection) cần có cơ chế kiểm soát truy cập (admission control), phân mảnh (partitioning) hoặc hàng đợi (queue) phù hợp.

## Khả năng quan sát (Observability)

Cần ghi nhận:

- ID của thao tác/command, khóa aggregate và số thứ tự lần thử;
- version đã tải / version kỳ vọng và loại xung đột;
- trạng thái/định danh hoàn thành của transaction trong các công cụ chẩn đoán;
- thời gian ở bên ngoài transaction dành cho backoff;
- tình trạng hết lần thử (attempts exhausted), quá hạn (deadline exceeded) và kết quả nghiệp vụ cuối cùng;
- tỷ lệ xung đột theo các khóa nóng (hot key);
- `UnexpectedRollbackException` hoặc các lỗi hệ quả của trạng thái aborted-transaction.

Không coi thông báo "lần thử 2 bắt đầu" là bằng chứng của một lượt retry sạch. Bằng chứng tốt nhất là quá trình rollback trước đó đã hoàn tất, transaction mới đã bắt đầu, version được tải lại và kết quả nghiệp vụ được commit chính xác.
