# Cạm Bẫy Đọc Đi Đọc Lại (DB-003): Non-repeatable read - Đem Râu Ông Nọ Cắm Cằm Bà Kia

## 1. Tóm tắt câu chuyện (Lỗi gì đây?)

Hãy tưởng tượng bạn (Ông A) là hệ thống tự động hoàn tiền. Chính sách của cửa hàng `M-42` đang cho phép bạn duyệt tự động tối đa `100.00` (Phiên bản chính sách số `7`).
Bạn liếc mắt vào chính sách, thấy số tiền khách yêu cầu là `80.00` đủ điều kiện (nhỏ hơn 100.00). Bạn tự tin gật gù: "Duyệt!".

Nhưng khoan! Đúng lúc bạn đang hì hục cắm cúi viết giấy chứng nhận "Đã duyệt", thì Sếp B (Ông B - Quản trị viên rủi ro) chạy ngang qua, đổi rẹt chính sách xuống còn `50.00`, tăng Phiên bản thành `8` rồi chốt sổ.

Bạn viết xong giấy duyệt, quay lên nhìn lại cơ sở dữ liệu để lấy số Phiên bản ghi vào giấy cho chắc ăn (SELECT lần hai). Mức cô lập mặc định của PostgreSQL (`READ COMMITTED`) cấp cho bạn một bức ảnh mới toanh (snapshot mới), bạn thấy Phiên bản hiện tại là `8`. Bạn hồn nhiên ghi `8` vào giấy chứng nhận.

Kết quả? Tờ giấy của bạn ghi: "Hoàn tiền `80.00`, dựa theo Phiên bản `8`". Trong khi Phiên bản `8` chỉ cho phép tối đa `50.00`! Bạn vừa tạo ra một Chứng cứ giả mạo mà ở bất kỳ thời điểm nào trong lịch sử cũng không hề tồn tại. Đem râu ông nọ cắm cằm bà kia là đây!

Nguyên tắc bắt buộc (Invariant):

```text
Mỗi quyết định hoàn tiền phải dựa trên MỘT bức ảnh chính sách (snapshot) đồng nhất.

Nếu quyết định = APPROVED (Duyệt):
  Số tiền phải <= Hạn mức (autoRefundLimit) CỦA ĐÚNG CÁI PHIÊN BẢN (policyRevision) được lưu cùng quyết định đó.

KHÔNG ĐƯỢC PHÉP:
  Lấy luật của Phiên bản 7 ra xét duyệt...
  Nhưng lại điền Phiên bản 8 vào Sổ Nhật ký (Audit) để nộp lên sếp.
```

> **Nói ngắn gọn:** Đừng ngây thơ tin rằng hai câu lệnh SELECT trong cùng một Giao dịch (`READ COMMITTED`) sẽ tự động nhìn về cùng một bức ảnh! Việc bạn bóc dữ liệu của lần Đọc Trước ghép với lần Đọc Sau sẽ đẻ ra một con quái vật dữ liệu không bao giờ hợp lệ.

## 2. Diễn viên và Đạo cụ (Actors và shared state)

| Thành phần | Vai trò |
| --- | --- |
| Ông A (Refund evaluator) | Đọc luật, xét duyệt và viết giấy hoàn tiền |
| Ông B (Risk administrator) | Người chuyên đi thay đổi Hạn mức hoàn tiền |
| `merchant_refund_policy` | Bảng Luật (chính sách hiện tại) |
| `refund_decision` | Bảng Sổ Nhật ký (lưu lại bằng chứng quyết định) |
| Chức năng MVCC của PostgreSQL | Quản gia chuyên phát "Ảnh chụp" (statement snapshot) cho mỗi lệnh SELECT |
| Giao dịch của Spring | Kẻ bao bọc cả 2 lần SELECT và 1 lần INSERT của Ông A vào một Giao dịch duy nhất |

Sổ Luật Ban Đầu (Đã chốt):

```text
merchant_id       = M-42
auto_refund_limit = 100.00
active            = true
revision          = 7
```

Sổ Luật Mới bị Ông B sửa (Vừa chốt xong):

```text
merchant_id       = M-42
auto_refund_limit = 50.00
active            = true
revision          = 8
```

Tờ Quyết định Thảm họa:

```text
amount             = 80.00
outcome            = APPROVED
evaluated_limit    = 100.00  (Lấy từ lúc đầu)
policy_revision    = 8       (Lấy từ lúc sau)
```

## 3. Hiện trường vụ án (Transaction boundary và contention point)

Ông A chạy qua Proxy của Spring:

```text
BẮT ĐẦU VỚI MỨC CÔ LẬP 'READ COMMITTED'
  LẦN ĐỌC 1: Đọc Luật                           -> Thấy hạn mức=100, phiên bản=7
  Trong RAM: Xét duyệt APPROVED (80 < 100)
  ... Đùng một cái! Ông B thay đổi luật và chốt Sổ (COMMIT) ...
  LẦN ĐỌC 2: Lại Đọc Luật để lấy Phiên bản điền Audit  -> Thấy hạn mức=50, phiên bản=8
  GHI VÀO SỔ NHẬT KÝ (INSERT)                   -> Ghi: APPROVED, phiên bản=8
CHỐT SỔ (COMMIT)
```

Ông B chạy ở một Giao dịch độc lập khác hoàn toàn:

```text
BẮT ĐẦU
  CẬP NHẬT merchant_refund_policy
     SET auto_refund_limit=50, revision=8
CHỐT SỔ (COMMIT)
```

Điểm tử huyệt ở đây là dòng dữ liệu của ông `merchant_refund_policy(M-42)`. Hai cú đọc SELECT chay của ông A KHÔNG HỀ giữ khóa (row lock) bảo vệ. Ông B thoải mái nhào vô sửa và chốt (commit) ngay giữa hai lệnh Đọc của A. Cuối cùng ông A ghi vào bảng Nhật ký (`refund_decision`) - một bảng khác hoàn toàn, nên DB chả thấy có va chạm Ghi Đè (write conflict) trực tiếp nào để mà PostgreSQL từ chối cả. Mọi thứ được duyệt cho qua lọt khe!

## 4. Ảo mộng và Đời thực (Expected và actual)

| Bước | Ảo tưởng của bạn (Broken assumption) | Đời thực phũ phàng của PostgreSQL `READ COMMITTED` |
| --- | --- | --- |
| Lần đọc 1 của A | Luật lệ sẽ không đổi cho đến hết giao dịch này | Quản gia phát Bức Ảnh số 1 (thấy Phiên bản 7) |
| Ông B chốt sổ | Việc của B không ảnh hưởng tới A | Phiên bản 8 chính thức có hiệu lực toàn cục |
| Lần đọc 2 của A | Luật vẫn là luật ban nãy | Quản gia phát Bức Ảnh số 2 (thấy Phiên bản 8 mới tinh) |
| Ông A ghi sổ (INSERT) | Quyết định và Bằng chứng luôn nhất quán | Gắn râu ông nọ (Phiên bản 8) vào cằm bà kia (Quyết định của Phiên bản 7) |
| Tín hiệu Cảnh báo lỗi | Hệ thống sẽ báo lỗi văng Exception nếu luật bị đổi | Đâu có ai lỗi đâu, A và B cùng vui vẻ báo Thành công! |

## 5. Từ điển Bỏ túi (Thuật ngữ cần biết)

| Thuật ngữ | Diễn giải dân dã |
| --- | --- |
| non-repeatable read (Đọc không lặp lại được) | Vừa đọc xong quay lại đọc dòng đó thêm lần nữa thì thấy dữ liệu bị đứa khác sửa mất tiêu rồi |
| statement snapshot (Ảnh chụp cho từng câu lệnh) | Bức ảnh toàn cảnh CSDL mà PostgreSQL phát riêng lẻ cho *mỗi* câu lệnh Đọc (nếu đang ở mức `READ COMMITTED`) |
| transaction snapshot (Ảnh chụp cho toàn giao dịch) | Bức ảnh ổn định xài chung từ đầu tới cuối Giao dịch (nếu dùng mức Cô lập xịn hơn là `REPEATABLE READ`) |
| coherent snapshot (Ảnh chụp Đồng nhất) | Tất cả các cột dữ liệu lấy ra phải thuộc về Cùng MỘT phiên bản luật |
| revalidation (Xác nhận lại trước giờ G) | Kiểm tra lại lần chót các điều kiện ngay tại khoảnh khắc đặt bút Ghi dữ liệu |
| policy revision (Phiên bản chính sách) | Con số định danh phiên bản của Luật dùng để ra quyết định |
| row lock (Khóa Dòng) | Đặt chỗ cấm đứa khác sửa vào dòng này cho đến khi mình chốt sổ hoặc hủy bỏ (commit/rollback) |
| serialization order (Thứ tự Tuần tự hóa) | Sắp xếp các giao dịch chạy song song sao cho ra kết quả giống hệt như bắt tụi nó chạy xếp hàng từng đứa một |

## 6. Đi đâu tiếp theo? (Điều hướng)

- [Đoạn Code Thảm Họa (Broken two-read decision)](broken-code.md)
- [Mổ Xẻ Nguyên Nhân Tận Gốc (Statement-snapshot analysis)](analysis.md)
- [Thuốc Đặc Trị: Chụp Ảnh, Xác Nhận Lại, và Khóa (Snapshot, validation and locking solutions)](solutions.md)
- [Phòng Thí Nghiệm Đập Tan Ảo Tưởng (Deterministic PostgreSQL experiments)](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Mức Độ Cô Lập (Isolation levels)](../../concepts/isolation-levels.md)
- [Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Hậu Quả Để Lại Cho Công Ty (Hậu quả production)

- Sổ Nhật ký (Audit) lưu lại bằng chứng giả: Trích dẫn Phiên bản Luật không hề cho phép quyết định đó.
- Hoàn tiền lố tay nhưng lại được công bố là "Tự Động Duyệt Hợp Lệ".
- Đối soát cuối tháng (Reconciliation) khóc thét vì không sao giải thích nổi lý do hoàn tiền từ Bằng chứng lưu trong DB.
- Cùng một lệnh xử lý nhưng trả về cho người dùng (response), bắn sự kiện (event) và ghi log lại mang các Phiên bản khác nhau loạn xị ngầu.
- Lỗi này chạy im re không hề quăng Exception, làm đội Dev cứ vò đầu bứt tai tưởng logic bị bug ngẫu nhiên.
- Càng nhiều Server chạy (nhiều application instances) thì tỷ lệ dính chưởng càng cao. Dùng khóa trong RAM Java (JVM-local lock) hoàn toàn phế võ công vì không bảo vệ được dữ liệu xài chung dưới DB.

## 8. Hướng Chữa Cháy (Hướng sửa khuyến nghị)

Khoan vội gõ phím, hãy chọn chiến thuật trước:

1. **Chụp 1 tấm ảnh duy nhất:** Nếu quyết định chỉ cần hợp lệ với luật "Tại thời điểm xét duyệt", thì hãy Đọc Bảng Luật đúng MỘT lần duy nhất. Giữ chặt cái kết quả đọc được đó (immutable snapshot) xài xuyên suốt và lưu đầy đủ thông tin vào Sổ Audit. Các phiên bản cũ của Luật nên được giữ lại (đừng xóa đi) để sau này còn có cái mà đối soát (audit).
2. **Luật mới nhất mới chịu:** Nếu bắt buộc quyết định phải tuân theo Luật Tươi Mới nhất ngay tại lúc Ghi sổ, hãy dùng truy vấn kiểm tra điều kiện (conditional validation) ngay dưới database. Nếu báo về số dòng bị ảnh hưởng (affected-row) là `0` (tức là luật đổi rồi), thì lập tức mở một giao dịch mới làm lại từ đầu.
3. **Chiếm khóa xếp hàng:** Nếu cấm tuyệt đối không cho ai đổi Luật khi bạn đang bận Xét duyệt, hãy dùng lệnh đọc chiếm khóa `FOR SHARE` (pessimistic read). Thằng nào muốn cập nhật luật (Updater) sẽ phải ngậm đắng nuốt cay chờ bạn chốt sổ xong hoặc bị timeout.
4. **Tăng Cấp Cô Lập:** Mức `REPEATABLE READ` quả thật chặn đứng được lỗi Đọc Lại (non-repeatable read), nhưng NHỚ KỸ: Nó giữ nguyên Bức Ảnh Cũ Rích từ đầu chí cuối, chứ nó không tự biến Bức Ảnh cũ đó thành "Luật mới nhất ở thời điểm Chốt sổ" (latest policy at commit) đâu nha!

Cơ chế phải khớp với Nguyên tắc bắt buộc (invariant), không chỉ làm trò lừa tình để hai cú SELECT trả về kết quả giống nhau.

## 9. Phạm Vi Câu Chuyện (Phạm vi)

Bài này chỉ bàn tới lỗi trên MỘT DÒNG DUY NHẤT (logical row) bị ai đó sửa khi bạn đọc lại. Nếu bạn dùng lệnh lọc (predicate query) quét cả Bảng và trả về tập kết quả bị thêm/bớt số lượng dòng, thì đó là Bóng Ma (phantom) của bài DB-004. Lỗi ghi đè (lost update) thuộc DB-001; Còn vụ mong chờ Đọc dữ liệu Tạm mà không ra (dirty-read expectation) nằm ở bài DB-002 nhé.
