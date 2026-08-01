# Dòng thời gian tranh chấp và nguyên nhân gốc

## Trạng thái ban đầu

Giả sử hệ thống đang như thế này:

```text
limit = 10
active = 9
pending = 0
used = active + pending = 9
```

Lúc này, hai request T1 và T2 cùng lúc muốn chen chân giành lấy một kết nối. Cả hai cùng gọi hàm `tryReserveCreation()` trên cái Spring singleton đó.

## Interleaving thứ nhất: kiểm tra rồi increment (check-then-increment)

| Bước | T1 (Thread 1) | T2 (Thread 2) | Tình hình hệ thống (State) |
| --- | --- | --- | --- |
| 1 | Thấy `active = 9` | | `active=9, pending=0` |
| 2 | Thấy `pending = 0`, cộng lại là 9 nên nghĩ là okay (pass) | | Vẫn chưa chốt đơn (reserve) |
| 3 | (Khựng lại chút) | Đọc thấy `active = 9` và `pending = 0`, cũng thấy okay | Cả hai thanh niên đều tin là còn dư slot |
| 4 | Gọi `pending.incrementAndGet()` → 1 | | Tổng lên 10 |
| 5 | | Gọi `pending.incrementAndGet()` → 2 | Bùm! Tổng 11, vượt quá limit |
| 6 | Trả về `true` | Trả về `true` | Cả hai cùng nhào vô tạo kết nối |

Mọi hàm `get` và `incrementAndGet` thì bản thân chúng làm rất tốt nhiệm vụ nguyên tử (atomic). Nhưng cái lỗi "chí mạng" là hành động nghiệp vụ bao gồm: kiểm tra xem còn chỗ không (check), rồi mới giành chỗ (reserve).

> **Nói ngắn gọn:** Hai thread này không đánh nhau ở bước cộng biến đếm; tụi nó đánh nhau ở cái quyền quyết định rằng "chỗ cuối cùng này là của tui".

## Interleaving thứ hai: Khe hở dung lượng ảo (capacity gap)

Giờ tình huống là nhà đã đầy:

```text
active = 9, pending = 1, used = 10
```

T1 bắt tay (handshake) thành công và tiến hành chuyển kết nối từ `pending` sang `active`. Code bị lỗi của chúng ta sẽ làm chuyện này bằng cách giảm biến này rồi mới tăng biến kia:

| Bước | T1 (Đang hoàn tất) | T2 (Request mới) | Tình hình hệ thống |
| --- | --- | --- | --- |
| 1 | `pending.decrementAndGet()` → 0 | | `active=9, pending=0` (Ối dồi ôi, tự nhiên dư ra 1 chỗ ảo!) |
| 2 | (Khựng lại chút) | Đọc thấy tổng là 9, nhào vô giành slot: pending → 1 | `active=9, pending=1` |
| 3 | T1 tỉnh dậy gọi `active.incrementAndGet()` → 10 | | `active=10, pending=1` |
| 4 | T1 hoàn thành chốt sổ | T2 cũng hớn hở vào nhà | Chúc mừng, tổng là 11 |

Xét về logic nghiệp vụ, cái kết nối đang pending kia nó chỉ "biến hình" sang active thôi. Nó không có vụ trả vé rồi xin lấy vé mới. Do thao tác cập nhật của bạn bị gãy làm đôi, một trạng thái "ảo tung chảo" chưa từng tồn tại đã lòi ra cho thiên hạ thấy.

## Kết quả mong đợi và kết quả thực tế

| Điểm cần xét | Lẽ ra phải thế này (Mong đợi) | Code bị lỗi nó lại thế này |
| --- | --- | --- |
| Sức chứa (Capacity) | `active + pending <= limit` mọi lúc mọi nơi | Tổng lâu lâu lại vượt limit |
| Giành slot (Reservation)| Slot cuối chỉ một người được ăn | Cả đám xúm vào và cùng tưởng mình đã có slot (`true`) |
| Chuyển trạng thái | Pending hóa active thì tổng số vé vẫn vậy | Lòi ra khe hở dung lượng ảo do giảm xong mới tăng |
| Xem nhanh (Snapshot) | Health check xem là phải ra số chuẩn | Gọi `get()` hai lần rải rác có thể ghép ra kết quả tào lao |
| Nhả slot (Release) | Xong một vòng đời (lifecycle) thì nhả đúng 1 slot | Callback gọi đúp làm số bị âm hoặc tạo ra slot ma |
| Hủy (Failure) | Tạo thất bại thì tự giác nhả slot pending đó ra | Mấy cái biến đếm tổng cục này làm sao biết slot đó thuộc về ai |

## Nguyên nhân theo từng lớp

### Volatile

Đọc volatile thì luôn thấy giá trị mới nhất, đúng luật Java Memory Model. Nhưng cái trò `used++` lại gồm tới ba bước: đọc, cộng, ghi. Thread khác hoàn toàn có thể chen ngang. Nhìn thấy mọi thứ (visibility) không có nghĩa là không ai được xen vào (atomicity).

### AtomicInteger

Nó chỉ bảo vệ cho đúng một số nguyên thôi. Đừng mơ nó tự tạo ra một cái lồng bảo vệ (transaction) cho 2 biến đếm cùng lúc hay gom cái đám `active.get() + pending.get()` lại thành một thao tác liền mạch.

Ngay cả cái hàm `view()` cũng đọc active rồi tới pending. Lỡ như ở giữa 2 lần đọc đó có cái transition nào chạy ngang, thì hàm `view` có thể bốc nhầm râu ông nọ cắm cằm bà kia.

Để hiểu sâu hơn, ghé đọc [Java Memory Model, volatile và atomic variable](../../concepts/java-memory-model-and-atomicity.md).

### Compound invariant (Quy tắc ghép)

Quy tắc này nằm ở một `BudgetState` tổng thể, thế nên vòng bảo vệ cũng phải bao trọn cái state đó. Có 3 cách hay xài:
- Gom tất cả vào một object bất biến (immutable) và xài CAS trên đó.
- Xài một cái lock (ổ khóa) để bảo vệ tất cả.
- Đổi cách làm: nếu chỉ quan tâm tới tổng số vé, quy nó về một biến đếm duy nhất thôi.

### Spring

Gắn Singleton thì mọi thằng đều gọi chung một cái budget, nhưng nó KHÔNG hề bắt tụi nó xếp hàng (serialize) gọi hàm. Mấy cái thread tạo request, check health, hay callback đều có thể đâm chém nhau loạn xạ.

### Transaction

Ở đây không có database transaction hay rollback gì ráo. `@Transactional` không có phép thần thông khóa mấy biến trong heap của Java lại đâu. Còn dọn dẹp lỗi thì phải tự code tử tế vào lifecycle của kết nối.

## Linearization point (Điểm chốt sổ) của lời giải CAS

Nếu dùng `AtomicReference<BudgetState>`:
1. Thread lấy một trạng thái bất biến (chứa cả active và pending).
2. Kiểm tra quy tắc và tính toán trạng thái tiếp theo lưu vào một biến nội bộ.
3. Chơi trò `compareAndSet(hiện_tại, tiếp_theo)` để cập nhật.
4. Nếu thất bại (do có ai nhanh tay hơn), đọc lại từ đầu.

Cái khoảnh khắc CAS thành công chính là linearization point. Tại đó việc kiểm tra sức chứa và giữ chỗ diễn ra như một cái chớp mắt. Lúc chuyển từ pending sang active cũng gom 2 biến lại đổi cùng lúc, dẹp luôn cái khe hở dung lượng ma kia.

Cái hàm tính toán trạng thái có thể chạy lại chục lần, nên đừng có rảnh mà bỏ log nghiệp vụ hay gọi API lung tung trong đó nhé (không sinh ra side effect).

## Kẻ thua cuộc (Loser), thử lại (retry) và contention

- Thằng nào thua CAS sẽ phải lóc cóc chạy lại với data mới nhất.
- Nếu thấy hết chỗ (full) thì nó trả về `false`; đây là báo hết quota (capacity rejection), không phải lỗi hệ thống (technical error).
- Vòng lặp CAS không thèm quan tâm chuyện công bằng (fairness). Đứa nào xui thì retry mệt xỉu luôn.
- Đừng lặp CAS cho những thứ dính tới kết nối mạng xa xôi; nó chỉ dành cho những cú chớp nhoáng trong memory thôi.
- Nếu bạn đề cao tính công bằng và không muốn bị chiếm tài nguyên, thì xài Lock hoặc Semaphore cho lẹ.

## Thất bại (Failure), timeout, crash và double callback

Phải xí chỗ trước khi đi bắt tay (handshake):
- Thành công: pending hóa active, tổng số vé giữ nguyên.
- Thất bại/timeout: nhả 1 pending.
- Đóng kết nối: nhả 1 active.
- Callback vô tình gọi đúp: phải có cách chặn lại (ví dụ dùng token), cấm được trừ biến đếm hai lần.

Mấy cái biến đếm tổng (aggregate counter) thì đếm mù thôi, nó chả biết callback này của thằng nào. Nếu bạn sợ vụ callback đúp, hãy xài permit handle (như `AtomicBoolean released`) hoặc gán ID cho từng reservation.

Nếu hệ thống sập (crash) thì biến đếm trong máy bay màu. HĐH sẽ ngắt kết nối, nhưng provider bên kia có thể sẽ phải chờ timeout mới biết. Nhớ là chúng ta không có tính lưu trữ dài lâu (durable lease) ở đây.

## Khi có nhiều application instance

Mỗi cái JVM tự quản budget riêng. Giả sử 3 con server cấu hình limit là 10, thì hệ thống bạn mở tới 30 kết nối. Mấy cái CAS, lock hay semaphore này chả biết đi tán dóc với server khác đâu.

Nếu provider chỉ cho 10 kết nối tổng cộng cho cả cụm server, thì bạn phải chia đều quota ra cho từng server, hoặc xài công cụ bên ngoài (Redis, Zookeeper...) để quản lý. Bài này chỉ nói chuyện trong phạm vi 1 máy chủ thôi.

## Hậu quả

### Hậu quả kỹ thuật

- Bão kết nối, vượt định mức, bị provider cấm cửa (throttling).
- Số liệu (metric) lấy sai bét do đọc lở dở giữa chừng.
- Counter bị âm hoặc rò rỉ dung lượng (leak) do gọi nhầm callback.
- Khi hệ thống đầy tràn (saturated), CPU bị đốt do vòng lặp CAS hoặc nghẽn lock.
- Bug chập chờn, chỉ lộ mặt khi limit sát nút.

### Hậu quả nghiệp vụ

- Lag sấp mặt, lỗi lan rộng ra các hệ thống khác.
- Lố quota của provider, các cửa hàng (merchant) khác trên cùng server chịu vạ lây.
- Alert/Autoscaling nhận data sai và hành động tào lao.
- Timeout xong retry lại càng sinh rác kết nối.
- Lỗi tại mình nhưng sếp tưởng do provider sập.

## Phạm vi không được giải quyết trong tình huống này

Chúng ta KHÔNG bàn chuyện kho bãi (inventory) lưu dài hạn, tiền nong (account balance), transaction của database, hay quota cho hệ thống phân tán. Cái chúng ta lo là quy tắc chùm (compound state) trong một căn nhà (JVM) thôi.
