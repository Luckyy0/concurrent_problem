# Mức độ cách ly (Isolation level) và Khả năng nhìn thấy dữ liệu

## Mục tiêu

Tài liệu này sẽ giải thích các từ vựng chung khi nói về "Mức độ cách ly" (Isolation Level) của giao dịch (Transaction) trong Spring Boot và PostgreSQL. Khi làm việc với nhiều luồng chạy cùng lúc, làm sao để luồng này không nhìn thấy dữ liệu đang chạy dở dang của luồng kia? Đó chính là câu chuyện của Isolation.

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Mức độ cách ly (`isolation level`) | Luật lệ quy định xem một giao dịch này được phép "nhìn trộm" bao nhiêu phần trăm dữ liệu đang thay đổi của giao dịch khác. |
| Đọc dữ liệu rác (`dirty read`) | Lỗi xảy ra khi bạn đọc nhầm dữ liệu mà người khác đang sửa dở dang (chưa lưu/commit). |
| Đọc không nhất quán (`non-repeatable read`) | Lỗi xảy ra khi trong cùng một giao dịch, bạn SELECT một dòng 2 lần, nhưng lần thứ hai dữ liệu lại biến thành số khác (do có ai đó vừa sửa và commit xen vào giữa). |
| Bóng ma (`phantom read`) | Tương tự như lỗi trên, nhưng thay vì sai một dòng, lần này là bạn SELECT ra cả một danh sách, tự nhiên lần hai danh sách đó mọc thêm (hoặc mất đi) vài dòng (do người khác vừa INSERT/DELETE). |
| Lệch ghi (`write skew`) | Lỗi rất khó chịu: Hai luồng cùng đọc chung một điều kiện, rồi mỗi luồng đi cập nhật một dòng khác nhau, dẫn đến phá hỏng quy tắc chung ban đầu. |
| Ảnh chụp (`snapshot`) | Giống như chụp màn hình. Khi giao dịch bắt đầu, nó sẽ "chụp" lại trạng thái Database lúc đó và chỉ làm việc trên bức ảnh đó. |
| Lỗi tuần tự hóa (`serialization failure`) | Lỗi khi Database thấy các giao dịch dẫm chân lên nhau quá nguy hiểm, nó quyết định giết (abort) một giao dịch để đảm bảo an toàn. |
| Cách ly thực tế (`effective isolation`) | Mức độ cách ly thực sự đang được áp dụng khi code của bạn chạy dưới Database (có thể khác với những gì bạn nghĩ). |

## Các cấp độ cách ly trong PostgreSQL

PostgreSQL hỗ trợ các cấp độ sau (từ lỏng lẻo đến chặt chẽ nhất):

- `READ UNCOMMITTED` (Đọc dữ liệu chưa lưu): PostgreSQL thực chất không hỗ trợ mức này, nếu bạn chọn nó, nó sẽ tự động chạy hệt như `READ COMMITTED`.
- `READ COMMITTED` (Đọc dữ liệu đã lưu - **Mặc định**): Mỗi khi bạn chạy một lệnh `SELECT`, Database sẽ "chụp" một bức ảnh mới nhất. Do đó, nếu bạn gọi hai lệnh `SELECT` cách nhau vài giây trong cùng một giao dịch, bạn có thể nhận về kết quả khác nhau nếu có ai đó vừa commit xen vào giữa.
- `REPEATABLE READ` (Đọc lặp lại được): Giao dịch sẽ chỉ dùng đúng MỘT bức ảnh chụp từ lúc bắt đầu cho đến khi kết thúc. Đảm bảo đọc 100 lần kết quả vẫn y hệt nhau. Tuy nhiên, nó có thể báo lỗi nếu phát hiện ai đó vừa ghi đè lên dữ liệu bạn định sửa.
- `SERIALIZABLE` (Tuần tự hóa): Mức khắt khe nhất. Đảm bảo mọi giao dịch chạy song song cho ra kết quả y hệt như thể chúng chạy xếp hàng nối đuôi nhau. Rất an toàn, nhưng bù lại Database sẽ văng lỗi (serialization failure) liên tục và ép code bạn phải "thử lại" (retry).

> **Nói ngắn gọn:** Nâng mức cách ly lên cao KHÔNG có nghĩa là Database sẽ khóa (lock) tất cả mọi thứ lại. Nó chỉ che giấu dữ liệu tốt hơn, và đổi lại, thay vì bị sai lệch dữ liệu, code của bạn sẽ bị văng lỗi nhiều hơn và bắt buộc phải xử lý try-catch để chạy lại (retry).

## Sự khác biệt giữa Ảnh chụp (Snapshot), Khóa (Lock) và Lưu (Commit)

- **Ảnh chụp (Snapshot)** chỉ quyết định việc bạn *nhìn thấy* dữ liệu phiên bản nào.
- **Khóa (Row locks)** mới là thứ quyết định bạn có bị *đứng chờ* người khác sửa xong hay không. Dùng lệnh `SELECT FOR UPDATE` là tạo khóa chặn người khác, chứ không phải là nâng mức cách ly lên `REPEATABLE READ`.
- Khi bạn gọi hàm sửa dữ liệu (Flush/Save), dữ liệu mới chỉ đẩy tạm xuống Database chứ chưa được công bố. Chỉ khi giao dịch kết thúc (Commit), các giao dịch khác mới bắt đầu nhìn thấy sự thay đổi này (tùy theo luật cách ly của họ).

## Mức độ cách ly thực tế trong Spring Boot

Trong Spring Boot, Mức độ cách ly (Isolation) chỉ thực sự có tác dụng ở **giao dịch ngoài cùng** (khi kết nối database được tạo ra lần đầu). 
Nếu bạn gọi một hàm con có đánh dấu `@Transactional(propagation = REQUIRED)`, hàm con đó sẽ xài ké luôn kết nối của hàm cha, và nó cũng phải chịu chung mức cách ly của hàm cha (bất kể bạn có ghi `@Transactional(isolation = ...)` khác đi nữa). Nếu không chỉnh gì, Spring sẽ xài mức mặc định của Database (thường là `READ COMMITTED`).

**Mẹo xử lý:** 
- Bạn có thể bật cấu hình `validateExistingTransaction = true` để Spring báo lỗi rầm rĩ nếu hàm con dám khai báo mức cách ly khác với hàm cha, thay vì im lặng xài ké.
- Nếu dùng `REQUIRES_NEW`, Spring sẽ ngắt giao dịch hiện tại, mở một kết nối hoàn toàn mới với mức cách ly độc lập và tự commit riêng. Hãy cẩn thận vì nó tốn thêm kết nối (connection) tới Database.

## Cách chọn Mức độ cách ly phù hợp

Đừng bao giờ cứ nhắm mắt chọn mức cao nhất (`SERIALIZABLE`) cho an toàn! Việc đó làm hệ thống rất chậm và sinh ra vô số lỗi bắt buộc phải retry.
Hãy cân nhắc xem: Bạn cần đọc ghi những gì? Bạn sợ gặp lỗi nào (dirty read, phantom)? 
Thông thường, việc dùng các lệnh SQL cập nhật xí chỗ trực tiếp (conditional atomic update) hoặc các Ràng buộc (Constraint) tạo sẵn trong Database sẽ bảo vệ tính đúng đắn tốt hơn và chạy mượt hơn nhiều so với việc chỉ mù quáng nâng mức cách ly lên cao.

## Lời khuyên khi Kiểm thử (Testing)

Khi test các vấn đề này, hãy dùng Testcontainers với PostgreSQL thật. Chạy thử hai kết nối (connection) độc lập. Dùng các chốt chặn (latch) để ép luồng 1 commit ngay giữa hai lệnh SELECT của luồng 2 để xem chuyện gì xảy ra.
Bạn có thể dùng câu lệnh `SELECT current_setting('transaction_isolation');` để in ra xem mức cách ly hiện tại đang là bao nhiêu. 
Hãy luôn kiểm tra xem kết quả dữ liệu cuối cùng (business outcome) có bị sai lệch không, thay vì chỉ tin tưởng vào code. Đặc biệt chú ý, coi chừng giao dịch sinh ra tự động bởi `@DataJpaTest` của tầng ngoài vô tình che khuất giao dịch của code thực tế bên trong.

## Liên kết tài liệu tham khảo

- [SPR-004 — Isolation boundary mismatch](../spring/isolation-boundary-mismatch/README.md)
- [DB-001 — Lost update under MVCC](../postgresql/lost-update-mvcc/README.md)
- [DB-002 — Dirty-read expectations](../postgresql/dirty-read-expectation/README.md)
- [DB-003 — Non-repeatable read](../postgresql/non-repeatable-read/README.md)
- [DB-004 — Phantom capacity check](../postgresql/phantom-capacity-check/README.md)
- [DB-005 — Write skew](../postgresql/write-skew/README.md)
- [DB-009 — Serializable abort and safe retry](../postgresql/serializable-retry/README.md)
- [Spring transaction boundaries](spring-transaction-boundaries.md)
- [Concurrency testing](concurrency-testing.md)
