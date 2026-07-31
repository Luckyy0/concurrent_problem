# LOCK-001 — Bảo vệ dữ liệu cơ bản bằng Khóa Lạc Quan (`@Version`)

## Tóm tắt

Tưởng tượng có hai nhân viên (A và B) cùng mở một sản phẩm (offer `42`), đang có giá là `100`, phiên bản (version) là `7`. 
- Nhân viên A đổi giá thành `90` và bấm Lưu.
- Nhân viên B ngâm trang web hơi lâu, sửa giá thành `80` và cũng bấm Lưu sau đó.

Nếu code bạn không có rào chắn `version`, CẢ HAI lệnh UPDATE đều báo sửa thành công 1 dòng (affected rows `1`). Hậu quả là B âm thầm đè bẹp công sức của A mà A không hề hay biết!

Chỉ cần bạn thêm Annotation `@Version` vào Code, Hibernate sẽ tự động chế ra câu SQL lợi hại như thế này:

```sql
update product_offer
set price = 90.00,
    version = 8      -- tự động tăng version lên
where offer_id = 42
  and version = 7;   -- BẮT BUỘC version trong DB lúc này phải đúng là 7
```

Lúc này, A chạy trước sẽ sửa được 1 dòng. Đến lượt B chạy sau, vì B vẫn dùng version cũ `7` để đi tìm, nhưng DB lúc này đã lên `8` rồi, nên câu lệnh của B sửa được `0` dòng! 
Ngay lập tức, Hibernate sẽ ném thẳng cái ngoại lệ `OptimisticLockException` vào mặt B, Spring sẽ dịch nó thành `ObjectOptimisticLockingFailureException`, và giao dịch của B sẽ bị Hủy bỏ (rollback) toàn tập.

> **Nói ngắn gọn:** Khóa Lạc Quan `@Version` KHÔNG NGĂN CẢN 2 người cùng đọc chung dữ liệu; nhưng nó giúp **phát hiện ra sự đụng chạm (conflict)** khi ghi, thay vì nhắm mắt cho phép ghi đè trong im lặng (silent overwrite).

## Các "Diễn viên" và Dữ liệu dùng chung (Actor và trạng thái)

| Thành phần | Giá trị hiện tại |
| --- | --- |
| Sản phẩm `product_offer` | `offer_id=42`, Giá đang là `100`, Phiên bản `version=7` |
| Người dùng A (Editor A) | Nhập giá muốn đổi thành `90` |
| Người dùng B (Editor B) | Nhập giá muốn đổi thành `80` |
| Kẻ phán xử (Authoritative state) | Dòng dữ liệu dưới database PostgreSQL |

Điểm tranh chấp sứt đầu mẻ trán diễn ra NGAY LÚC câu lệnh UPDATE mang version được nã xuống DB (khi chạy hàm flush hoặc commit), **CHỨ KHÔNG PHẢI** lúc bạn gọi hàm Set trên Object Java hay gọi hàm `repository.save()`.

## Các Luật Bất biến (Invariant)

```text
Một thao tác lưu chỉ được phép chốt sổ (commit) nếu sản phẩm trong DB VẪN ĐANG Ở ĐÚNG PHIÊN BẢN (expected version) mà người dùng đó đã thấy lúc tải trang web.

Tuyệt đối không một thao tác nào mang dữ liệu ôi thiu (stale edit) được phép báo Thành Công, nếu đã có người khác nhanh tay chốt sổ trước đó.

Số Version CHỈ DO bản thân Framework (Hibernate) tự động quản lý tăng lên; Lập trình viên không bao giờ được viết code tự gán hay tự sửa số này.
```

Trong Case Study này, chúng ta chọn cách báo lỗi "Đụng độ, xin vui lòng tải lại trang và làm lại" (conflict), CHỨ KHÔNG tự động thử lại (retry) hành động gán giá này. 
Lý do là nếu bạn code cho nó "mù quáng" tự retry đè lên cái giá `90` của người kia thành `80` thì hệ thống sẽ loạn. (Muốn biết cách thử lại an toàn, hãy sang bài `LOCK-002`).

## Ranh giới Giao dịch (Transaction Boundaries)

Mỗi lần một nhân viên bấm Lưu, luồng xử lý qua Spring Proxy sẽ diễn ra y như sau:

```text
BẮT ĐẦU GIAO DỊCH (BEGIN)
1. Tải Sản phẩm kèm Version mới nhất từ DB lên
2. So sánh với cái Version cũ (expectedVersion) mà API đẩy lên
3. Đổi giá mới vào Object Java
4. Bắn câu SQL UPDATE có version xuống DB (flush)
5. Nếu suôn sẻ -> CHỐT SỔ (COMMIT). Nếu bị DB báo sửa 0 dòng -> Ném lỗi đụng độ Khóa Lạc Quan và HỦY BỎ (ROLLBACK).
```

Lỗi đụng độ có thể văng ra ở bước 4 (lúc xả rác flush) hoặc ở bước 5 (lúc commit). Nhớ rằng, Code của bạn chỉ được báo về Client là Thành Công SAU KHI cục Proxy Giao Dịch kia báo đã Commit xong xuôi.

## Bức tường ranh giới của Client (Detached/client boundary)

Việc bạn nhét `@Version` vào Entity thôi là **CHƯA ĐỦ AN TOÀN**. Tưởng tượng nếu bạn viết API hời hợt, không bắt Client (màn hình) gửi kèm cái Version mà họ đang xem:

```text
1. Màn hình máy B đang hiển thị version 7
2. Màn hình máy A chốt giá thành công, version trong DB nhảy lên 8
3. Máy B bấm Lưu, nhưng gửi API mà KHÔNG kẹp cái version 7 kia theo.
4. Code Backend của bạn ngây thơ chui xuống DB lấy cái v8 mới nhất lên, gán giá = 80 vào.
5. Code ra lệnh UPDATE với version = 8. THÀNH CÔNG! Đè bẹp luôn công sức của A!
```

Vì vậy, mọi lệnh sửa dữ liệu (command/API) **BẮT BUỘC** phải mang theo `expectedVersion` (cái Version mà người dùng đang nhìn thấy). Service Backend phải đối chiếu cái này với cái Version vừa lôi từ DB lên; còn cái `@Version` của Hibernate sẽ làm nhiệm vụ bảo vệ dữ liệu ở khúc chạy đua từ lúc Backend bốc dữ liệu lên đến lúc đẩy lệnh UPDATE xuống (luồng A và luồng B giành giật nhau ở dưới DB).

## Các Thuật ngữ dân trong nghề hay dùng

| Thuật ngữ | Ý nghĩa trong ngữ cảnh này |
| --- | --- |
| Khóa Lạc Quan (optimistic locking) | Cứ cho phép nhiều người cùng đọc thoải mái, nhưng ai lưu sau thì báo đụng độ (conflict) |
| Cột Version (version column) | Số đếm lịch sử thay đổi do Hibernate (persistence provider) tự xử |
| Version kỳ vọng (expected version) | Cái Version mà màn hình Client đang giữ (lúc họ bấm nút sửa) |
| Lệnh ôi thiu (stale entity/edit) | Lệnh gửi lên mang theo một cái Version lỗi thời, không còn là mới nhất |
| Số dòng được tác động (affected-row count) | = `1` nghĩa là bạn thắng; = `0` nghĩa là bạn thua vì sai Version |
| `OptimisticLockException` | Mã tín hiệu (signal) báo có đụng độ từ Jakarta Persistence ném ra |
| Flush | Giây phút hệ thống xả đống lệnh SQL bẩn (dirty) cất trên RAM xuống Database |
| Trạng thái detached | Cái Object Java bị rớt lại bên ngoài Giao dịch (ngoài persistence context) |

## Sơ đồ Bản đồ (Điều hướng)

- [Code viết sai và Lỗ hổng API không truyền Version](broken-code.md)
- [Phân tích Dòng thời gian khóa dòng và đụng độ MVCC](analysis.md)
- [Cách cấu hình `@Version`, Thiết kế API và Bắt mã lỗi (outcome mapping)](solutions.md)
- [Thực nghiệm bằng PostgreSQL Testcontainers](experiments.md)
- [Khái niệm: Khóa Lạc quan (Optimistic locking) và đụng độ phiên bản](../../concepts/optimistic-locking.md)
- [Khái niệm: Cơ chế MVCC của PostgreSQL](../../concepts/postgresql-mvcc.md)
- [Khái niệm: Kiểm thử đồng thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## Hậu quả nếu code ẩu đưa lên Production

- Nhân viên A nhận thông báo Thành Công, nhưng sáng mai ra check thì thấy bị đổi thành giá khác (của B).
- Lịch sử thay đổi (Audit) không hề lưu vết của B, không ai biết B đã đè lên A lúc nào!
- Bộ đệm (Cache) hoặc các hệ thống nhận Event báo lung tung không khớp với Database hiện tại.
- Lỗi đụng độ bị chôn vùi mãi đến tận lúc chốt sổ (commit), văng ra cho Client cái mã `500 Internal Server Error` chung chung khó hiểu.
- Cố tình viết vòng lặp "thử lại" trong một Giao dịch đã chết, kết quả chỉ nhận thêm 1 rổ Exception!
- Tranh chấp ở một cái Ví siêu Hot sẽ tạo ra hiệu ứng khuếch đại (amplification) làm cháy máy chủ.
- Dùng `Lock` của Java trên 1 máy nhưng có đến tận 5 máy cùng chạy chung 1 cái DB thì vô nghĩa (node-local lock không chống được tranh chấp).

## Hướng sửa chữa Khuyến nghị

1. Thêm cái cột `version not null` (tuyệt đối không null) và Annotation `@Version` vào class trọng tâm (aggregate root).
2. Viết file migration SQL điền version = 0 cho các dòng dữ liệu cũ TRƯỚC KHI bật Hibernate lên.
3. API và Request JSON **bắt buộc** phải có trường chứa Version.
4. Phải chủ động gọi `flush` ngay trong thân hàm để cục Proxy văng lỗi cho mình còn bắt được sớm. Phải nhớ bao bọc khối `catch` thật kỹ vì lúc `commit` nó cũng có thể ném ra Exception.
5. Bóp chết Giao Dịch (Rollback) kẻ đi sau và chuyển mã lỗi thành `409 Conflict` (Xung đột) hoặc `412 Precondition Failed` báo cho Frontend biết đường xử lý.
6. **TUYỆT ĐỐI KHÔNG tự động thử lại** những thao tác nhập liệu của con người (user edit); Hãy báo cho họ biết bản mới nhất trên DB là gì và để họ tự quyết định nạp lại hay hợp nhất (merge).

## Khi nào nên xài bài này?

Khóa Lạc Quan sinh ra là để xử lý các bảng dữ liệu ít/vừa bị tranh chấp (contention thấp/vừa), và áp dụng cho mấy món mà kẻ đi sau (loser) có thể bấm Hủy, tải lại trang, hoặc nhìn màn hình để tự sửa tay. 
Nếu bạn đang code hệ thống Ví tiền (hot counter) hay đếm số lượng Tồn kho (stock delta) chạy liên tục hàng nghìn click mỗi giây, hãy chuyển qua xài bài Cộng Trừ Nguyên Tử (SQL) hoặc Khóa Bi Quan xếp hàng.

## Phạm vi bài học

Case này tập trung hướng dẫn bạn BẮT TẬN TAY CÁI LỖI ĐỤNG ĐỘ. 
Nếu bạn muốn code cho nó "Tự động thử lại" (Retry có jitter), mời đọc `LOCK-002`; Nếu muốn khóa mõm Bi quan (`FOR UPDATE`), xin mời qua `LOCK-003`; Nếu muốn Cập nhật Nguyên tử (Atomic mutation), mời đọc `LOCK-004`.
