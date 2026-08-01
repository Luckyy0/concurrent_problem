# SPR-003 — Propagation tạo workflow partial-commit

## Tóm tắt

Quá trình checkout đổi đơn hàng (order) từ trạng thái `PENDING` sang `PAID`. Bên trong, bean audit dùng `REQUIRES_NEW` ghi `PAYMENT_COMPLETED` và commit độc lập. Nếu transaction ngoài (outer transaction) thất bại, order sẽ rollback về `PENDING` nhưng bản ghi audit “completed” vẫn tồn tại.

Trường hợp này cũng phân tích chiều ngược lại: phương thức bên trong (inner method) dùng `REQUIRED`, ném ra ngoại lệ runtime qua proxy và đánh dấu transaction vật lý là rollback-only. Khối mã bên ngoài (outer code) bắt ngoại lệ (catch exception) rồi tiếp tục, nhưng lần commit cuối cùng vẫn ném `UnexpectedRollbackException`.

Bất biến (Invariant):

```text
Bản ghi mô tả sự thành công của nghiệp vụ chỉ tồn tại khi transaction nghiệp vụ commit.
Công việc cần atomicity phải cùng một transaction vật lý.
Công việc cố ý tồn tại sau khi rollback phải có semantics trung thực như ATTEMPT/FAILURE (thử nghiệm/thất bại).
Phía gọi (caller) không được nhận kết quả thành công khi transaction đã bị đánh dấu rollback-only.
```

> **Nói ngắn gọn:** propagation không chỉ quyết định “có transaction hay không”;
> nó quyết định phần nào có thể commit/rollback độc lập.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| phạm vi transaction logic (logical transaction scope) | Phạm vi phương thức/advice mang cấu hình propagation |
| transaction vật lý (physical transaction) | Transaction/connection thật sự của database được commit hoặc rollback |
| `REQUIRED` | Tham gia vào transaction vật lý hiện có hoặc tạo mới |
| `REQUIRES_NEW` | Tạm dừng (suspend) transaction ngoài và tạo transaction vật lý độc lập |
| rollback-only | Transaction vật lý đã bị đánh dấu là không thể commit |
| `UnexpectedRollbackException` | Mã nguồn bên ngoài yêu cầu commit nhưng transaction đã bị đánh dấu rollback-only |
| partial commit | Một phần của workflow được commit dù phần khác bị rollback |
| transaction bị tạm dừng (suspended transaction) | Transaction ngoài tạm ngừng trong khi transaction bên trong chạy |

## Bối cảnh và ranh giới

| Thành phần | Giá trị |
| --- | --- |
| Bên ngoài (Outer) | Checkout Tx-O: order PENDING→PAID, trạng thái thanh toán/kho hàng |
| Bên trong (Inner) | Audit Tx-A: thêm `PAYMENT_COMPLETED` bằng `REQUIRES_NEW` |
| Lỗi (Failure) | Bước xác thực/kho hàng bên ngoài ném ngoại lệ sau khi audit |
| Kết quả thực tế | Tx-A được commit, Tx-O bị rollback |
| Database | PostgreSQL |
| Phạm vi (Scope) | Mô hình transaction của một ứng dụng/database |

## Điều hướng

- [Mã nguồn propagation bị lỗi](broken-code.md)
- [Phân tích transaction vật lý/logic](analysis.md)
- [Lựa chọn ranh giới chính xác](solutions.md)
- [Thử nghiệm tích hợp PostgreSQL](experiments.md)
- [Ranh giới transaction trong Spring](../../concepts/spring-transaction-boundaries.md)
- [Kiểm thử đồng thời](../../concepts/concurrency-testing.md)

## Hậu quả trên production

- sự kiện/audit báo thanh toán hoàn tất cho đơn hàng vẫn đang chờ xử lý (pending);
- các hệ thống phía sau (downstream) xử lý bản ghi đã commit độc lập;
- khối mã bên ngoài bắt lỗi REQUIRED từ bên trong nhưng cuối cùng vẫn bị rollback;
- `REQUIRES_NEW` giữ thêm một connection và có thể làm cạn kiệt pool;
- transaction bên trong bị chặn (block) bởi lock do transaction ngoài (đang bị tạm dừng) nắm giữ;
- việc thử lại (retry) tạo ra bản ghi audit trùng lặp nếu không có tính lũy đẳng (idempotency).

## Hướng sửa khuyến nghị

Các công việc mô tả cùng một kết quả nghiệp vụ nên dùng `REQUIRED` và chung một transaction ngoài, hoặc xuất bản sự kiện thành công sau khi commit/outbox. Chỉ dùng `REQUIRES_NEW` khi commit độc lập là một yêu cầu thực sự; bản ghi phải mô tả lần thử/thất bại một cách độc lập, không giả mạo sự thành công.
Đừng bắt ngoại lệ từ phạm vi REQUIRED bên trong rồi trả về kết quả thành công; hãy lan truyền (propagate) ngoại lệ hoặc đổi nhánh dự kiến thành kết quả rõ ràng (explicit outcome) mà không đánh dấu rollback-only.

## Phạm vi

Trường hợp này không thiết kế mô hình saga/compensation phân tán (`DIST-004`) và không giải quyết các bất thường về cô lập (isolation anomaly) (`DB-*`).
