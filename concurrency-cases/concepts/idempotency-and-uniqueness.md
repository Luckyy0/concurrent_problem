# Tính lũy đẳng (Idempotency), Tính duy nhất (Uniqueness) và Xí chỗ dữ liệu (Atomic claim)

## Mục tiêu

Tài liệu này giải thích các từ vựng và khái niệm chung khi bạn cần đảm bảo một thao tác chỉ được thực hiện duy nhất một lần (chống trùng lặp), và cách xử lý khi có ai đó (hoặc hệ thống khác) gửi cùng một yêu cầu (request) nhiều lần.

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Khóa nghiệp vụ (`business key`) | Một hoặc vài cột ghép lại để xác định một dữ liệu duy nhất trong thực tế (không nhất thiết phải là ID chính - primary key). Ví dụ: số CCCD. |
| Ràng buộc duy nhất (`unique constraint`) | Quy tắc cấu hình dưới Database để cấm không cho 2 dòng có chung Khóa nghiệp vụ. |
| Xí chỗ dữ liệu (`atomic claim`) | Một thao tác ghi xuống Database nhằm tuyên bố: "Tôi là người duy nhất đang xử lý dữ liệu này". |
| Khóa lũy đẳng (`idempotency key`) | Một mã (thường do Client gửi lên) để báo cho Server biết: "Đây vẫn là yêu cầu cũ đang gửi lại, đừng làm lại từ đầu". |
| Dấu vân tay yêu cầu (`request fingerprint`) | Một mã băm (hash) của nội dung yêu cầu, dùng để kiểm tra xem Client có gian lận gửi nội dung khác nhưng xài lại cái mã cũ không. |
| Phát lại (`replay`) | Khi nhận được yêu cầu trùng, thay vì chạy lại code, ta chỉ cần lôi kết quả cũ đã lưu ra trả về lại. |
| Đang xử lý (`in-progress`) | Trạng thái báo hiệu: Yêu cầu này đã được xí chỗ, đang chạy dở dang, chưa có kết quả cuối cùng. |
| Ghi đè hoặc thêm mới (`upsert`) | Lệnh `INSERT ... ON CONFLICT` trong SQL: Nếu chưa có thì thêm mới, nếu có rồi thì cập nhật hoặc bỏ qua. |

## Tính duy nhất (Uniqueness) khác Tính lũy đẳng (Idempotency) ở chỗ nào?

- Ràng buộc **Tính duy nhất** (Uniqueness) ở Database chỉ quan tâm 1 câu hỏi: *"Có được phép thêm dòng thứ 2 cho dữ liệu này không?"* (Đáp án là Không).
- Hợp đồng **Tính lũy đẳng** (Idempotency) lại quan tâm toàn diện hơn:
  - *"Nếu gửi trùng yêu cầu thì tôi trả về kết quả gì?"*
  - *"Nếu gửi trùng mã (key) nhưng ruột dữ liệu (payload) bị thay đổi thì xử lý sao?"*
  - *"Nếu lần chạy trước bị lỗi hoặc đang chạy dở thì lần này gửi lại có được chạy tiếp không?"*

> **Nói ngắn gọn:** Uniqueness ngăn Database không bị đẻ thêm dòng rác; Idempotency định nghĩa cách Server ứng xử khéo léo khi Client lỡ tay bấm nút "Thanh toán" hai lần.

## Đừng bao giờ kiểm tra rồi mới thêm (Check-then-Insert)

Cách làm sai kinh điển của người mới (rất dễ bị lỗi khi chạy đồng thời - race condition):
```text
(1) Luồng 1 dùng SELECT hỏi Database: "Có ai dùng mã này chưa?" -> DB trả lời: "Chưa"
(2) Cùng lúc đó, Luồng 2 cũng SELECT hỏi câu y hệt -> DB vẫn trả lời: "Chưa"
(3) Cả hai luồng cùng đinh ninh chưa có ai dùng nên cùng gọi lệnh INSERT -> BÙM! Lỗi trùng lặp dữ liệu.
```
Lý do: Lệnh `SELECT` đơn thuần không có tác dụng khóa giữ chỗ.

**Cách làm đúng:** Phải dùng Ràng buộc duy nhất (Unique Constraint) tạo sẵn trong Database, hoặc lệnh xí chỗ trực tiếp (INSERT ... ON CONFLICT) của Database.
```sql
INSERT INTO command_claim(scope_id, idempotency_key, status)
VALUES (:scopeId, :key, 'IN_PROGRESS')
ON CONFLICT (scope_id, idempotency_key) DO NOTHING;
```
Lệnh này trả về số lượng dòng bị ảnh hưởng. Ai nhận được số `1` thì người đó thắng. Nhận được số `0` nghĩa là đã có người khác xí chỗ trước. 

## Thiết kế Khóa nghiệp vụ (Business key)

Khóa phải xác định đúng phạm vi (scope). Ví dụ:
- Dùng `(tenant_id, external_reference)` thay vì chỉ dùng `external_reference`.
Nếu bạn chỉ dùng mã đơn hàng làm khóa duy nhất trên toàn hệ thống (Global), có thể bạn sẽ chặn nhầm mã đơn hàng của hai công ty (tenant) khác nhau.

Lưu ý khi thiết kế: Cẩn thận với chữ hoa/thường, khoảng trắng, hoặc giá trị `NULL`. (Trong PostgreSQL, nhiều giá trị `NULL` vẫn không bị coi là trùng nhau trừ khi bạn có cấu hình rõ ràng).

## Ràng buộc và Chỉ mục (Constraint & Index)

Để cấu hình tính duy nhất cho Database, **bắt buộc phải viết lệnh SQL Migration** (như dùng Flyway/Liquibase).
```sql
ALTER TABLE work_item
    ADD CONSTRAINT uk_work_item_business
    UNIQUE (tenant_id, external_reference);
```
**Lưu ý quan trọng:** Việc đặt thẻ `@UniqueConstraint` trên đầu class Entity của Hibernate/JPA chỉ có ý nghĩa làm tài liệu cho bạn đọc. Nó **KHÔNG CÓ TÁC DỤNG** ép Database tự tạo ràng buộc ở môi trường Production. Đừng trông cậy vào Hibernate để sửa index!

## Database xử lý trùng lặp ra sao?

Khi 2 luồng cùng lúc gọi lệnh INSERT chung một mã xí chỗ:
- Luồng 1 chạy vào trước, nó sẽ khóa giữ chỗ mục đó.
- Luồng 2 chạy vào sau, nó sẽ **bị treo lại, đứng chờ** Luồng 1 xử lý xong.
- Nếu Luồng 1 lưu thành công (commit): Luồng 2 sẽ bị văng lỗi mã `23505` (Trùng lặp Unique Constraint) hoặc bị bỏ qua tùy theo logic `ON CONFLICT` của bạn.
- Nếu Luồng 1 bị lỗi và hủy bỏ (rollback): Luồng 2 sẽ được giải phóng đi tiếp và trở thành người thắng cuộc.

## Cẩn thận với bộ đệm của Spring Data JPA/Hibernate

Lệnh `repository.save(entity)` thường chỉ lưu tạm dữ liệu vào bộ đệm của Java (Persistence Context), chưa hề chạy câu lệnh SQL INSERT xuống Database. 
Chỉ khi bạn dùng `saveAndFlush()` hoặc giao dịch thực sự kết thúc (commit), lệnh SQL mới bắn xuống Database và lỗi trùng lặp mới xảy ra.

**Lưu ý cực kỳ quan trọng:** Nếu Database đã báo lỗi trùng lặp (SQLSTATE 23505), toàn bộ giao dịch (Transaction) của Spring đó đã bị hỏng (rollback-only). Bạn **không thể** dùng khối `try-catch` bắt lỗi đó lại, rồi hồn nhiên gọi lệnh `SELECT` tiếp ở ngay bên trong giao dịch đó. 
Nếu muốn bắt lỗi và đọc lại dữ liệu, bạn phải tách việc "lưu thử" (insert attempt) và việc "đọc lại kết quả cũ" ra thành các giao dịch riêng biệt.

## Các lựa chọn khi gặp xung đột (`ON CONFLICT`)

### `DO NOTHING` (Bỏ qua)

Rất thích hợp khi bạn dùng lệnh xí chỗ (claim):
```sql
INSERT ... ON CONFLICT (scope_id, key) DO NOTHING
RETURNING claim_id;
```
Nếu có `claim_id` trả về nghĩa là bạn đã thắng (xí chỗ thành công). Nếu trả về rỗng, bạn là người thua vì đã có người đến trước.

### `DO UPDATE` (Cập nhật đè)

Chỉ dùng cách này khi bạn THỰC SỰ có nhu cầu lấy dữ liệu mới trộn lẫn (merge) với dữ liệu cũ. 
Tuyệt đối không dùng lệnh cập nhật giả (no-op) chỉ để ép Database trả về cái ID. Việc này sẽ tạo ra hàng loạt bản nháp (tuple) vô dụng, gây nhiễu log hệ thống và làm Database phình to (bloat). Đặc biệt lưu ý: Đừng bao giờ lấy dữ liệu bị trùng lặp ghi đè lên lịch sử nội dung yêu cầu (fingerprint) ban đầu.

## Vòng đời của một bản ghi Idempotency (Idempotency lifecycle)

Một bảng lưu thông tin Idempotency bền vững thường cần các cột:
```text
scope/key: Khóa phân loại và mã xí chỗ
request fingerprint: Dấu vân tay để đối chiếu nội dung gửi
status: Trạng thái (IN_PROGRESS | SUCCEEDED | FAILED)
resource/response reference: Lưu lại kết quả để sau này dùng lại (replay)
created_at, completed_at, expires_at: Mốc thời gian
owner/attempt metadata: Lưu thông tin ai đang xử lý để phục vụ cứu hộ (recovery)
```

Hợp đồng (Contract) code của bạn bắt buộc phải định nghĩa rõ cách xử lý cho các trường hợp:
- Gửi trùng key + trùng dấu vân tay.
- Gửi trùng key + NHƯNG khác dấu vân tay (gian lận).
- Bị trùng khi trạng thái vẫn đang là `IN_PROGRESS`.
- Phân biệt lỗi có thể thử lại (retryable) và lỗi chết hẳn (terminal failure).
- Cách lưu trữ và dựng lại kết quả cũ (response reconstruction).
- Hạn sử dụng (expiry), dọn dẹp (cleanup) và gửi lại quá trễ (late retry).
- Sập nguồn ngay giữa lúc xí chỗ và lúc lưu dữ liệu.

**Mẹo:** Lệnh xí chỗ và lệnh lưu dữ liệu thật nên được gộp chung vào một giao dịch (transaction) nếu dùng chung Database. Nếu gọi qua hệ thống khác, bạn phải dùng các kỹ thuật riêng như Inbox/Outbox hoặc luồng công việc (Workflow).

## Xử lý sự cố và cứu hộ (Failure và recovery)

Khi có sự cố xảy ra, đây là cách hệ thống nên hoạt động:
- **Lệnh xí chỗ (insert claim) bị hoàn tác (rollback):** Key đó coi như chưa có ai sở hữu, người khác có thể vào lấy.
- **Người thắng cuộc sập nguồn trước khi lưu (commit):** Người đang đứng chờ (waiter) sẽ được hệ thống cho phép đi tiếp.
- **Người thắng cuộc lưu xong nhưng kết quả báo về bị mất mạng:** Lần sau Client gửi lại (duplicate), Server chỉ cần lôi kết quả đã lưu ra trả về.
- **Yêu cầu bị kẹt mãi ở trạng thái `IN_PROGRESS`:** Cứu hộ bằng cách dựa vào trạng thái đã lưu, thời gian quá hạn (timeout) và quyền sở hữu (owner). Đừng bao giờ viết code xóa các lệnh xí chỗ này một cách mù quáng.
- **Dọn dẹp (Cleanup):** Chỉ xóa dữ liệu xí chỗ khi thời hạn cho phép lưu trữ (retention window) đã hết.

## Kiểm thử (Testing)

Khi viết bài kiểm thử cho Database bằng PostgreSQL Testcontainers, bạn phải đảm bảo có đủ các kịch bản:
1. Dùng 2 kết nối (connections) cùng lúc gửi chung một mã business key.
2. Dùng chốt (latch) để điều khiển chính xác thứ tự ai chạy trước, ai commit, ai rollback.
3. Đảm bảo cuối cùng chỉ có đúng 1 dòng dữ liệu tồn tại.
4. Kiểm tra đúng kết quả trả về cho người thắng và người thua.
5. Kiểm tra cảnh người thua phải đứng chờ người thắng commit, và chờ người thắng rollback.
6. Kiểm tra đúng tên Ràng buộc (constraint name) và mã lỗi SQLSTATE (ví dụ 23505).
7. Mọi hàm chờ (futures/waits) đều phải có thời gian tối đa (bounded timeout).

## Theo dõi hệ thống (Observability)

Nên thiết lập các chỉ số (metric) để theo dõi các sự kiện sau:
```text
claim.created (Tạo mới thành công)
claim.duplicate (Gửi trùng lặp)
claim.payload_mismatch (Gửi trùng nhưng sai nội dung vân tay)
claim.in_progress (Đang xử lý)
claim.replayed (Phát lại kết quả cũ)
unique_violation_by_constraint (Báo lỗi vi phạm khóa duy nhất)
claim_wait_duration (Thời gian phải đứng chờ khóa)
stuck_claim_age (Thời gian bị kẹt quá lâu)
```

**Lưu ý bảo mật:** Tuyệt đối không in nguyên văn mã xí chỗ (idempotency keys) hoặc nội dung (payload) ra log nếu chúng chứa mật khẩu hay dữ liệu cá nhân. Hãy dùng chuỗi mã hóa (hash) hoặc mã định danh (correlation ID) phù hợp.

## Liên kết tài liệu tham khảo

- [DB-006 — Unique constraint concurrency](../postgresql/unique-constraint-concurrency/README.md)
- [Concurrency testing](concurrency-testing.md)
- [Case catalog](../CONCURRENCY_CASE_LIBRARY.md)
