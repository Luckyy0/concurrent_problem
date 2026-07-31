# Ranh giới Giao dịch (Transaction) trong Spring

## Mục tiêu

Tài liệu này giải thích cách Spring Framework kẻ ra "biên giới" cho một giao dịch (transaction) thông qua cơ chế người đóng thế (proxy), cách lan truyền (propagation), cách đẩy dữ liệu (flush/commit) và cách nó gắn liền với từng luồng (thread). Hiểu rõ nền tảng này sẽ giúp bạn không lặp lại các lỗi kinh điển khi dùng `@Transactional`.

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Người đóng thế (`transactional proxy`) | Một đối tượng trung gian đứng ra chặn đường (intercept) trước khi gọi hàm thật để lo việc Mở và Đóng giao dịch. |
| Tự gọi hàm (`self-invocation`) | Lỗi kinh điển: Từ bên trong một hàm, bạn gọi thẳng một hàm khác của chính class đó (`this.method()`). Lúc này "người đóng thế" bị bỏ qua. |
| Ranh giới giao dịch (`transaction boundary`) | Phạm vi làm việc chung: Cùng thành công (commit) hoặc cùng thất bại (rollback). |
| Lan truyền (`propagation`) | Luật phân xử xem một hàm nên "ké" vào giao dịch đang có, hay tạo mới một giao dịch riêng. |
| Đẩy dữ liệu xuống DB (`flush`) | Đẩy các lệnh SQL thay đổi từ bộ nhớ xuống Database. Lưu ý: Nó CHƯA phải là chốt sổ (commit)! |
| Chốt sổ (`commit`) | Dấu chấm hết giao dịch, làm cho mọi thay đổi thực sự có hiệu lực và người khác nhìn thấy được. |
| Bị đánh dấu hỏng (`rollback-only`) | Giao dịch đã bị lỗi ở đâu đó và bị hệ thống ghim lại: "Cấm không cho commit nữa". |
| Bám dính vào luồng (`thread-bound context`) | Giao dịch và bộ đệm của Spring luôn luôn gắn liền với cái luồng (thread) đang chạy hiện tại. |

## Cơ chế chặn đường của Người đóng thế (Proxy interception)

Theo mặc định, từ khóa `@Transactional` chỉ linh nghiệm khi có người từ bên ngoài gọi vào hàm của bạn thông qua "người đóng thế" (Spring-managed proxy).
Nếu bạn viết lệnh `this.method()`, bạn đang gọi thẳng code Java nội bộ. Trình chặn đường (interceptor) sẽ hoàn toàn "mù" và không thể mở giao dịch giúp bạn. Ngoài ra, việc dùng đúng quyền truy cập hàm (public) hay khai báo Bean chuẩn cũng là bắt buộc.

> **Nói ngắn gọn:** Chữ `@Transactional` viết ra chỉ để "làm kiểng" trang trí. Cái proxy đóng thế đứng bên ngoài chộp lấy lời gọi hàm mới chính là cỗ máy chạy giao dịch thực sự.

## Lan truyền (Propagation) và Hoàn tác (Rollback)

- Lệnh `REQUIRED`: (Mặc định) Sẽ xin đi ké vào giao dịch đang có, nếu chưa có thì tạo mới.
- Lệnh `REQUIRES_NEW`: Sẽ tạm dừng giao dịch hiện tại, mở một giao dịch hoàn toàn mới và hoạt động độc lập (thắng/thua tự chịu).
Hãy chọn luật lan truyền dựa trên phạm vi nghiệp vụ bạn muốn gom chung lại. Đừng dùng nó như một mẹo để "chữa cháy" che giấu các lỗi mơ hồ.

**Quy tắc báo lỗi (Rollback):**
Mặc định Spring sẽ chỉ hoàn tác (rollback) khi gặp lỗi hệ thống (unchecked exception như `RuntimeException`). Đối với lỗi nghiệp vụ (checked exception), bạn BẮT BUỘC phải khai báo rõ ràng bằng `rollbackFor = Exception.class` nếu muốn nó hoàn tác.
Cực kỳ cẩn thận: Nếu bạn dùng `try-catch` "nuốt" mất cái exception ở bên trong hàm, Spring sẽ tưởng là không có lỗi và vẫn cố đi Commit, hoặc sẽ văng lỗi muộn vì giao dịch đã bị đánh dấu là `rollback-only`.

## Đẩy dữ liệu (Flush), Chốt (Commit) và Tầm nhìn (Visibility)

Hibernate có khả năng tự động đẩy SQL (flush) trước mỗi khi bạn chạy query hoặc lúc commit. Lệnh Flush mới chỉ gửi SQL tới database chứ giao dịch CHƯA hề được chốt (commit); nghĩa là bạn vẫn đang giữ khóa, và người khác vẫn chưa nhìn thấy dữ liệu theo luật của Database.

Đừng bao giờ dùng hàm `saveAndFlush` của Spring Data JPA để thay thế cho ranh giới giao dịch (transaction boundary). 
Nếu tầng Service của bạn quên mất chữ `@Transactional`, bản thân hàm repository proxy của Spring sẽ tự động chia nhỏ thành các giao dịch lắt nhắt cho từng câu lệnh. Hậu quả là tiến trình nghiệp vụ của bạn lưu được một nửa (partial state) rồi văng lỗi, và bên ngoài đã nhìn thấy cái trạng thái dở dang đó!

## Luồng (Thread) và các tác vụ bất đồng bộ (Async boundary)

Hãy nhớ nằm lòng: Giao dịch LUÔN LUÔN bám dính vào luồng hiện tại (thread-bound).
Các từ khóa như `@Async`, các luồng executor task, hay `CompletableFuture` đều bứng code của bạn sang chạy ở một luồng (thread) hoàn toàn mới. Và thế là nó **không thể** ké vào giao dịch của luồng cũ được nữa.
Tuyệt đối không được truyền cái kết nối Database (EntityManager) hoặc các thực thể chưa tải hết dữ liệu (entity lazy state) từ luồng này sang luồng khác rồi hy vọng nó vẫn chung một giao dịch!

## Lời khuyên khi Kiểm thử (Testing)

Khi viết test tích hợp (Integration test), bạn phải gọi bean thông qua Proxy của Spring, phải dùng các giao dịch riêng biệt để đọc/ghi, và kiểm tra các dữ liệu nghiệp vụ sau khi đã lưu xong (committed).
Cảnh báo: Nếu bạn hồn nhiên gắn chữ `@Transactional` lên ngay trên cái test method, toàn bộ quá trình chạy test đó sẽ bị nhốt chung vào 1 giao dịch. Khi đó, test của bạn sẽ vượt qua dễ dàng và **che giấu** luôn cả những lỗi tự gọi hàm (self-invocation) hay lỗi quên commit ở code thật.

## Liên kết tài liệu tham khảo

- [SPR-001 — Transactional self-invocation](../spring/transactional-self-invocation/README.md)
- [SPR-005 — Checked exception rollback](../spring/checked-exception-rollback/README.md)
- [SPR-006 — Retry transaction boundary](../spring/retry-transaction-boundary/README.md)
- [LOCK-002 — Bounded optimistic retry](../locking/optimistic-retry-contention/README.md)
- [SPR-007 — Long transaction pool exhaustion](../spring/connection-pool-long-transaction/README.md)
- [Kiểm thử đồng thời](concurrency-testing.md)
