# DB-009 — Cơ Chế Khóa SERIALIZABLE và Chiến Lược Thử Lại (Serializable and Retry)

## 1. Tóm tắt (Overview)

Hãy xem xét kịch bản: Một Khách hàng (Merchant) có hạn mức đặt cọc (reservation limit) là `100`. Tổng số tiền đang được kích hoạt (active) là `60`. Có hai giao dịch (command) diễn ra đồng thời, mỗi giao dịch yêu cầu đặt cọc thêm `30`:

```text
Giao dịch T1 kiểm tra tổng là 60 -> CHẤP NHẬN (ACCEPTED) -> Ghi nhận đặt cọc C1.
Giao dịch T2 cũng kiểm tra tổng là 60 -> CHẤP NHẬN (ACCEPTED) -> Ghi nhận đặt cọc C2.
```

Nếu cả hai giao dịch đều được hoàn tất (commit) thành công, tổng số tiền cọc sẽ tăng lên `120`, vượt quá hạn mức cho phép.
Lúc này, cơ chế `SERIALIZABLE` của PostgreSQL (sử dụng Serializable Snapshot Isolation - `SSI`) sẽ kiểm tra các thao tác đọc và ghi. Nếu hệ thống phát hiện các giao dịch đồng thời (concurrent history) này KHÔNG THỂ thực thi theo một trật tự tuần tự hợp lệ, nó sẽ tự động hủy (abort) một trong các giao dịch và trả về mã lỗi: SQLSTATE `40001`.

Tức là, mức cô lập `SERIALIZABLE` đảm bảo tính toàn vẹn dữ liệu cho các giao dịch đã hoàn tất (commit), nhưng không đảm bảo mọi giao dịch đều sẽ thành công. Nhiệm vụ của ứng dụng (Application) là phải bắt lỗi, hủy bỏ (rollback) trạng thái hiện tại, tạm nghỉ (backoff) và thực hiện lại (retry) logic trong một giao dịch HOÀN TOÀN MỚI.

> **Ghi chú quan trọng:** Mức cô lập `SERIALIZABLE` kiểm soát lỗi bằng cách biến một bất thường dữ liệu (anomaly) thành một lỗi có chủ đích (controlled failure); Thiết lập ranh giới giao dịch và cơ chế thử lại (retry boundary) là cách để ứng dụng biến lỗi đó thành một kết quả thành công cuối cùng.

## 2. Các thực thể và Trạng thái chia sẻ (Actors and shared state)

| Thực thể (Actor/State) | Trạng thái hiện hành |
| --- | --- |
| `merchant_limit` | Merchant `7`, hạn mức `100` |
| `credit_reservation` | Có sẵn cọc `ACTIVE` với tổng `60` |
| Giao dịch T1 (App-1) | Lệnh C1, yêu cầu `30` |
| Giao dịch T2 (App-2) | Lệnh C2, yêu cầu `30` |
| Cơ sở dữ liệu (Authoritative store) | PostgreSQL |

Vấn đề tranh chấp ở đây KHÔNG PHẢI là hai giao dịch cập nhật cùng một dòng dữ liệu (same-row update). Cả hai đều đánh giá dựa trên cùng một điều kiện truy vấn (predicate):

```sql
select coalesce(sum(amount), 0)
from credit_reservation
where merchant_id = 7
  and status = 'ACTIVE';
```

Sau khi đọc dữ liệu, mỗi giao dịch sẽ tạo ra một bản ghi đặt cọc HOÀN TOÀN KHÁC NHAU. Tuy nhiên, bản ghi mới của giao dịch này lại làm thay đổi kết quả tổng mà giao dịch kia vừa đọc.

## 3. Nguyên tắc thiết kế cốt lõi (Invariant)

```text
Tổng số tiền cọc ACTIVE cho một merchant tuyệt đối KHÔNG ĐƯỢC vượt quá hạn mức (limit).

Mỗi mã lệnh (Command ID) chỉ được phép có ĐÚNG 1 kết quả (durable decision): CHẤP NHẬN (ACCEPTED) hoặc TỪ CHỐI (REJECTED).

Bất kỳ lần thử lại (retry) nào cũng bắt buộc phải thực hiện toàn bộ quy trình Đọc -> Quyết định -> Ghi trong một giao dịch MỚI; KHÔNG sử dụng lại trạng thái của lần thử trước đó.
```

Giả sử giao dịch T1 commit thành công. Lần thử lại của T2 lúc này sẽ tính được tổng là `90`, dẫn đến quyết định `90 + 30 > 100` và trả về kết quả `REJECTED`. Kết quả cuối cùng: tổng cọc là `90`, C1 và C2 đều có kết quả xác định.

## 4. Ranh giới giao dịch (Transaction boundaries)

Thành phần điều phối thử lại (Retry coordinator) nằm ở TẦNG NGOÀI và không chứa giao dịch cơ sở dữ liệu. Mỗi lần thử lại sẽ gọi đến một Proxy Service (`SerializableAttemptService`) đóng vai trò là một giao dịch vật lý hoàn chỉnh:

```text
Lần thử N:
  BẮT ĐẦU GIAO DỊCH SERIALIZABLE
  kiểm tra xem lệnh đã được xử lý chưa
  đọc hạn mức (limit) và tổng active hiện tại
  ra quyết định (decide)
  lưu bản ghi reservation nếu được chấp nhận
  lưu kết quả decision của command
  đẩy dữ liệu (flush)
  THÀNH CÔNG (COMMIT) hoặc GẶP LỖI 40001 VÀ HỦY (ROLLBACK)
```

Xung đột có thể xảy ra khi gửi lệnh SQL (statement), khi Hibernate gọi flush, hoặc tại thời điểm commit. Thành phần điều phối chỉ nhận kết quả SAU KHI lớp Proxy đã hoàn tất rollback (nếu có lỗi). Hệ thống KHÔNG ĐƯỢC tái sử dụng `EntityManager` hoặc các Entities từ lần thử đã thất bại.

## 5. Mong đợi và Thực tế lỗi (Expected vs actual outcomes)

| Khía cạnh | Hoạt động đúng (Expected) | Sai lầm phổ biến (Broken assumptions) |
| --- | --- | --- |
| Cơ chế `SERIALIZABLE` | Kết quả lịch sử giao dịch tương đương với việc chạy tuần tự (Serializable history) | Tất cả các giao dịch sẽ tự động khóa, chờ nhau và đều thành công. |
| Xử lý lỗi `40001` | Lỗi được nhận diện, rollback hoàn toàn và tạo giao dịch mới để retry | Xem đây là lỗi kết nối ngẫu nhiên hoặc bỏ qua (nuốt lỗi). |
| Chu trình thử lại (Retry) | Tạo Transaction và Snapshot mới hoàn toàn, tính toán lại logic | Lặp lại thao tác trong cùng một phương thức có `@Transactional` ban đầu. |
| Tính lũy đẳng (Idempotency) | Giữ nguyên Command ID để đảm bảo tính nhất quán của kết quả (Durable decision) | Tạo Command ID mới cho mỗi lần retry. |
| Giới hạn thử lại (Exhaustion) | Dừng thử lại sau một số lần nhất định và trả về lỗi rõ ràng (Explicit failure) | Thử lại vô hạn gây treo hệ thống. |

## 6. Thuật ngữ quan trọng (Terminology)

| Thuật ngữ | Ý nghĩa |
| --- | --- |
| Serializable Snapshot Isolation (SSI) | Cơ chế của PostgreSQL sử dụng Snapshot và kiểm tra xung đột (dependency checks) để đảm bảo tính tuần tự. |
| Serialization failure | Lỗi xảy ra khi PostgreSQL hủy giao dịch do không thể sắp xếp lịch sử thực thi hợp lệ (SQLSTATE `40001`). |
| Predicate lock (`SIReadLock`) | Cơ chế theo dõi các bản ghi thỏa mãn điều kiện đọc. Không phải là khóa ngăn chặn (blocking lock). |
| Read-write conflict (rw-conflict) | Xung đột xảy ra khi một giao dịch ghi vào tập dữ liệu mà một giao dịch khác vừa đọc. |
| Chu kỳ nguy hiểm (Serialization cycle) | Chuỗi xung đột chéo giữa các giao dịch, buộc hệ thống phải hủy (abort) để bảo vệ tính nhất quán. |
| Thử lại toàn bộ giao dịch (Whole-transaction retry) | Quy trình hủy hoàn toàn giao dịch lỗi và thực thi lại toàn bộ logic trong một giao dịch SQL mới. |
| Xử lý lũy đẳng (Idempotent attempt) | Xử lý lệnh dựa trên mã định danh duy nhất (Command ID) để không gây hiệu ứng phụ khi gọi nhiều lần. |
| Nghỉ chờ có giới hạn (Bounded backoff) | Khoảng thời gian trì hoãn giữa các lần retry, thường kèm yếu tố ngẫu nhiên (jitter) và giới hạn thời gian (deadline). |

## 7. Hướng dẫn tham khảo (Navigation)

- [Phân Tích Lỗi Thiết Kế Giao Dịch (broken-code.md)](broken-code.md)
- [Phân Tích Cơ Chế SSI Và Lỗi 40001 (analysis.md)](analysis.md)
- [Giải Pháp Retry Toàn Bộ Transaction Kèm Idempotency (solutions.md)](solutions.md)
- [Thực Nghiệm Hành Vi SSI Trong PostgreSQL (experiments.md)](experiments.md)
- [Nền Móng Mức Cô Lập (Isolation levels)](../../concepts/isolation-levels.md)
- [Xử Lý Deadlocks (Deadlocks and retries)](../../concepts/deadlocks-and-retries.md)
- [Kiểm Thử Tương Tranh (Concurrency testing)](../../concepts/concurrency-testing.md)

## 8. Tác động tới hệ thống (Production impact)

- Lỗi `40001` có thể bị rò rỉ ra ngoài và trả về mã lỗi HTTP 500, dù yêu cầu có thể thành công nếu được thử lại đúng cách.
- Thử lại bên trong một giao dịch đã lỗi (Doomed transaction) sẽ dẫn đến lỗi `25P02`.
- Cơ chế thử lại lặp lại liên tục và dồn dập có thể gây quá tải cơ sở dữ liệu (conflict amplification).
- Đọc dữ liệu cũ từ Snapshot của giao dịch thất bại sẽ dẫn đến các quyết định nghiệp vụ sai lệch.
- Gửi thông báo (Notification) trước khi giao dịch commit có thể dẫn đến việc phát đi thông tin sai lệch khi giao dịch bị rollback.
- Yêu cầu xử lý không có tính lũy đẳng (Idempotency) khiến Client thử lại và tạo ra dữ liệu trùng lặp.

## 9. Khuyến nghị khắc phục (Remediation steps)

1. Chỉ sử dụng `SERIALIZABLE` khi nghiệp vụ có các ràng buộc dữ liệu (invariants) phức tạp, chéo nhau giữa tập đọc và tập ghi (read-write set). Ưu tiên các giải pháp cập nhật nguyên tử (atomic/conditional update) hoặc ràng buộc mức độ cấu trúc (constraints) nếu có thể.
2. Phương thức xử lý logic nghiệp vụ (transactional attempt bean) cần có khả năng xác nhận nó đang chạy trong mức cô lập đúng thông qua `current_setting('transaction_isolation')`.
3. Tách biệt thành phần điều phối thử lại (Coordinator) ở TẦNG NGOÀI (ngoài ranh giới giao dịch).
4. Phân loại ngoại lệ để bắt chính xác lỗi SQLSTATE `40001`, dọn dẹp các bộ đệm bộ nhớ và khởi tạo giao dịch MỚI cho toàn bộ dữ liệu.
5. Thiết lập chính sách thử lại: Giới hạn số lần (Attempt Cap), độ trễ số mũ (Exponential Backoff), yếu tố ngẫu nhiên (Jitter) và thời hạn tối đa (Deadline).
6. Duy trì Command ID gốc từ phía Client; Lưu kết quả quyết định cuối cùng (Outcome) vào chung giao dịch thành công.
7. KHÔNG thử lại các lỗi liên quan đến vi phạm quy tắc nghiệp vụ (Business rejection) hoặc lỗi dữ liệu đầu vào.

## 10. Khi nào nên áp dụng (Applicability)

Giải pháp `SERIALIZABLE` phù hợp để bảo vệ tính nhất quán dữ liệu cho các truy vấn theo điều kiện (predicate) hoặc liên quan đến nhiều bản ghi, nơi các ranh giới đọc-ghi chéo nhau. Nó đòi hỏi hệ thống phải triển khai một khung xử lý Rollback/Retry vững chắc. Đối với các hệ thống có mức độ tương tranh quá lớn (hot spots), nên chuyển sang sử dụng khóa nguyên tử (atomic updates) hoặc hàng đợi (queue) để tránh quá tải do thử lại (retry storm).
