# Bài toán LOCK-001 — Bảo vệ dữ liệu cơ bản bằng Khóa Lạc Quan (`@Version`)

## 1. Tóm tắt vấn đề

Giả sử có hai nhân viên (A và B) cùng truy cập vào một sản phẩm `42` với mức giá hiện tại là `100` và phiên bản (version) là `7`. 
- Nhân viên A thay đổi mức giá thành `90` và lưu lại.
- Nhân viên B giữ màn hình một thời gian, sau đó thay đổi mức giá thành `80` và thực hiện lưu.

Nếu hệ thống thiếu cơ chế quản lý version, CẢ HAI lệnh UPDATE đều thông báo thực thi thành công 1 bản ghi (số dòng bị ảnh hưởng là `1`). Hệ quả là thao tác của B ghi đè kết quả của A mà A không hề được thông báo.

Khi bổ sung annotation `@Version` vào mã nguồn, Hibernate (hoặc các trình cung cấp JPA khác) sẽ tự động tạo ra câu lệnh SQL như sau:

```sql
update product_offer
set price = 90.00,
    version = 8      -- tự động tăng version
where offer_id = 42
  and version = 7;   -- BẮT BUỘC version trong database phải khớp với 7
```

Trong trường hợp này, khi A thực thi trước, câu lệnh sẽ cập nhật thành công 1 bản ghi. Khi đến lượt B thực thi, do B vẫn sử dụng version cũ là `7` làm điều kiện WHERE, nhưng dữ liệu trong database đã được cập nhật lên `8`, câu lệnh của B sẽ không tìm thấy bản ghi nào hợp lệ (cập nhật `0` bản ghi). 
Lập tức, hệ thống ORM sẽ ném ngoại lệ `OptimisticLockException` (được Spring chuyển đổi thành `ObjectOptimisticLockingFailureException`), và transaction của B sẽ bị rollback.

> **Ghi chú quan trọng:** Khóa lạc quan (`@Version`) KHÔNG NGĂN CẢN các luồng đọc dữ liệu đồng thời; mục đích của nó là phát hiện xung đột tại thời điểm ghi dữ liệu, thay vì cho phép ghi đè ngầm.

## 2. Các thực thể và trạng thái chia sẻ

| Thành phần | Trạng thái hiện tại |
| --- | --- |
| Sản phẩm `product_offer` | `offer_id=42`, Giá hiện hành: `100`, Version: `7` |
| Người dùng A | Yêu cầu cập nhật giá thành `90` |
| Người dùng B | Yêu cầu cập nhật giá thành `80` |
| Trạng thái cơ sở dữ liệu | Bản ghi vật lý trong hệ quản trị CSDL (ví dụ: PostgreSQL) |

Tình trạng tranh chấp tài nguyên xảy ra NGAY TẠI THỜI ĐIỂM câu lệnh UPDATE mang theo version được gửi tới database (khi thực thi hàm `flush` hoặc `commit`), KHÔNG PHẢI tại thời điểm thay đổi thuộc tính trên đối tượng Java hay gọi hàm `repository.save()`.

## 3. Các quy tắc bất biến

```text
Một thao tác cập nhật chỉ được xác nhận (commit) nếu trạng thái bản ghi trong database VẪN KHỚP VỚI PHIÊN BẢN (expected version) mà phía gọi đã tải.

Bất kỳ thao tác nào mang dữ liệu lỗi thời đều không được phép ghi nhận thành công nếu đã có một transaction khác commit trước đó.

Thuộc tính version BẮT BUỘC phải được quản lý và gia tăng tự động bởi framework (ví dụ: Hibernate); Lập trình viên không được phép gán hoặc sửa đổi giá trị này bằng tay.
```

Trong kịch bản này, giải pháp tiêu chuẩn là từ chối transaction, thông báo xung đột và yêu cầu người dùng nạp lại dữ liệu, THAY VÌ thử lại tự động thao tác gán giá trị tuyệt đối. Việc tự động thử lại thao tác ghi đè tuyệt đối (ví dụ: ghi đè giá `90` bằng `80`) sẽ phá vỡ tính nhất quán nghiệp vụ. (Để tìm hiểu kỹ thuật tự động thử lại an toàn đối với các phép toán tương đối, tham khảo bài `LOCK-002`).

## 4. Ranh giới transaction

Quy trình xử lý một yêu cầu cập nhật đi qua Spring proxy được mô tả như sau:

```text
BẮT ĐẦU TRANSACTION
1. Truy xuất đối tượng sản phẩm và version mới nhất từ database.
2. Xác minh đối chiếu với phiên bản kỳ vọng được phía gọi truyền lên.
3. Cập nhật các trường dữ liệu trên đối tượng entity.
4. Đồng bộ (flush) câu lệnh SQL UPDATE chứa mệnh đề điều kiện version.
5. Nếu cập nhật thành công (số dòng bị ảnh hưởng = 1) -> Xác nhận (commit). 
   Nếu cập nhật thất bại (số dòng bị ảnh hưởng = 0) -> Ném ngoại lệ khóa lạc quan và rollback.
```

Ngoại lệ có thể phát sinh tại bước 4 (trong quá trình `flush`) hoặc tại bước 5 (khi thực hiện `commit`). Tầng API chỉ được phép trả về trạng thái thành công sau khi quá trình commit của proxy transaction hoàn tất.

## 5. Ranh giới phía gọi

Việc khai báo `@Version` trên entity là ĐIỀU KIỆN CẦN nhưng CHƯA ĐỦ. Thiết kế API không đồng bộ hóa phiên bản hiển thị tại phía gọi sẽ dẫn đến rò rỉ dữ liệu:

```text
1. Màn hình của B tải dữ liệu tại version 7.
2. A hoàn tất cập nhật, version trong database tăng lên 8.
3. B yêu cầu lưu dữ liệu nhưng API KHÔNG ĐÍNH KÈM thông số version (expected version).
4. Máy chủ tải bản ghi version 8 mới nhất từ database, và áp dụng giá trị 80 của B lên đó.
5. Lệnh UPDATE được thực thi thành công với version 8. Kết quả của A bị ghi đè hoàn toàn.
```

Do đó, mọi lệnh sửa đổi dữ liệu BẮT BUỘC phải mang theo thông số phiên bản kỳ vọng (phiên bản mà người dùng đang thao tác). Tầng dịch vụ có nhiệm vụ đối chiếu phiên bản kỳ vọng với version hiện tại từ database; trong khi đó, `@Version` của Hibernate đóng vai trò khóa chặn các giao dịch cạnh tranh song song trong khoảng thời gian từ lúc tải dữ liệu đến lúc thực thi UPDATE.

## 6. Thuật ngữ

| Thuật ngữ | Ý nghĩa trong ngữ cảnh |
| --- | --- |
| Khóa lạc quan | Kỹ thuật kiểm soát đa truy cập: cho phép đọc song song nhưng kiểm tra xung đột ở thời điểm ghi. |
| Cột version | Bộ đếm nội bộ dùng để theo dõi sự thay đổi của thực thể, được quản lý tự động bởi trình cung cấp JPA. |
| Phiên bản kỳ vọng | Phiên bản dữ liệu mà phía gọi đang sở hữu và căn cứ vào đó để yêu cầu thay đổi. |
| Dữ liệu lỗi thời | Đối tượng mang phiên bản đã cũ, không còn khớp với trạng thái mới nhất trong database. |
| Số dòng bị ảnh hưởng | Giá trị trả về từ database: `1` xác nhận cập nhật thành công, `0` chỉ định thất bại do xung đột phiên bản. |
| `OptimisticLockException` | Ngoại lệ tiêu chuẩn của Jakarta Persistence báo hiệu sự cố vi phạm khóa lạc quan. |
| Đồng bộ dữ liệu (flush) | Tiến trình chuyển đổi trạng thái của các đối tượng trong bộ đệm thành các lệnh SQL tương ứng để gửi tới database. |
| Trạng thái tách rời (detached) | Tình trạng của một đối tượng entity không còn nằm trong sự quản lý của ngữ cảnh transaction hiện tại. |

## 7. Điều hướng tài liệu

- [Phân Tích Lỗi Thiết Kế Và Lỗ Hổng API Không Truyền Version (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Dòng Thời Gian Và Cơ Chế MVCC (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình `@Version` Và Xử Lý Ngoại Lệ (solutions.md)](solutions.md)
- [Thực Nghiệm Với PostgreSQL Testcontainers (experiments.md)](experiments.md)
- [Khóa Lạc Quan (Optimistic locking) Và Xung Đột Phiên Bản](../../concepts/optimistic-locking.md)
- [Cơ Chế Điều Khiển Đa Phiên Bản (MVCC) Của PostgreSQL](../../concepts/postgresql-mvcc.md)
- [Kiểm Thử Tương Tranh (Concurrency testing)](../../concepts/concurrency-testing.md)

## 8. Tác động tới hệ thống

Hệ quả của việc thiết kế thiếu khóa lạc quan hoặc kiểm soát phiên bản yếu kém:
- Dữ liệu bị ghi đè ngầm, dẫn đến việc người dùng nhận được phản hồi thành công nhưng dữ liệu cuối cùng lại sai lệch.
- Nhật ký kiểm toán không ghi nhận đầy đủ luồng tác động, gây mất tính khả xuất.
- Bộ nhớ đệm hoặc các hệ thống hướng sự kiện xử lý thông tin không nhất quán với database gốc.
- Lỗi xung đột chỉ bộc phát tại thời điểm commit, trả về mã lỗi `500 Internal Server Error` không thể hiện rõ nguyên nhân cho phía gọi.
- Áp dụng cơ chế thử lại sai cách vào một transaction đã hỏng (rollback-only) gây ra lỗi lan truyền.
- Bỏ qua cơ chế phiên bản đối với các thực thể cạnh tranh cao (dữ liệu nóng) có thể làm tê liệt hệ thống.
- Nhầm lẫn khi áp dụng cơ chế khóa mức ứng dụng (ví dụ: `synchronized`) trên kiến trúc triển khai nhiều instance, dẫn đến tình trạng cạnh tranh dữ liệu không kiểm soát.

## 9. Khuyến nghị áp dụng

1. Thiết lập cột `version` với ràng buộc `NOT NULL` và áp dụng annotation `@Version` cho tất cả các thực thể gốc tập hợp (aggregate root).
2. Xây dựng kịch bản migration để gán giá trị mặc định (ví dụ: `version = 0`) cho các bản ghi hiện có TRƯỚC KHI triển khai tính năng khóa lạc quan.
3. Yêu cầu thiết kế API (payload) BẮT BUỘC phải đính kèm thuộc tính `version`.
4. Chủ động gọi phương thức `flush` trước khi kết thúc tiến trình nghiệp vụ nhằm phát hiện và bắt sớm các ngoại lệ khóa lạc quan.
5. Xử lý triệt để các transaction thất bại, rollback thay đổi và chuyển đổi ngoại lệ thành phản hồi HTTP chuẩn như `409 Conflict` hoặc `412 Precondition Failed`.
6. KHÔNG tự động khởi động cơ chế thử lại đối với các lệnh gán trạng thái trực tiếp của người dùng; quy trình đúng là hiển thị trạng thái mới nhất và trao quyền quyết định hợp nhất lại cho phía gọi.

## 10. Phạm vi áp dụng

Khóa lạc quan được thiết kế tối ưu cho các mô hình dữ liệu có độ cạnh tranh truy cập ở mức thấp hoặc trung bình, đặc biệt đối với các yêu cầu chỉnh sửa thủ công từ phía người dùng.
Đối với các kịch bản cạnh tranh cao với tần suất hàng ngàn transaction/giây (ví dụ: quản lý số dư, quản lý tồn kho), các mô hình kiến trúc như cập nhật nguyên tử hoặc khóa bi quan sẽ đem lại hiệu quả cao hơn.

Chi tiết về tự động thử lại với xử lý độ trễ ngẫu nhiên được trình bày tại `LOCK-002`. Các hệ thống áp dụng cơ chế khóa bi quan được phân tích trong `LOCK-003`, và phương pháp cập nhật nguyên tử tại `LOCK-004`.
