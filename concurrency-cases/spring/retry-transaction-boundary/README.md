# SPR-006 — Retry chạy bên trong một doomed transaction

## Tóm tắt

Hai reservation commands cùng load SKU `BOOK-42` ở `version = 7`. Command A commit
trước, làm stock `2 -> 1` và version `7 -> 8`. Command B flush stale update với
predicate `version = 7`, nhận optimistic conflict.

Broken service catch exception trong một retry loop nằm bên trong cùng
`@Transactional` method. Attempt tiếp theo reuse physical transaction đã
`rollback-only` và persistence context vừa failed flush. `clear()` hay reload
không biến transaction đó thành clean attempt.

Invariant:

```text
Mỗi retry attempt phải chạy trong physical transaction và persistence context mới.
Một returned success phải tương ứng đúng một committed reservation/decrement.
Mỗi attempt phải reload current version và re-evaluate available stock.
Retry phải bounded bởi attempt limit và operation deadline.
```

> **Nói ngắn gọn:** rollback attempt cũ trước, rồi mới retry; loop không phải
> transaction boundary.

## Actors và shared state

| Thành phần | Vai trò |
| --- | --- |
| Command A | Winner commit update từ version 7 lên 8 |
| Command B | Loser conflict, sau đó retry trên version mới |
| `inventory_item` | Authoritative stock và `@Version` |
| `reservation_record` | Durable result keyed theo command ID |
| Spring transaction proxy | Tạo/rollback transaction cho từng attempt |
| Hibernate persistence context | Giữ managed entity snapshot trong một attempt |
| PostgreSQL | Kiểm tra version predicate và committed row state |

Initial state:

```text
SKU BOOK-42: available = 2, version = 7
reservation records: none
```

Correct final state khi cả hai distinct commands được phép reserve:

```text
available = 0, version = 9
reservation records = {A, B}
```

## Transaction boundary và contention point

Contention point là row `inventory_item.sku = BOOK-42`. Mỗi attempt thực hiện:

```text
load current version -> validate stock -> decrement -> insert reservation -> flush/commit
```

Optimistic conflict xảy ra tại versioned UPDATE, không phải lúc Java entity được
mutate. Retry coordinator phải nằm ngoài transaction attempt:

```text
correct: Retry coordinator -> Tx attempt 1 -> rollback
                           -> Tx attempt 2 -> reload -> commit

broken:  One Tx -> attempt 1 fails -> attempt 2 reuses doomed Tx
```

## Expected và actual

| | Failed attempt | Retry attempt | Caller/final state |
| --- | --- | --- | --- |
| Expected | Rollback và close context | New Tx, reload version 8 | Commit B; stock 0 |
| Broken | Exception bị catch trong Tx | Reuse rollback-only/context cũ | Late failure; B không commit |

Nếu flush chỉ xảy ra ở outer commit, method-local catch còn không nhìn thấy
conflict; exception xuất hiện sau khi loop đã return.

## Thuật ngữ cần biết

| Thuật ngữ | Giải thích |
| --- | --- |
| doomed transaction | Transaction đã rollback-only hoặc không còn usable để commit |
| attempt boundary | Physical transaction bao quanh đúng một execution attempt |
| retry ordering | Thứ tự retry interceptor và transaction interceptor |
| fresh snapshot | State reload trong persistence context/transaction mới |
| optimistic conflict | Versioned UPDATE ảnh hưởng 0 rows |
| bounded backoff | Delay có giới hạn, attempt limit và thường có jitter |
| retryable failure | Failure tạm thời được policy cho phép chạy lại |
| business revalidation | Kiểm tra lại stock/rule trên state mới |

## Điều hướng

- [Broken retry placement](broken-code.md)
- [Doomed transaction analysis](analysis.md)
- [New transaction per attempt](solutions.md)
- [Deterministic optimistic-conflict experiments](experiments.md)
- [Optimistic locking](../../concepts/optimistic-locking.md)
- [Spring transaction boundaries](../../concepts/spring-transaction-boundaries.md)
- [Deadlock and safe retry](../../concepts/deadlocks-and-retries.md)
- [Concurrency testing](../../concepts/concurrency-testing.md)

## Hậu quả production

- Retry counter tăng nhưng không attempt nào có thể commit.
- Caller nhận `UnexpectedRollbackException`/persistence error muộn.
- Stale entity làm business validation dùng dữ liệu cũ.
- Hot key tạo retry amplification và database load.
- External side effect có thể lặp dù database attempt rollback.
- Advisor ordering khác giữa configurations làm behavior khó đoán.

## Hướng sửa khuyến nghị

Dùng non-transactional retry coordinator gọi một proxied `reserveOnce()` trên bean
khác. Mỗi call `reserveOnce()` dùng `@Transactional`, để conflict thoát ra, rollback
hoàn tất, rồi coordinator mới backoff và thử lại.

Phân loại exception cụ thể, giới hạn attempts/deadline, reload aggregate và không
đặt remote side effect trong retryable transaction.

## Phạm vi

Case tập trung vào retry mechanics. Việc một business command có idempotent và an
toàn để retry hay không vẫn phải được quyết định trong domain case tương ứng.
