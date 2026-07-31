# DB-008 — Kẹt Xe Database: Án Mạng Deadlock Vì Khóa Ngược Chiều (PostgreSQL deadlock do khóa row ngược thứ tự)

## 1. Tóm tắt (Bức tranh toàn cảnh)

Tưởng tượng em có hai lệnh chuyển tiền chạy cùng lúc, giữa hai cái ví, nhưng mà đi ngược chiều nhau.
Và mỗi Đứa chạy Giao Dịch (transaction) đều ôm khư khư cái ví Nguồn (source) trước, rồi mới ngó sang ví Đích (destination):

```text
Thằng T1: Chuyển từ ví A -> ví B. Chộp lấy ví A, đứng chờ ví B mở khóa.
Thằng T2: Chuyển từ ví B -> ví A. Chộp lấy ví B, đứng chờ ví A mở khóa.
```

Đấy! Hai thằng đứng nhìn nhau say đắm!
PostgreSQL nhìn thấy cảnh **Chờ Đợi Lẫn Nhau** (wait-for cycle) này ngứa mắt quá, bèn rút súng bắn bỏ một thằng (gọi là victim) để dẹp đường. Thằng đó sẽ văng cái lỗi SQLSTATE `40P01`. Thằng còn sót lại (may mắn) sẽ chạy tiếp sau khi xác của thằng kia bị dọn đi (rollback) và Khóa (lock) được nhả ra.

Bài học này dạy em 3 Nguyên tắc Vàng:

```text
1. Mọi con đường code mà phải Khóa nhiều ví cùng lúc, THÌ BẮT BUỘC phải tuân theo 1 Thứ Tự Chuẩn (canonical order) thống nhất (ví dụ: So sánh ID, nhỏ khóa trước, lớn khóa sau).

2. Đã gọi là chuyển tiền thì hễ Ghi nhận (commit) là phải trừ bên này và cộng bên kia. Có biến là xé bỏ (rollback) toàn bộ!

3. Nếu muốn cho thằng bị bắn chết (victim) sống lại (retry), thì mỗi lần thử lại phải mở một Phiên Giao Dịch Mới Tinh (new transaction), tải lại toàn bộ trạng thái mới nhất, và thử lại số lần có hạn thôi!
```

> **Nói ngắn gọn:** Chữ "Transaction" không phải bùa hộ mệnh chống Deadlock. Mọi Actor (nhân viên) phải khóa tài nguyên theo CÙNG 1 THỨ TỰ. Và cái trò Bấm Nút Làm Lại (retry) chỉ là lớp Cấp Cứu có giới hạn.

## 2. Diễn viên và Đồ Dùng Chung (Actor và trạng thái dùng chung)

Cái kho `account` là Cuốn Sổ Cái Quyền Lực (authoritative shared state):

| Cái Ví (Account) | Mã Số (ID) | Tiền ban đầu (Balance) |
| --- | ---: | ---: |
| A | `101` | `1_000` |
| B | `202` | `1_000` |

Hai luồng chuyển tiền chui qua 2 cái App khác nhau cùng lao vào xâu xé CSDL:

| Nhân Viên (Actor) | Lệnh Mệnh (Command) | Thao tác Chọc Gậy Bánh Xe (Thứ tự broken) |
| --- | --- | --- |
| Lính T1 trên App-1 | Chuyển `100` từ A sang B | Chộp khóa `101`, rồi thò tay khóa `202` |
| Lính T2 trên App-2 | Chuyển `70` từ B sang A | Chộp khóa `202`, rồi thò tay khóa `101` |

Điểm Chạm Trán xịt khói (`contention point`) chính là lúc Tụi nó gọi câu lệnh `SELECT ... FOR UPDATE` THỨ HAI.
Bởi vì lúc này mỗi đứa ĐỀU ĐANG ÔM một Ổ Khóa Dòng (row-level lock) mà thằng kia đang khao khát.

## 3. Ranh Giới Giao Dịch (Ranh giới transaction)

Một lần thử nghiệm ĐÚNG CHUẨN chỉ được xài duy nhất MỘT gói Spring transaction:

```text
BẮT ĐẦU (BEGIN)
  Khóa cái ví có Mã Số (ID) NHỎ HƠN
  Khóa cái ví có Mã Số (ID) LỚN HƠN
  Thẩm định xem thằng Chuyển còn đủ tiền không (trên dữ liệu vừa khóa)
  Trừ tiền thằng Chuyển
  Cộng tiền thằng Nhận
  Đẩy dữ liệu (flush)
CHỐT SỔ (COMMIT)
```

Nhớ nhé, cái mác `@Transactional` phải được gắn trên cái Hàm Xử Lý (worker bean). Thằng Trưởng Phòng Điều Phối Retry phải Đứng Ngoài Giao Dịch đó. Có vậy thì cái Tội Lỗi (exception) của lần chạy trước mới được xí xóa (rollback) sạch sẽ trước khi bắt đầu thử lại.

Trong bài test này, mình giả định xài hàng mặc định của PostgreSQL là `READ COMMITTED`. Ở cái level này, mỗi phát `SELECT ... FOR UPDATE` sẽ bốc Dữ Liệu Tức Thời (statement snapshot) rồi Cắm Cọc Chờ Ổ Khóa Dòng. Ổ Khóa đó sẽ Được Ngậm cho tới khi Em Nhấn Nút `COMMIT` hoặc Bị Ép `ROLLBACK`, CHỨ KHÔNG PHẢI chạy xong cái Repository Method là nó nhả Khóa ra đâu nha!

## 4. Luật Thép Bất Di Bất Dịch (Invariant và kết quả mong đợi)

Ví dụ này tóm gọn cái Deadlock, không phải dạy viết App Ngân Hàng Hoành Tráng. Trong khuôn khổ bài này, em phải đảm bảo:

- Tổng tiền của A và B vĩnh viễn là `2_000`;
- Hễ báo chuyển thành công là Bắt Buộc Tiền bên A giảm và bên B tăng đúng số;
- Thằng Nạn Nhân (deadlock victim) tuyệt đối KHÔNG ĐƯỢC để lại bãi rác Dữ Liệu dở dang;
- Sau khi đánh lộn (và làm lại vài lần), hai Lệnh Chuyển tiền đều phải chạy Xong Hết, Hoặc báo Thất Bại Tức Tưởi (exhaustion);
- CẤM TIỆT mấy trò Gửi Email/Bắn Sự Kiện ra ngoài Trước Khi `COMMIT`, trừ phi rành rẽ món Nhắn Tin Ngoài Hộp (outbox/idempotency).

Nếu code lởm (Broken implementation), em mong tụi nó tự xếp hàng (serialize)? Đừng nằm mơ, PostgreSQL sẽ Đấm Vỡ Mặt (abort) một thằng. Nếu code App nhắm mắt làm ngơ nuốt lỗi, chạy lại ngay trên cái Đống Đổ Nát (transaction cũ), hoặc dối trá báo Thành Công sớm, thì Luật Thép bị Phá Vỡ Tan Tành!

## 5. Từ Lóng Giang Hồ Phải Biết (Thuật ngữ cần biết)

| Từ Lóng | Giải nghĩa bình dân |
| --- | --- |
| Kẹt Xe (deadlock) | Mấy Giao Dịch cắn đuôi chờ nhau thành 1 Vòng Tròn Luẩn Quẩn |
| Lưới Chờ (wait-for graph) | Cái Sơ Đồ Vẽ Thằng Nào Đang Ngồi Chờ Thằng Nào |
| Chuẩn Xếp Hàng (canonical lock order) | Luật Tôn Ti Trật Tự; Mã Số (ID) NHỎ được Mời Vô Phòng Khóa TRƯỚC |
| Kẻ Chết Thay (victim) | Đứa bị DB Bắn Bỏ để dẹp Đường Kẹt Xe |
| SQLSTATE `40P01` | Mã Báo Tử của PostgreSQL cho tội Kẹt Xe (`deadlock_detected`) |
| Giao dịch đứt gánh (aborted transaction) | Lệnh không cho chạy tiếp nữa, ép phải Rollback |
| Thử Lại Có Hạn (bounded retry) | Trò làm lại nhưng giới hạn Số Lần, Dãn Cách (backoff) và Có Mốc Hết Giờ |
| Làm Lại Cuộc Đời (fresh attempt) | Chạy lại trên một Môi Trường Mới Tinh Tươm (transaction/persistence context mới) |

## 6. Bản Đồ Kho Báu (Điều hướng)

- [Broken Spring/JPA implementation](broken-code.md)
- [Timeline, detector và rollback analysis](analysis.md)
- [Canonical ordering và safe retry](solutions.md)
- [PostgreSQL Testcontainers experiments](experiments.md)
- [PostgreSQL locks và lock lifetime](../../concepts/postgresql-locks.md)
- [Deadlock và retry an toàn](../../concepts/deadlocks-and-retries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## 7. Thảm Họa Trên Chiến Trường (Hậu quả trong production)

- Khách hàng bị văng mã `40P01`, Mọi thứ bị Rollback, Ứng Dụng chạy Rề Rề (latency tăng).
- Trẻ trâu Code Lại (retry) sai quy trình: Làm lại trên Cái Đống Rác của Giao Dịch cũ, Rác chất thành núi!
- Viết Bơm Dữ Liệu liên tục không dãn cách (không backoff) sinh ra Cơn Bão Làm Lại (retry storm) Bắn Chết Tài Khoản Đang Nóng.
- Kẻ đứng chờ Ngậm Hết Kết Nối (Connection), Làm Cạn Hồ Bơi Kết Nối trước khi Trái Tim (CPU) của DB kịp nhồi máu cơ tim.
- Gửi Email Báo Chuyển Tiền cho Đã, Xong Giây Cuối Bị Rollback -> Tiền Không Giảm Mà Khách Thì Vỗ Tay Mừng Rỡ.
- Đám Code Khắp Nơi Khóa Không Theo 1 Tôn Ti Trật Tự Nào -> Bắt Bug Kẹt Xe Khó Hơn Lên Trời!
- Đẩy Lên Nhiều Máy Chủ (Scale-out) Càng Tăng Quân Ăn Cướp; Xài `synchronized` Rẻ Tiền Trong Code App ĐÉO THỂ BẢO VỆ Nổi Dữ Liệu Thật Sự Dưới DB.

## 8. Đơn Thuốc Cứu Sinh (Hướng sửa khuyến nghị)

1. Cứ thấy Hai cái ID, là nhắm mắt Sắp Xếp: `firstId = min(fromId, toId)` và `secondId = max(fromId, toId)`.
2. Mọi ngóc ngách Code, Bắt buộc Móc Khóa (`FOR UPDATE`) cái Nhỏ Trước, Lớn Sau.
3. Ôm ĐỦ 2 CHÌA KHÓA rồi, lúc đó muốn đổi lại cái nào là Đích, Cái nào là Nguồn, Thẩm định Cỡ Nào Thì Tùy Tức.
4. Rút Ngắn Vòng Đời Giao Dịch; CẤM GỌI API (Remote Service) Trong Lúc Đang Cầm Ổ Khóa DB.
5. Nhận diện Mã Lỗi Báo Tử `40P01`; Khóc Lóc Dọn Dẹp (Rollback), Xong Kéo Chăn Làm Lại Giao Dịch Khác Với Số Lần Thử Giới Hạn.
6. Canh Chỉnh Khóa Giờ Cẩn Thận (`lock_timeout`, `statement_timeout`) Cho Phù Hợp Độ Trễ Hệ Thống; Dùng Timeout ÉO PHẢI LÀ LẤP LIẾM Cho Việc Không Biết Xếp Hàng!

Xếp hàng Chuẩn (Canonical ordering) đập tan Cơn Kẹt Xe. Nhưng Vòng Đời Thử Lại (Bounded retry) vẫn vô cùng Quan Trọng Vì Code Thực Tế Nó Đẻ Ra Trăm Ngàn Bệnh Khác (Khóa Khóa Ngoại, Bảo Trì, Vòng Lặp 3 - 4 Bảng Đan Chéo).

## 9. Khu Vực Giới Hạn (Phạm vi)

Bài này Sếp Chỉ Xoáy Sâu vào Vụ Kẹt Xe Ở PostgreSQL, Vụ DB Phán Tử Hình (victim abort) Và Cách Code Nút Thử Lại (transaction retry).
Còn mấy trò Múa Phụ Vụ Phức Tạp Của Dân Kế Toán Ngân Hàng, Sổ Cái, Cầm Cố, Đấu Trừ thì đi đọc ở case `BANK-003`. Chơi Kẹt Xe Khóa Mềm Trong Máy Ảo Java thì đọc `JVM-007`. Còn Sợ Mức Lỗi `SERIALIZABLE` Và Bắn Ngoại Lệ SSI thì qua đọc `DB-009`.
