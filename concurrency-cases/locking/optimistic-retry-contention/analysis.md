# Phân tích Bão Thử lại (Retry Amplification) và Tiến độ Hệ thống

## Trạng thái ban đầu giả định

Chiếc Ví (Wallet) mang mã `77`: Đang có `100` điểm, phiên bản (version) `10`.
Cùng lúc đó, có 8 lệnh (command) khác nhau đồng loạt muốn cộng `10` điểm vào ví này.

## Dòng thời gian đan xen (Interleaving)

| Giai đoạn | Kẻ thắng (1 luồng) | Kẻ thua (7 luồng) |
| --- | --- | --- |
| Tải dữ liệu | Đọc lên `100` điểm, bản `v10` | Cũng đọc y xì `100` điểm, bản `v10` |
| Bơm xuống DB (Flush) | Thành công sửa `1` dòng, lên bản `v11` | Thất bại vì đụng `v11` của kẻ thắng (sửa `0` dòng) |
| Thử lại ngay lập tức | Đã chốt sổ xong xuôi | Đồng loạt tải lại bản `v11` |
| Bơm xuống DB lần 2 | 1 kẻ thua cũ vươn lên thắng (lên `v12`) | 6 kẻ thua còn lại tiếp tục đập đầu vào tường |

Bạn thấy đấy, nếu tất cả kẻ thua cứ húc đầu vào thử lại cùng 1 nhịp, số lần hệ thống phải phục vụ (attempts) sẽ lớn hơn rất nhiều so với số lượng lệnh ban đầu. 
Dữ liệu thì vẫn đúng nhờ có rào chắn `version`, nhưng tốc độ hệ thống sẽ chậm như rùa bò và thời gian chờ (tail latency) dài thê thảm.

> **Nói ngắn gọn:** Khóa Lạc quan (Optimistic locking) bảo vệ dữ liệu cực tốt, nhưng nó làm hệ thống ì ạch nếu đụng độ cao. Việc bạn thiết lập "luật thử lại" chính là bản hợp đồng sống còn của hệ thống.

## An toàn và Khả năng sống sót (Safety và liveness)

- **An toàn (Safety):** Tuyệt đối không mất điểm, không cộng điểm 2 lần; số phiên bản (version) tăng đúng với số lần cộng.
- **Khả năng sống sót (Liveness):** Một lệnh đi vào thì phải có kết thúc: hoặc thành công, hoặc bị trùng mã (replay), hoặc báo "kiệt sức" (exhaustion) trong khoảng thời gian cho phép.
- **Tính công bằng (Fairness):** Đời không như mơ, không có gì đảm bảo ai đến trước sẽ thắng trước; một luồng đen đủi có thể liên tục bị đứa khác nẫng tay trên cho tới khi chết đói (starve).

Thành công cuối cùng chưa nói lên được điều gì; Khi viết test hay đo lường, bạn bắt buộc phải đo số lần đập đầu (attempts) và tỷ lệ kiệt sức (exhaustion).

## Vì sao BẮT BUỘC phải mở Giao dịch (Transaction) mới?

Ngay khi gặp lỗi đụng độ Khóa Lạc quan:

- Giao dịch cũ đã bị đóng mộc tử hình (rollback-only);
- Bộ đệm (persistence context) trên RAM đang chứa dữ liệu bị ôi thiu;
- Cái dòng lịch sử (command record) mà bạn vừa `INSERT` thử cũng đã bay màu theo;
- Lần thử lại (retry) bắt buộc phải có một "bức ảnh" (snapshot) mới tinh để thấy được kết quả của người vừa thắng.

Đó là lý do Class Chỉ huy (Coordinator) phải đứng ĐỢI BÊN NGOÀI để bắt lỗi (catch) sau khi proxy đã hủy giao dịch xong xuôi. Việc "ngủ một chút" (backoff) diễn ra ở ngoài, không chiếm dụng Database. Khi Giao dịch mới được mở, nó sẽ kiểm tra chống trùng trước, rồi mới tải Ví lên để đánh giá lại từ đầu.

## Mô hình Khuếch đại Tải (Amplification model) định tính

Cứ tưởng tượng bạn có `R` yêu cầu (requests) và cho phép mỗi cái được thử tối đa `A` lần. Khối lượng công việc database phải gánh sẽ xấp xỉ `R × A`, chưa kể đống lệnh `SELECT` và `FLUSH`.
Công thức này không dùng để đo lường chính xác, nhưng nó cảnh tỉnh bạn: **Cứ vung tay tăng số lần thử lại tối đa (attempt cap) lên, bạn đang trực tiếp nhân lên thảm họa cho Database.**

Nếu bạn code cho chúng nó thử lại ngay lập tức (immediate retry), chúng sẽ lại lao vào đâm nhau ở cùng một thời điểm. Việc cho chúng "ngủ theo hàm mũ" (exponential backoff) giúp tách bọn chúng ra xa nhau dần; còn việc "cộng thêm độ lệch ngẫu nhiên" (jitter) đảm bảo chúng không bao giờ báo thức dậy cùng 1 lúc. Nhớ rằng: Đi ngủ (Backoff) không cứu được hệ thống nếu số lượng đơn hàng đổ vào liên tục lớn hơn tốc độ xử lý của server.

## Giới hạn số lần thử và Thời gian tối đa (Deadline)

Phải dùng kết hợp cả hai bảo bối này:

- **Giới hạn số lần (attempt cap):** Chặn 1 lệnh ngớ ngẩn nào đó tạo ra vòng lặp vô hạn.
- **Thời gian tối đa (overall deadline):** Chặn những vụ thử lại bị kéo dài lê thê do mạng chậm hoặc chờ ngủ (backoff) quá lâu, làm lố ngân sách thời gian (latency budget) của API.
- **Nút Hủy (cancellation/interrupt):** Dừng ngay mọi thứ nếu người dùng hoặc hệ thống bên trên đã cắt kết nối (không thèm chờ nữa).
- Phải chừa lại chút thời gian trong deadline để dọn dẹp rác.

Luật chơi là: Kiểm tra deadline trước mỗi lần thử lại, và kiểm tra lại lần nữa trước khi quyết định đi ngủ (backoff). Thời gian ngủ không bao giờ được phép dài hơn thời gian còn lại của Deadline.

## Chống trùng lặp (Idempotency)

Khóa chính `command_id` của bảng `reward_credit` được cắm cùng một Giao dịch (transaction) với lệnh UPDATE ví.

- Nếu đụng độ bị hủy (rollback): Cả điểm được cộng và dòng lịch sử lưu vết ĐỀU BIẾN MẤT CÙNG NHAU;
- Nếu khách bấm đúp gửi lại cùng mã ID (retry same ID): Điểm chỉ được cộng đúng 1 lần;
- Giao dịch xong, xui xẻo rớt mạng làm mất Response: Lần gọi tiếp theo sẽ trả về đúng kết quả cũ (replay record);
- Gửi 2 mã ID giống nhau cùng 1 tích tắc: Database sẽ tát 1 thằng văng ra bằng lỗi Trùng Khóa (unique conflict), thằng đó thử lại sẽ đọc được kết quả của thằng đi trước.

Tuy nhiên, bảng chống trùng này KHÔNG ngăn được 2 Lệnh KHÁC NHAU cùng nhào vô giành giật 1 cái Ví.

## Xác nhận lại Luật Kinh Doanh (Business revalidation)

Một lần "thử lại sạch sẽ" (fresh attempt) không chỉ đơn giản là tải lại cái Version mới. Nó còn phải xét nét: Ví này có bị khóa chưa? Lệnh này nãy giờ có ai làm giùm chưa? Điểm cộng này còn hợp lệ không? 
Ví dụ: Thằng đi trước đã cộng điểm tới mức trần tối đa (cap reached). Lúc này, luồng đi sau khi tỉnh dậy phải ngoan ngoãn báo lỗi "Quá hạn mức" (domain rejection) chứ không được nhắm mắt thử lại tiếp!

## Phân loại Mã Lỗi (Exception classification)

Chỉ cho phép thử lại với duy nhất mã lỗi `ObjectOptimisticLockingFailureException` (hoặc lỗi đụng độ tương đương).
**TUYỆT ĐỐI KHÔNG thử lại** với các trường hợp:

- Dữ liệu bị đình chỉ hoặc không hợp lệ;
- Lỗi trùng Khóa (unique) mà nguyên nhân không phải do chính mình (same command);
- Lỗi đứt mạng/timeout nếu luật chưa cho phép;
- Yêu cầu đã bị Hủy (cancellation);
- Lỗi cú pháp code, lỗi map object ngu ngốc;
- Lỗi gọi API bên thứ 3 mà không biết bên kia đã nhận được hay chưa (ambiguous).

Nhớ là Exception thường chỉ lòi đuôi ra khi gọi hàm `flush` hoặc `commit`, nên khối `catch` phải bao trọn gói vòng đời của cái proxy đó.

## Đứng chờ (Backoff) và Giải phóng Tài nguyên (Resource lifetime)

Khi Giao dịch thất bại bị Hủy (rollback), PostgreSQL sẽ nhả ngay lập tức cái Khóa dòng (row lock) và hệ thống trả Connection về hồ bơi (pool) TRƯỚC KHI đi ngủ. 
Lúc ngủ, chúng ta chỉ giữ lại luồng (thread) của ứng dụng hoặc lịch hẹn giờ (scheduled continuation), tuyệt đối KHÔNG GIỮ Giao dịch (Transaction). Đừng tưởng xài Luồng Ảo (Virtual threads) của Java 21 thì có quyền ngâm Database, database có giới hạn đấy!

## Sập nguồn (Crash) và Chốt sổ mập mờ (Ambiguous commit)

Nếu sập nguồn trước lúc chốt sổ (commit), toàn bộ lần thử đó biến mất như chưa từng tồn tại. 
Nếu sập nguồn ngay SAU KHI chốt sổ mà chưa kịp gửi Response cho khách, lệnh này rơi vào trạng thái "Mập mờ" (ambiguous). Khi khách hàng tự gọi lại lệnh đó với cùng `command_id`, hệ thống phải móc bảng `reward_credit` ra để trả về (replay). Bạn tuyệt đối KHÔNG ĐƯỢC dùng số đếm lần thử (Attempt number) làm định danh chống trùng lặp.

Việc gửi tin nhắn ra hệ thống khác (External events) phải xài kỹ thuật Hộp thư (outbox) nằm bên trong giao dịch Thành công. Lệnh thử lại tuyệt đối KHÔNG ĐƯỢC gọi API gọi ra ngoài (remote service) trước khi có được cái `commit`.

## Môi trường Đa Máy chủ (Multi-instance)

Chỉ có Cơ sở dữ liệu PostgreSQL (nhờ Version và Unique constraints) mới đủ sức đứng ra làm trọng tài phân xử cho n máy chủ. 
Độ lệch ngẫu nhiên (Backoff jitter) phải tự động sinh ra độc lập trên từng máy; Dùng mấy cái Semaphore nội bộ trên RAM chỉ giúp hạn chế tắc đường ở máy đó chứ không bảo vệ được sự chính xác (correctness) của toàn hệ thống.

Càng chạy nhiều máy (Scale-out), số lượng thằng giành giật cái Ví hot càng đông. Nếu đụng độ và kiệt sức diễn ra liên tục, bắt buộc bạn phải đổi chiến thuật (ví dụ dùng hàng đợi queue, chia nhỏ ví partition ownership, hoặc quay về khóa bi quan như ở bài `LOCK-005`).

## Truy tìm Nguồn gốc theo Từng Lớp (Root cause theo layer)

| Lớp (Layer) | Vai trò làm gì? |
| --- | --- |
| Code của bạn (Application) | Cài đặt số lần thử, chống trùng (idempotency), hẹn giờ (deadline) và xét lại luật kinh doanh |
| Spring Boot | Dùng Proxy/Advisor để tạo ra hoặc bẻ gãy ranh giới của một Giao dịch sạch sẽ (fresh boundary) |
| Hibernate | Bắn cờ phiên bản (Versioned flush) và khóa mõm giao dịch thành `rollback-only` khi có biến |
| PostgreSQL | Giữ khóa dòng (Row lock), đếm số dòng sửa được và ném lỗi đụng độ phiên bản/khóa duy nhất |

## Các Kết quả có thể xảy ra (Failure outcomes)

| Kết quả cuối cùng (Outcome) | Hành vi của hệ thống |
| --- | --- |
| Ăn ngay từ lần đầu (First-attempt win) | Chốt sổ cái rụp, khỏi cần chờ (no backoff) |
| Bị đụng độ rồi mới thắng | Hủy bỏ, ra ngoài đứng chờ, vào tạo Giao dịch mới rồi Chốt sổ |
| Bấm gửi trùng 2 lần | Bị chặn lại, lôi lịch sử cũ ra trả về cho khách (Replay) |
| Vi phạm luật (Business rejection) | Chốt sổ hoặc văng lỗi theo đúng luật, DỪNG THỬ LẠI |
| Hết giờ / Hết số lần (Exhausted) | Dừng lại, báo lỗi "Hệ thống đang bận/Tranh chấp quá tải" |
| Bị ngắt ngang (Interrupted) | Truyền tiếp lệnh hủy báo lên trên, dọn dẹp và nghỉ chơi |

## Lệnh Chết đói (Starvation) và Bão thử lại (Retry storm)

Độ lệch ngẫu nhiên (Jitter) giúp bọn nó không dẫm đạp lên nhau, nhưng KHÔNG đảm bảo tính công bằng (fairness). Hãy theo dõi tuổi thọ của Lệnh (command age), số lượng lần thử bị phân tán và hiện tượng lặp lại lỗi "Kiệt sức". 
Khi có một cái Ví quá hot ngày này qua tháng nọ, bạn phải nghĩ đến chuyện giới hạn đầu vào (Admission control), tạo hàng đợi riêng cho nó (per-key queue) hoặc đổi chiến thuật ngay lập tức.

## Giám sát (Observability)

Những chỉ số máu chốt cần đo:

- Số lượng Lệnh thực tế (commands) SO VỚI Tổng số lần húc đầu (attempts);
- Tần suất đụng độ Khóa Lạc quan (optimistic conflicts);
- Tỷ lệ ăn điểm ở lần thử thứ mấy;
- Tỷ lệ Kiệt sức (exhaustion) / Quá hạn (deadline) / Bị hủy ngang (cancellation);
- Thời gian ngủ chờ (backoff duration);
- Số lượng khách hối hả bấm đúp (duplicate replay);
- Thời gian chạy giao dịch / câu lệnh và số luồng bị xếp hàng chờ trong hồ bơi kết nối;
- Dùng công cụ Trace mẫu (sampled trace) để bắt quả tang cái Ví nào đang bị nghẽn, tuyệt đối không quăng thẳng ID ví vào bộ đếm metric (tránh sập bộ nhớ monitor).

Đừng bị lừa: Tỷ lệ báo Thành Công vẫn có thể rất cao 99%, nhưng hệ thống của bạn thực chất đang rên rỉ gánh chịu thảm họa Khuếch đại tải (amplification) vì dưới gầm bàn nó phải thử lại liên tục!
