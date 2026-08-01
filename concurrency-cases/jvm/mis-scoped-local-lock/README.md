# Bài toán JVM-006 — Cấu trúc `synchronized` và `ReentrantLock` Đặt Sai Phạm Vi

## 1. Tóm tắt vấn đề (Overview)

Hãy tưởng tượng bạn có một Spring service làm nhiệm vụ tạo file đối soát cuối ngày (settlement artifact) và lưu vào kho dùng chung. Luật là: Hai yêu cầu cùng khóa (artifactKey) không được phép chạy tạo file và ghi đè cùng lúc. Nhiều anh em dev hay dùng `synchronized` hoặc `ReentrantLock` để khóa lại. Tuy nhiên, lỗi hay xảy ra ở chỗ: (1) tạo một cái khóa mới tinh ở mỗi lần gọi hàm, (2) khóa nhầm đối tượng, hoặc (3) ảo tưởng rằng khóa trên một server (local lock) có thể bảo vệ được dữ liệu khi chạy trên nhiều server (distributed).

Khóa chỉ có tác dụng khi các luồng (threads) cùng tranh nhau **một ổ khóa duy nhất** và vùng khóa phải bao trọn được dữ liệu dùng chung. Hai object khác nhau không thể làm chung một khóa chỉ vì hàm `equals()` của chúng giống nhau; và dĩ nhiên, hai máy chủ JVM khác nhau không thể xài chung một cái khóa nội bộ được.

Các quy tắc kinh điển cần nhớ:

```text
- Trong một ứng dụng (JVM), chỉ một luồng xử lý cùng một artifactKey được phép đi vào vùng nhạy cảm (vùng tới hạn).
- Việc kiểm tra dữ liệu đã có chưa, tạo dữ liệu, và lưu dữ liệu phải nằm trọn trong cùng một vòng bảo vệ của khóa.
- Phải luôn mở khóa (release) trong mọi tình huống: dù thành công, có lỗi, hết hạn hay bị ngắt.
- Đừng lầm tưởng khóa nội bộ trên một máy có thể cản được các máy khác trong hệ thống phân tán.
```

> **Nguyên tắc kỹ thuật:** Gõ thêm chữ `synchronized` là chưa đủ; mọi luồng phải cùng bám vào một "ổ khóa", và ổ khóa đó phải bao bọc toàn bộ chu kỳ sống của dữ liệu dùng chung.

## 2. Các Thuật ngữ Chuyên ngành (Terminology)

| Thuật ngữ | Ý nghĩa vui vẻ, dễ hiểu |
| --- | --- |
| Định danh khóa (`monitor identity`) | Địa chỉ thật sự của cái ổ khóa trong bộ nhớ máy tính. |
| Phạm vi khóa (`lock scope`) | Những dòng code và luồng nào thật sự được khóa bảo vệ. |
| Vùng tới hạn (`critical section`) | Vùng code "cấm địa", cấm nhiều luồng nhảy vào cùng lúc. |
| Khóa theo định danh (`keyed lock`) | Mỗi mã nghiệp vụ sẽ có riêng một ổ khóa tương ứng. |
| Khóa phân dải (`striped lock`) | Chia khóa thành vài nhóm (dải) cố định. Mã nghiệp vụ sẽ được băm (hash) để chọn một dải, giúp tiết kiệm RAM. |
| Loại trừ lẫn nhau (`mutual exclusion`) | Tại một thời điểm, chỉ một anh được phép giữ khóa. |
| Khóa cục bộ (`local lock`) | Cái khóa chỉ tồn tại và có tác dụng trên một máy chủ (JVM) duy nhất. |
| Khởi tạo có điều kiện (`conditional create`) | Lưu vào kho với điều kiện: "Chỉ ghi nếu chưa có ai ghi". |
| Thẻ chắn (`fencing token`) | Giống như cái vé có đánh số thứ tự để kho từ chối mấy anh chậm chân, hết hạn. |

## 3. Bối cảnh nghiệp vụ (Business Context)

Ứng dụng của chúng ta chạy tính toán cuối ngày:
- Client hoặc hệ thống tự động gọi hàm `generate("settlement/2026-07-29.csv")`.
- Code sẽ check xem file đã có chưa.
- Nếu chưa có, tiến hành đọc data, xử lý (rất nặng) và đẩy file lên Kho (Object store).
- Nếu bị gọi trùng, phải trả về file đã làm xong thay vì hì hục làm lại từ đầu.
- Hệ thống này được deploy trên nhiều máy chủ (Multi-instance).

Việc xử lý tạo file ngốn cực nhiều CPU và Database. Nếu để các máy cùng chạy trùng một file, không những làm sập server mà còn có nguy cơ ghi đè dữ liệu sai lệch lên kho chứa.

## 4. Các Thực thể và Trạng thái chia sẻ (Shared state & Contention)

| Thành phần | Nó là cái gì? |
| --- | --- |
| Tài nguyên Logic | Chính là cái `artifactKey` (mã file). |
| Chia sẻ Nội bộ (Local) | Bộ tạo file, Cache và cái Map chứa khóa. |
| Chia sẻ Phân Tán (Global) | Kho chứa file (Object store). |
| Chuỗi thao tác hay lỗi | `exists(key) → render(key) → put(key, bytes)` |
| Chủ Thể Cạnh Tranh (Actor) | User request, Job tự động, hoặc Luồng tự retry. |
| Vùng Tới Hạn Nội Bộ | Toàn bộ khâu kiểm tra và lưu file cho cùng một khóa. |
| Ranh Giới Uy Quyền (Authoritative) | Database hoặc Store (nơi quyết định cuối cùng). |

Nếu bạn tạo khóa mới mỗi lần chạy thì khóa coi như vô dụng. Còn nếu xài một khóa bự (Global lock) cho mọi luồng thì hệ thống sẽ bị chậm rì (như kẹt xe). Khóa phân dải (Striped lock) là một cách cực hay để cân bằng giữa việc chạy song song và đảm bảo tính đúng đắn trên một máy.

## 5. Giới hạn Áp dụng (Out of Scope)

Trong bài này, chúng ta sẽ mổ xẻ:
- Phân biệt giữa "Cùng một ổ khóa" và "Hai chuỗi giống nhau".
- Sự khác nhau giữa khóa cấp biến (field) và khóa cục bộ trong hàm.
- Tác hại của việc khóa vùng code quá ngắn.
- Cách thiết kế Khóa phân dải và quản lý vòng đời của khóa.
- Luật "trả khóa" khi gặp lỗi, hết giờ.
- Ranh giới thật sự của khóa trên một máy.

Bài này KHÔNG dạy cách làm hệ thống khóa phân tán (như xài Redis hay Zookeeper) — cái đó thuộc về bài `DIST-001`. Cơ chế Database khóa hay Store khóa có điều kiện chỉ được nhắc tới như một giải pháp chốt chặn cuối cùng.

## 6. Điều hướng Tài liệu (Navigation)

- [Phân Tích Lỗi Thiết Kế Hệ Thống (broken-code.md)](broken-code.md)
- [Phân Tích Chuyên Sâu Luồng Tương Tranh (analysis.md)](analysis.md)
- [Giải Pháp Cấu Hình Phù Hợp (solutions.md)](solutions.md)
- [Thực Nghiệm Khóa Bi Quan Với Testcontainers (experiments.md)](experiments.md)
- [Tổng Quan Về Mô Hình Bộ Nhớ Java (Java Memory Model)](../../concepts/java-memory-model-and-atomicity.md)
- [Phương Pháp Kiểm Thử Đồng Thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## 7. Tác Động Tới Hệ Thống (Production Impact)

### Hệ Quả Kỹ Thuật
- Đốt CPU vì chạy tính toán trùng lặp; nghẽn mạng do ghi đè file liên tục.
- Lỗi "Ai ghi sau thì thắng", làm mất sạch dữ liệu của người chạy trước.
- Khóa toàn cục làm đứng im toàn bộ request, kể cả mấy request chả liên quan gì đến nhau.
- Map chứa khóa bị phình to làm tràn RAM, hoặc xóa khóa sai thời điểm đẻ ra 2 khóa song song cho 1 dữ liệu.
- Luồng bị treo vĩnh viễn vì không cài đặt timeout.
- Lỗi xảy ra không chịu mở khóa (quên `finally`), làm khóa bị chết kẹt luôn.

### Hệ Quả Nghiệp Vụ
- File đối soát bị lỗi, sai lệch dữ liệu.
- Các hệ thống phía sau nhận được hàng loạt tín hiệu "File xong rồi" trùng lặp.
- Đánh gục Database hoặc Kho lưu trữ ngay lúc cao điểm.
- Lúc test trên 1 máy ở Dev thì chạy mượt, nhưng lên Production chạy nhiều máy thì toang.

## 8. Khuyến Nghị Phân Lớp Áp Dụng (Best Practices)

Trên một máy chủ (Nội trong Mạch JVM):
- Dùng `ReentrantLock` hay Monitor (`synchronized`) kiểu Private Final để khóa diện rộng.
- Xài Khóa Phân Dải (Striped locks) nếu lượng khóa nhiều và muốn chạy song song mạnh mẽ.
- Xài Map giữ khóa cố định nhưng phải biết cách dọn dẹp cẩn thận.
- Đã khóa thì phải khóa trọn vẹn từ lúc "Check" đến lúc "Lưu" thì mới an toàn.

Giữa nhiều máy chủ (Xuyên suốt Mạng Đa Nút):
- Giao phó việc quyết định xung đột cho Database hoặc Store: Ghi có điều kiện, đánh Unique index hoặc xài Khóa phân tán (Distributed lease).
- Khóa nội bộ trên máy chỉ giúp cản bớt việc tính toán thừa, nó CHƯA BAO GIỜ là cái khiên bảo vệ tuyệt đối cho toàn bộ hệ thống.
