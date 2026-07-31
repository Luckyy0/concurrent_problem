# SPR-004 — Isolation annotation không khớp transaction boundary thật

## Tóm tắt

Hàm `ReportFacade.generate()` mở một giao dịch vòng ngoài (outer transaction) bằng mức độ cô lập (isolation) mặc định của PostgreSQL là `READ COMMITTED`. Sau đó, nó gọi đến bean `SnapshotQueryService.readTwice()` vốn được đánh dấu (annotated) mức cô lập là `REPEATABLE_READ`. Ngặt nỗi, vì cơ chế lan truyền (propagation) mặc định là `REQUIRED`, phương thức vòng trong (inner method) đành ngậm ngùi tham gia (join) vào giao dịch vật lý đã được mở sẵn; thế nên cái annotation `REPEATABLE_READ` kia bị bơ đẹp và hoàn toàn không nâng (upgrade) mức cô lập lên.

Hậu quả là: Có một luồng ghi (Writer) chen ngang chốt (commit) giá mới ngay giữa hai lệnh SELECT. Báo cáo (Report) ngơ ngác nhận về giá trị `100` ở lần đọc một rồi giật thót với giá `120` ở lần hai, trái ngược hoàn toàn với kỳ vọng về một bản ảnh tĩnh bất biến (stable snapshot).

Rào chắn tính đúng đắn (Invariant):

```text
Hai lần đọc dữ liệu hòng tạo ra cùng một bản báo cáo bắt buộc phải sử dụng chung hệ quy chiếu ngữ nghĩa bản ảnh (snapshot semantics) đã được công bố.
Mức cô lập thực tế (Effective isolation) phải được ấn định ngay tại cái nôi nơi giao dịch vật lý được khai sinh (physical transaction được tạo).
Sự vênh nhau về mức độ cô lập (Isolation mismatch) tuyệt đối không được phép ậm ừ cho qua (silently chấp nhận) ở những ranh giới (boundary) sống còn.
Các bài kiểm thử (Test) không những phải xác minh mức cài đặt thực tế (effective setting) mà còn phải rà soát chặt chẽ cả kết quả đọc nghiệp vụ (business read outcome).
```

> **Nói ngắn gọn:** Mức cô lập (isolation) là đặc quyền sống còn gắn liền với bản thân giao dịch (transaction), chứ nó không thèm thuộc về riêng lẻ một đoạn phương thức (method) nào đó chỉ vì nó được trang hoàng cái annotation đẹp đẽ nhất.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| effective isolation | Mức cô lập thực tế: Mức độ cô lập đang thực sự vận hành của giao dịch vật lý hiện tại |
| isolation declaration | Khai báo cô lập: Mức cô lập mà phương thức ao ước mỏi mòn được set trong `@Transactional` |
| isolation inheritance | Kế thừa cô lập: Phương thức vòng trong mang mác REQUIRED đành phải dùng ké mức cô lập của giao dịch vòng ngoài đã có sẵn |
| datasource default | Mặc định của nguồn dữ liệu: Mức cô lập được lôi ra dùng làm bia đỡ đạn khi phần khai báo là `DEFAULT` |
| non-repeatable read | Đọc không lặp lại: Thảm cảnh cùng một dòng dữ liệu (row) đọc hai lần nhưng lại lòi ra hai giá trị đã commit (committed values) khác nhau |
| transaction creation point | Nơi khai sinh giao dịch: Điểm gọi qua proxy nơi mà người quản lý giao dịch (transaction manager) chính thức làm lễ mở giao dịch vật lý |
| validation existing transaction | Xác nhận giao dịch có sẵn: Cơ chế đạp thắng (fail-fast) tức thì khi phát hiện các thuộc tính (attributes) của vòng trong ngỗ nghịch không tương thích với vòng ngoài |

## Bối cảnh và ranh giới (Bối cảnh và boundary)

| Thành phần | Giá trị |
| --- | --- |
| Điểm vào vòng ngoài (Outer entry) | `ReportFacade.generate`, đeo nhãn `@Transactional` DEFAULT |
| Mặc định của cơ sở dữ liệu (Database default) | PostgreSQL `READ COMMITTED` |
| Vòng trong (Inner) | `readTwice`, đeo nhãn `REPEATABLE_READ`, nhưng mang kiếp propagation REQUIRED |
| Luồng ghi (Writer) | Một giao dịch chễm chệ khác âm thầm cập nhật giá và chốt commit lọt thỏm giữa hai cú reads |
| Kết cục thực tế (Effective result) | Cả hai lệnh reads ngoan ngoãn bò trong khuôn khổ outer `READ COMMITTED` |
| Phạm vi (Scope) | Nằm vùng ở khâu cấu hình Spring (Spring configuration); còn các nghịch lý chi tiết (anomaly) thì xin nhường sân khấu cho mục DB cases |

## Hướng dẫn điều hướng

- [Đặt annotation sai bét nhè (Broken annotation placement)](broken-code.md)
- [Phân tích mức độ cô lập thực tế (Effective isolation analysis)](analysis.md)
- [Thiết lập ranh giới đúng và các đòn fail-fast (Correct boundary and fail-fast options)](solutions.md)
- [Thí nghiệm tích hợp với PostgreSQL (PostgreSQL integration experiments)](experiments.md)
- [Các mức độ cô lập (Isolation levels)](../../concepts/isolation-levels.md)
- [Ranh giới giao dịch trong Spring (Spring transaction boundaries)](../../concepts/spring-transaction-boundaries.md)
- [Kiểm thử vấn đề đồng thời (Concurrency testing)](../../concepts/concurrency-testing.md)

## Hậu quả trong production nếu làm sai

Báo cáo/kết quả đối soát (Report/reconciliation) vô tình băm vằm chắp vá dữ liệu nhặt nhạnh từ nhiều khoảnh khắc chốt hạ (committed moments) khác nhau; hội đồng duyệt mã (code review) lại mù quáng trao niềm tin nhầm chỗ cho cái annotation hào nhoáng; mặc định của nguồn dữ liệu (datasource default) lén lút tráo trở xoay vần giữa các môi trường (environment); lời răn dạy khai báo vòng trong (inner declaration) bị ném vào thùng rác (ignore); ngông cuồng nâng cấp bừa bãi bằng `REQUIRES_NEW` vô tình đẻ ra một đống chốt hạ nửa vời (partial commit) và đốt phá tài nguyên vô bổ (resource cost).

## Hướng sửa chữa (Khuyến nghị)

Gắn chặt mức độ cô lập (isolation) ngay trên chóp cái phương thức cửa ngõ công khai (public entry method) - kẻ đích thực đứng ra thành lập giao dịch và phái lệnh gọi qua Spring proxy. Nhớ bật lẫy `validateExistingTransaction` làm hàng rào thép gai canh gác (fail-fast) cho các ranh giới trọng yếu. Chỉ rước `REQUIRES_NEW` về dinh nếu bản ảnh/chốt hạ (snapshot/commit) độc lập là một mệnh lệnh bắt buộc (requirement). Lúc nào cũng phải nện integration test với PostgreSQL và moi móc dò xét gắt gao mức độ cô lập thực tế (query effective isolation).

## Phạm vi

Bài toán này sẽ không dông dài kể lể hết cặn kẽ về lost update, write skew hay phantom; bởi mấy cái dị tật (anomaly) đó đã bị nhốt chung ở chuồng `DB-001`–`DB-005`.
