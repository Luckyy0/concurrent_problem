# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

Giả sử ban đầu hệ thống của chúng ta đang có thông số thế này:

```text
nextSequence  = 41
lastCustomerId = null

T1 input = "alice"
T2 input = "bob"
```

Bạn nhìn dòng code `++nextSequence` có vẻ vô hại như một câu lệnh đơn, nhưng máy ảo nó xử lý thành 3 bước thế này:

```text
read nextSequence → add 1 → write nextSequence
```

## Thứ tự thực thi xen kẽ

Chuyện gì xảy ra khi hai ông luồng (thread) chạy song song và tranh nhau xử lý? Nó sẽ giống như vầy:

```text
T1: alice                                  T2: bob
----------------------------------------   ----------------------------------
lastCustomerId = "alice"
read nextSequence -> 41
                                             lastCustomerId = "bob"
                                             read nextSequence -> 41
                                             add 1 -> 42
                                             write nextSequence = 42
add 1 -> 42
write nextSequence = 42
read lastCustomerId -> "bob"
return ReceiptDraft(42, "bob")
                                             read lastCustomerId -> "bob"
                                             return ReceiptDraft(42, "bob")
```

## Kết quả mong đợi và kết quả thực tế

Cái chúng ta hy vọng:

```text
Expected:
  alice -> ReceiptDraft(42, "alice")
  bob   -> ReceiptDraft(43, "bob")
  unique sequences = 2
```

Và đời không như mơ:

```text
Actual:
  alice -> ReceiptDraft(42, "bob")
  bob   -> ReceiptDraft(42, "bob")
  unique sequences = 1
```

Thế là toang cả hai quy tắc sống còn của chúng ta:
- Cái ID `42` bị lấy xài tận 2 lần vì luồng nọ ghi đè luồng kia (`lost update`).
- Tội nghiệp cô Alice, bản nháp trả về tự nhiên lại cõng cái tên "bob".

> **Nói ngắn gọn:** Hai luồng thi nhau đọc con số cũ, thi nhau ghi đè con số mới. Đã vậy ông B còn nhanh tay sửa cái tên trước khi ông A kịp trả kết quả ra ngoài.

## Nguyên nhân theo từng lớp

### Vòng đời Spring bean

Class đánh `@Service` trong Spring thì vòng đời mặc định là `singleton` - tức là chỉ có duy nhất 1 cục object. Spring không tạo mới service cho mỗi HTTP request. Do đó, nếu bạn có biến gì trong cục service đó, các request cứ thế mà dùng chung. Spring chỉ đảm bảo lúc khởi tạo nó an toàn thôi, còn lúc request lao vào thì bạn tự lo.

### JVM và Java Memory Model

Đây gọi là tình trạng **tranh chấp dữ liệu** (`data race`). Hai luồng cùng sờ vào một biến, có ít nhất một đứa nhảy vào sửa, mà chẳng có khoá (lock) gì cả để xếp hàng.

Tổng hợp lại ta có hai cái lỗi bự:
1. `++nextSequence` tưởng là một bước nhưng lại là 3 bước đọc-sửa-ghi, nên rất dễ đụng hàng.
2. `lastCustomerId` đáng lẽ phải là đồ riêng của từng request, bạn lại đem lưu vào biến chung, đợi lúc đọc lại thì nó đã bị đứa khác sửa mất tiêu rồi.

Quá trình "phá hoại" diễn ra theo trình tự này:

```text
shared field write → interleaving → compound update/read shared field
```

Tóm lại, không phải cứ có đông request mới sinh lỗi, mà là do các request nhảy vào sửa trạng thái chung mà không thèm "đóng cửa phòng" lại.

### Spring transaction, Hibernate và database

Trong ví dụ này chúng ta không dùng DB, nên mấy cái xịn xò như `persistence context`, `MVCC` hay `row lock` đều vứt. Dù bạn có quăng `@Transactional` vào thì mỗi request vẫn mở một transaction riêng, nhưng chúng lại trỏ về cùng cái object trên RAM. Database có rollback đi nữa thì mấy cái biến trên Java heap cũng chẳng thể quay về giá trị cũ đâu.

## Ảnh hưởng của commit, rollback, timeout và crash

- **Commit/rollback:** Nó chỉ tác dụng với database thôi. Mấy biến cục bộ chạy trên RAM không có "sổ ghi nợ" (transaction log) để khôi phục đâu.
- **Lỗi (Exception):** Nếu hệ thống đang đếm sequence lên mà bị dính lỗi cái ầm, số đó sẽ bị hụt mất, và không ai thèm rollback lại.
- **Timeout/retry:** Request bị timeout, client bấm gọi lại (retry), hệ thống lại đẻ ra thêm một bản nháp mới tinh. Vụ này không chuẩn `idempotent` (chạy lại n lần vẫn ra 1 kết quả) chút nào.
- **Sập nguồn/Restart:** Bùm, bộ đếm trên RAM quay về số không, và thế là nó lại lôi mấy sequence cũ ra cấp lại từ đầu.

Đấy, bạn thấy biến bộ đếm trên RAM lỏng lẻo cỡ nào chưa? Đừng bao giờ dùng nó để tạo ID quan trọng nhé.

## Khi có nhiều application instance

Bây giờ sếp bắt chạy hệ thống ra 2 máy chủ (node) khác nhau:

```text
App A: nextSequence = 41, monitor/AtomicLong A
App B: nextSequence = 41, monitor/AtomicLong B
```

Dù bạn có nhét `synchronized` hay xài `AtomicLong` khét lẹt trên code, thì cả 2 máy chủ cũng vẫn cấp ra cái sequence y chang nhau thôi! Ổ khoá nhà ông A làm sao mà khoá cửa nhà ông B được?

> **Nói ngắn gọn:** Khoá cục bộ (local lock) không có cửa khi chạy đa máy chủ.

## Hậu quả

### Hậu quả kỹ thuật

- Mất cập nhật, sequence trùng nhau rần rần;
- Dữ liệu bị cắm sừng, râu ông nọ cắm cằm bà kia;
- Nhìn log chẳng khác gì nồi cám heo, không truy vết được;
- Bug này "hên xui" theo lịch trình chạy của hệ điều hành, có lúc bị lúc không;
- Server sập phát là bay sạch dữ liệu.

### Hậu quả nghiệp vụ

- Khách hàng hốt hoảng khi thấy dữ liệu của người khác trong tài khoản mình;
- Dữ liệu rác đẩy xuống các hệ thống liên quan gây loạn cào cào;
- Việc điều tra sự cố gặp bế tắc vì log ghi sai bét;
- Rò rỉ dữ liệu (data leak) cực kỳ nguy hiểm.

## Vì sao lỗi khó tái hiện bằng kiểm thử đơn vị thông thường

Nếu bạn chỉ viết unit test tuần tự kiểu:

```text
call A hoàn tất → call B bắt đầu
```

Thì làm sao bạn chọc đúng vào cái tích tắc giữa bước đọc và bước ghi được? Để móc được cái lỗi luồng này ra ánh sáng, bạn phải dùng các chiêu trò ép ép (barrier, latch) để các luồng đụng nhau đúng thời điểm. Bí kíp nằm ở [phần kiểm thử](experiments.md) nhé!
