# Phân Tích Chuyên Sâu — Hành Vi Từng Chiến Lược Dưới Tải Tương Tranh Cao

## 1. Trạng thái Ban đầu

```text
Sản phẩm SKU-2024:
  Số lượng có sẵn (available_quantity)  = 500
  Đã giữ chỗ (reserved_quantity)       = 0
  Phiên bản (version)                  = 0

Lưu lượng đồng thời: 800 yêu cầu mua 1 sản phẩm mỗi yêu cầu
Số máy chủ ứng dụng: 3 instance
Connection pool mỗi instance: 20 kết nối (tổng 60 kết nối)
Mức cô lập PostgreSQL: READ COMMITTED (mặc định)
```

Mỗi yêu cầu được xử lý bởi một transaction độc lập. Tất cả đều nhắm vào cùng một bản ghi (`product_id = 2024`). Đây là bài toán hot-row thuần túy.

## 2. Kịch bản A — Lock lạc quan (@Version) dưới tải cao

### Dòng thời gian

| Bước | Transaction 1–50 (Đợt 1) | PostgreSQL |
| --- | --- | --- |
| 1 | 50 transaction cùng `SELECT` → đọc `version = 0` | Cung cấp snapshot cho mỗi transaction |
| 2 | 50 transaction tính toán trên RAM: `500 >= 1` → hợp lệ | Database không can thiệp |
| 3 | 50 transaction phát lệnh `UPDATE ... WHERE version = 0` | Transaction đầu tiên chiếm lock bản ghi |
| 4 | Transaction 1 commit → `version = 1` | 49 transaction chờ, sau đó đánh giá lại |
| 5 | 49 transaction: `WHERE version = 0` → 0 dòng bị ảnh hưởng | Hibernate phát hiện xung đột version |
| 6 | 49 transaction nhận `OptimisticLockException` | Transaction rollback |

Kết quả đợt 1: 1 thành công, 49 thất bại, tỷ lệ thất bại 98%.

### Hiệu ứng cộng dồn khi thử lại

```text
Đợt 1: 50 yêu cầu → 1 thành công, 49 thất bại
Đợt 2 (retry): 49 yêu cầu + yêu cầu mới → 1 thành công, ~48 thất bại
Đợt 3 (retry): ~48 yêu cầu + retry từ đợt 2 → 1 thành công, ~47 thất bại
...
```

Mỗi đợt chỉ có 1 transaction thắng cuộc. Các transaction thua phải mở transaction mới, tải lại dữ liệu, tính toán lại, rồi thử lại. Tổng số lượt truy vấn database tăng gấp nhiều lần so với số yêu cầu ban đầu.

### Phân tích tải

Giả sử mỗi yêu cầu được phép thử lại tối đa 5 lần:

```text
Yêu cầu ban đầu:                800
Tỷ lệ xung đột trung bình:      > 90% (tùy mức đồng thời tức thời)
Tổng lượt transaction ước tính:  800 × (1 + tỷ_lệ_thất_bại × số_lần_thử)
Tải database thực tế:            Gấp 3–5 lần so với lưu lượng gốc
```

> **Nói ngắn gọn:** Lock lạc quan biến 800 yêu cầu thành hàng nghìn lượt transaction, phần lớn là thất bại. Database phải xử lý tải cao hơn nhiều lần nhưng thông lượng thực tế không tăng.

### Tại sao backoff và jitter không đủ

Backoff với jitter giúp phân tán thời điểm thử lại, giảm xác suất xung đột đồng thời. Tuy nhiên, khi số lượng yêu cầu đang chờ thử lại lớn hơn nhiều so với thông lượng xử lý (1 transaction/lần), backoff chỉ kéo dài thời gian chờ mà không giảm đáng kể tổng tải. Phía gọi (caller) phải chờ lâu hơn và vẫn có thể thất bại sau khi hết số lần thử lại cho phép.

## 3. Kịch bản B — Lock bi quan (FOR UPDATE) dưới tải cao

### Dòng thời gian

| Bước | Transaction 1–50 (Đồng thời) | PostgreSQL |
| --- | --- | --- |
| 1 | 50 transaction phát lệnh `SELECT ... FOR UPDATE` | Transaction 1 chiếm lock, 49 transaction xếp hàng |
| 2 | Transaction 1 xử lý logic nghiệp vụ (5ms) | 49 transaction chờ lock |
| 3 | Transaction 1 commit, giải phóng lock | Transaction 2 chiếm lock |
| 4 | Transaction 2 xử lý (5ms), commit | Transaction 3 chiếm lock |
| ... | Tuần tự, mỗi lần 5ms | Hàng đợi giảm dần |
| 50 | Transaction 50 hoàn tất | Tổng thời gian: ~250ms |

Tất cả 50 transaction đều thành công (nếu còn hàng). Không có thử lại. Nhưng:

### Phân tích nghẽn kết nối

```text
50 transaction xếp hàng trên cùng bản ghi
Mỗi transaction giữ 1 kết nối database trong suốt thời gian chờ
Transaction 50 chờ: 49 × 5ms = 245ms (chỉ chờ lock)
Connection pool: 20 kết nối

Khi 800 yêu cầu đồng thời:
  20 kết nối bị giữ chờ lock trên sản phẩm SKU-2024
  Yêu cầu thứ 21 trở đi: chờ kết nối từ pool
  Các yêu cầu khác (không liên quan flash-sale): cũng bị chặn do hết kết nối
```

> **Nói ngắn gọn:** Lock bi quan tuần tự hóa mọi yêu cầu qua hàng đợi lock. Mỗi vị trí trong hàng đợi tiêu thụ một kết nối database. Khi hàng đợi dài hơn connection pool, toàn hệ thống bị nghẽn.

### Lock timeout

Nếu thiết lập `lock_timeout = 200ms`:

```text
Transaction 1–40:  Hoàn tất trong 200ms (mỗi transaction 5ms)
Transaction 41+:   Chờ > 200ms → lỗi 55P03 (lock timeout)
```

Lock timeout bảo vệ hệ thống khỏi chờ vô hạn, nhưng biến lỗi thành `55P03` thay vì thành công. Ứng dụng cần quyết định: thử lại (tăng tải) hay từ chối (mất đơn).

## 4. Kịch bản C — Cập nhật nguyên tử (Conditional UPDATE) dưới tải cao

### Dòng thời gian

| Bước | Transaction 1–50 | PostgreSQL |
| --- | --- | --- |
| 1 | 50 transaction phát lệnh `UPDATE ... WHERE available >= 1` | Transaction 1 chiếm lock bản ghi |
| 2 | Transaction 1: điều kiện hợp lệ → cập nhật, chưa commit | 49 transaction chờ lock |
| 3 | Transaction 1 commit, giải phóng lock | Transaction 2 tự động đánh giá lại (recheck) |
| 4 | Transaction 2: `499 >= 1` → hợp lệ → cập nhật, commit | Transaction 3 đánh giá lại |
| ... | Mỗi transaction đánh giá lại và cập nhật tuần tự | |
| 50 | Transaction 50: `451 >= 1` → hợp lệ → cập nhật, commit | |

Kết quả: 50 transaction đều thành công. Không có thử lại. Không có ngoại lệ.

### So sánh với lock bi quan

Hành vi xếp hàng tương tự `FOR UPDATE`, nhưng có điểm khác biệt quan trọng:

```text
Lock bi quan (FOR UPDATE):
  1. SELECT FOR UPDATE → chờ lock → đọc dữ liệu → xử lý trên RAM → UPDATE → COMMIT
  Transaction giữ lock từ bước SELECT đến COMMIT

Atomic UPDATE:
  1. UPDATE WHERE ... → chờ lock (nếu bị chặn) → đánh giá lại → cập nhật → COMMIT
  Transaction giữ lock từ thời điểm UPDATE đến COMMIT
```

Thời gian giữ lock với atomic UPDATE ngắn hơn vì không có bước đọc-tính toán trên RAM giữa lúc chiếm lock và commit. Lock chỉ được giữ trong khoảng thời gian thực thi `UPDATE` đến `COMMIT`.

### Khi hàng hết

```text
Transaction 501 trở đi:
  UPDATE ... WHERE available_quantity >= 1
  → Đánh giá lại: available_quantity = 0 >= 1 → Sai
  → Trả về 0 dòng bị ảnh hưởng
  → Không cần thử lại, không phát sinh ngoại lệ
```

Phía thua nhận kết quả `0` dòng ngay lập tức sau khi lock được giải phóng. Ứng dụng phản hồi `OUT_OF_STOCK` mà không tốn thêm tài nguyên.

### Vấn đề còn tồn tại

- Mặc dù thời gian giữ lock ngắn hơn, 800 yêu cầu đồng thời vẫn tạo hàng đợi chờ lock trên bản ghi.
- Mỗi vị trí trong hàng đợi vẫn giữ một kết nối database.
- Khi tải vượt xa khả năng xử lý tuần tự của database (1 transaction tại một thời điểm trên cùng bản ghi), connection pool vẫn có thể cạn kiệt.
- Transaction chứa thêm logic sau `UPDATE` (ghi lịch sử, tạo outbox event) kéo dài thời gian giữ lock.

## 5. Kịch bản D — Hàng đợi tuần tự (Serial queue) tại tầng ứng dụng

### Ý tưởng

Thay vì để 800 transaction đồng thời tranh chấp tại database, tầng ứng dụng tuần tự hóa các yêu cầu trước khi gửi xuống:

```text
800 yêu cầu → Bộ điều phối (Dispatcher) → Hàng đợi (Queue) → Xử lý tuần tự
                                                                 ↓
                                                            1 transaction tại một thời điểm
```

### Hành vi

- Không có xung đột lock tại database (chỉ 1 transaction tại một thời điểm).
- Không có thử lại do xung đột.
- Thông lượng bị giới hạn bởi tốc độ xử lý tuần tự của một luồng.
- Độ trễ tăng tuyến tính theo vị trí trong hàng đợi.

### Vấn đề

- Đòi hỏi hạ tầng bổ sung (message queue hoặc in-memory queue).
- Trong kiến trúc đa máy chủ, cần cơ chế định tuyến (routing) để đảm bảo tất cả yêu cầu cho cùng sản phẩm đi vào cùng một hàng đợi.
- Nếu bộ xử lý gặp sự cố, hàng đợi cần cơ chế phục hồi.
- Phức tạp vận hành cao hơn đáng kể so với ba chiến lược trên.

## 6. Bảng so sánh định tính (Qualitative comparison)

| Tiêu chí | Lock lạc quan | Lock bi quan | Atomic UPDATE | Hàng đợi |
| --- | --- | --- | --- | --- |
| Tính đúng đắn | ✓ | ✓ | ✓ | ✓ |
| Hành vi phía thua | Ngoại lệ, thử lại | Chờ lock | 0 dòng, không thử lại | Chờ trong hàng đợi |
| Tải thử lại | Cao, nhân bản | Không có | Không có | Không có |
| Thời gian giữ lock | Ngắn (flush→commit) | Dài (select→commit) | Trung bình (update→commit) | Không có lock tranh chấp |
| Tiêu thụ kết nối khi chờ | Không (thử lại sau) | Có (giữ kết nối) | Có (giữ kết nối) | Không (chờ ngoài database) |
| Thông lượng tương tranh thấp | Tốt | Tốt | Tốt | Tốt |
| Thông lượng tương tranh cao | Sụp đổ do retry storm | Giảm do lock queue | Giảm nhẹ | Ổn định |
| Ảnh hưởng đến yêu cầu khác | Gián tiếp (tải CPU/IO) | Trực tiếp (hết kết nối) | Trực tiếp (hết kết nối) | Không |
| Độ phức tạp triển khai | Thấp | Thấp | Thấp | Cao |
| Yêu cầu hạ tầng bổ sung | Không | Không | Không | Message queue/Router |

## 7. Phân tích nguyên nhân gốc rễ theo tầng (Root cause by layer)

| Tầng xử lý | Vấn đề |
| --- | --- |
| Ứng dụng | Chọn chiến lược lock dựa trên tính đúng đắn mà không đánh giá hành vi dưới tải cao. |
| Spring | Retry interceptor không phân biệt lỗi tương tranh với lỗi hệ thống, thiếu backoff/jitter. |
| Hibernate | `@Version` phát hiện xung đột tại thời điểm flush, không phải tại thời điểm đọc. Trên bản ghi nóng, gần như mọi flush đều xung đột. |
| PostgreSQL | Lock bản ghi là cơ chế tuần tự hóa cấp thấp nhất. Khi N transaction tranh chấp, N-1 phải chờ. Database không có cơ chế kiểm soát đầu vào. |
| Connection pool | Kích thước cố định. Khi lock queue dài hơn pool, hệ thống nghẽn toàn bộ. |

## 8. Mức cô lập và ảnh hưởng

### READ COMMITTED (mặc định)

- Lock lạc quan: Xung đột phát hiện tại Hibernate (version mismatch), không phải PostgreSQL.
- Lock bi quan: `FOR UPDATE` chờ lock, sau đó đọc dữ liệu mới nhất.
- Atomic UPDATE: Chờ lock, sau đó PostgreSQL đánh giá lại mệnh đề `WHERE` trên phiên bản đã commit.

### REPEATABLE READ

- Lock lạc quan: Hành vi tương tự `READ COMMITTED` vì xung đột vẫn do version mismatch.
- Atomic UPDATE: Thay vì trả về 0 dòng, PostgreSQL có thể phát sinh lỗi `40001` (serialization failure) khi bản ghi đã bị cập nhật bởi transaction khác. Ứng dụng phải thử lại toàn bộ transaction.

### SERIALIZABLE

- Tỷ lệ abort `40001` tăng tương tự lock lạc quan, nhưng cơ chế phát hiện khác (SSI thay vì version).
- Không phải giải pháp phù hợp cho bản ghi nóng.

## 9. Kiến trúc đa máy chủ (Multi-instance behavior)

Ba chiến lược đầu (lock lạc quan, lock bi quan, atomic UPDATE) đều an toàn trong kiến trúc đa máy chủ vì cơ chế bảo vệ nằm tại database. Tuy nhiên:

- Lock lạc quan: Mỗi instance tạo ra retry storm độc lập, tổng tải nhân theo số instance.
- Lock bi quan: Lock queue tại database chứa transaction từ tất cả instance, tổng chiều dài queue nhân theo số instance.
- Atomic UPDATE: Tương tự lock bi quan nhưng thời gian giữ lock ngắn hơn.

Hàng đợi tuần tự cần cơ chế phân phối (partitioning) để đảm bảo tất cả yêu cầu cho cùng sản phẩm đi vào cùng một điểm xử lý, bất kể instance nào nhận yêu cầu.

## 10. Khi nào mỗi chiến lược sụp đổ

| Chiến lược | Ngưỡng sụp đổ | Triệu chứng |
| --- | --- | --- |
| Lock lạc quan | Khi tỷ lệ xung đột > 50% | Retry storm, tải gấp bội, latency tăng vọt |
| Lock bi quan | Khi lock queue > connection pool | Connection pool exhaustion, cascading timeout |
| Atomic UPDATE | Khi lock queue > connection pool | Tương tự lock bi quan nhưng ngưỡng cao hơn |
| Hàng đợi | Khi tốc độ đầu vào >> tốc độ xử lý | Hàng đợi tích lũy, latency tăng tuyến tính, risk tràn bộ nhớ |

## 11. Giới hạn phân tích (Scope boundaries)

Bài phân tích này không đưa ra con số hiệu năng cụ thể. Tất cả ước tính đều mang tính minh họa cơ chế, không phải kết quả đo lường trên hạ tầng thực tế. Quyết định chiến lược phải dựa trên:
- Đo lường tải thực tế (load testing) trên hạ tầng cụ thể.
- Giám sát hành vi production (observability).
- Đặc điểm nghiệp vụ: mức chấp nhận rủi ro, thời gian phản hồi tối đa, tỷ lệ thất bại cho phép.

Các quyết định cụ thể cho tồn kho, thanh toán, booking thuộc phạm vi của các bài toán nghiệp vụ tương ứng (`ECOM-*`, `BANK-*`, `BOOK-*`).
