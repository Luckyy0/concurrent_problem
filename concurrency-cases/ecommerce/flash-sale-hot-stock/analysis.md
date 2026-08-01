# Phân tích tranh chấp trên sản phẩm nóng

## 1. Trạng thái ban đầu

```text
Sản phẩm FLASH-77:
  on_hand_quantity   = S
  available_quantity = S
  reserved_quantity  = 0

N yêu cầu cùng mua sản phẩm FLASH-77
N lớn hơn số kết nối dành cho đường xử lý này
Mỗi yêu cầu dùng một giao dịch READ COMMITTED
```

Ký hiệu được dùng thay cho số liệu benchmark. Giá trị thật phụ thuộc phần cứng,
cấu hình PostgreSQL, vùng kết nối, thời lượng giao dịch và dạng tải thực tế.

## 2. Hai vấn đề độc lập

ECOM-002 phải giữ tách biệt hai câu hỏi:

| Câu hỏi | Cơ chế trả lời |
| --- | --- |
| Có bán vượt số lượng không? | `UPDATE` có điều kiện và giao dịch cơ sở dữ liệu |
| Hệ thống có chịu được đợt yêu cầu không? | Điều tiết đầu vào, giới hạn chờ và cô lập tài nguyên |

Một hệ thống có thể trả lời đúng câu thứ nhất nhưng thất bại ở câu thứ hai. Khi
đó không có dữ liệu âm, nhưng phần lớn yêu cầu hết thời gian hoặc không lấy được
kết nối.

## 3. Nút thắt không thể chạy song song

Tất cả yêu cầu cùng sửa một dòng. PostgreSQL phải sắp thứ tự các lần thay đổi
dòng đó, dù ứng dụng có bao nhiêu luồng, máy chủ hay kết nối.

Nếu tốc độ yêu cầu đến trong một khoảng thời gian lớn hơn tốc độ giao dịch có
thể lần lượt chốt trên dòng nóng, lượng công việc chờ sẽ tăng. Đây là giới hạn
của tài nguyên tuần tự, không phải lỗi có thể chữa bằng cách tạo thêm luồng.

## 4. Dòng thời gian với câu cập nhật có điều kiện

| Bước | Giao dịch đầu | Các giao dịch khác | PostgreSQL |
| --- | --- | --- | --- |
| 1 | Chạy `UPDATE` | Cùng chạy `UPDATE` | Bên đầu lấy khóa dòng |
| 2 | Ghi bản giữ hàng | Chờ khóa | Khóa được giữ tới lúc chốt |
| 3 | Chốt | Một bên được đánh thức | Kiểm tra lại `WHERE` |
| 4 | Hoàn tất | Bên kế tiếp cập nhật hoặc nhận `0` dòng | Hàng đợi tiến từng vị trí |

Câu SQL này không tạo bão thử lại do xung đột phiên bản. Tuy nhiên, các bên chờ
vẫn giữ kết nối sau khi đã vào giao dịch. Nếu giao dịch còn ghi lịch sử hoặc hộp
thư đi, khóa được giữ đến khi các thao tác đó hoàn tất.

## 5. Hàng đợi chuyển từ tầng này sang tầng khác

Không có hệ thống nào loại bỏ hoàn toàn việc chờ. Thiết kế chỉ quyết định nơi
chờ và giới hạn của nó:

| Nơi chờ | Tài nguyên bị giữ | Hậu quả khi không giới hạn |
| --- | --- | --- |
| Luồng HTTP | Bộ nhớ và luồng xử lý | Tăng số yêu cầu đang mở |
| Hàng đợi trong JVM | Bộ nhớ tiến trình | Tăng bộ nhớ, mất việc khi tiến trình dừng |
| Vùng kết nối | Luồng ứng dụng | Hết thời gian lấy kết nối |
| Khóa PostgreSQL | Kết nối và giao dịch | Chiếm hết kết nối đang hoạt động |
| Hàng đợi bền vững | Dung lượng lưu trữ và độ trễ | Tồn đọng tăng nếu đầu vào luôn lớn hơn đầu ra |

Điều tiết đầu vào đặt một hàng đợi nhỏ hoặc một điểm từ chối trước vùng kết nối.
Nhờ đó PostgreSQL không phải làm nơi hấp thụ toàn bộ lưu lượng đột biến.

## 6. Cách vùng kết nối bị chiếm hết

Gọi `P` là ngân sách kết nối của đường flash sale:

1. Một giao dịch giữ khóa dòng.
2. Tối đa `P - 1` giao dịch khác lấy kết nối rồi chờ khóa.
3. Vùng kết nối đạt giới hạn dù chỉ một giao dịch đang làm thay đổi hữu ích.
4. Yêu cầu tiếp theo chờ lấy kết nối.
5. Truy vấn không liên quan nhưng dùng cùng vùng kết nối cũng bị chậm.

Tăng `P` làm hàng đợi trong PostgreSQL dài hơn. Nó chỉ có ích khi nút thắt thật
sự là thiếu kết nối cho công việc độc lập; nó không làm một dòng nóng xử lý nhiều
lần ghi cùng lúc.

## 7. Vì sao `@Version` dễ tạo bão thử lại

Với khóa lạc quan, nhiều giao dịch cùng đọc một phiên bản. Chỉ bên chốt trước cập
nhật thành công; các bên còn lại phát hiện xung đột ở lúc `flush()`.

Nếu mọi bên lập tức mở giao dịch mới:

```text
yêu cầu gốc
→ đọc và làm việc
→ xung đột phiên bản
→ mở giao dịch mới
→ cùng tranh chấp lại
```

Tài nguyên được dùng cho nhiều lần đọc và nhiều giao dịch thất bại. Khoảng lùi và
độ lệch ngẫu nhiên có thể giảm việc các bên quay lại cùng lúc, nhưng không thay
đổi giới hạn tuần tự của dòng. Khi xung đột đã trở thành trạng thái bình thường,
`UPDATE` có điều kiện phù hợp hơn vì bên không còn hàng nhận kết quả nghiệp vụ mà
không cần thử lại.

## 8. Vì sao `FOR UPDATE` có thể giữ khóa lâu hơn

`FOR UPDATE` lấy khóa ở bước đọc. Mọi tính toán, tạo đối tượng, truy vấn bổ sung
và thao tác ghi sau đó diễn ra trong thời gian giữ khóa.

Cập nhật có điều kiện lấy khóa tại câu ghi, nên phần chuẩn bị có thể hoàn tất
trước khi vào đoạn tranh chấp. Với quy tắc chỉ là “còn đủ hàng”, `FOR UPDATE`
không đem lại thêm tính đúng đắn nhưng thường mở rộng đoạn giữ khóa.

Nếu Java thật sự cần đọc nhiều trường rồi quyết định, `FOR UPDATE` vẫn có thể là
lựa chọn đúng. Khi đó phải giữ đoạn sau khóa ngắn và đặt giới hạn chờ.

## 9. Ba loại thời hạn khác nhau

| Giới hạn | Bảo vệ điều gì |
| --- | --- |
| Thời gian lấy kết nối | Không để luồng chờ vùng kết nối vô hạn |
| `lock_timeout` | Không để giao dịch giữ kết nối chỉ để chờ khóa quá lâu |
| `statement_timeout` | Giới hạn thời gian chạy toàn bộ câu lệnh |

Ba giới hạn phải nằm trong thời hạn phản hồi tổng. Nếu cổng API hết thời gian
trước PostgreSQL, phía gọi có thể bỏ cuộc trong khi truy vấn cũ vẫn tiếp tục làm
việc. Lần thử lại mới sau đó chồng lên công việc chưa kết thúc.

Giá trị cụ thể phải được đo và cấu hình theo hệ thống. Case này không cung cấp
một bộ số dùng chung.

## 10. Thử lại làm tăng tải như thế nào

Thử lại chỉ hợp lý khi lỗi có khả năng biến mất và yêu cầu còn thời hạn. Trong
đợt bán cao điểm:

- `OUT_OF_STOCK` không được thử lại;
- `BUSY` không được máy chủ tự lặp ngay trong cùng yêu cầu;
- `55P03`, `40P01`, `40001` chỉ được thử lại có giới hạn, trong giao dịch mới;
- lỗi lấy kết nối thường là dấu hiệu cần từ chối tải, không phải tín hiệu mở thêm
  nhiều lần thử ngay lập tức.

Phải tính cả lần thử từ khách hàng, cổng API, thư viện HTTP và ứng dụng. Mỗi tầng
đều thử lại “một ít” có thể tạo số lần thực thi lớn hơn nhiều số yêu cầu gốc.

## 11. Điều tiết đầu vào đồng bộ

Cổng điều tiết cấp một số quyền xử lý hữu hạn trước khi mở giao dịch:

```text
yêu cầu
→ xin quyền xử lý
   → không có quyền: trả BUSY ngay
   → có quyền: mở giao dịch và chạy UPDATE có điều kiện
→ trả quyền trong finally
```

Điều tiết không quyết định ai được mua hàng và không thay thế câu SQL. Nó chỉ
giới hạn số bên được phép tranh chấp tài nguyên cùng lúc.

Từ chối nhanh là một phần của hợp đồng sản phẩm. API phải cho phía gọi biết đó là
quá tải tạm thời hay hết hàng, và công bố cách gửi lại phù hợp nếu có.

## 12. Giới hạn cục bộ trong hệ thống nhiều máy chủ

Nếu mỗi máy có `K` quyền và có `I` máy, giới hạn lý thuyết toàn cụm là tổng quyền
của các máy đang hoạt động. Khi tự động tăng số máy, tổng lưu lượng vào PostgreSQL
cũng có thể tăng dù cấu hình mỗi máy không đổi.

Vì vậy cần:

- đặt ngân sách dựa trên tổng số máy và giới hạn của PostgreSQL;
- dành phần vùng kết nối cho chức năng khác;
- quan sát tổng tải tại cơ sở dữ liệu, không chỉ chỉ số từng máy;
- không gọi cổng cục bộ là khóa toàn cụm.

Một bộ điều tiết phân tán có thể tạo giới hạn chung nhưng bổ sung thêm một hệ
thống phải nhất quán và phục hồi. Với bất biến tồn kho đã nằm tại PostgreSQL, lớp
này chỉ đáng dùng khi lợi ích vận hành lớn hơn độ phức tạp.

## 13. Cô lập tài nguyên

Nếu đường flash sale và API thông thường dùng chung toàn bộ vùng kết nối, tải nóng
có thể làm cả hai cùng ngừng tiến lên. Có thể tách ngân sách bằng:

- cổng điều tiết chỉ dành cho sản phẩm nóng;
- vùng kết nối hoặc dịch vụ triển khai riêng cho đường bán cao điểm;
- giới hạn đồng thời riêng cho truy vấn nền và yêu cầu người dùng;
- giới hạn tại cổng API trước khi yêu cầu tới ứng dụng.

Cô lập tài nguyên không làm tăng thông lượng của dòng nóng. Nó giới hạn phạm vi
ảnh hưởng khi dòng đó bị nghẽn.

## 14. Gợi ý hết hàng để từ chối sớm

Sau khi câu cập nhật trả `0` cho một chiến dịch không bổ sung thêm hàng, ứng dụng
có thể ghi nhớ gợi ý “đã hết” và từ chối các yêu cầu sau trước cơ sở dữ liệu.

Gợi ý này an toàn để tối ưu chỉ khi:

- vòng đời chiến dịch và phiên bản tồn kho được xác định;
- có đường xóa hoặc đổi phiên bản khi bổ sung hàng;
- dữ liệu “còn hàng” trong bộ nhớ không bao giờ được dùng để bỏ qua câu SQL;
- chấp nhận được việc một máy chậm nhận thông tin hết hàng và vẫn gọi PostgreSQL.

PostgreSQL vẫn là nguồn sự thật. Gợi ý sai theo hướng “còn hàng” chỉ làm tăng tải;
gợi ý sai theo hướng “hết hàng” có thể từ chối khách dù kho đã được bổ sung.

## 15. Hàng đợi bền vững theo sản phẩm

Khi không muốn từ chối yêu cầu và có thể trả kết quả sau, hệ thống có thể đưa
yêu cầu vào hàng đợi được định tuyến theo `product_id`:

```text
API tiếp nhận
→ lưu yêu cầu bền vững
→ trả mã theo dõi
→ bộ xử lý của phân vùng lần lượt chạy giao dịch tồn kho
→ lưu kết quả
→ phía gọi tra cứu hoặc nhận thông báo
```

Tất cả yêu cầu cùng sản phẩm phải đi qua cùng một thứ tự xử lý. Nhiều sản phẩm
khác nhau vẫn có thể nằm ở các phân vùng khác và chạy song song.

Hàng đợi đưa thời gian chờ ra ngoài vùng kết nối, nhưng tạo thêm yêu cầu:

- chống tiếp nhận và xử lý trùng;
- giới hạn lượng tồn đọng;
- phục hồi khi bộ xử lý dừng giữa chừng;
- xác định thứ tự, hủy yêu cầu và thời hạn;
- theo dõi khoảng cách giữa đầu vào và đầu ra.

Ngay cả bộ xử lý tuần tự vẫn nên dùng `UPDATE` có điều kiện vì gửi lại, chuyển
phân vùng hoặc thao tác quản trị có thể tạo xử lý lặp.

## 16. Công bằng và thứ tự

Không nên hứa thứ tự “đến trước mua trước” chỉ vì các giao dịch đang chờ khóa.
Thứ tự đến bộ cân bằng tải, lấy kết nối và được PostgreSQL đánh thức không tạo một
hợp đồng công bằng bền vững.

Nếu nghiệp vụ cần thứ tự rõ ràng, hệ thống phải định nghĩa điểm tiếp nhận, số thứ
tự, cách xử lý trùng và quy tắc khi người mua hủy hoặc hết thời hạn. Đây là một
hợp đồng hàng đợi, không phải thuộc tính ngầm của khóa dòng.

## 17. Chốt, hoàn tác và mất phản hồi

- Giao dịch hoàn tác giải phóng khóa và trả tồn kho về trạng thái trước đó.
- Bên chờ tiếp tục và kiểm tra lại điều kiện sau khi khóa được giải phóng.
- Hết thời gian chờ khóa làm giao dịch hiện tại thất bại; không được lưu
  bản ghi giữ hàng thành công.
- Mất phản hồi quanh `COMMIT` tạo kết quả chưa rõ. Phía gọi phải tra cứu bằng mã
  yêu cầu thay vì tạo một ý định mua mới.
- Cổng điều tiết phải trả quyền trong `finally`, kể cả khi giao dịch thất bại.
- Tiến trình dừng sẽ làm mất hàng đợi trong bộ nhớ; công việc cần bảo đảm tiếp
  nhận phải được lưu bền vững trước khi xác nhận.

## 18. Nguyên nhân gốc theo tầng

| Tầng | Sai lầm thường gặp |
| --- | --- |
| API | Nhận mọi yêu cầu dù hệ thống không còn ngân sách xử lý |
| Spring | Thử lại rộng, không xét thời hạn và trạng thái quá tải |
| Hibernate | Dùng `@Version` cho dòng có xung đột là trạng thái bình thường |
| Vùng kết nối | Cho tải nóng chiếm toàn bộ ngân sách dùng chung |
| PostgreSQL | Trở thành hàng đợi cuối cho mọi yêu cầu cùng một dòng |
| Vận hành | Tăng kết nối hoặc số máy mà không tính tổng tải vào cơ sở dữ liệu |

## 19. Dữ liệu cần theo dõi

- số yêu cầu vào, được cấp quyền, bị báo bận, hết hàng và thành công;
- số lần thực thi cơ sở dữ liệu trên mỗi yêu cầu gốc;
- số kết nối đang dùng, còn rảnh và số luồng chờ kết nối;
- thời gian chờ khóa và số phiên đang chờ cùng `product_id`;
- SQLSTATE `55P03`, `40P01`, `40001`;
- thời lượng giao dịch từ câu cập nhật đến lúc chốt;
- ảnh hưởng đến API không thuộc đợt bán;
- lượng tồn đọng và tuổi bản ghi lâu nhất nếu dùng hàng đợi bền vững.

Không đặt ngưỡng cảnh báo chỉ dựa trên số liệu trong tài liệu. Ngưỡng phải xuất
phát từ thời hạn phản hồi, ngân sách tài nguyên và phép đo của hệ thống thật.

## 20. Phạm vi phân tích

Case này không so sánh benchmark giữa các chiến lược và không hứa một kích thước
vùng kết nối tối ưu. Nó giải thích cơ chế để đội phát triển tự đo và chọn giới
hạn.

Tính đúng đắn của phép trừ kho thuộc ECOM-001. Chống yêu cầu trùng thuộc
ECOM-003. Chi tiết giao hàng qua Kafka, thứ tự phân vùng và xử lý lại thuộc các
case `MSG-*`.
