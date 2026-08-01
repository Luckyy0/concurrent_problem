# SPR-006 — Retry chạy bên trong một doomed transaction

## Tóm tắt

Hai reservation command cùng đọc SKU `BOOK-42` ở `version = 7`. Command A commit trước, làm stock giảm `2 -> 1` và version tăng `7 -> 8`. Command B flush một stale update với predicate `version = 7`, dẫn đến optimistic conflict (xung đột lạc quan).

Service bị lỗi đã catch ngoại lệ trong một vòng lặp retry nằm bên trong cùng phương thức `@Transactional`. Lần thử (attempt) tiếp theo sử dụng lại physical transaction đã bị đánh dấu `rollback-only` và persistence context vừa bị lỗi khi flush. Việc gọi `clear()` hay tải lại (reload) không biến transaction đó thành một lần thử hợp lệ (clean attempt).

Invariant (Bất biến):

```text
Mỗi lần thử retry phải chạy trong một physical transaction và persistence context mới.
Một kết quả thành công trả về phải tương ứng đúng với một decrement/reservation đã được commit.
Mỗi lần thử phải tải lại version hiện tại và đánh giá lại lượng stock khả dụng.
Retry phải bị giới hạn bởi số lần thử và thời gian timeout của operation.
```

> **Nói ngắn gọn:** cần rollback lần thử cũ trước, rồi mới retry; vòng lặp không phải là ranh giới của transaction.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Command A | Tiến trình thắng, commit cập nhật từ version 7 lên 8 |
| Command B | Tiến trình thua do xung đột, sau đó retry trên version mới |
| `inventory_item` | Dữ liệu stock có thẩm quyền và `@Version` |
| `reservation_record` | Kết quả được lưu trữ định danh theo command ID |
| Spring transaction proxy | Tạo/rollback transaction cho từng lần thử |
| Hibernate persistence context | Giữ managed entity snapshot trong một lần thử |
| PostgreSQL | Kiểm tra version predicate và trạng thái dòng đã commit |

Trạng thái ban đầu:

```text
SKU BOOK-42: available = 2, version = 7
reservation records: none
```

Trạng thái cuối cùng hợp lệ khi cả hai command riêng biệt được phép reserve:

```text
available = 0, version = 9
reservation records = {A, B}
```

## Transaction boundary và contention point

Điểm tranh chấp (contention point) là dòng `inventory_item.sku = BOOK-42`. Mỗi lần thử thực hiện:

```text
đọc version hiện tại -> kiểm tra stock -> giảm số lượng -> thêm reservation -> flush/commit
```

Optimistic conflict xảy ra tại câu lệnh UPDATE có chứa version, không phải lúc Java entity bị thay đổi. Coordinator điều phối retry phải nằm ngoài transaction của lần thử:

```text
Đúng: Retry coordinator -> Tx attempt 1 -> rollback
                        -> Tx attempt 2 -> tải lại -> commit

Sai:  Một Tx chung -> attempt 1 lỗi -> attempt 2 dùng lại Tx đã hỏng (doomed Tx)
```

## Expected và actual

| | Lần thử bị lỗi | Lần thử retry | Phía gọi (caller) / trạng thái cuối |
| --- | --- | --- | --- |
| Kỳ vọng (Expected) | Rollback và đóng context | Transaction mới, tải lại version 8 | Commit B; stock 0 |
| Lỗi (Broken) | Ngoại lệ bị catch trong Tx | Dùng lại rollback-only/context cũ | Lỗi trả về muộn; B không commit |

Nếu flush chỉ xảy ra ở bước commit bên ngoài, khối catch cục bộ trong phương thức còn không nhìn thấy xung đột; ngoại lệ xuất hiện sau khi vòng lặp đã trả về.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| doomed transaction | Transaction đã bị đánh dấu rollback-only hoặc không còn khả dụng để commit |
| attempt boundary | Physical transaction bao quanh đúng một lần thử thực thi |
| retry ordering | Thứ tự giữa retry interceptor và transaction interceptor |
| fresh snapshot | Trạng thái được tải lại trong persistence context/transaction mới |
| optimistic conflict | Câu lệnh UPDATE có chứa version ảnh hưởng đến 0 dòng |
| bounded backoff | Thời gian chờ có giới hạn, có số lần thử tối đa và thường có độ trễ ngẫu nhiên (jitter) |
| retryable failure | Lỗi tạm thời được policy cho phép chạy lại |
| business revalidation | Kiểm tra lại dữ liệu stock/quy tắc nghiệp vụ trên trạng thái mới |

## Điều hướng

- [Vị trí đặt retry bị lỗi](broken-code.md)
- [Phân tích doomed transaction](analysis.md)
- [Transaction mới cho mỗi lần thử](solutions.md)
- [Các thử nghiệm optimistic-conflict tất định](experiments.md)
- [Optimistic locking](../../concepts/optimistic-locking.md)
- [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md)
- [Deadlock and safe retry](../../concepts/deadlocks-and-retries.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Bộ đếm retry tăng nhưng không có lần thử nào có thể commit.
- Phía gọi nhận `UnexpectedRollbackException` hoặc lỗi persistence muộn.
- Thực thể lỗi thời (stale entity) làm quá trình kiểm tra nghiệp vụ dùng dữ liệu cũ.
- Dữ liệu bị truy cập nhiều (hot key) tạo ra retry khuyếch đại và tăng tải database.
- Side effect bên ngoài có thể bị lặp lại dù database attempt bị rollback.
- Thứ tự advisor khác nhau giữa các cấu hình làm hành vi khó đoán.

## Hướng sửa khuyến nghị

Dùng non-transactional retry coordinator để gọi một phương thức `reserveOnce()` đã được proxy trên một bean khác. Mỗi lời gọi `reserveOnce()` sử dụng `@Transactional`, để xung đột thoát ra ngoài, quá trình rollback hoàn tất, rồi coordinator mới chờ một khoảng backoff và thử lại.

Phân loại ngoại lệ cụ thể, giới hạn số lần thử/thời gian, tải lại aggregate và không đặt remote side effect trong một transaction có thể retry.

## Phạm vi

Trường hợp này tập trung vào cơ chế retry. Việc một command nghiệp vụ có tính lũy đẳng (idempotent) và an toàn để retry hay không vẫn phải được quyết định trong từng trường hợp cụ thể của domain.
