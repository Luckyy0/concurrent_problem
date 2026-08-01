# Phân Tích Chuyên Sâu: Chu Trình Chờ, Giao Dịch Nạn Nhân và Quá Trình Rollback (Wait-for graph, victim and rollback analysis)

## 1. Trạng thái khởi điểm (Initial state)

Cơ sở dữ liệu lưu trữ hai bản ghi tài khoản đã được commit:

```text
Tài khoản 101 (A): Số dư = 1_000
Tài khoản 202 (B): Số dư = 1_000
```

Hai luồng xử lý đồng thời khởi tạo giao dịch:
- Luồng T1 thực hiện lệnh chuyển tiền `transfer(101, 202, 100)`.
- Luồng T2 thực hiện lệnh chuyển tiền `transfer(202, 101, 70)`.

Mỗi luồng tạo một kết nối vật lý độc lập (physical connection) và khởi tạo một giao dịch (transaction) ở mức cô lập `READ COMMITTED` thông qua Spring proxy.

## 2. Diễn biến hình thành Deadlock trong PostgreSQL (Timeline of deadlock creation)

| Bước | Luồng T1 (Chuyển A -> B) | Luồng T2 (Chuyển B -> A) | Đồ thị chờ khóa (Wait-for graph) |
| ---: | --- | --- | --- |
| 1 | Thực thi `BEGIN` | Thực thi `BEGIN` | Không có tranh chấp |
| 2 | Cấp khóa độc quyền tài khoản A (`FOR UPDATE`) | | T1 giữ khóa A |
| 3 | | Cấp khóa độc quyền tài khoản B (`FOR UPDATE`) | T2 giữ khóa B |
| 4 | Yêu cầu khóa B và chuyển sang trạng thái CHỜ | | T1 chờ T2 giải phóng B |
| 5 | | Yêu cầu khóa A và chuyển sang trạng thái CHỜ | Hình thành chu trình chờ (Circular wait) |
| 6 | Deadlock detector phát hiện chu trình | Deadlock detector phát hiện chu trình | PostgreSQL chọn một giao dịch làm nạn nhân (victim) |
| 7 | T1 tiếp tục chờ | T2 bị hủy (nhận mã lỗi `40P01`) | Giao dịch T2 chuyển sang trạng thái `aborted` |
| 8 | T1 được cấp khóa B | T2 thực hiện rollback, giải phóng khóa B | Chu trình chờ bị phá vỡ |
| 9 | T1 cập nhật số dư, thực thi `COMMIT` | | Tài khoản A còn `900`, B tăng lên `1_100` |

Trong tình huống trên, giao dịch T1 cũng có thể bị chọn làm nạn nhân thay vì T2. Lập trình viên không thể và không nên thiết kế logic ứng dụng dựa trên giả định rằng một kết nối cụ thể nào đó sẽ luôn là nạn nhân hoặc luôn thành công.

> **Ghi chú quan trọng:** PostgreSQL giải quyết deadlock bằng cách chủ động hủy (abort) một giao dịch để các giao dịch khác có thể tiếp tục. Trách nhiệm xử lý sau khi giao dịch bị hủy (thử lại hay trả lỗi về cho người dùng) hoàn toàn thuộc về mã nguồn ứng dụng (application code), không phải của cơ sở dữ liệu.

## 3. Mong đợi và Thực tế (Expected vs. Actual outcomes)

| Yếu tố | Mong đợi trong thiết kế lỗi | Thực tế diễn ra |
| --- | --- | --- |
| Cơ chế khóa (Locking) | Hai lệnh chuyển tiền tự động xếp hàng tuần tự. | Mỗi lệnh giữ một khóa và yêu cầu khóa còn lại, gây ra bế tắc. |
| Tiến độ (Progress) | Cả hai lệnh đều ghi nhận thành công. | Một giao dịch bị hủy và trả về lỗi `40P01`. |
| Tính nguyên tử (Atomicity) | Thao tác trừ tiền và cộng tiền luôn đi kèm với nhau. | PostgreSQL tự động xử lý rollback dữ liệu của giao dịch nạn nhân, nhưng ứng dụng phải quản lý logic phục hồi. |
| Cơ chế thử lại (Retry) | Xử lý `catch` ngoại lệ và thử lại ngay trong giao dịch hiện tại. | Giao dịch hiện tại đã bị vô hiệu hóa; ứng dụng phải khởi tạo một giao dịch mới (fresh attempt) và tải lại dữ liệu. |
| Mở rộng hệ thống (Scale-out) | Sử dụng từ khóa `synchronized` để ngăn lỗi. | Khóa cục bộ không có tác dụng trong môi trường phân tán; bế tắc vẫn xảy ra tại tầng cơ sở dữ liệu. |

Annotation `@Transactional` đảm bảo tính nguyên tử trong việc rollback khi có ngoại lệ xảy ra, nhưng nó không có khả năng sắp xếp lại thứ tự yêu cầu khóa (canonical lock order) hay ngăn chặn deadlock.

## 4. Phân tích đồ thị chờ khóa (Wait-for graph)

Khi luồng T1 thực thi câu lệnh:

```sql
select * from account where id = 202 for update;
```

Tài khoản B đã bị T2 khóa. T1 bắt buộc phải chuyển sang trạng thái chờ cho đến khi T2 kết thúc giao dịch (commit hoặc rollback). Đồng thời, T2 cũng đang chờ T1 giải phóng khóa tài khoản A. Quá trình này tạo ra một đồ thị phụ thuộc vòng tròn (circular dependency):

```text
T1 (Giữ khóa A) ──────── Yêu cầu khóa B (đang bị T2 giữ)
  ▲                                                   │
  └──────── T2 (Giữ khóa B) yêu cầu khóa A ───────────┘
```

PostgreSQL không liên tục kiểm tra deadlock để tiết kiệm tài nguyên. Quá trình kiểm tra chỉ được kích hoạt sau khi một tiến trình chờ đợi vượt quá khoảng thời gian `deadlock_timeout` (mặc định là 1 giây). Khi phát hiện chu trình, bộ dò tìm sẽ phát tín hiệu hủy bỏ (abort) một giao dịch.

Cần lưu ý rằng `deadlock_timeout` là thông số dùng để kích hoạt bộ kiểm tra lỗi, không phải là hạn mức thời gian của nghiệp vụ (business deadline). Giảm tham số này không ngăn chặn được deadlock, mà chỉ làm tăng tần suất tiêu thụ CPU cho quá trình kiểm tra.

## 5. Các loại khóa tham gia tranh chấp (Lock types involved)

Khi thực thi `SELECT ... FOR UPDATE`, cơ sở dữ liệu sẽ:

- Cấp khóa chia sẻ cấp bảng (`ROW SHARE` lock) trên bảng `account`.
- Cấp khóa độc quyền cấp dòng (row-level lock) trên các bản ghi thỏa mãn điều kiện.
- Nếu bản ghi đã bị khóa bởi một giao dịch khác, luồng yêu cầu sẽ chuyển sang trạng thái chờ giao dịch nắm giữ khóa (transaction ID) kết thúc.

Khóa bảng `ROW SHARE` của T1 và T2 không xung đột với nhau. Sự cố deadlock phát sinh hoàn toàn từ sự cạnh tranh ở mức khóa dòng và chờ giao dịch.

Để theo dõi, quản trị viên cơ sở dữ liệu có thể truy vấn các view hệ thống như `pg_stat_activity`, `pg_locks` và `pg_blocking_pids()`. Các thông tin này giúp xác định mối quan hệ chặn khóa giữa các tiến trình (PIDs) trong thời gian thực.

## 6. MVCC và Statement Snapshot

Ở mức cô lập `READ COMMITTED`, PostgreSQL tạo một snapshot mới tại thời điểm bắt đầu của mỗi câu lệnh. Tuy nhiên, lệnh `SELECT ... FOR UPDATE` không chỉ đọc phiên bản dữ liệu hiện hành (visible version), mà còn yêu cầu khóa cấp dòng.

Lịch sử tương tác dữ liệu:

- Snapshot của T1 khi truy vấn tài khoản A sẽ đọc được số dư là `1_000`.
- Snapshot của T2 khi truy vấn tài khoản B cũng đọc được số dư là `1_000`.
- Khi các luồng thực thi lệnh truy vấn thứ hai, chúng đều nhận thức được sự tồn tại của bản ghi, nhưng phải chờ để cấp khóa.
- Các lệnh truy vấn thông thường (không kèm `FOR UPDATE`) vẫn có thể đọc dữ liệu đã commit mà không bị chặn, và không tham gia vào chu trình deadlock.
- Khi giao dịch nắm giữ khóa kết thúc, giao dịch đang chờ sẽ nhận được khóa và đọc lại phiên bản dữ liệu mới nhất đã được commit (re-evaluate) để đảm bảo tính nhất quán trước khi phản hồi về ứng dụng.

Việc tăng cấp độ cô lập lên `REPEATABLE READ` hoặc `SERIALIZABLE` không giải quyết được vấn đề khóa ngược chiều (opposite ordering). Ngược lại, chúng có thể dẫn đến các lỗi tranh chấp (serialization failures) phức tạp hơn. (Tham khảo tài liệu `DB-009`).

## 7. Trạng thái Giao dịch bị hủy (Aborted transaction behavior)

Khi PostgreSQL quyết định hủy một giao dịch do deadlock, nó sẽ trả về mã lỗi `40P01`. Toàn bộ thao tác thực hiện trong giao dịch đó (bao gồm các cập nhật dữ liệu) đều bị hủy bỏ.

Nếu ứng dụng tiếp tục gửi bất kỳ lệnh SQL nào (ví dụ: `SELECT 1`) trên cùng một kết nối của giao dịch đã bị hủy, PostgreSQL sẽ từ chối bằng lỗi:

```text
25P02: current transaction is aborted, commands ignored until end of transaction block
```

Lớp quản lý giao dịch của Spring (Transaction Interceptor) sẽ nắm bắt ngoại lệ này và tự động phát lệnh `ROLLBACK`. Sau khi rollback:

- Tất cả khóa cấp dòng, cấp bảng và cấp giao dịch đều được giải phóng.
- Persistence context (bộ đệm của Hibernate) trong giao dịch hiện tại bị vô hiệu hóa.
- Giao dịch đang chờ (trong trường hợp này là T1) sẽ được cấp khóa và tiếp tục xử lý.
- Nếu ứng dụng muốn thực hiện retry, nó bắt buộc phải khởi tạo một giao dịch hoàn toàn mới (fresh transaction context) và tải lại toàn bộ trạng thái dữ liệu.

Phương thức `EntityManager.clear()` chỉ làm sạch persistence context, không thay thế được thao tác `ROLLBACK` của JDBC. Ứng dụng không được bắt ngoại lệ và tiếp tục sử dụng logic nghiệp vụ bên trong cùng một phương thức `@Transactional` đã thất bại.

## 8. Kết quả sau tranh chấp (Outcomes of contention)

### Giao dịch thành công (Committed transaction)

Giao dịch giành được khóa (T1) sẽ tiến hành kiểm tra lại số dư, thực hiện các lệnh `UPDATE` tương ứng và thực thi `COMMIT`. Các thay đổi (cộng và trừ tiền) sẽ được áp dụng nguyên tử và hiển thị cho các luồng xử lý khác.

### Giao dịch bị hủy không thực hiện thử lại (Victim rollback without retry)

Nếu không có cơ chế retry, giao dịch T2 bị hủy và ứng dụng trả về lỗi cho người dùng. Dữ liệu tài khoản B và A không bị ảnh hưởng bởi T2. Mọi thay đổi dự kiến đều bị loại bỏ. Chú ý, ứng dụng tuyệt đối không được ghi nhận giao dịch là thành công hoặc gửi thông báo (email) trước khi có xác nhận commit từ cơ sở dữ liệu.

### Giao dịch bị hủy thực hiện thử lại (Victim rollback with retry)

Ứng dụng khởi tạo một giao dịch mới cho T2, với persistence context mới và dữ liệu được nạp lại. T2 lúc này sẽ thấy các thay đổi (nếu có) từ T1 và phải thực hiện lại toàn bộ quá trình thẩm định nghiệp vụ (business validation) trước khi tiến hành cập nhật.

Nếu các dữ liệu đầu vào của T1 và T2 hoàn toàn hợp lệ, việc thực hiện lại T2 sẽ kết thúc thành công:

```text
Hoàn tất T1: A giảm còn 900, B tăng lên 1100
Hoàn tất T2 (Retry): A tăng lên 970, B giảm còn 1030
(Tổng số dư luôn được bảo toàn là 2000)
```

Điều quan trọng nhất là tính nguyên tử (atomicity) và tính bảo toàn dữ liệu (conservation) luôn được hệ thống duy trì chặt chẽ.

## 9. Phân biệt Deadlock và Timeout

| Trạng thái lỗi | SQLSTATE | Phân tích nguyên nhân | Hướng xử lý đề xuất |
| --- | --- | --- | --- |
| Phát hiện Deadlock (Deadlock detected) | `40P01` | Cơ sở dữ liệu phát hiện chu trình chờ và hủy một giao dịch. | Rollback giao dịch hiện tại; Khởi tạo giao dịch mới để thử lại (nếu nghiệp vụ cho phép). |
| Quá hạn chờ khóa (Lock timeout) | `55P03` | Quá thời gian quy định (`lock_timeout`) mà không lấy được khóa; Không phát hiện chu trình chờ. | Rollback giao dịch; Phân tích các tác vụ chiếm giữ khóa kéo dài hoặc tối ưu hiệu năng truy vấn. |
| Giao dịch bị gián đoạn (Statement canceled) | `57014` | Quá thời gian thực thi lệnh (`statement_timeout`) hoặc bị hủy do người quản trị. | Rollback giao dịch; Tối ưu hóa truy vấn để đáp ứng yêu cầu về thời gian thực thi. |
| Lỗi phân lập tuần tự (Serialization failure) | `40001` | Phát hiện xung đột dữ liệu khi áp dụng các mức cô lập nghiêm ngặt (SSI). | Rollback và tiến hành cơ chế Retry toàn diện (Tham khảo case DB-009). |

Ứng dụng cần phân loại rõ ràng các mã lỗi kỹ thuật (technical contention) này, không nhầm lẫn với các lỗi nghiệp vụ (business validation error) như "tài khoản không đủ tiền".

## 10. Trình tự khóa theo chuẩn giải quyết bế tắc (Why canonical order prevents deadlocks)

Áp dụng quy tắc xếp hạng tài nguyên theo định danh bất biến (stable unique identifier):

```text
First Lock  = Khóa tài khoản có min(fromId, toId)
Second Lock = Khóa tài khoản có max(fromId, toId)
```

Với quy tắc này, bất kể lệnh chuyển tiền từ A sang B hay B sang A, hệ thống đều yêu cầu khóa tài khoản A (ID 101) trước, sau đó mới yêu cầu khóa B (ID 202).

- Nếu luồng T1 đang giữ khóa A, luồng T2 sẽ phải chờ tại bước yêu cầu khóa A.
- Do T2 chưa nắm giữ khóa B, luồng T1 có thể lấy khóa B một cách an toàn và hoàn tất giao dịch.
- Đồ thị chờ sẽ chỉ có một chiều (T2 chờ T1), phá vỡ hoàn toàn chu trình khép kín.

Giải pháp này chỉ hiệu quả khi toàn bộ hệ thống sử dụng chung một tiêu chuẩn sắp xếp (comparator) và một hệ quy chiếu định danh. Việc áp dụng không nhất quán (ví dụ: dùng ID nội bộ ở một service và mã thẻ ở một service khác) sẽ phá vỡ trật tự khóa và gây ra deadlock trở lại.

## 11. Xử lý ranh giới Hibernate và Spring (Spring/Hibernate boundaries)

Sử dụng `@Lock(PESSIMISTIC_WRITE)` trong Spring Data JPA sẽ phát sinh lệnh `SELECT ... FOR UPDATE` ngay tại thời điểm gọi hàm. Tuy nhiên, hành vi hoãn cập nhật (deferred dirty checking) của Hibernate sẽ chỉ phát sinh các lệnh `UPDATE` tại thời điểm xả bộ đệm (flush/commit).

Nếu mã nguồn thay đổi đối tượng (trừ tiền) và hệ thống kích hoạt tự động xả dữ liệu (auto-flush) trước khi yêu cầu khóa tài khoản đích, trình tự sinh ra các lệnh SQL có thể thay đổi và gây bế tắc tại lệnh UPDATE. Việc mã hóa cần được thực hiện minh bạch để kiểm soát trình tự phát sinh SQL.

Ngoại lệ cấp thấp từ JDBC thường được Spring bọc lại trong `CannotAcquireLockException` hoặc `DataAccessException`. Để đảm bảo cơ chế retry hoạt động chính xác, bộ điều phối (retry coordinator) phải kiểm tra thuộc tính `SQLSTATE` của ngoại lệ gốc (`SQLException#getSQLState()`) để xác định mã `40P01` thay vì chỉ phụ thuộc vào cấu trúc class của ngoại lệ Spring.

## 12. Rủi ro xử lý ngoại vi (External side effects)

Khi PostgreSQL thực thi rollback, mọi tương tác với cơ sở dữ liệu bị thu hồi, nhưng các thao tác I/O từ xa (như gửi email, gọi API đối tác) không thể hoàn tác.

Nếu luồng T2 gọi một API bên thứ ba trước khi commit, và sau đó bị PostgreSQL hủy giao dịch:
- Giao dịch cơ sở dữ liệu sẽ quay về trạng thái ban đầu.
- Tuy nhiên, hệ thống đối tác hoặc người dùng đã nhận được thông báo "chuyển tiền thành công".
- Nếu luồng T2 thực hiện retry, hệ thống có thể gửi thông báo lần hai, dẫn đến trùng lặp dữ liệu (duplicate side effect).

Thiết kế chuẩn yêu cầu:
- Tuyệt đối không thực hiện các cuộc gọi ngoại vi bên trong phạm vi giao dịch cơ sở dữ liệu.
- Sử dụng các mẫu thiết kế an toàn như Outbox pattern để lưu trữ sự kiện cùng với giao dịch, đảm bảo tính nguyên tử (atomicity) giữa việc xử lý dữ liệu và gửi thông báo.

## 13. Tác động của gián đoạn kết nối (Process crash and lost connections)

Nếu tiến trình ứng dụng gặp sự cố (crash) hoặc mạng bị đứt, kết nối đến cơ sở dữ liệu sẽ bị ngắt. PostgreSQL sẽ tự động rollback giao dịch chưa commit và giải phóng các khóa.

Trong trường hợp sự cố xảy ra ngay sau khi ứng dụng phát lệnh `COMMIT` nhưng trước khi nhận được phản hồi, ứng dụng sẽ rơi vào trạng thái không xác định (ambiguous outcome). Để phòng ngừa rủi ro xử lý kép khi ứng dụng phục hồi, tất cả các tác vụ cập nhật quan trọng phải tích hợp cơ chế khóa trung gian (idempotency key) và khả năng đối soát trạng thái (status lookup).

## 14. Môi trường triển khai đa máy chủ (Multi-instance concurrency)

Các giải pháp đồng bộ hóa dựa trên bộ nhớ cục bộ (như `synchronized` hay `ReentrantLock` trong Java) chỉ quản lý trạng thái trong phạm vi một JVM duy nhất. Trong hệ thống phân tán, các yêu cầu xử lý từ nhiều máy chủ khác nhau vẫn có thể dẫn đến bế tắc tại cơ sở dữ liệu.

Do đó, cơ sở dữ liệu phân tán cần áp dụng các chuẩn mực:
- Trình tự khóa chuẩn (Canonical row order) phải là quy tắc thiết kế toàn hệ thống.
- Các khoảng thời gian chờ (timeout) tại Connection Pool phải được điều chỉnh tương thích với `lock_timeout` của cơ sở dữ liệu để ngăn chặn hiện tượng treo kết nối kéo dài.

## 15. Xác định nguyên nhân gốc rễ (Root cause mapping)

| Thành phần | Biểu hiện / Hành vi |
| --- | --- |
| Lớp Ứng dụng (Application code) | Yêu cầu tài nguyên theo thứ tự động (phụ thuộc vào luồng dữ liệu đầu vào), gây ra chu trình chờ (circular wait). |
| Lớp Quản lý Giao dịch (Spring) | Cấu trúc ranh giới giao dịch chưa tối ưu, không tách biệt rõ ràng tiến trình xử lý và tiến trình thử lại. |
| Lớp ORM (Hibernate/JPA) | Che khuất lệnh phát sinh SQL, phức tạp hóa quá trình bắt và phân tích ngoại lệ. |
| Lớp Cơ sở dữ liệu (PostgreSQL) | Phát hiện bế tắc, hủy một giao dịch để giải phóng hệ thống, trả về mã lỗi `40P01`. |

Nguyên nhân cốt lõi không phải do cơ sở dữ liệu phản hồi chậm, mà do kiến trúc ứng dụng yêu cầu nhiều tài nguyên phân tán mà không tuân thủ một trật tự sắp xếp toàn cục (total order).

## 16. Yêu cầu giám sát (Observability requirements)

Hệ thống giám sát cần thiết lập các tiêu chí sau:

- Theo dõi số lượng sự kiện `deadlock_detected` (metric `deadlock_detected_total`). Không đưa ID tài khoản vào nhãn (label) của metric để tránh quá tải do độ phân tán (high-cardinality) quá lớn.
- Ghi log chi tiết tiến trình retry: Lần thử (attempt number), kết quả, lý do và thời gian xử lý.
- Cung cấp dữ liệu chi tiết về SQLSTATE, Transaction ID và bối cảnh xảy ra lỗi để thuận tiện truy vết.
- Cấu hình thông số `log_lock_waits = on` trong PostgreSQL để ghi nhận thông tin chậm trễ khi lấy khóa.
- Triển khai các kịch bản tự động truy vấn `pg_stat_activity`, `pg_locks` và `pg_blocking_pids` trong quá trình xảy ra sự cố để phục vụ công tác phân tích.
- Theo dõi các số liệu liên quan đến Connection Pool (pending, active, timeout) để có biện pháp can thiệp kịp thời.

Mã lỗi hệ thống (deadlock) cung cấp thông tin quý giá cho việc tái cấu trúc (refactoring). Khả năng giám sát và xử lý lỗi chuyên sâu là cơ sở vững chắc để thiết kế kiến trúc xử lý tài chính an toàn và nhất quán.
