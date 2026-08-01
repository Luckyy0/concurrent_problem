# Cạm Bẫy Đọc Đi Đọc Lại (DB-003): Non-repeatable read - Dữ Liệu Bất Nhất Giữa Hai Lần Đọc

## 1. Tóm tắt câu chuyện (Lỗi gì đây?)

Hãy tưởng tượng bạn là hệ thống tự động hoàn tiền. Chính sách của cửa hàng `M-42` đang cho phép duyệt tự động tối đa `100.00` (Phiên bản chính sách số `7`).
Bạn kiểm tra chính sách, thấy số tiền khách yêu cầu là `80.00` đủ điều kiện (nhỏ hơn `100.00`), và tiến hành phê duyệt.

Tuy nhiên, ngay lúc bạn đang chuẩn bị ghi nhận quyết định "Đã duyệt" vào cơ sở dữ liệu, một quản trị viên rủi ro cập nhật chính sách, giảm hạn mức xuống còn `50.00`, tăng phiên bản thành `8` và commit giao dịch.

Tiếp theo, bạn thực hiện câu lệnh SELECT lần hai để lấy số phiên bản chính sách ghi vào sổ nhật ký (audit). Với mức cô lập mặc định của PostgreSQL (`READ COMMITTED`), hệ thống cấp cho bạn một snapshot mới. Bạn nhận được phiên bản hiện tại là `8` và ghi nhận phiên bản này vào bản ghi quyết định.

Kết quả? Quyết định của bạn ghi nhận: "Hoàn tiền `80.00`, dựa theo Phiên bản `8`". Tuy nhiên, Phiên bản `8` chỉ cho phép tối đa `50.00`. Bạn vừa tạo ra một bản ghi dữ liệu mâu thuẫn và không hợp lệ trong suốt vòng đời của hệ thống. 

Nguyên tắc bắt buộc (Invariant):

```text
Mỗi quyết định hoàn tiền phải được đánh giá dựa trên MỘT snapshot chính sách đồng nhất.

Nếu quyết định = APPROVED (Duyệt):
  Số tiền phải <= Hạn mức (autoRefundLimit) CỦA ĐÚNG PHIÊN BẢN (policyRevision) được lưu cùng quyết định đó.

KHÔNG ĐƯỢC PHÉP:
  Sử dụng hạn mức của Phiên bản 7 để xét duyệt, nhưng lại ghi nhận Phiên bản 8 vào Nhật ký (Audit).
```

> **Ghi chú quan trọng:** Đừng giả định rằng hai câu lệnh SELECT trong cùng một giao dịch ở mức `READ COMMITTED` sẽ tự động trỏ về cùng một snapshot. Việc lấy một phần dữ liệu từ lần đọc trước kết hợp với dữ liệu từ lần đọc sau sẽ dẫn đến trạng thái dữ liệu bất nhất.

## 2. Các thành phần tham gia (Actors và shared state)

| Thành phần | Vai trò |
| --- | --- |
| Quy trình xét duyệt (Refund evaluator) | Đọc chính sách, đánh giá và ghi nhận quyết định hoàn tiền |
| Quản trị viên (Risk administrator) | Cập nhật hạn mức hoàn tiền |
| `merchant_refund_policy` | Bảng lưu trữ chính sách hiện hành |
| `refund_decision` | Bảng nhật ký ghi nhận quyết định hoàn tiền |
| Cơ chế MVCC của PostgreSQL | Quản lý và cung cấp snapshot (statement snapshot) cho mỗi lệnh SELECT |
| Giao dịch của Spring | Đóng gói các lệnh SELECT và lệnh INSERT của quy trình xét duyệt vào một Giao dịch duy nhất |

Trạng thái chính sách ban đầu:

```text
merchant_id       = M-42
auto_refund_limit = 100.00
active            = true
revision          = 7
```

Trạng thái chính sách sau khi quản trị viên cập nhật:

```text
merchant_id       = M-42
auto_refund_limit = 50.00
active            = true
revision          = 8
```

Bản ghi quyết định lỗi:

```text
amount             = 80.00
outcome            = APPROVED
evaluated_limit    = 100.00  (Lấy từ lần đọc 1)
policy_revision    = 8       (Lấy từ lần đọc 2)
```

## 3. Hiện trường tranh chấp (Transaction boundary và contention point)

Luồng xử lý xét duyệt thông qua Spring Proxy:

```text
BẮT ĐẦU GIAO DỊCH VỚI MỨC CÔ LẬP 'READ COMMITTED'
  LẦN ĐỌC 1: Đọc Chính sách                      -> Kết quả: Hạn mức=100, Phiên bản=7
  Trong RAM: Xét duyệt APPROVED (80 < 100)
  ... Quản trị viên cập nhật chính sách và COMMIT ...
  LẦN ĐỌC 2: Đọc Chính sách để lấy Phiên bản     -> Kết quả: Hạn mức=50, Phiên bản=8
  GHI NHẬN QUYẾT ĐỊNH (INSERT)                   -> Ghi nhận: APPROVED, phiên bản=8
COMMIT GIAO DỊCH
```

Luồng cập nhật của quản trị viên (chạy độc lập):

```text
BẮT ĐẦU GIAO DỊCH
  CẬP NHẬT merchant_refund_policy
     SET auto_refund_limit=50, revision=8
COMMIT GIAO DỊCH
```

Điểm xung đột nằm ở bản ghi `merchant_refund_policy(M-42)`. Hai lệnh SELECT của quy trình xét duyệt KHÔNG giữ khóa (row lock). Quản trị viên có thể cập nhật và commit ngay giữa hai lệnh SELECT này. Cuối cùng, hệ thống ghi vào bảng `refund_decision` - một bảng riêng biệt, do đó cơ sở dữ liệu không phát hiện xung đột ghi (write conflict) trực tiếp. Trạng thái bất nhất được lưu trữ thành công.

## 4. Kỳ vọng và Thực tế (Expected vs. Actual)

| Bước | Kỳ vọng của lập trình viên (Broken assumption) | Thực tế hoạt động của PostgreSQL `READ COMMITTED` |
| --- | --- | --- |
| Lần đọc 1 (Xét duyệt) | Chính sách sẽ không thay đổi trong suốt giao dịch | PostgreSQL cấp Snapshot số 1 (Phiên bản 7) |
| Quản trị viên commit | Không ảnh hưởng đến giao dịch đang xét duyệt | Phiên bản 8 chính thức có hiệu lực trên toàn hệ thống |
| Lần đọc 2 (Xét duyệt) | Dữ liệu trả về giống hệt lần đọc 1 | PostgreSQL cấp Snapshot số 2 (Phiên bản 8 mới nhất) |
| Xét duyệt ghi sổ (INSERT) | Quyết định và dữ liệu audit nhất quán | Dữ liệu hỗn hợp: Quyết định dựa trên Phiên bản 7, audit ghi Phiên bản 8 |
| Cảnh báo lỗi | Cơ sở dữ liệu sẽ chặn hoặc văng Exception | Không có lỗi, cả hai giao dịch đều hoàn thành thành công |

## 5. Từ vựng cốt lõi (Terminology)

| Thuật ngữ | Diễn giải |
| --- | --- |
| non-repeatable read (Đọc không lặp lại) | Hiện tượng đọc lại một bản ghi nhưng nhận được dữ liệu khác do có giao dịch khác đã cập nhật bản ghi đó. |
| statement snapshot (Ảnh chụp cấp lệnh) | Trạng thái cơ sở dữ liệu được PostgreSQL cấp riêng biệt cho *từng* câu lệnh trong giao dịch (ở mức `READ COMMITTED`). |
| transaction snapshot (Ảnh chụp cấp giao dịch) | Trạng thái ổn định duy trì xuyên suốt toàn bộ giao dịch (khi sử dụng mức `REPEATABLE READ`). |
| coherent snapshot (Ảnh chụp đồng nhất) | Trạng thái yêu cầu mọi thuộc tính dữ liệu lấy ra phải thuộc về cùng một phiên bản logic. |
| revalidation (Xác nhận lại) | Việc kiểm tra lại điều kiện kinh doanh ngay tại thời điểm ghi dữ liệu xuống cơ sở dữ liệu. |
| policy revision (Phiên bản chính sách) | Thuộc tính định danh sự thay đổi của quy tắc kinh doanh. |
| row lock (Khóa cấp dòng) | Cơ chế ngăn chặn các giao dịch khác sửa đổi bản ghi cho đến khi giao dịch hiện tại hoàn tất (commit/rollback). |
| serialization order (Thứ tự tuần tự hóa) | Sắp xếp lịch trình thực thi đồng thời sao cho kết quả tương đương với việc chạy từng giao dịch một cách tuần tự. |

## 6. Điều hướng (Navigation)

- [Đoạn mã có lỗi (Broken two-read decision)](broken-code.md)
- [Phân tích nguyên nhân gốc rễ (Statement-snapshot analysis)](analysis.md)
- [Các giải pháp: Snapshot, validation và locking (Solutions)](solutions.md)
- [Thực nghiệm và tái hiện lỗi (Deterministic PostgreSQL experiments)](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Các mức cô lập (Isolation levels)](../../concepts/isolation-levels.md)
- [Kiểm thử đồng thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Hậu quả thực tế (Production impact)

- Bảng Audit ghi nhận bằng chứng không hợp lệ: Trích dẫn phiên bản chính sách không hỗ trợ cho hạn mức đã duyệt.
- Xảy ra thất thoát do hoàn tiền vượt hạn mức hiện tại nhưng vẫn được ghi nhận là hợp lệ.
- Quy trình đối soát (reconciliation) gặp khó khăn do dữ liệu lưu trữ bị mâu thuẫn.
- Cùng một lệnh xử lý nhưng dữ liệu trả về, sự kiện (event) và log có thể chứa các phiên bản khác nhau.
- Lỗi xảy ra âm thầm không ném Exception, gây khó khăn cho việc gỡ lỗi.
- Môi trường đa máy chủ (multi-instance) tăng tần suất xuất hiện lỗi. Các cơ chế khóa nội bộ JVM hoàn toàn vô hiệu.

## 8. Hướng khắc phục (Recommendation)

Cần lựa chọn chiến thuật phù hợp với yêu cầu kinh doanh:

1. **Sử dụng duy nhất một snapshot:** Nếu quyết định chỉ cần hợp lệ với chính sách "Tại thời điểm bắt đầu xét duyệt", hãy Đọc bảng chính sách MỘT lần duy nhất. Sử dụng bản ghi đó (immutable snapshot) cho mọi bước tính toán và lưu trữ. Yêu cầu hệ thống lưu lại lịch sử các phiên bản chính sách.
2. **Xác nhận lại trước khi cập nhật:** Nếu quyết định yêu cầu phải tuân theo chính sách mới nhất tại thời điểm ghi, hãy sử dụng truy vấn có điều kiện (conditional validation). Nếu số lượng dòng ảnh hưởng (affected-row) trả về là 0, tức là chính sách đã đổi, cần hủy giao dịch và thực hiện lại từ đầu.
3. **Chiếm khóa bi quan (Pessimistic lock):** Nếu cần chặn tuyệt đối mọi thay đổi chính sách trong lúc xét duyệt, hãy sử dụng lệnh khóa dòng `FOR SHARE`. Giao dịch muốn cập nhật chính sách sẽ phải chờ.
4. **Nâng cấp mức cô lập:** Mức `REPEATABLE READ` ngăn chặn được non-repeatable read, giữ cho dữ liệu ổn định từ đầu đến cuối giao dịch, NHƯNG nó không đảm bảo đây là dữ liệu mới nhất (latest-at-commit). 

Việc áp dụng cơ chế phải tương thích với Invariant của hệ thống, không chỉ đơn thuần là làm cho hai lệnh SELECT trả về kết quả giống nhau.

## 9. Phạm vi bài viết (Scope)

Bài viết này tập trung vào sự thay đổi trên MỘT BẢN GHI CỤ THỂ (logical row) bị cập nhật khi đọc lại. Nếu việc truy vấn dựa trên điều kiện (predicate query) trả về tập kết quả thay đổi số lượng dòng, đó là hiện tượng Phantom read (DB-004). Lỗi ghi đè trực tiếp (lost update) thuộc DB-001; Kỳ vọng đọc dữ liệu tạm (dirty-read expectation) thuộc DB-002.
