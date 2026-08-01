# Phân tích quá trình lấy việc, tính công bằng và phục hồi sự cố

## Trạng thái ban đầu

Giả sử chúng ta có các công việc từ J1 đến J4 đều đang ở trạng thái `READY`, cùng độ ưu tiên, và thời gian sẵn sàng (`available_at`) tăng dần. Có hai tiến trình là Tiến trình A và Tiến trình B đang liên tục kiểm tra hàng đợi trên hai kết nối hoặc transaction hoàn toàn độc lập với nhau.

## Timeline dùng `SELECT` thông thường tạo dữ liệu trùng lặp

Nếu chúng ta chỉ dùng lệnh `SELECT` thông thường, kịch bản sau sẽ xảy ra:

| Bước | Tiến trình A | Tiến trình B |
| ---: | --- | --- |
| 1 | đọc J1 | |
| 2 | | đọc J1 |
| 3 | gọi dịch vụ bên ngoài cho J1 | gọi cùng dịch vụ đó cho J1 |
| 4 | cập nhật trạng thái thành DONE | cập nhật trạng thái thành DONE |

Ở đây, chuỗi hành động Đọc → Xử lý → Ghi không phải là một khối nguyên tử. Dù cuối cùng dòng dữ liệu chỉ ghi trạng thái `DONE` một lần, nhưng thực tế tác vụ gọi dịch vụ bên ngoài đã bị thực thi trùng lặp.

## Timeline `FOR UPDATE` tạo lock convoy

Nếu chúng ta dùng `FOR UPDATE` để khóa dòng nhưng không dùng `SKIP LOCKED`:

| Bước | Tiến trình A | Tiến trình B |
| ---: | --- | --- |
| 1 | khóa J1 | |
| 2 | giữ J1 để xử lý | yêu cầu J1 và bị chặn lại |
| 3 | J2 và J3 vẫn đang READY | không thể bỏ qua để xử lý J2 |
| 4 | commit transaction | bây giờ mới lấy được J1 hoặc đánh giá lại |

Trường hợp này không gây ra deadlock (vì không có vòng lặp chờ nhau), nhưng một dòng dữ liệu đang xử lý chậm sẽ làm tất cả các tiến trình khác phải xếp hàng chờ đợi theo thứ tự. Hậu quả là làm cạn kiệt các kết nối trong pool.

## Timeline với `SKIP LOCKED`

Khi chúng ta thêm `SKIP LOCKED` vào truy vấn:

| Bước | Tiến trình A | Tiến trình B |
| ---: | --- | --- |
| 1 | khóa J1 | |
| 2 | | bỏ qua J1 đang bị khóa, tiến tới khóa J2 |
| 3 | cập nhật J1 thành PROCESSING | cập nhật J2 thành PROCESSING |
| 4 | commit việc lấy J1 | commit việc lấy J2 |
| 5 | xử lý J1 bên ngoài transaction | xử lý J2 bên ngoài transaction |

Hai tiến trình sẽ lấy được hai dòng dữ liệu riêng biệt. Lưu ý rằng khóa dòng chỉ tồn tại trong vòng đời của transaction lấy việc, hoàn toàn không kéo dài đến bước xử lý công việc bên ngoài.

> **Nói ngắn gọn:** `SKIP LOCKED` thay đổi tư duy từ "chờ đúng dòng đầu tiên" thành "bỏ qua và lấy một dòng đang rảnh rỗi". Bù lại, chúng ta phải chấp nhận việc hệ thống không còn đảm bảo thứ tự vào trước ra trước một cách tuyệt đối nữa.

## MVCC và hành vi của khóa

Với mức độ cô lập `READ COMMITTED`, câu lệnh lấy việc sẽ đọc dữ liệu tại thời điểm câu lệnh bắt đầu. Quá trình quét tìm ứng viên diễn ra như sau:

1. Lọc các dòng đã được commit thỏa mãn điều kiện `READY` và `available_at`.
2. Duyệt theo thứ tự `ORDER BY` để tìm dòng phù hợp.
3. Thử đặt khóa `FOR UPDATE` lên dòng đó.
4. Nếu dòng đang bị khóa bởi transaction khác, ngay lập tức bỏ qua.
5. Dừng lại khi đã lấy đủ số lượng theo `LIMIT`.
6. Thực thi `UPDATE ... RETURNING` để đổi quyền sở hữu ngay trong cùng một câu lệnh.

Tiến trình khác sẽ không nhìn thấy trạng thái `PROCESSING` đang cập nhật dở dang của tiến trình này, nhưng khi chạm vào khóa dòng, nó sẽ lướt qua luôn. Khi transaction commit, dòng dữ liệu đó sẽ không còn thỏa mãn điều kiện `READY` nữa.

Cần nhớ rằng `SKIP LOCKED` chỉ bỏ qua các khóa cấp độ dòng. Lệnh `SELECT FOR UPDATE` vẫn yêu cầu khóa cấp độ bảng là `ROW SHARE`; do đó, nếu có một thao tác cấu trúc dữ liệu đang chạy, truy vấn vẫn sẽ bị chặn. Hơn nữa, thời gian chờ lấy kết nối, I/O, thời gian thực thi truy vấn và commit vẫn tạo ra độ trễ.

## Vì sao lấy việc bằng CTE là nguyên tử?

Việc sử dụng Common Table Expression (CTE) giúp gom mọi thứ vào một khối lệnh duy nhất:

```text
Khóa các dòng ứng viên
→ Cập nhật chính các dòng vừa khóa
→ Trả về thẻ định danh và trạng thái
→ Transaction commit
```

Cấu trúc này không chừa ra khe hở nào để ứng dụng kịp lấy ID về rồi một transaction khác lại chen vào lấy mất trước khi có lệnh UPDATE.
Nếu transaction bị rollback hoặc tiến trình gặp sự cố trước khi commit, toàn bộ trạng thái, thẻ định danh, số lần thử đều được hoàn tác, và các khóa cũng được giải phóng sạch sẽ.

Mỗi dòng trả về cho ứng dụng chắc chắn đã được gắn thẻ định danh mới. Ứng dụng chỉ bắt đầu xử lý bên ngoài khi quá trình commit thành công trót lọt.

## Góc nhìn không nhất quán là một tính năng có chủ ý

Khi dùng `SKIP LOCKED`, một tiến trình sẽ không thấy các dòng `READY` đang bị khóa trong kết quả trả về. Vì thế:

- Không nên dùng kết quả này để tính toán tổng số lượng trong hàng đợi, để tính tiền hay báo cáo doanh nghiệp.
- Đừng vội kết luận "không còn công việc" nếu kết quả trả về là rỗng; có thể các công việc chỉ đang tạm thời bị khóa.
- Đừng áp đặt yêu cầu vào trước ra trước làm tiêu chuẩn bắt buộc.
- Cơ chế này cực kỳ tối ưu khi mục tiêu chính là phân phối các công việc đang khả dụng cho nhiều tiến trình cùng một lúc.

Để xây dựng dashboard báo cáo, bạn nên dùng các truy vấn đếm thông thường hoặc sử dụng bản sao database riêng biệt, không nên dựa vào kết quả của truy vấn `SKIP LOCKED`.

## Sự công bằng và tình trạng chết đói

Câu lệnh `ORDER BY priority DESC, available_at, job_id` thiết lập một thứ tự ưu tiên ổn định, nhưng không mang lại sự công bằng tuyệt đối. Công việc J1 có thể liên tục bị bỏ qua nếu:

- Một transaction khác đang giữ khóa quá lâu.
- J1 liên tục bị lỗi và được trả lại hàng đợi ngay lập tức với độ ưu tiên cao.
- Các tiến trình luôn lấp đầy lô công việc bằng những công việc mới có độ ưu tiên cao.
- Có sự bất đồng bộ giữa kế hoạch thực thi truy vấn và index.

Để kiểm soát rủi ro này, chúng ta cần:

- Giữ transaction lấy việc thật ngắn.
- Giới hạn kích thước lô công việc.
- Giãn cách thời gian thử lại bằng thời gian chờ (cập nhật lại `available_at`).
- Giới hạn số lần thử và đưa vào trạng thái `DEAD`.
- Tăng dần độ ưu tiên theo thời gian chờ hoặc dành riêng tiến trình cho các độ ưu tiên thấp.
- Giám sát độ tuổi của công việc chờ lâu nhất, chứ không chỉ đếm số lượng hàng đợi.
- Dùng một tiến trình quét để phát hiện và xử lý các công việc đã quá hạn thuê.

Tóm lại, sự công bằng là một chính sách bạn phải tự thiết kế và giám sát, không phải là thứ có sẵn miễn phí khi dùng `SKIP LOCKED`.

## Thời hạn thuê và chu kỳ sở hữu

Mỗi lần lấy việc thành công, chúng ta sinh ra một `claim_token` mới và cập nhật `lease_until`. Thẻ định danh này đại diện cho một chu kỳ sở hữu:

```text
J1/token-A hết thời hạn thuê
→ Tiến trình quét trả J1 về READY
→ Tiến trình B lấy J1, nhận token-B
→ Tiến trình A đột nhiên tỉnh lại và tiếp tục xử lý với token-A
→ Khi Tiến trình A gửi lệnh hoàn thành (kèm token-A), database trả về số dòng bị ảnh hưởng = 0 (thất bại)
```

Thẻ định danh ngăn chặn các tiến trình cũ ghi đè lên hàng đợi. Tuy nhiên, nó không thể ngăn hệ thống bên ngoài nhận lại lời gọi cũ; do đó, các dịch vụ bên ngoài vẫn phải tự xử lý tính lũy đẳng.

Thời hạn thuê phải đủ dài để xử lý công việc bình thường, nhưng không được quá dài để tránh việc phục hồi sau sự cố trở nên quá chậm. Đối với các công việc cần thời gian dài, bạn nên có cơ chế kiểm tra tín hiệu hoạt động để gia hạn thuê, và phải kiểm tra điều kiện thẻ định danh hiện tại, đồng thời áp dụng chính sách thời gian chạy tối đa.

## Ma trận sự cố

| Điểm gặp sự cố | Trạng thái database | Phục hồi |
| --- | --- | --- |
| Trước khi commit việc lấy công việc | Hoàn tác, công việc vẫn là READY | Tiến trình khác lấy việc |
| Sau khi commit việc lấy công việc, trước khi xử lý | Nằm ở PROCESSING cho đến hết thời hạn thuê | Tiến trình quét sẽ đưa lại vào hàng đợi |
| Đang xử lý bên ngoài | Kết quả xử lý có thể bị lỗi hoặc không rõ | Thử lại với cơ chế tính lũy đẳng |
| Xử lý xong nhưng chưa hoàn thành | Công việc sẽ được chạy lại | Dịch vụ bên ngoài sẽ tự lọc trùng dựa trên ID của công việc hoặc tác vụ |
| Tiến trình cũ gọi lệnh hoàn thành | Thẻ định danh không khớp, số dòng bị ảnh hưởng = `0` | Tiến trình bỏ qua kết quả, ghi log cảnh báo |
| Commit hoàn thành xong nhưng rớt mạng | Đã ghi nhận DONE an toàn | Hệ thống đọc lại trạng thái/thẻ định danh để đồng bộ |
| Tiến trình quét gặp sự cố | Các công việc hết hạn vẫn kẹt ở PROCESSING | Tiến trình quét khác sẽ xử lý hoặc chờ lần chạy tiếp theo |

## Ít nhất một lần, không phải chính xác một lần

Transaction database không bao trùm các dịch vụ bên ngoài. Nếu hệ thống gặp sự cố sau khi đã gọi dịch vụ nhưng chưa kịp ghi `DONE`, việc gọi lại là hoàn toàn hợp lệ. Các giải pháp xử lý bao gồm:

- Gửi kèm mã nhận diện lũy đẳng tương ứng với ID của công việc.
- Ghi dữ liệu vào hàng đợi cục bộ trước rồi mới giao tiếp với bên ngoài.
- Dịch vụ đích có hỗ trợ thao tác có điều kiện.
- Đối soát định kỳ để phát hiện độ lệch giữa hai bên.

Đừng bao giờ cố gắng gọi dịch vụ bên ngoài ngay bên trong transaction database với hy vọng đạt được "chính xác một lần". Cách làm này chỉ khiến khóa database kéo dài và vẫn không thể đảm bảo an toàn như transaction phân tán.

## Hoàn thành, thất bại và thử lại

Cả hai thao tác báo hoàn thành hoặc báo lỗi đều phải kèm theo thẻ định danh:

```sql
where job_id = :jobId
  and status = 'PROCESSING'
  and claim_token = :claimToken
```

Xử lý các tình huống lỗi:

- Lỗi có thể thử lại: Chuyển trạng thái về READY, cộng thêm thời gian chờ `available_at = now() + backoff`, và xóa quyền sở hữu.
- Lỗi vĩnh viễn: Chuyển trạng thái sang DEAD, lưu lại thông tin lỗi đã che bảo mật.
- Mất quyền sở hữu: Khi cập nhật trả về `0` dòng, không được phép thay đổi gì thêm.

Cơ chế thời gian chờ sẽ giúp hàng đợi không bị tắc nghẽn bởi những công việc liên tục bị lỗi. Các bản ghi lỗi tuyệt đối không được chứa thông tin bảo mật hay dữ liệu nhạy cảm.

## Hết giờ, deadlock và cô lập

Truy vấn lấy việc thường dùng mức độ cô lập `READ COMMITTED`. Dù `SKIP LOCKED` giúp giảm bớt thời gian chờ khóa dòng, nó không thể loại trừ hoàn toàn các vấn đề deadlock hoặc chờ khóa bảng. Luôn thiết lập giới hạn thời gian thực thi `statement_timeout`; còn `lock_timeout` thì vẫn rất hữu ích cho các loại khóa khác.

Nếu thao tác lấy việc vô tình kích hoạt các bảng khác, hãy quy định rõ thứ tự khóa và chuẩn bị xử lý mã lỗi `40P01`. Mức độ cô lập `SERIALIZABLE` là không cần thiết cho mục đích lấy việc, và nó còn có thể làm tăng tỷ lệ hủy transaction.

## Sự cố và giải phóng transaction

Nếu kết nối bị đứt trước khi commit, PostgreSQL sẽ tự động rollback và giải phóng các khóa dòng. Nhưng sau khi đã commit, nếu tiến trình xử lý gặp sự cố, database sẽ không tự rollback quyền sở hữu vì dữ liệu đã được ghi nhận an toàn; lúc này cơ chế phục hồi qua thời hạn thuê là bắt buộc.

Luôn sử dụng thời gian của database (`clock_timestamp()` hoặc `CURRENT_TIMESTAMP` tùy theo nhu cầu) để tính toán, giúp tránh được độ trễ thời gian giữa các máy chủ. Bạn cần cân nhắc kỹ khi nào dùng thời gian của transaction và khi nào dùng thời gian thực trong cùng một câu lệnh.

## Đa máy chủ

Các khóa dòng của PostgreSQL tự động phối hợp mọi bản sao ứng dụng. Việc dùng mutex ở cấp độ thread là không cần thiết, và đôi khi còn làm giảm hiệu năng vì bắt một máy chủ phải chạy tuần tự không đáng có.

Khi mở rộng hệ thống, số lượng truy vấn lấy việc sẽ tăng lên. Bạn phải giới hạn số lượng tiến trình, kích thước lô công việc, khoảng thời gian giữa các lần thăm dò và tổng số lượng kết nối. Nếu lấy về lô rỗng, cần giãn cách thời gian chờ, tuyệt đối không tạo vòng lặp bận.

## Nguyên nhân gốc theo tầng

| Tầng | Vấn đề / Cơ chế |
| --- | --- |
| Ứng dụng | Lỗi phổ biến là Đọc/Xử/Ghi tuần tự, hoặc giữ transaction quá lâu khi gọi dịch vụ bên ngoài. |
| Spring | Proxy chịu trách nhiệm đảm bảo transaction lấy việc đã commit TRƯỚC KHI xử lý bên ngoài. |
| Hibernate/JDBC | Đảm bảo ánh xạ đúng dữ liệu trả về từ Native CTE, số dòng bị ảnh hưởng và thẻ định danh. |
| PostgreSQL | Chịu trách nhiệm quản lý MVCC, khóa dòng, cơ chế bỏ qua khóa và lệnh UPDATE nguyên tử. |
| Điểm đến bên ngoài | Quyết định tính an toàn thông qua tính năng kiểm tra tính lũy đẳng. |

## Khả năng quan sát

Cần theo dõi các chỉ số sau:

- Độ sâu của hàng đợi và tuổi của công việc chờ lâu nhất theo từng độ ưu tiên.
- Kích thước lô công việc, độ trễ và số lần truy vấn trả về rỗng.
- Thời gian xử lý công việc, số công việc hết hạn thuê, số lần lấy lại công việc và số công việc thất bại vĩnh viễn.
- Số lần thao tác hoàn thành bị từ chối do quá hạn (với số dòng bị ảnh hưởng = `0`).
- Phân bố số lần thử lại và số lần tái thực thi ở hệ thống bên ngoài.
- Số lượng kết nối đang hoạt động/chờ trong pool, số lượng câu lệnh bị hủy do hết giờ và mã lỗi deadlock `40P01`.
- Tra cứu `pg_stat_activity`, `pg_locks`, `pg_blocking_pids` khi truy vấn lấy việc bị treo bất thường.

Tuyệt đối không sử dụng ID của tiến trình hoặc ID của công việc làm nhãn giám sát vì sẽ gây bùng nổ dữ liệu; thay vào đó, hãy lưu chúng trong log hoặc trace có cấu trúc kèm theo cơ chế lấy mẫu.
