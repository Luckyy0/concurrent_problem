# DB-002 — Lầm tưởng về Đọc Bẩn (Dirty Read) và Sự Thật Ở PostgreSQL

## Tóm tắt

Giả sử có một tiến trình (Job) tên là `IMPORT-42` đang chạy và đã hoàn thành `20%`. Một luồng xử lý (Processor A) tiến hành cập nhật tiến độ (progress) lên `80%`. Luồng này đã đẩy (flush) lệnh SQL xuống cơ sở dữ liệu nhưng **chưa chốt giao dịch (chưa commit)** vì đang xử lý một Giao dịch (Transaction) dài.

Cùng lúc đó, một luồng giám sát (Watchdog B) kiểm tra xem tiến trình có bị treo hay không. Luồng B này sử dụng mức độ cô lập `@Transactional(isolation = READ_UNCOMMITTED)` (cho phép đọc bẩn - dirty read) với kỳ vọng sẽ đọc được con số `80%` đang nằm trong giao dịch chưa commit của A, nhằm xác nhận tiến trình vẫn đang hoạt động.

Tuy nhiên, PostgreSQL xử lý mức độ cô lập này theo cách riêng: Nó kiên quyết **không bao giờ** trả về dữ liệu chưa commit, bất chấp việc bạn cấu hình isolation level là `READ_UNCOMMITTED`. Do đó, luồng B chỉ nhận được con số `20%` (dữ liệu đã commit gần nhất). Luồng B kết luận sai rằng tiến trình đã bị treo và kích hoạt cơ chế khôi phục (recovery), dẫn đến việc chạy trùng lặp (duplicate task).

Điều kiện bất biến (Invariant):

```text
Mọi quyết định nghiệp vụ quan trọng của hệ thống phải dựa trên dữ liệu đã được lưu trữ an toàn (committed durable state).
Tuyệt đối không sử dụng khả năng Đọc Bẩn (dirty-read) để kiểm tra trạng thái hoạt động của hệ thống.
Để báo cáo tiến độ, hãy đóng mở các giao dịch ngắn (short commit) hoặc sử dụng cơ chế nhịp tim (heartbeat).
```

> **Nói ngắn gọn:** Yêu cầu `READ_UNCOMMITTED` ở PostgreSQL không mang lại hiệu ứng "Đọc bẩn" như ở các hệ quản trị cơ sở dữ liệu khác (như SQL Server hay MySQL). PostgreSQL có thể chấp nhận cấu hình này nhưng cách hành xử thực tế luôn tương đương với `READ COMMITTED` (chỉ đọc dữ liệu đã commit).

## Các thành phần tham gia (Actors và shared state)

| Thành phần | Vai trò |
| --- | --- |
| Processor A | Luồng xử lý công việc, cập nhật tiến độ nhưng giữ giao dịch quá lâu chưa commit. |
| Watchdog B | Luồng giám sát tiến độ để xác định tiến trình đang hoạt động hay bị treo. |
| Recovery C | Luồng cứu hộ, tự động chạy lại công việc nếu B báo "Treo". |
| Bảng `job_run` | Nơi lưu trữ trạng thái tiến độ cuối cùng. |
| PostgreSQL MVCC | Cơ chế quản lý đa phiên bản, quyết định B được nhìn thấy dữ liệu nào. |
| Spring/JDBC | Truyền cấu hình mức độ cô lập (isolation level) xuống CSDL. |

Trạng thái ban đầu (đã commit):

```text
job_id      = IMPORT-42
status      = RUNNING
progress    = 20
generation  = 7
```

Processor A cập nhật (chưa commit):

```text
progress = 80
```

Kết quả Watchdog B đọc được:

```text
progress = 20
```

## Ranh giới giao dịch và điểm tranh chấp (Transaction boundary và contention point)

**Luồng thực thi của A:**

```text
BẮT ĐẦU GIAO DỊCH -> UPDATE progress=80 -> FLUSH xuống DB -> Xử lý tiếp -> COMMIT (hoặc ROLLBACK)
```

**Luồng thực thi của B:**

```text
BẮT ĐẦU GIAO DỊCH (mức READ_UNCOMMITTED)
  -> Thực thi lệnh SELECT thông thường (plain SELECT)
  -> Nhận kết quả 20%
COMMIT
```

Vấn đề nằm ở dòng dữ liệu `job_run(IMPORT-42)`. Do cơ chế MVCC của PostgreSQL, lệnh SELECT thông thường của B không hề bị chặn (block) bởi khóa (lock) của A. Nó sẽ đọc ngay phiên bản dữ liệu cũ (20%) đã được commit. Đây là sự khác biệt về tầm nhìn dữ liệu (visibility mismatch), không phải lỗi do chờ khóa (lock timeout).

## Kỳ vọng theo thiết kế lỗi và Thực tế (Expected vs Actual)

| Tiêu chí | Kỳ vọng (Thiết kế lỗi) | Thực tế ở PostgreSQL |
| --- | --- | --- |
| Ý nghĩa của Isolation | DB sẽ cho phép Đọc Bẩn! | Nhận cấu hình nhưng xử lý như `READ_COMMITTED` |
| B đọc khi A chưa commit | Thấy giá trị 80 | Chỉ thấy giá trị 20 |
| Nếu A bị lỗi Rollback | Ứng dụng phải tự xử lý số liệu sai (80) | Số 80 chưa từng hiển thị, không gây lỗi logic |
| Lệnh SELECT có bị Block?| Không | Không, tự động đọc phiên bản dữ liệu cũ |
| Tính tương thích (Portability)| Mọi RDBMS đều xử lý `READ_UNCOMMITTED` giống nhau | Sai lầm, mỗi hệ quản trị CSDL áp dụng luật MVCC khác nhau |

Nếu A commit thành công, một giao dịch mới của B (bắt đầu sau đó) mới có thể đọc được `80`. Nếu A rollback, B vẫn tiếp tục đọc được `20`.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| Dirty read (Đọc bẩn) | Đọc dữ liệu từ một giao dịch khác chưa được commit. |
| Aborted version | Phiên bản dữ liệu của một giao dịch bị hủy (rollback). |
| Requested isolation | Mức độ cô lập mà mã nguồn ứng dụng yêu cầu cấp phép. |
| Reported label | Mức độ cô lập mà CSDL báo cáo lại (có thể chỉ là hình thức). |
| Effective semantics | Cách hành xử thực sự của CSDL với dữ liệu. |
| Committed-only | Nguyên tắc chỉ trả về các dữ liệu đã được commit an toàn. |
| Progress checkpoint | Báo cáo tiến độ riêng biệt, lưu ngay lập tức, không đợi toàn bộ công việc xong. |
| Portability assumption | Giả định sai lầm rằng các hệ quản trị CSDL khác nhau sẽ hoạt động hoàn toàn giống nhau với cùng một cấu hình chuẩn. |

## Điều hướng

- [Mã nguồn gây lỗi Ảo tưởng Đọc Bẩn (Broken dirty-read design)](broken-code.md)
- [Phân tích Tầm nhìn MVCC (PostgreSQL visibility analysis)](analysis.md)
- [Giải pháp báo cáo Tiến Độ Chuẩn (Committed coordination and progress solutions)](solutions.md)
- [Các bài kiểm thử Tái hiện lỗi (Deterministic visibility experiments)](experiments.md)
- [PostgreSQL MVCC](../../concepts/postgresql-mvcc.md)
- [Các mức độ cô lập (Isolation levels)](../../concepts/isolation-levels.md)
- [Kiểm thử tương tranh (Concurrency testing)](../../concepts/concurrency-testing.md)

## Hậu quả trên môi trường Production

- Watchdog báo động nhầm khiến hệ thống kích hoạt cơ chế khôi phục (Recovery), dẫn đến việc xử lý trùng lặp dữ liệu và lãng phí tài nguyên.
- Giao diện Dashboard hiển thị tiến độ bị kẹt ở con số cũ, và chỉ nhảy lên 100% khi toàn bộ quá trình kết thúc.
- Việc chuyển đổi hệ quản trị cơ sở dữ liệu (Database Migration) sang PostgreSQL sẽ gây ra hàng loạt lỗi logic ẩn nếu mã nguồn phụ thuộc vào khả năng Đọc Bẩn.
- Ứng dụng triển khai cơ chế hỏi vòng (polling) liên tục nhưng vô ích, vì CSDL luôn trả về dữ liệu cũ cho đến khi giao dịch kia commit.
- Duy trì các giao dịch dài hạn (long-running transactions) gây tắc nghẽn tài nguyên và làm hệ thống thiếu tính minh bạch.

## Hướng sửa chữa khuyến nghị

Thiết kế lại cơ chế giám sát mà không phụ thuộc vào dữ liệu chưa commit. Có 3 hướng tiếp cận chính:

1. **Tuân thủ Committed-only:** Chấp nhận thực tế rằng chỉ có dữ liệu đã commit mới đáng tin cậy để ra quyết định.
2. **Cơ chế Máy trạng thái (Checkpoints/Chunks):** Chia nhỏ công việc thành các giao dịch độc lập ngắn hơn, lưu trạng thái hoàn thành (checkpoint) và commit ngay lập tức.
3. **Cơ chế Nhịp tim (Independent Heartbeat):** Sử dụng một bảng riêng để lưu "Nhịp tim" báo cáo tiến độ, cập nhật qua các giao dịch độc lập cực ngắn (REQUIRES_NEW) mà không cản trở giao dịch chính.

Hãy tránh việc gộp quá nhiều logic tính toán dài hạn vào trong một giao dịch cơ sở dữ liệu duy nhất.

## Phạm vi của bài toán

Tài liệu này tập trung vào cách PostgreSQL vô hiệu hóa khả năng Đọc Bẩn (dirty-read) và sự khác biệt hành vi giữa các RDBMS (tính portability).
- Các vấn đề khi hai giao dịch cùng thao tác và gây hiện tượng Đọc Không Lặp Lại (Non-repeatable read) sẽ được phân tích ở case `DB-003`.
- Các lỗi Mất Dữ Liệu Cập Nhật (Lost update) hoặc Ghi Bẩn (Dirty writes) được trình bày trong các bài viết riêng biệt.
