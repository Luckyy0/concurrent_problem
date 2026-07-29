# SPR-005 — Checked exception khiến transaction commit ngoài dự kiến

## Tóm tắt

`PayoutPreparationService.prepare()` chuyển payout sang `PROCESSING`, giữ tiền
trong wallet và thêm một ledger entry. Sau đó local beneficiary policy phát hiện
người nhận bị chặn và ném `BeneficiaryRejectedException`, một checked exception.

Caller nhận exception nên tin rằng operation đã rollback. Nhưng Spring mặc định
chỉ rollback với unchecked exception và `Error`; transaction interceptor commit
các thay đổi rồi mới truyền checked exception ra ngoài.

Invariant:

```text
Nếu prepare() kết thúc bằng BeneficiaryRejectedException:
  payout không được ở trạng thái executable;
  available balance/hold không được thay đổi;
  không có PAYOUT_HOLD ledger entry được commit;
  dispatcher không được quan sát payout như một công việc hợp lệ.
```

> **Nói ngắn gọn:** method outcome nói “rejected” nhưng transaction outcome lại là
> “committed”; lỗi nằm ở rollback classification, không nằm ở PostgreSQL.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Request A | Chuẩn bị payout và nhận checked business exception |
| Dispatcher B | Đọc committed `PROCESSING` payout để gửi sang execution pipeline |
| `payout_request` | State machine của payout |
| `wallet_account` | Available balance/hold projection |
| `ledger_entry` | Append-only audit entry cho funds hold |
| Spring transaction proxy | Quyết định commit hay rollback theo exception type/rule |
| PostgreSQL | Authoritative committed state được mọi app instance cùng quan sát |

## Transaction boundary và contention point

`prepare()` là public method gọi qua Spring proxy, propagation `REQUIRED`. Nó tạo
một physical transaction ở PostgreSQL. Beneficiary policy là local, không thực
hiện remote I/O trong transaction.

Contention point là cùng payout/wallet rows và điều kiện dispatcher
`status = 'PROCESSING'`. Row lock có thể serialize hai writers, nhưng không sửa
rollback rule: state sai vẫn được commit và trở thành visible sau khi lock release.

## Expected và actual

| | Caller | Database sau method | Dispatcher |
| --- | --- | --- | --- |
| Expected | Nhận rejection | State ban đầu hoặc `REJECTED`, không có hold | Không dispatch |
| Broken actual | Nhận rejection | `PROCESSING`, balance giảm, ledger có hold | Có thể dispatch |

Khi transaction proxy xử lý checked exception, commit hoàn tất trước khi exception
được rethrow tới caller. Vì vậy retry, alert hoặc compensation dựa trên exception
có thể chạy trên giả định sai.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| checked exception | Exception phải được catch/declare; không kế thừa `RuntimeException` |
| unchecked exception | `RuntimeException` hoặc subclass |
| rollback rule | Policy ánh xạ throwable thành rollback hoặc commit |
| rollback-only | Cờ cho biết transaction không còn được phép commit |
| failure contract | Quan hệ đã công bố giữa method outcome và durable state |
| logical scope | Phạm vi `@Transactional` của một intercepted method |
| physical transaction | Database transaction thật được commit/rollback |
| executable state | State mà dispatcher coi là đủ điều kiện xử lý |

## Điều hướng

- [Broken checked-exception path](broken-code.md)
- [Rollback classification analysis](analysis.md)
- [Explicit failure contracts and fixes](solutions.md)
- [PostgreSQL integration experiments](experiments.md)
- [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Funds bị hold dù payout bị báo rejected.
- Dispatcher thực thi công việc caller tin rằng đã thất bại.
- Client retry có thể tạo duplicate command hoặc conflict với state đã commit.
- Reconciliation thấy exception log nhưng ledger lại có entry hợp lệ về kỹ thuật.
- Multi-instance làm khoảng lệch rõ hơn vì node khác thấy commit ngay lập tức.
- Compensation trở nên bắt buộc cho lỗi lẽ ra phải atomic rollback.

## Hướng sửa khuyến nghị

Nếu checked exception biểu thị toàn bộ unit phải thất bại, khai báo
`rollbackFor = BeneficiaryRejectedException.class` ngay trên public transaction
boundary và để exception thoát qua proxy.

Nếu rejection là expected business outcome cần lưu lại, đừng dùng exception với
hàm ý rollback. Ghi state `REJECTED`, không tạo executable state/hold, commit một
kết quả domain rõ ràng và return `Rejected`.

## Khi áp dụng

Áp dụng cho service methods khai báo checked business exception sau khi mutate
database. Case không xử lý remote timeout ambiguity, unknown outcome từ payment
provider hay compensation cho external side effect; đó là failure model khác.
