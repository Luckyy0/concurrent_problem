# LOCK-002 — Thử lại Có chừng mực (Bounded Retry) khi dùng Khóa Lạc quan

## Tóm tắt

Tưởng tượng có nhiều luồng (command) cùng lúc muốn cộng điểm thưởng (reward points) vào cùng một cái ví (wallet) đang được bảo vệ bằng `@Version` (Khóa Lạc Quan). 
Vì bản chất của việc cộng điểm là lấy "số dư hiện tại + điểm mới", nên nếu một luồng bị báo lỗi do có người khác chen ngang (version conflict), luồng đó hoàn toàn có thể thử lại (retry). 
Nhưng nếu bạn code vòng lặp thử lại vô tội vạ, chạy lại ngay lập tức không chớp mắt (immediate/unbounded retry), thì đám luồng bị rớt này sẽ lại lao vào, lại đụng độ, rồi lại văng ra... tạo ra một đợt "bão đụng độ" làm sập luôn database!

Luồng thử lại (Retry) đúng chuẩn phải đi qua các bước sau:

```text
Lần thử thứ N (Tx-N): Tải ví hiện tại + Lưu lịch sử lệnh
→ Tính toán cộng điểm + Ghi xuống DB (flush)
→ BÙM! Có thằng chen ngang (conflict): Hủy (rollback) sạch sẽ toàn bộ
→ Kiểm tra xem đã thử quá số lần chưa / quá thời gian chưa
→ Ra khỏi Giao dịch (transaction), đứng chờ một khoảng thời gian ngẫu nhiên (backoff có jitter)
→ Bắt đầu lần thử thứ N+1 (Tx-N+1): Tải lại ví từ đầu
```

> **Nói ngắn gọn:** `@Version` giúp dữ liệu không bị sai lệch; còn vòng lặp thử lại có giới hạn (bounded fresh retry) giúp hệ thống không bị nghẽn mạng vì bão đụng độ (retry storm).

## Các "Diễn viên" và Dữ liệu dùng chung (Actor và trạng thái)

| Thành phần | Trạng thái hiện tại |
| --- | --- |
| `reward_wallet` (Ví điểm) | Ví `77`, Điểm `100`, Phiên bản (version) `10` |
| Các Lệnh C1…Cn | Mỗi lệnh có ID duy nhất và số điểm dương cần cộng thêm |
| `reward_credit` (Lịch sử) | Bảng ghi nhận lịch sử cộng điểm chống trùng lặp theo ID Lệnh |
| App-1…App-N | Các máy chủ Spring Boot chạy song song |

Điểm đánh nhau sứt đầu mẻ trán chính là câu lệnh UPDATE kiểm tra phiên bản trên cùng 1 dòng Ví. Khi vòng lặp đang trong thời gian nghỉ ngơi (backoff), nó KHÔNG ĐƯỢC ngâm khóa, nhưng mỗi lần tỉnh dậy để thử lại thì nó lại tốn 1 kết nối (connection) và 1 đợt truy vấn.

## Các Luật Bất biến (Invariant)

```text
Điểm tổng cuối cùng = Điểm ban đầu + Tổng điểm của tất cả các Lệnh thành công hợp lệ (unique committed commands).

Mỗi mã lệnh (command ID) chỉ được phép tạo ra tối đa MỘT dòng lịch sử reward_credit.

Mỗi lần thử lại (retry) PHẢI MỞ RA MỘT Giao dịch hoàn toàn mới (persistence context mới) và bắt buộc phải Tải lại Ví từ DB lên.

Việc thử lại phải tự giác dừng khi hết số lần hoặc hết thời gian tối đa; Nếu cạn kiệt (exhaustion) thì không được báo thành công.
```

## Ranh giới Giao dịch (Transaction Boundaries)

Class chỉ huy `RewardCreditCoordinator` hoàn toàn KHÔNG CÓ `@Transactional`. Nó đứng ngoài để gọi vào Class lính đánh thuê `RewardCreditAttempt.creditOnce()` qua proxy của Spring. 

Hàm của thằng lính này được gắn cờ `REQUIRES_NEW` (Bắt buộc mở Giao dịch mới). Trong đó, nó sẽ kiểm tra xem lệnh này chạy chưa, tải Ví lên, cộng điểm, lưu lịch sử, ép xả rác xuống DB (flush) và cuối cùng là Chốt sổ (commit) hoặc Hủy bỏ (rollback).

Việc "Đứng chờ" (Backoff) nằm ở Class chỉ huy, sau khi hàm lính đánh thuê đã Hủy bỏ (rollback) xong xuôi, nhờ vậy hệ thống không bị ngâm kết nối Database một cách lãng phí vô ích. Người dùng gọi API chỉ nhận được kết quả cuối cùng sau khi Chốt sổ thành công.

## Khi nào thì Thử lại (Retry) mới An Toàn?

Cái lệnh kiểu `Cộng thêm 10 điểm` là tính toán dựa trên số dư mới nhất ở thời điểm hiện tại và không đổi mã `commandId`, nên cho dù có thử lại 100 lần thì vẫn an toàn. NHƯNG tuyệt đối không được áp dụng trò Thử lại cho những câu lệnh kiểu gán cứng `Gán giá bằng 80`, hoặc những lệnh có gọi API bên ngoài (remote side effect) không hỗ trợ chống trùng, hoặc những lỗi do vi phạm luật kinh doanh (ví dụ: cấm cộng điểm quá 1000).

Mỗi lần thử lại, bạn bắt buộc phải kiểm chứng lại từ đầu:

- Lệnh này thực ra đã chốt thành công chưa (lỡ lần trước chạy xong rớt mạng)?
- Ví này có còn hoạt động hay bị khóa rồi?
- Điểm cộng này còn hợp lệ theo policy không?
- Đã quá hạn thời gian hay bị người dùng bấm Hủy (cancel) chưa?

## Các Thuật ngữ dân trong nghề hay dùng

| Thuật ngữ | Ý nghĩa trong ngữ cảnh này |
| --- | --- |
| Load amplification (Khuếch đại tải) | 1 request gửi lên nhưng sinh ra hàng chục lần vòng lặp thử lại/truy vấn/ghi |
| Retry storm (Bão thử lại) | Quá nhiều luồng thất bại tự động thử lại CÙNG MỘT LÚC khiến database sụp đổ |
| Fresh attempt (Thử lại sạch sẽ) | Tạo mới hoàn toàn Giao dịch, lấy ảnh chụp mới và bộ nhớ đệm (persistence context) mới |
| Bounded retry (Thử lại có chừng mực) | Giới hạn số lần thử tối đa và tổng thời gian tối đa |
| Exponential backoff (Chờ theo hàm mũ) | Lần thử sau phải chờ lâu hơn lần thử trước |
| Jitter (Độ lệch ngẫu nhiên) | Cộng thêm thời gian chờ ngẫu nhiên để các luồng không ồ ạt thức dậy cùng 1 lúc |
| Starvation (Chết đói) | Một lệnh đen đủi liên tục bị thằng khác tranh mất suất cho đến khi cạn kiệt |
| Idempotency record (Lịch sử chống trùng) | Dòng dữ liệu lưu lại để đảm bảo 1 mã lệnh không cộng điểm 2 lần |
| Exhaustion (Kiệt sức) | Hết số lần thử hoặc hết giờ mà lệnh vẫn chưa được chốt |

## Sơ đồ Bản đồ (Điều hướng)

- [Code viết sai và Thảm họa khuếch đại tải (load amplification)](broken-code.md)
- [Phân tích Dòng thời gian, rollback, tính công bằng và sập nguồn (crash)](analysis.md)
- [Mã nguồn Chỉ huy/Lính đánh thuê, luật và thời gian chờ (backoff)](solutions.md)
- [Thực nghiệm bằng PostgreSQL Testcontainers](experiments.md)
- [Khái niệm: Khóa Lạc quan (Optimistic locking) và đụng độ phiên bản](../../concepts/optimistic-locking.md)
- [Khái niệm: Ranh giới giao dịch trong Spring](../../concepts/spring-transaction-boundaries.md)
- [Khái niệm: Kiểm thử đồng thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## Hậu quả nếu code ẩu đưa lên Production

- CPU, số lượng query, và lượt flush DB tăng vọt gấp nhiều lần số lượng Request đẩy vào;
- Hồ bơi kết nối (connection pool) cạn kiệt vì bị đám luồng thử lại vây hãm;
- Những ví nào hot sẽ gặp tình trạng giật lag kéo dài (tail latency) hoặc tỷ lệ lỗi "Kiệt sức" cao chót vót;
- Đặt vòng lặp Thử lại NGAY TRONG GIAO DỊCH khiến bộ nhớ đệm bị đóng dấu `rollback-only`, thử bao nhiêu lần cũng phế;
- Lỗi ngu ngốc: Mỗi lần thử lại tự tạo luôn mã Lệnh (command ID) mới, dẫn đến 1 giao dịch thành công nhiều lần;
- Vòng lặp chờ đều răm rắp (đồng nhịp) khiến một vài lệnh bị "Chết đói";
- Giao dịch xong, rớt mạng mất Response, client gửi lại mà không có chống trùng (replay by command ID) thì không biết đường nào mà lần.

## Hướng sửa chữa Khuyến nghị

1. CHỈ được phép Thử lại với lỗi đụng độ Khóa Lạc quan (optimistic conflict).
2. TÁCH BIỆT Class Chỉ huy (KHÔNG dùng `@Transactional`) và Class Lính đánh thuê thực thi 1 lần (Có `@Transactional`).
3. MỖI lần thử lại bắt buộc phải Tải lại đối tượng Ví và Lệnh từ DB.
4. GIỮ NGUYÊN mã Lệnh (command ID) và lợi dụng Ràng buộc Duy nhất (unique constraint) của DB để chống trùng.
5. SỬ DỤNG luật: Giới hạn số lần thử, Giới hạn tổng thời gian, Đứng chờ theo hàm mũ và CÓ TRỘN ĐỘ LỆCH NGẪU NHIÊN (jitter).
6. PHẢI LOG lại số lần thử, báo cáo "Thành công sau N lần thử", báo "Kiệt sức", và tỷ lệ đụng độ để giám sát.
7. Khi tranh chấp quá căng thẳng kéo dài, HÃY ĐỔI CHIẾN THUẬT (ví dụ dùng Hàng đợi) chứ đừng có mù quáng tăng số lần thử lại lên.

## Phạm vi bài học

Case này áp dụng cực ngon cho tình huống tỷ lệ tranh chấp Thấp/Vừa (low/moderate contention) và dành cho các Lệnh an toàn khi làm lại (retry-safe). Nếu bạn muốn xem cách bắt lỗi cơ bản, hãy xem bài `LOCK-001`. Nếu hệ thống tranh chấp cực kỳ khủng khiếp (high contention), hãy xem chiến thuật ở bài `LOCK-005`. Chi tiết về thứ tự xếp lớp của Spring Advisor, xem ở `SPR-006`.
