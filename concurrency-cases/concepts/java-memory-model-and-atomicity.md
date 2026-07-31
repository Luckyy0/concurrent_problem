# Mô hình bộ nhớ Java (Java Memory Model), Tính nguyên tử và Trạng thái dùng chung

## Mục tiêu

Tài liệu này giải thích các nền tảng cơ bản khi bạn có nhiều luồng (thread) cùng lúc truy cập vào một đối tượng (object) trong ứng dụng Java. Mục tiêu quan trọng nhất là giúp bạn phân biệt rõ 3 đặc tính rất hay bị nhầm lẫn với nhau:

- **Tính nguyên tử** (`atomicity`): Thao tác có thể bị luồng khác chen ngang phá bĩnh vào giữa chừng hay không?
- **Khả năng nhìn thấy** (`visibility`): Nếu một luồng vừa sửa dữ liệu xong, luồng khác có nhìn thấy ngay sự thay đổi đó hay không?
- **Thứ tự quan sát** (`ordering`): Máy ảo Java (JVM) và CPU được phép tự do sắp xếp lại thứ tự các dòng code của bạn như thế nào để tối ưu tốc độ?

Một đoạn code có thể bảo đảm mọi người đều nhìn thấy dữ liệu mới (visibility), nhưng vẫn bị sai về tính nguyên tử (atomicity). 
Ví dụ kinh điển: Dùng từ khóa `volatile long counter` giúp các luồng luôn nhìn thấy giá trị mới nhất. Nhưng lệnh `counter++` thực chất lại gồm 3 bước rời rạc:
```text
(1) Đọc counter hiện tại → (2) Cộng thêm 1 → (3) Ghi lại vào counter
```
Giữa 3 bước này, một luồng khác hoàn toàn có thể chen ngang vào!

## Các thuật ngữ chính cần hiểu

| Thuật ngữ | Giải thích dễ hiểu |
| --- | --- |
| Mô hình bộ nhớ Java (`Java Memory Model` - JMM) | Bộ luật của Java quy định xem một luồng được phép nhìn thấy giá trị nào do luồng khác ghi xuống. |
| Tính nguyên tử (`atomicity`) | Một thao tác là nguyên tử nếu nó chạy một mạch từ đầu đến cuối, không ai có thể chen ngang. |
| Khả năng nhìn thấy (`visibility`) | Đảm bảo khi luồng A sửa biến X, luồng B ngay lập tức nhìn thấy X đã thay đổi. |
| Thứ tự quan sát (`ordering`) | Trật tự mà các hành động thực sự diễn ra dưới góc nhìn của các luồng khác. |
| Xảy ra-trước (`happens-before`) | Một quy tắc cốt lõi: Nếu A "xảy ra-trước" B, thì B chắc chắn sẽ nhìn thấy mọi thay đổi do A làm ra. |
| Thao tác ghép (`compound operation`) | Một chuỗi xử lý logic được tạo thành từ nhiều bước nhỏ (ví dụ: `counter++` hoặc `if(x > 0) x--`). |
| Công bố an toàn (`safe publication`) | Cách bạn chia sẻ một object cho các luồng khác sao cho chúng thấy được trạng thái hoàn chỉnh, không bị "mẻ" dữ liệu. |
| Giam giữ trong luồng (`thread confinement`) | Chiến thuật an toàn nhất: Tuyệt đối chỉ cho đúng một luồng được quyền giữ và sửa dữ liệu đó. |

## Mô hình bộ nhớ Java (JMM) và quy tắc Xảy ra-trước (happens-before)

JMM sử dụng một khái niệm gọi là quan hệ **xảy ra-trước** (`happens-before`) để kết nối các hành động giữa nhiều luồng với nhau. 

Dưới đây là một số luật "xảy ra-trước" quan trọng nhất:
- Hành động **mở khóa** (unlock) luôn xảy ra-trước hành động **khóa** (lock) tiếp theo trên cùng một ổ khóa đó.
- Hành động **ghi** vào một biến `volatile` luôn xảy ra-trước hành động **đọc** tiếp theo trên chính biến đó.
- Bất cứ hành động nào bạn làm trước khi gọi `Thread.start()` đều xảy ra-trước mọi hành động bên trong luồng mới được tạo.
- Mọi hành động bên trong một luồng đã kết thúc đều xảy ra-trước thời điểm một luồng khác thoát khỏi hàm `Thread.join()`.
- Các câu lệnh viết liền nhau trong cùng một luồng thì lệnh trên xảy ra-trước lệnh dưới (tuân theo đúng thứ tự code của bạn).

Quy tắc này giúp đảm bảo luồng đi sau sẽ nhận được dữ liệu đúng của luồng đi trước. Tuy nhiên, nó **không** biến một chuỗi nhiều bước thành một thao tác nguyên tử.

> **Nói ngắn gọn:** Nhìn thấy đúng giá trị mới nhất KHÔNG có nghĩa là luồng của bạn sẽ không bị chen ngang giữa chừng khi thực hiện các bước xử lý.

## Thao tác nguyên tử và các quy tắc gồm nhiều bước

Một thao tác đọc hoặc ghi biến đơn giản thì có thể là nguyên tử. Nhưng lỗi thường xảy ra ở các khối code có điều kiện logic:

```java
if (remaining > 0) {
    remaining--;
}
```

Đoạn code trên bao gồm một chuỗi: Đọc kiểm tra → Quyết định → Ghi trừ đi. 
Để bảo vệ toàn bộ chuỗi này khỏi việc bị luồng khác chen ngang, TẤT CẢ các luồng tham gia phải dùng chung một "ổ khóa", ví dụ như:
- Dùng chung khóa qua từ khóa `synchronized`.
- Dùng chung đối tượng `Lock` (như ReentrantLock).
- Dùng vòng lặp so-sánh-và-cập-nhật (`compare-and-set` - CAS).
- Thiết kế code sao cho chỉ một luồng duy nhất được quyền sửa biến đó.
- **Tốt nhất:** Thiết kế code không có trạng thái dùng chung (loại bỏ hoàn toàn các biến toàn cục có thể thay đổi).

**Lưu ý:** Việc bạn dùng một biến an toàn (thread-safe) không có nghĩa là toàn bộ logic của bạn an toàn. Ví dụ: Dùng `AtomicLong sequence` chỉ bảo vệ được việc tăng số `sequence`. Nó không thể bảo vệ được một quy tắc đòi hỏi phải cập nhật cùng lúc hai biến `(sequence, customerId)`.

## Bảng so sánh Volatile, Biến Atomic và Khóa

| Cơ chế | Điểm mạnh (Bảo đảm được) | Điểm yếu (Không tự bảo đảm được) |
| --- | --- | --- |
| Từ khóa `volatile` | Đảm bảo tính nhìn thấy (visibility) và thứ tự (ordering) cho duy nhất một biến. | KHÔNG bảo vệ được chuỗi nhiều bước (`counter++`). |
| `AtomicLong` | Bảo vệ hoàn hảo (atomic/CAS) khi tính toán trên một giá trị duy nhất. | KHÔNG bảo vệ được quy tắc liên quan đến nhiều biến cùng lúc. |
| Khối `synchronized` | Bảo vệ độc quyền (mutual exclusion) và thứ tự xảy ra-trước trên cùng một ổ khóa (monitor). | KHÔNG thể phối hợp khóa giữa 2 server (JVM) khác nhau. |
| `ReentrantLock` | Bảo vệ độc quyền, kèm theo tính năng nâng cao như giới hạn thời gian chờ (timeout) hoặc ngắt (interrupt). | Vẫn sẽ sai nếu luồng khác cố tình dùng một ổ khóa khác. |
| Biến bất biến (`Immutable object`) | Trạng thái không bao giờ bị đổi sau khi khởi tạo thành công (safe publication). | KHÔNG giúp chuỗi xử lý logic ở bên ngoài object trở thành nguyên tử. |

*(Một vài định nghĩa bổ sung cho người mới:)*
- `Mutual exclusion` (Độc quyền): Tại một thời điểm, chỉ cho phép đúng 1 luồng được đi qua cửa để vào khối code.
- `Compare-and-set` (CAS): Kỹ thuật "So sánh rồi mới cập nhật". Nó chỉ ghi giá trị mới xuống nếu giá trị hiện tại dưới RAM vẫn y hệt như giá trị mà nó vừa đọc lúc nãy.

**Lời khuyên vàng:** Hãy ưu tiên thiết kế các service phi trạng thái (stateless) hoặc dùng các object bất biến (immutable) trước khi nghĩ đến việc dùng khóa (lock). Không có biến dùng chung thì sẽ không bao giờ có lỗi tranh chấp luồng!

## Công bố đối tượng an toàn (Safe publication)

Một đối tượng (object) chỉ thực sự an toàn và bất biến (immutable) trong mắt các luồng khác khi đạt đủ 4 điều kiện:
1. Toàn bộ dữ liệu của object phải được gán xong xuôi đâu vào đấy trước khi được tung ra cho luồng khác xài.
2. Các biến bên trong nên được khai báo là `final`.
3. Trong lúc hàm khởi tạo (`constructor`) đang chạy, tuyệt đối không được để lọt từ khóa `this` ra ngoài.
4. Object phải được chia sẻ cho luồng khác thông qua một quan hệ "xảy ra-trước" hợp lệ.

Trong Spring Boot, các Bean (dạng singleton) được Spring công bố rất an toàn sau khi khởi tạo xong. Tuy nhiên, điều đó **chỉ** đảm bảo các luồng nhìn thấy Bean đã được tạo hoàn chỉnh. Nếu bên trong Bean đó bạn cố tình viết một biến có thể thay đổi (mutable field) rồi sửa nó khi xử lý request, thì code của bạn vẫn sẽ bị lỗi đa luồng như thường!

## Ranh giới giữa một máy ảo (JVM) và nhiều máy chủ (Application Instances)

Mỗi khi bạn chạy một ứng dụng Java, nó sẽ nằm trong một máy ảo (JVM). Mỗi máy ảo có vùng nhớ (heap), ổ khóa (monitor) và biến atomic CỦA RIÊNG NÓ:

```text
Cân bằng tải (Load Balancer)
    ├── Máy chủ A: counter = 42, giữ ổ khóa A
    └── Máy chủ B: counter = 42, giữ ổ khóa B
```

Việc bạn dùng `AtomicLong`, `synchronized` hay `ReentrantLock` trong code của Máy chủ A **hoàn toàn không có tác dụng gì** đối với Máy chủ B. 
Nếu bạn có một quy tắc nghiệp vụ cần chạy chung cho cả 2 máy chủ (ví dụ: trừ tiền tài khoản), bạn BẮT BUỘC phải phối hợp khóa ở tầng cơ sở dữ liệu chung, ví dụ như:
- Các ràng buộc của cơ sở dữ liệu (Database constraint).
- Lệnh SQL cập nhật có điều kiện (Conditional SQL).
- Lệnh khóa dòng (Row lock).
- Bảng lưu dấu vân tay chống trùng lặp (Durable idempotency record).
- Cơ chế thuê quyền (Protocol lease/fencing) phù hợp khi có máy chủ bị sập.

> **Nói ngắn gọn:** Các loại Khóa trong Java (như `synchronized`) chỉ bảo vệ được các luồng chạy bên trong cùng một cái máy tính (tiến trình). Nó vô dụng khi bạn chạy ứng dụng trên nhiều máy chủ khác nhau!

## Liên kết tài liệu tham khảo

- Case áp dụng trực tiếp: [JVM-001](../jvm/spring-singleton-mutable-state/README.md)
- Kỹ thuật kiểm thử: [Kiểm thử đồng thời](concurrency-testing.md)
- Các case liên quan trong [catalog](../CONCURRENCY_CASE_LIBRARY.md): `JVM-005`, `JVM-006`, `DB-001`
