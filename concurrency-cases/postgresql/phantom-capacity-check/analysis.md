# Phân Tích Dưới Lăng Kính MVCC (Phân tích MVCC predicate và capacity invariant)

## 1. Trạng Thái Ban Đầu (Initial state)

```text
Bể xử lý processing_pool(P-42)
  Sức chứa: capacity = 10

Danh sách phân bổ slot_allocation
  Đang có: 9 dòng thỏa điều kiện pool_id=P-42 and status=ACTIVE
```

Các request A và B đều hợp lệ và sử dụng các ID khác nhau. Nếu chạy tuần tự, mỗi request đều có thể được cấp phát vào slot cuối cùng. Tuy nhiên, khi chạy đồng thời, chúng sẽ gây ra lỗi.

## 2. Hai Kịch Bản Tương Tranh Cần Phân Biệt (Hai timelines cần phân biệt)

### Kịch Bản 1 — Hiện Tượng Dòng Bóng Ma (Visible phantom ở `READ COMMITTED`)

| Bước | Đọc Dữ Liệu (Reader A) | Ghi Dữ Liệu (Writer B) |
| --- | --- | --- |
| 1 | `BEGIN READ COMMITTED` | |
| 2 | Đếm số lượng S1 = `9` | |
| 3 | | INSERT một dòng ACTIVE mới |
| 4 | | `COMMIT` thành công |
| 5 | Đếm số lượng lần hai S2 = `10` | |

Ở mức `READ COMMITTED`, khi giao dịch A thực hiện lệnh đếm thứ hai, nó sẽ nhìn thấy dòng mới mà B vừa commit (vì dòng này thỏa mãn điều kiện lọc). Sự thay đổi kết quả của câu lệnh `COUNT` do dòng mới được thêm vào này chính là hiện tượng dòng bóng ma (phantom row).

### Kịch Bản 2 — Vi Phạm Kiểm Tra Sức Chứa (Capacity violation)

| Bước | Cấp Phát (Allocator A) | Cấp Phát (Allocator B) |
| --- | --- | --- |
| 1 | BẮT ĐẦU GIAO DỊCH (BEGIN) | BẮT ĐẦU GIAO DỊCH (BEGIN) |
| 2 | Đếm kết quả = 9 | |
| 3 | | Đếm kết quả = 9 |
| 4 | Quyết định ACCEPT | Quyết định ACCEPT |
| 5 | Chèn dòng A (INSERT) | Chèn dòng B (INSERT) |
| 6 | Đóng giao dịch COMMIT | Đóng giao dịch COMMIT |
| 7 | | Tổng số lượng cuối cùng = 11 |

Xung đột kiểm tra sức chứa (Capacity race) này xảy ra mà **không cần** bất kỳ giao dịch nào phải đọc lại lần thứ hai. Căn nguyên (Root cause) là cả hai luồng xử lý cùng dựa trên một dữ liệu trạng thái (9 < 10) để đưa ra quyết định, sau đó cùng thực hiện thao tác INSERT. Vì chúng INSERT các dòng khác nhau (khóa chính khác nhau), database không thể tự phát hiện ra xung đột ghi (write conflict) để ngăn chặn.

> **Giải thích thuật ngữ:** *Dòng bóng ma* (phantom) mô tả sự thay đổi của tập kết quả trả về do dữ liệu mới xuất hiện. *Vi phạm kiểm tra tổng thể* (invariant violation) là hậu quả khi hai luồng xử lý cùng sử dụng một trạng thái cũ để cập nhật dữ liệu mới mà không có cơ chế khóa (authoritative write conflict) đủ để bảo vệ quy tắc tổng (ví dụ: tổng dòng phải <= 10).

## 3. Điều Kiện Lọc Chính Là Dữ Liệu Dùng Chung (Predicate là shared state thật sự)

Ứng dụng kiểm tra sức chứa bằng một câu lệnh:

```text
COUNT(các dòng có pool=P-42 và status=ACTIVE) < pool.capacity
```

Trong cách thiết kế thiếu an toàn này, **không có bất kỳ một dòng cụ thể nào** lưu trữ thông tin về "slot trống cuối cùng". Hệ thống PostgreSQL MVCC chỉ nhìn thấy các thao tác độc lập:

- Giao dịch A đọc một số phiên bản dòng (tuple versions) cũ;
- Giao dịch B cũng đọc các phiên bản dòng cũ đó;
- Giao dịch A chèn một dòng hoàn toàn mới (new tuple);
- Giao dịch B chèn một dòng hoàn toàn mới khác.

Vì ID là duy nhất nên cả dữ liệu dòng và các chỉ mục (Index) đều không gây ra xung đột (không bị dẫm chân lên nhau). Kể cả khi bạn thiết lập `CHECK constraint` trên bảng `slot_allocation`, nó cũng chỉ kiểm tra được dữ liệu của chính dòng đang được chèn, chứ không thể kiểm tra chéo các dòng khác (aggregate rows).

## 4. MVCC Snapshot Ở Mức READ COMMITTED (MVCC snapshots ở `READ COMMITTED`)

Khi lệnh `COUNT` được thực thi, PostgreSQL tạo ra một snapshot tại thời điểm bắt đầu câu lệnh. Nếu cả hai lệnh `COUNT` của A và B đều chạy trước khi bất kỳ thao tác `COMMIT` nào diễn ra:

```text
A đọc bằng Snapshot(A) -> Thấy 9 dòng ACTIVE
B đọc bằng Snapshot(B) -> Thấy 9 dòng ACTIVE
```

Thao tác `INSERT` chưa được commit của A hoàn toàn vô hình đối với lệnh `SELECT` của B. Ngay khi A commit, B mới có khả năng nhìn thấy dòng dữ liệu đó, nhưng ứng dụng đã thực thi xong lệnh `COUNT` và đang dùng kết quả cũ để ra quyết định mà không kiểm tra lại (revalidate).

## 5. Hành Vi Khóa Trong PostgreSQL (Lock behavior)

### Đọc dữ liệu thông thường (Plain predicate read)

Lệnh `COUNT` thông thường không tạo ra bất kỳ khóa cấp dòng (row lock) nào để ngăn các giao dịch khác chèn dữ liệu. Khóa `ACCESS SHARE` mà lệnh SELECT tạo ra chỉ áp dụng ở cấp độ bảng (table-level) để ngăn các thay đổi về cấu trúc bảng (schema changes), chứ không ngăn cản các lệnh INSERT.

### Chèn dữ liệu mới (INSERT)

Mỗi lệnh `INSERT` chỉ tạo khóa trên chính dòng dữ liệu mới mà nó sinh ra, cho đến khi giao dịch kết thúc. Các khóa chính (Primary keys) khác biệt đảm bảo rằng hai lệnh INSERT sẽ không bao giờ khóa lẫn nhau.

### Khóa các dòng hiện có (Lock existing rows)

Lệnh `SELECT ... FOR UPDATE` có tác dụng tạo khóa độc quyền cấp dòng (exclusive row locks) trên **những dòng đã thực sự tồn tại** tại thời điểm truy vấn. Nó không thể khóa những dòng vô hình sẽ được chèn trong tương lai. Do đó, để tạo ra điểm tranh chấp (contention point), chúng ta phải áp dụng khóa lên một đối tượng đã tồn tại sẵn, ví dụ như dòng cha (shared parent row) hoặc các dòng slot trống được tạo sẵn (pre-created slot rows).

## 6. Góc Khuất Của Mức REPEATABLE READ (PostgreSQL `REPEATABLE READ` nuance)

Mức cô lập `REPEATABLE READ` trong PostgreSQL duy trì một snapshot nhất quán trong suốt vòng đời của giao dịch. Điều này khiến cho lệnh `COUNT` thứ hai (nếu có) sẽ **không bao giờ** nhìn thấy dòng bóng ma:

```text
A thực hiện COUNT #1 -> Kết quả: 9
B thực hiện INSERT và COMMIT 
A thực hiện COUNT #2 -> Kết quả vẫn là: 9
```

Điều này tưởng chừng như an toàn, nhưng thực tế cả hai giao dịch vẫn chèn dữ liệu dựa trên kết quả đếm cũ, và vì chúng chèn các dòng độc lập, chúng vẫn commit thành công. Đây là một điểm yếu của Snapshot Isolation, được gọi là `serializability anomaly`, khi các thao tác đọc và ghi dựa dẫm vào nhau nhưng không gây ra xung đột ghi trên cùng một dòng (same-row write conflict).

Vì vậy, cần ghi nhớ:

```text
"Không nhìn thấy dòng bóng ma khi đọc lại"
KHÔNG ĐỒNG NGHĨA VỚI VIỆC
"Tổng sức chứa cuối cùng sẽ được bảo đảm" (aggregate capacity luôn đúng).
```

## 7. PostgreSQL SERIALIZABLE Và Xử Lý Xung Đột (PostgreSQL `SERIALIZABLE` và SSI)

Cơ chế Serializable Snapshot Isolation (SSI) của PostgreSQL sử dụng các khóa ảo (metadata SIReadLock) để theo dõi các thao tác đọc. Khác với khóa thực thụ (`FOR UPDATE`) chặn các truy vấn khác, SIReadLock chỉ âm thầm ghi nhận để phát hiện các mẫu xung đột phụ thuộc rủi ro (dangerous dependency structures).

Trong kịch bản tranh giành sức chứa, SSI phát hiện:

```text
A đọc dữ liệu và không thấy dòng mới do B chèn.
B chèn dòng mới vào đúng phạm vi lọc mà A đã đọc.

B đọc dữ liệu và không thấy dòng mới do A chèn.
A chèn dòng mới vào đúng phạm vi lọc mà B đã đọc.
```

Điều này tạo ra cấu trúc xung đột phụ thuộc chéo (rw-antidependencies) dẫn đến chu trình rủi ro. PostgreSQL sẽ tự động bảo vệ hệ thống bằng cách Hủy (abort) một trong các giao dịch, tung ra lỗi serialization failure (`40001`), để đảm bảo các giao dịch còn lại giữ đúng thứ tự tuần tự (serial order).

Ứng dụng của bạn phải có trách nhiệm xử lý khi bị hủy giao dịch:

- Giao dịch bị lỗi (loser) sẽ bị rollback;
- Bắt đầu một giao dịch cơ sở dữ liệu (physical transaction) hoàn toàn mới;
- Tải lại toàn bộ dữ liệu hiện tại (reload current state);
- Thực thi lại logic kiểm tra sức chứa;
- Áp dụng thời gian trễ ngẫu nhiên (`backoff/jitter`) để giảm thiểu xung đột lặp lại;
- Nếu đếm lại thấy sức chứa đã đạt giới hạn (FULL), trả về thông báo lỗi cho người dùng và KHÔNG retry nữa.

Đừng dựa dẫm vào việc giả định giao dịch nào sẽ bị hủy; PostgreSQL quyết định dựa trên tối ưu hệ thống.

## 8. Nguyên Nhân Gốc Rễ Ở Các Tầng (Root cause theo layer)

### Tầng PostgreSQL

Ở cả hai mức `READ COMMITTED` và `REPEATABLE READ`, thao tác count-then-insert không tạo ra xung đột ghi để bảo vệ điều kiện tập hợp (Aggregate Predicate). Đây là hành vi hợp lệ và được thiết kế theo đúng mô hình MVCC (permitted concurrency behavior).

### Tầng Spring Framework

Sử dụng `@Transactional` chỉ gom nhóm các thao tác vào chung một giao dịch để đảm bảo tính nguyên tử (Atomic). Nó **không thay đổi** hành vi cô lập của cơ sở dữ liệu. Vòng lặp `count -> compare -> insert` vẫn gặp lỗi nếu không đi kèm cơ chế khóa hợp lý.

Lưu ý: Nếu một phương thức bên trong được thiết lập `@Transactional(isolation = Isolation.SERIALIZABLE)` và propagation là `REQUIRED`, nhưng được gọi từ một phương thức bên ngoài đã có giao dịch mở trước đó, cấu hình mức độ cô lập của nó sẽ bị phớt lờ (effective isolation của outer transaction sẽ được sử dụng).

### Tầng Hibernate / JPA

Việc gọi `save()` trong Hibernate chỉ đẩy Entity vào bộ đệm Persistence Context. Phải đợi đến lúc thao tác `flush/commit` diễn ra thì lệnh INSERT mới được gửi xuống DB. Hibernate không có tính năng tự động nhận diện hay bảo vệ các quy tắc tính toán sức chứa nghiệp vụ như đếm số lượng slot.

### Tầng Application Model

Vấn đề cốt lõi là thiết kế hệ thống đang tính toán tổng sức chứa dựa trên các dòng con (child rows) mà **không có** một điểm chốt chặn dữ liệu (authoritative point) độc nhất nào. Không có biến đếm nguyên tử (atomic counter), không khóa dòng cha (parent row lock), không tạo sẵn slot (finite slot identity) và cũng không kích hoạt `SERIALIZABLE` để kiểm soát lỗi.

## 9. Kỳ Vọng và Thực Tế Trái Ngược (Expected và actual)

Kỳ vọng chuẩn mực (Expected):

```text
Số request được chấp nhận (accepted) = 1
Số request bị từ chối / đầy (full) = 1
Tổng số active cuối cùng (final active) = 10
```

Hậu quả thực tế khi bị lỗi (Broken):

```text
Số request được chấp nhận (accepted) = 2
Số request bị từ chối / đầy (full) = 0
Tổng số active cuối cùng (final active) = 11
Lỗi diễn ra âm thầm, không có ngoại lệ (Exception) nào được văng ra.
```

## 10. Xử Lý Khi Giao Dịch Chậm Trễ Và Lỗi (Commit và rollback)

### Lỗi trên một INSERT (Một INSERT rollback)

Nếu một giao dịch INSERT thất bại, dòng dữ liệu đó sẽ bị rollback và vô hình với bên ngoài, tổng sức chứa có thể quay về `10`. Tuy nhiên, sự "đúng đắn" ngẫu nhiên do thất bại này không thay thế cho một kiến trúc chính xác. Hệ thống phải đảm bảo tính đúng đắn dựa trên thiết kế, không phải dựa vào tỷ lệ xuất hiện lỗi.

### Cả hai giao dịch đều Commit thành công

Vi phạm sức chứa đã được lưu vĩnh viễn (durable) vào cơ sở dữ liệu. Khi giao dịch đã commit, các thao tác kiểm tra (validation) cục bộ không thể thu hồi. Bạn chỉ có thể sửa sai bằng quy trình rà soát toàn cục (global reconciliation).

### Lỗi xảy ra sau khi đếm (Counter claim rồi allocation INSERT fail)

Khi sử dụng bộ đếm nguyên tử (phương pháp giải quyết tối ưu số 1), cả lệnh tăng biến đếm (counter increment) và lệnh chèn dữ liệu (INSERT) phải nằm trong cùng một giao dịch. Bất kỳ lỗi Runtime/DataAccess nào cũng phải rollback toàn bộ thao tác, qua đó biến đếm sẽ quay về như cũ. Việc cố tình "catch" lỗi và ỉm đi sẽ làm biến đếm bị lệch (counter drift).

### Khôi phục sức chứa khi xả slot (Release)

Thao tác xả slot phải là quá trình 1 chiều: Cập nhật trạng thái allocation `ACTIVE -> RELEASED` có điều kiện. **Chỉ khi** quá trình cập nhật trạng thái này thành công, bạn mới tiến hành giảm bộ đếm (counter). Chạy các lệnh giảm bộ đếm một cách mù quáng (ví dụ khi có thao tác retry bất thường) sẽ gây ra tình trạng giảm kép (double-decrement), làm hỏng bộ đếm sức chứa.

## 11. Các Rủi Ro Trì Hoãn Và Khóa Chéo (Timeout và deadlock)

Khi áp dụng khóa cấp cha (Parent lock) hoặc bộ đếm nguyên tử, nhiều giao dịch sẽ xếp hàng chờ trên cùng một dòng. Lưu ý các nguyên tắc sau:

- **Giữ giao dịch ngắn (Short transaction):** Tránh kéo dài thời gian giữ khóa.
- **Áp dụng Timeout:** Sử dụng `lock_timeout` và `query_timeout` để ngăn chặn việc các kết nối chờ đợi vô thời hạn, giúp ứng dụng nhận phản hồi lỗi sớm.
- **Tránh I/O từ xa (Remote I/O):** Không thực hiện các lệnh gọi API mạng từ bên ngoài (như gọi API thanh toán) trong lúc đang giữ các khóa database quan trọng.
- **Đồng nhất thứ tự khóa (Deterministic Lock Order):** Nếu thao tác của bạn cấp phát trên nhiều pool cùng lúc, hãy thống nhất thứ tự (ví dụ: theo pool ID) để tránh khóa chéo (deadlock).

Nếu hệ thống của bạn là một bãi chứa cực nóng (Hot pool) với lưu lượng lớn, sự tắc nghẽn ở kết nối (Connection pool exhaustion) là điều không thể tránh khỏi. Cân nhắc dùng phương pháp `SKIP LOCKED` với các slot tạo sẵn để bỏ qua sự tắc nghẽn, từ chối ngay các request tới sau thay vì bắt chúng chờ đợi.

## 12. Xử Lý Khi Lỗi Hệ Thống (Crash và duplicate)

Nếu máy chủ sập (crash) ngay trước khi commit, toàn bộ giao dịch sẽ rollback và slot chưa được cấp. Nếu máy chủ sập sau khi DB đã commit nhưng trước khi ứng dụng gửi phản hồi thành công, request của khách hàng có thể bị gửi lại (caller retry).

Do đó, bạn phải dựa vào ràng buộc `(pool_id, request_id)` hoặc sử dụng bảng Idempotency riêng lẻ để nhận diện và trả về kết quả thành công mà không cấp phát dư thừa (replay accepted result). 

Tuy nhiên, ràng buộc Duplicate này chỉ ngăn trùng một yêu cầu, chứ không ngăn được vi phạm tổng sức chứa từ các yêu cầu khác biệt.

## 13. Hạn Chế Khi Xử Lý Tương Tranh Ứng Dụng Đa Phiên Bản (Multi-instance)

Nếu hệ thống của bạn có nhiều instance (pods) cùng trỏ vào một DB, các cơ chế khóa ứng dụng nội bộ (như `synchronized` hay biến `Mutex` cấp Process) sẽ không bảo vệ được gì. Bạn buộc phải đẩy ranh giới điều phối (Coordination Boundary) xuống cơ sở dữ liệu như PostgreSQL.

Việc dùng một biến đếm cache ở bộ nhớ (local cache active count) rất dễ bị lệch nhịp do độ trễ và sự cố mất đồng bộ (cache invalidation delay), từ đó gây tràn giới hạn sức chứa. Bộ đếm sức chứa là một dữ liệu nhạy cảm cao, không nên sử dụng caching cho quyết định cấp phát.

## 14. Chỉ Số Quan Sát Vận Hành (Observability)

Giám sát các chỉ số nghiệp vụ (Business metrics):

- `pool.capacity`: Giới hạn cấu hình.
- `pool.used_slots`: Sức chứa đang được sử dụng (nếu có dùng bộ đếm).
- `pool.active_allocation_count`: Số lượng dòng ACTIVE thực tế.
- Số lượng các request trả về: `accepted`, `full`, `duplicate`.
- **Đặc biệt lưu ý:** `capacity.invariant_violation` (khi active count > capacity).

Giám sát các chỉ số hạ tầng cơ sở dữ liệu (Database metrics):

- `conditional_update_affected_zero`: Số lần update gặp điều kiện trả về 0 (ví dụ hết capacity).
- `lock_wait_duration`: Thời gian trung bình giao dịch bị chờ khóa.
- `lock_timeout`: Số lần hết thời gian chờ khóa.
- `serialization_failure_40001`: Lỗi SSI.
- `deadlock_40P01`: Số lần phát hiện khóa chéo.
- `transaction_duration`: Thời gian chạy giao dịch trung bình.

Có thể sử dụng các công cụ như `pg_stat_activity` hoặc `pg_locks` để chẩn đoán. Tuy nhiên, cách mạnh mẽ nhất để nhận biết lỗi hệ thống là tạo các script định kỳ rà soát dữ liệu (Reconciliation script) để đối chiếu điều kiện `active_count <= capacity` trực tiếp trên database.
